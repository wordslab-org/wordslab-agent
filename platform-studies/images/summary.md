# Unified Image & Video AI API Specification

> **Aggregated from:** OpenAI, Google Gemini, xAI Grok Imagine, Black Forest Labs (FLUX), Ideogram, Recraft, Reve, DaVinci.ai, and the classical vision APIs (Replicate, Google Cloud Vision, Azure Computer Vision, Azure Custom Vision)
> **Purpose:** A single, exhaustive specification that encompasses every image/video capability, concept, parameter, and processing step described across the nine platform studies in `./platform-studies/images/`.

---

## How to Read This Document

This document is written for end users — developers, product managers, and architects who want to understand the **full landscape** of image and video AI capabilities before choosing a provider or building a system. It is organized as follows:

1. **Part I — Concepts & Vocabulary** — An approachable introduction to every concept you will encounter, with plain-language explanations and a glossary that maps the many different names providers use for the *same* idea.
2. **Part II — The Exhaustive Processing Pipeline** — Every image/video capability ordered into a single exhaustive end-to-end pipeline, from authentication through delivery. Each stage lists the alternative approaches available and the synonyms used by each platform.
3. **Part III — The Unified API Specification** — A detailed, provider-agnostic API reference that describes every endpoint, parameter, and data structure needed to implement a "super complete" image & video AI platform.

---

# Part I — Concepts & Vocabulary

## 1. What Is Image & Video AI?

Image and Video AI is the family of technologies that let machines **understand visual content**, **generate visual content**, and **transform existing visual content**. A complete image & video AI platform touches seven broad domains:

| Domain | One-line description |
|--------|----------------------|
| **Image Understanding (Vision)** | Analyze an existing image: classify, detect objects, segment regions, caption, OCR, recognize faces/landmarks/logos. |
| **Image Generation** | Synthesize a new raster image from text (text-to-image) and/or from one or more reference images. |
| **Image Editing & Transformation** | Modify an existing image: inpaint, outpaint, erase, restyle, relight, replace/remove background, upscale, deblur, virtual try-on. |
| **Image Format & Structure Conversion** | Vectorize raster→SVG, extract text layers, derive layouts, transparent-background generation. |
| **Video Understanding** | Analyze video frames: object tracking, scene segmentation, captioning (covered via vision models applied per-frame). |
| **Video Generation** | Synthesize a new video clip (with optional native audio) from text, an image (first frame), references, or interpolation between two frames. |
| **Video Editing & Extension** | Edit an existing video, extend it from its last frame, or interpolate between a first and last frame. |

A single product may span several domains. For example, a "generate → edit → animate" pipeline chains image generation, image editing, and image-to-video generation.

## 2. Core Concepts (Provider-Agnostic)

### 2.1 Visual Modalities & Data Flows
Image/video systems deal with six kinds of data flowing in and out:
- **Text prompt / instruction** — natural-language description of what to generate, edit, or analyze.
- **Structured prompt (JSON)** — a machine-readable prompt contract (Ideogram `V4JsonPrompt`, Reve `Layout`, FLUX JSON prompting) describing elements, positions, styles.
- **Reference image(s)** — existing images that guide generation/editing (style, identity, composition).
- **Mask** — a grayscale or alpha image defining which pixels to edit (white = edit/inpaint, black = keep).
- **Generated image** — output raster (PNG/JPEG/WebP) or vector (SVG).
- **Generated video** — output MP4 (with optional native audio), plus thumbnail/spritesheet.

### 2.2 Model
The **model** is the underlying AI. Providers expose multiple models tuned for different trade-offs:
- **Quality vs speed** — flagship (e.g. FLUX.2 [max], GPT Image 2, Nano Banana Pro, Recraft V4.1 Pro) vs fast/lightweight (FLUX.2 [klein], GPT Image 1 Mini, Nano Banana 2 Lite, Recraft V4.1, Reve Fast).
- **Generation vs editing** — some models are unified (FLUX.2, GPT Image, Grok Imagine, Reve); others split generation and editing into separate models (Ideogram generate vs remix, FLUX.1 Fill for inpainting).
- **Raster vs vector** — Recraft offers both raster (PNG/WEBP/JPG) and vector (SVG) model variants.
- **Versioning** — pinned dated snapshots (`reve-create@20250915`, `sonic-3.5-2026-05-04`) vs rolling aliases (`latest`, `auto`).

### 2.3 Understanding vs Generation — A Key Split
Most platforms clearly separate **understanding** (analysis of existing images) from **generation** (synthesis of new images):
- **Understanding** is a modality of mainline multimodal chat models (OpenAI GPT-5.x `input_image`, Google Gemini Interactions API, Azure Computer Vision, Google Cloud Vision, Replicate task models). It produces text/JSON/boxes/masks.
- **Generation** uses dedicated image-synthesis models (GPT Image, Nano Banana, FLUX, Ideogram, Recraft, Grok Imagine, Reve). It produces images.
- Some platforms bridge both: OpenAI's Responses API lets a mainline model both *see* (`input_image`) and *produce* (`image_generation` tool) images in one conversation. Google's Interactions API unifies understanding and generation in one call. Ideogram's `describe` round-trips image→structured prompt→image. Reve's `image_to_layout` derives structure from an image.

### 2.4 Synchronous vs Asynchronous
- **Synchronous** — the request blocks until the image/video is returned. Used by OpenAI Images API, Google Interactions, all Recraft endpoints, all Reve endpoints, all xAI image endpoints, Azure Computer Vision v4.0, Google Cloud Vision `images:annotate`.
- **Asynchronous (submit → poll)** — the request returns a job/operation id; the caller polls or receives a webhook. Used by all video generation (OpenAI Videos, Google Veo `predictLongRunning`, xAI Videos), all BFL FLUX endpoints, Replicate predictions, Ideogram async generate, Azure Custom Vision training, Google Cloud Vision async batch.
- **Webhook delivery** — instead of polling, the platform POSTs the result to a registered webhook (BFL, OpenAI Videos, Ideogram async, Replicate).

### 2.5 Prompt Enhancement
Automatic rewriting of the user's prompt for improved generation quality. Synonyms: *Prompt Upsampling* (BFL `prompt_upsampling` / `disable_pup`), *Magic Prompt* (Ideogram `magic_prompt: AUTO/ON/OFF`), *Enhance Prompt* (Recraft `/v1/prompts/enhance`), *Revised Prompt* (OpenAI `revised_prompt` — the rewritten prompt is returned), *Auto-enhanced prompt* (Reve — silent, not returned), *Thinking process* (Google — internal thought images, billed but not always surfaced).

### 2.6 Reference Images & Multi-Reference
Using one or more existing images to guide generation/editing. Roles vary by platform:
- **Style reference** — transfer visual aesthetic (Recraft `style_reference_images`, Google style reference, Ideogram `style_reference_images`).
- **Character reference** — keep a character consistent (Ideogram `character_reference_images`, Google character refs, OpenAI Sora `characters`, Reve `references`).
- **Object reference** — include a specific object (Google object refs, BFL `input_image_N`, Reve `<img>N</img>` tags).
- **First frame / last frame** — for video (OpenAI `input_reference`, Google `image`+`lastFrame`, xAI `image`).
- **Composite/multi-ref** — combine multiple references into one output (BFL up to 8, Ideogram up to 10, Reve up to 6, xAI up to 3, Google up to 14).

### 2.7 Mask
A mask defines *which pixels to edit*. Encodings:
- **Black/white grayscale** — white = edit/inpaint/remove, black = keep (BFL, Recraft, Ideogram inpaint, Reve).
- **Alpha channel** — transparent regions = inpaint (BFL Fill, OpenAI `input_image_mask`).
- **Polygon/mask image** — segmentation masks (Replicate SAM, Google `mask` polygon).
Mask must match image dimensions; some platforms require an alpha channel (OpenAI).

### 2.8 Bounding Boxes & Coordinate Systems
A critical source of divergence across platforms:

| Encoding | Format | Used by |
|----------|--------|---------|
| **XYXY** (pixel) | `[x1,y1,x2,y2]` | Replicate Grounding DINO, YOLO-World, Florence-2 |
| **XYWH** (pixel) | `{x,y,w,h}` | Azure Computer Vision, Replicate Semantic-Segment-Anything |
| **Bounding polygon** (pixel) | `[{x,y}...]` | Google `BoundingPoly`, Azure `boundingPolygon`, Florence-2 quad boxes |
| **Normalized 0–1** (float) | `[ymin,xmin,ymax,xmax]` or `[x,y,w,h]` | Google `NormalizedVertex`, Reve layout `[0,1]`, Recraft `text_layout` `[0,1]` |
| **Normalized 0–1000** (int) | `[ymin,xmin,ymax,xmax]` | Google Gemini object detection, Ideogram `V4JsonPrompt` bbox |

Always check the coordinate space (pixel vs normalized) and axis order (xyxy vs xywh vs yxyx).

### 2.9 Open-Vocabulary vs Fixed-Vocabulary Detection
- **Open-vocabulary** — detect arbitrary objects described by free-form text at inference, no retraining (Replicate Grounding DINO, YOLO-World, OWL-ViT, Grounded SAM).
- **Fixed-vocabulary** — provider-trained label set (Google Cloud Vision `LABEL_DETECTION`/`OBJECT_LOCALIZATION`, Azure `Tags`/`Objects`).
- **Custom-trained** — you define the vocabulary at training time (Azure Custom Vision, Recraft custom styles, Ideogram custom models, BFL LoRA finetunes).

### 2.10 Segmentation Mask Encodings
- **PNG image** — SAM-2 returns `combined_mask` and `individual_masks` as image URLs.
- **COCO RLE** — `{size:[h,w], counts:"<encoded>"}`; used by Semantic-Segment-Anything and SAMURAI for compact per-mask JSON.
- **Polygon** — list of `[x,y]` vertices normalized 0–1000 (Google Gemini).
- **Quality scores** — `predicted_iou` and `stability_score` (SAM family).

### 2.11 Style System
Mechanisms to steer the visual aesthetic:
- **Curated style names** — Recraft `style` (`Photorealism`, `Illustration`, `Vector art`), Ideogram `style_type` (`REALISTIC`/`DESIGN`/`FICTION`).
- **Style presets** — Ideogram ~60 named presets (`ART_DECO`, `WATERCOLOR`, `BAUHAUS`).
- **Style codes** — Ideogram 8-char hex codes (shareable, precise).
- **Style reference images** — Recraft, Ideogram, Google.
- **Custom style (style_id)** — Recraft `POST /v1/styles` from up to 5 reference images.
- **Color palette** — Recraft `controls.colors`, Ideogram `color_palette` (preset or hex+weight members).
- **Structured style description** — Ideogram V4 `style_description` (aesthetics, art_style, lighting, medium, photo).
- **Controls** — Recraft `controls` (colors, background_color, artistic_level, no_text).
- **Text layout** — Recraft `text_layout` placing individual words as 4-point polygons.
- **LoRA finetune** — BFL `finetune_id` + `finetune_strength`; custom style/identity model.
- **Custom model** — Ideogram `custom_model_uri` trained on 15–100 assets.

### 2.12 Moderation & Safety
- **Safety tolerance** — BFL `safety_tolerance` 0–5 (FLUX.2) / 0–6 (FLUX.1), 0 = strictest.
- **Moderation strictness** — OpenAI `moderation: auto|low`.
- **Content violation flag** — Reve `content_violation` / `X-Reve-Content-Violation`; xAI `respect_moderation`; Ideogram `is_image_safe`.
- **Moderation stages** — OpenAI `moderation_stage: input|output|unknown`, `moderation_details.categories`.
- **Copyright detection** — Ideogram `enable_copyright_detection` (Hive likeness + logo checks).
- **Person generation** — Google Veo `personGeneration: allow_all|allow_adult|dont_allow`.
- **Safe search** — Google Cloud Vision `SAFE_SEARCH_DETECTION` (adult/spoof/medical/violence/racy likelihood enum).
- **NSFW classification** — Replicate `falcons-ai/nsfw_image_detection`.
- **Watermarking** — Google SynthID (invisible watermark on all generated images and Veo outputs).

### 2.13 Files API / Storage
Persisting inputs and outputs for reuse across requests:
- **OpenAI Files API** — `purpose="vision"` uploads → `file_id` referenced by Responses API and video `input_reference`.
- **Google Files API** — resumable upload → `uri` + `name`; poll `state` PROCESSING→ACTIVE. Recommended for >100MB / reuse.
- **xAI Files API** — bidirectional: inputs substitute `file_id` for url/base64; outputs via `storage_options` with optional permanent public URL.
- **BFL** — `input_image_blob_path` (hosted blob) on FLUX.2 [flex].
- **Reve project references** — `ImageInput.ref = id:<uuid>` or `reference:@<name>`.
- **Ephemeral URLs** — all generated URLs expire (BFL, Ideogram, Recraft ~24h, OpenAI video 1h / batch 24h, xAI, Reve). Download promptly or persist via Files API.

### 2.14 Billing Units
- **Credits** — BFL (1 credit = $0.01), Recraft (1,000 units = $1), Reve (750 credits ≈ $1), Ideogram, DaVinci.
- **Per image** — most image generation (flat or megapixel-based on FLUX.2).
- **Per output-image tokens** — OpenAI GPT Image (driven by quality + size).
- **Per patch/tile (vision input)** — OpenAI mainline vision (32×32 patches or 512px tiles).
- **Per second (video)** — xAI video, Google Veo (driven by model/resolution/duration).
- **Per request** — Recraft transformations, Reve operations, BFL tools.
- **Surcharges** — OpenAI partial images (+100 output tokens each); Reve postprocessing (upscale/effect adds cost; fit_image free).

### 2.15 Aspect Ratio & Resolution
Output dimensions are specified by either:
- **Explicit pixel dimensions** — `1024x1024`, `2048x2048` (OpenAI, Recraft `WxH`, Ideogram `ResolutionV3`).
- **Aspect ratio** — `16:9`, `9:16`, `1:1` (BFL, Google, xAI, Reve, Recraft `w:h`, Ideogram `aspect_ratio`).
- **Size tier** — Google `image_size: 512px|1K|2K|4K`; OpenAI `quality: low|medium|high`.
- **Megapixel budget** — BFL FLUX.2 (cost scales with output MP); OpenAI `gpt-image-2` (any resolution within constraints).
- **`auto`** — model selects dimensions from the prompt (OpenAI, Google, Reve v2, xAI).

## 3. Cross-Provider Synonym Glossary

The table below maps the **many different names** providers use for the **same concept**. When reading any provider's docs, use this to translate to the unified vocabulary.

| Unified concept | OpenAI | Google Gemini | xAI | BFL (FLUX) | Ideogram | Recraft | Reve | Classical Vision (Google Cloud Vision / Azure / Replicate) |
|----------------|--------|---------------|-----|------------|----------|---------|------|------------------------------------------------------------|
| API key header | `Authorization: Bearer` | `x-goog-api-key` | `Authorization: Bearer` | `x-key` | `Api-Key` | `Authorization: Bearer` | `Authorization: Bearer` | `Authorization: Bearer` (GCV) / `Ocp-Apim-Subscription-Key` (Azure) / `Authorization: Token` (Replicate) |
| Text-to-image generation | `/v1/images/generations` (Images API) or `image_generation` tool (Responses) | Interactions API (`gemini-*-image`) | `/v1/images/generations` | `/v1/flux-2-{pro,max,flex,klein}` | `/v1/ideogram-v4/generate` | `/v1/images/generations` | `/v1/image/create` | — |
| Image editing (prompt-based) | `/v1/images/edits` or `image_generation` tool with `input_image` | Interactions (text+image input) | `/v1/images/edits` | same endpoint + `input_image` | `/v1/edit` or `/v1/ideogram-v4/remix` | `/v1/images/imageToImage` | `/v1/image/edit` | — |
| Multi-reference editing | `image` array (Images) / multiple `input_image` (Responses) | up to 14 reference image parts | up to 3 source images | `input_image` … `input_image_8` | up to 10 images | via `imageToImage` + style refs | 1–6 `reference_images` | — |
| Image understanding (vision) | `input_image` (Responses/Chat) | Interactions (input image part) | (mainline Grok chat, out of Imagine scope) | — | `/v1/ideogram-v4/describe` (→ V4JsonPrompt) | — (no vision API) | `/v2/image/image_to_layout` | `images:annotate` (GCV) / `imageanalysis:analyze` (Azure) / per-model (Replicate) |
| Mask | `input_image_mask.file_id` (alpha) | `mask` polygon | — | `mask` (B/W) or alpha | `mask` (B/W, black=edit) | `mask`/`mask_url` (grayscale, white=modify) | — (no mask endpoint) | — |
| Inpainting | Images `/edits` with mask | Interactions (image+prompt) | — | `/v1/flux-pro-1.0-fill` | `/v1/ideogram-v3/inpaint` | `/v1/images/inpaint` | — | — |
| Outpainting / border extension | — | — | — | `/v1/flux-tools/outpainting-v1` or `/v1/flux-pro-1.0-expand` | `/v1/ideogram-v3/reframe` | `/v1/images/outpaint` | — | — |
| Background removal | — | — | — | — | `/v1/remove-background` | `/v1/images/removeBackground` | postprocessing `remove_background` | — |
| Background replace | — | — | — | — | `/v1/ideogram-v3/replace-background` | `/v1/images/replaceBackground` | — | — |
| Object removal / erase | — | — | — | `/v1/flux-tools/erase-v1` | — | `/v1/images/eraseRegion` | — | — |
| Upscale (preserve content) | — | — | — | — | `/upscale` (`resemblance`/`detail`) | `/v1/images/crispUpscale` | postprocessing `upscale` | — |
| Upscale (regenerate detail) | — | — | — | — | `/upscale` (creative) | `/v1/images/creativeUpscale` | postprocessing `upscale` (factor 4) | — |
| Deblur | — | — | — | `/v1/flux-tools/deblur-v1` | — | — | — | — |
| Virtual try-on | — | — | — | `/v1/flux-tools/vto-v1` | — | — | — | — |
| Vectorization (raster→SVG) | — | — | — | — | — | `/v1/images/vectorize` | — | — |
| Transparent background gen | — | — | — | — | `/v1/ideogram-v3/generate-transparent` | — (use removeBackground) | postprocessing `remove_background` | — |
| Text layer extraction | — | — | — | — | `/v1/ideogram-v3/layerize-text` | — | — | OCR (Florence-2, GCV, Azure Read) |
| Prompt enhancement | `revised_prompt` (returned) | thinking process (internal) | — | `prompt_upsampling` / `disable_pup` | `magic_prompt` (AUTO/ON/OFF) + `/v1/ideogram-v4/magic-prompt` | `/v1/prompts/enhance` | auto-enhanced (silent) | — |
| Style — curated name | — | — | — | — | `style_type` | `style` | — | — |
| Style — preset | — | — | — | — | `style_preset` (~60) | — | — | — |
| Style — code | — | — | — | — | `style_codes` (8-char hex) | — | — | — |
| Style — reference images | — | style reference part | — | — | `style_reference_images` | `style_reference_images` | — | — |
| Custom style / model | — | — | — | LoRA `finetune_id` + `finetune_strength` | `custom_model_uri` (trained 15–100 imgs) | `style_id` (from `/v1/styles`) | — | Azure Custom Vision (train your own) |
| Color palette | — | — | — | hex colors in prompt | `color_palette` (preset or members) | `controls.colors` | — | — |
| Character reference | — | character ref part | — | — | `character_reference_images` (+mask) | — | — | — |
| Seed / determinism | `seed` | `seed` (Veo) | — | `seed` | `seed` | `random_seed` | — | — |
| Negative prompt | — | — | — | — (no negative prompts) | `negative_prompt` | `negative_prompt` | — | — |
| Aspect ratio | `size` (`1024x1024` etc.) or `auto` | `response_format.aspect_ratio` | `aspect_ratio` | `aspect_ratio` or `width`/`height` | `aspect_ratio` or `resolution` | `size` (`WxH` or `w:h`) | `aspect_ratio` | — |
| Resolution / quality | `quality: low/medium/high` | `image_size: 512px/1K/2K/4K` | `resolution: 1k/2k` | megapixel-based (any res within limits) | `resolution` enum | model variant (1MP vs 4MP Pro) | `test_time_scaling` (quality) | — |
| Output format | `output_format: png/jpeg/webp` | `response_format.mime_type` | `response_format: url/b64_json` | `output_format: jpeg/png/webp` | (URL only) | `response_format: url/b64_json` | `Accept` header (`image/png` etc.) | — |
| Number of images | `n` | — | `n` (up to 10) | — | `num_images` | `n` (1–6) | — | — |
| Streaming / partial images | `stream` + `partial_images` (0–3) | — | — | — | — | — | — | — |
| Video generation (text) | `/v1/videos` (`sora-2/2-pro`) | `models.generate_videos` (Veo) | `/v1/videos/generations` | — (no video API) | — (no video) | — (Studio-only) | — (no video) | — |
| Image-to-video (first frame) | `input_reference` | `image` | `image` | — | — | — | — | — |
| Last-frame interpolation | — | `image` + `lastFrame` | — | — | — | — | — | — |
| Reference images (video) | `characters` (non-human) | `referenceImages` (up to 3) | `reference_images` + `<IMAGE_N>` tags | — | — | — | — | — |
| Video extension | `/v1/videos/extensions` (+20s, ≤6×, ≤120s) | `video` (+7s, ≤20×, ≤148s, 720p) | `/v1/videos/extensions` (`duration` = extension only) | — | — | — | — | — |
| Video edit | `/v1/videos/edits` | — | `/v1/videos/edits` (inherits props, cap 720p/8.7s) | — | — | — | — | — |
| Async lifecycle | poll `/videos/{id}` or webhook | poll operation `done` | poll `/videos/{request_id}` (`pending/done/failed/expired`) | poll `/get_result?id` or webhook | poll `/generations/{id}` or webhook | — (synchronous) | — (synchronous) | Replicate poll predictions; GCV async batch |
| Files API (inputs) | `POST /v1/files` (`purpose=vision`) → `file_id` | `POST /upload/v1beta/files` → `uri`/`name` | `file_id` substitutes url/base64 | `input_image_blob_path` (flex) | — (URLs only) | — (multipart or URL) | project `id:<uuid>` / `reference:@<name>` | — |
| Files API (outputs) | — | — | `storage_options` → `file_output` + optional public URL | — | — | — (URL ~24h) | — | — |
| Bounding box (detection) | — | `[ymin,xmin,ymax,xmax]` 0–1000 (Gemini) | — | — | `bbox` `[y_min,x_min,y_max,x_max]` 0–1000 | — | layout `Bbox` `[0,1]` | XYXY/XYWH/polygon (varies) |
| Object detection | — (vision via chat) | Gemini object detection (boxes+labels) | — | — | — | — | layout regions | GCV `OBJECT_LOCALIZATION`, Azure `Objects`, Replicate Grounding DINO/YOLO-World/Florence-2 |
| Segmentation | — | Gemini segmentation (polygon mask) | — | — | — | — | — | Replicate SAM-2, Grounded SAM, Semantic-Segment-Anything |
| Image classification / tagging | — | — | — | — | — | — | — | GCV `LABEL_DETECTION`, Azure `Tags`, Replicate Florence-2, Custom Vision classifier |
| OCR | — (vision via chat) | — (vision via chat) | — | — | — | — | — | GCV `TEXT_DETECTION`/`DOCUMENT_TEXT_DETECTION`, Azure `Read`, Florence-2 `<OCR>` |
| Captioning | — (vision via chat) | — (vision via Gemini) | — | — | `describe` → V4JsonPrompt | — | — | Azure `Caption`/`denseCaptions`, Florence-2 `<CAPTION>` |
| Face / pose detection | — | — | — | — | — | — | — | GCV `FACE_DETECTION`, MediaPipe (Replicate), Azure Face API (separate) |
| Landmark / logo recognition | — | — | — | — | — | — | — | GCV `LANDMARK_DETECTION`, `LOGO_DETECTION` |
| Web / reverse image | — | — | — | — | — | — | — | GCV `WEB_DETECTION` |
| Safe search / content moderation | `moderation` | SynthID + safety filters | `respect_moderation` | `safety_tolerance` | `is_image_safe` + copyright detection | — | `content_violation` | GCV `SAFE_SEARCH_DETECTION`, Replicate NSFW ViT |
| Smart cropping | — | — | — | — | — | — | — | GCV `CROP_HINTS`, Azure `SmartCrops` |
| Object tracking (video) | — | — | — | — | — | — | — | Replicate SAMURAI, YOLO-World (video), SAM-2-video |
| Region caption / dense caption | — | — | — | — | — | — | — | Azure `denseCaptions`, Florence-2 `<DENSE_REGION_CAPTION>` |
| Caption-to-phrase grounding | — | — | — | — | — | — | — | Florence-2 `<CAPTION_TO_PHRASE_GROUNDING>`, Grounded SAM |
| Structured prompt (JSON) | — | — | — | JSON prompting (subject, background, lighting…) | `V4JsonPrompt` (high_level_description + style_description + compositional_deconstruction) | — | `Layout` (regions with label/prompt/bbox/region_type) | — |
| Layout rendering | — | — | — | — | — | — | `/v2/image/render` (layout→image) | — |
| Layout from image | — | — | — | — | `describe` (→ V4JsonPrompt with bboxes) | — | `/v2/image/image_to_layout` | — |
| Effects / filters | — | — | — | — | — | — | postprocessing `effect` (named presets) | — |
| Fit image (scale down) | — | — | — | — | — | — | postprocessing `fit_image` (free) | — |
| Webhooks | video `completed`/`failed` | — | — | `webhook_url` + `webhook_secret` | Ed25519-signed webhook + JWKS | — | — | Replicate webhooks |
| Grounding / web search | — | `google_search` tool (web + image search) | — | FLUX.2 [max] grounding search | — | — | — | — |

---

# Part II — The Exhaustive Processing Pipeline

This section orders **every** capability into a single end-to-end pipeline. The pipeline represents the full lifecycle of an image & video AI system, from setup through delivery. Each stage notes the **alternative approaches** available and the **synonyms** used across providers.

```
Stage 0:  Authentication & Access Control
Stage 1:  Model & Style Asset Management
Stage 2:  Image & Video Input Preprocessing
Stage 3:  Image Understanding (Vision / Analysis)
Stage 4:  Prompt Construction & Enhancement
Stage 5:  Image Generation
Stage 6:  Image Editing & Transformation
Stage 7:  Image Format & Structure Conversion
Stage 8:  Video Generation
Stage 9:  Video Editing, Extension & Interpolation
Stage 10: Postprocessing & Effects
Stage 11: Output Formatting & Delivery
Stage 12: Async Job Lifecycle, Storage & Management
```

---

## Stage 0 — Authentication & Access Control

### 0.1 API Key Authentication
Every request is authenticated with an API key. The header name and format differ by provider (see glossary §3).

**Key management capabilities:**
- **Organizations & Projects** — keys organized for access control (BFL).
- **Subscription-tiered access** — model availability scales with plan tier (DaVinci Basic→Creator→Pro→Ultimate; Recraft; Ideogram).
- **Verification gates** — OpenAI GPT Image requires Organization Verification; OpenAI uploaded-video editing and human-likeness characters are gated (contact sales).

### 0.2 Files API Tokens / Ephemeral References
For inputs that must be reused across requests, platforms provide upload-then-reference flows:
- **OpenAI** — `POST /v1/files` with `purpose="vision"` → `file_id`.
- **Google** — resumable upload → `uri` + `name`; poll `state` PROCESSING→ACTIVE.
- **xAI** — Files API with bidirectional input/output; `file_id` substitutes for url/base64; outputs via `storage_options` with optional permanent public URL and expiry.
- **Reve** — project references `id:<uuid>` (image/generation) and `reference:@<name>` (named entity).
- **BFL** — `input_image_blob_path` (hosted blob, FLUX.2 [flex] only).

### 0.3 Privacy, Data Residency & Training Opt-Out
- **Not used for training** — xAI, DaVinci (outputs private by default, never used for training).
- **Zero retention** — enterprise options (xAI SOC 2/HIPAA/GDPR with EU data residency).
- **Watermarking / provenance** — Google SynthID (invisible watermark on all generated images and Veo outputs).

### 0.4 API Versioning
- **Dated snapshots** — Reve `reve-create@20250915`, BFL model versions, Google `v1beta`/`v1alpha`.
- **Rolling aliases** — `latest`, `latest-fast`, `auto`.
- **No explicit versioning** — OpenAI, Ideogram, Recraft (use model IDs for reproducibility).
- **Deprecation notices** — OpenAI Sora 2 shuts down 2026-09-24; Google Imagen shut down 2026-08-17; Azure Custom Vision retirement 2025–2028.

---

## Stage 1 — Model & Style Asset Management

### 1.1 Model Catalog & Selection
Providers expose multiple models with different trade-offs. Selection is via `model` parameter or URL path.

**Image generation models (by provider):**

| Provider | Flagship (quality) | Default / balanced | Fast / lite |
|----------|--------------------|--------------------|-------------|
| OpenAI | `gpt-image-2` | `gpt-image-1.5` | `gpt-image-1-mini` |
| Google | Nano Banana Pro (`gemini-3-pro-image`) | Nano Banana 2 (`gemini-3.1-flash-image`) | Nano Banana 2 Lite (`gemini-3.1-flash-lite-image`) |
| xAI | `grok-imagine-image-quality` | same | — |
| BFL | FLUX.2 [max] | FLUX.2 [pro] | FLUX.2 [klein 4B/9B], FLUX.2 [flex] |
| Ideogram | Ideogram 4.0 (`QUALITY`) | Ideogram 4.0 (`DEFAULT`) | Ideogram 4.0 (`TURBO`/`FLASH`*) |
| Recraft | `recraftv4_1_pro` (4MP) | `recraftv4_1` (1MP) | — |
| Reve | `latest` (standard) | — | `latest-fast` |
| DaVinci | aggregates all above | — | — |

**Video generation models:** OpenAI Sora 2 / Sora 2 Pro; Google Veo 3.1 / 3.1 Fast / 3.1 Lite / Veo 3 / Veo 2; xAI `grok-imagine-video` / `grok-imagine-video-1.5` (1080p I2V only); plus DaVinci-aggregated Kling, Seedance, Wan.

### 1.2 Style Asset Management
Mechanisms to create, store, and reuse visual styles:

| Mechanism | Providers | Description |
|-----------|-----------|-------------|
| **Curated style names** | Recraft (`style`), Ideogram (`style_type`) | Named aesthetic categories |
| **Style presets (~60)** | Ideogram (`style_preset`) | Named artistic presets (ART_DECO, WATERCOLOR, BAUHAUS…) |
| **Style codes** | Ideogram (`style_codes`) | 8-char hex codes, shareable |
| **Style reference images** | Recraft, Ideogram, Google | Images whose style is transferred |
| **Custom style (style_id)** | Recraft (`POST /v1/styles`, up to 5 refs, V3/V3 Vector only) | Reusable UUID |
| **Color palette** | Recraft (`controls.colors`), Ideogram (`color_palette`) | Preset name or hex+weight members |
| **LoRA finetune** | BFL (`finetune_id` + `finetune_strength` 0–2) | Trained on user dataset |
| **Custom model** | Ideogram (`custom_model_uri`, 15–100 images) | Trained model enforcing brand style |
| **Azure Custom Vision** | Azure (deprecated 2025–2028) | Train-your-own classifier/detector, export to ONNX/TF/CoreML/Docker |

**Style exclusivity rules (Ideogram V3):** `style_codes` cannot combine with `style_reference_images` or `style_type`; `color_palette` cannot mix preset and members; `resolution` and `aspect_ratio` are mutually exclusive.

**Recraft style compatibility:** Styles (`style`/`style_id`) are **only supported on V2 and V3 models** — V4/V4.1 ignore style parameters.

### 1.3 Character Reference Management
Keep a character consistent across generations/edits:
- **Ideogram** — `character_reference_images` (currently 1) + optional `character_reference_images_mask` (grayscale, same dims).
- **Google** — character reference parts (up to 4–5 depending on model).
- **OpenAI Sora** — reusable `characters` assets (non-human, up to 2 per video, mention name in prompt).
- **DaVinci/Kling** — "locked character consistency" across variations.

### 1.4 Effects Library (Reve)
Named visual filter presets (built-in + project-saved), listed via `GET /v1/image/effect`, applied as a postprocessing step with optional `effect_parameters` overrides.

---

## Stage 2 — Image & Video Input Preprocessing

### 2.1 Input Image Formats

| Provider | Accepted formats | Max size |
|----------|------------------|----------|
| OpenAI (vision) | PNG, JPEG, WebP, non-animated GIF | 512 MB payload, 1500 images |
| OpenAI (Sora input_reference) | JPEG, PNG, WebP | must match `size` |
| Google | PNG, JPEG, WebP, HEIC, HEIF | inline <100MB; Files API 2GB |
| xAI | PNG, JPEG, WebP (images); MP4 (videos) | — |
| BFL | base64 or URL (jpeg/png/webp) | — |
| Ideogram | JPEG, PNG, WebP | 10 MB/image |
| Recraft | PNG, JPG, WEBP, SVG (some endpoints) | <10MB, ≤16MP, max dim ≤4096px, min dim ≥256px (crisp upscale ≥32px) |
| Reve | WEBP, JPEG, PNG, GIF, TIFF, AVIF (base64) | ≤40MB/image, ≤33,554,432 px; per-call ≤50,331,648 px / 100MB |
| Classical vision | varies (GCV: most; Azure: standard) | GCV async up to 2000 files |

### 2.2 Input Methods

| Method | Providers |
|--------|-----------|
| **Public URL** | OpenAI, Google, xAI, BFL, Ideogram (`image_urls`), Recraft (`image_url`), Reve (base64 only) |
| **Base64 data URL** | OpenAI, Google, xAI, BFL, Ideogram (multipart), Recraft (`data:image/...;base64`), Reve (base64 inline) |
| **File ID / URI** (Files API) | OpenAI (`file_id`), Google (`uri` from Files API), xAI (`file_id`), Reve (`id:<uuid>` / `reference:@<name>`) |
| **Multipart upload** | OpenAI Images, Ideogram, Recraft, Azure |
| **Hosted blob path** | BFL (`input_image_blob_path`, FLUX.2 [flex] only) |
| **Cloud Storage (GCS)** | Google Cloud Vision (`source.imageUri`) |

### 2.3 Vision Input Detail / Resolution Control (OpenAI)
`detail` parameter: `low` (512×512, fast/cheap), `high` (standard fidelity), `original` (preserves input dimensions, `gpt-5.4+`), `auto` (default; =`original` on 5.5/5.6, =`high` on 5.4).

### 2.4 Media Resolution (Google)
`media_resolution` (Gemini 3+) controls max tokens per input image/video frame — higher = better fine-text/detail reading but more tokens and latency.

### 2.5 Multichannel / Multi-Image Handling
- **OpenAI** — up to 1500 image inputs per request (vision).
- **Google** — up to 3,600 image files per request (vision).
- **Ideogram edit** — up to 10 images per call (files or URLs).
- **Reve remix** — 1–6 reference images.
- **xAI multi-image edit** — up to 3 source images (mix url/file_id kinds).
- **BFL FLUX.2** — up to 8 reference images (API) / 10 (playground); 9MP combined budget on [pro].
- **Google 3.1 Flash** — up to 14 object refs + 4 character refs + 3 style refs (roles).

### 2.6 Video Input Preprocessing
- **OpenAI Sora** — `input_reference` image must match target video `size`; supported `image/jpeg`, `image/png`, `image/webp`.
- **xAI** — image-to-video: image becomes first frame; `aspect_ratio` defaults to input image ratio.
- **Google Veo** — `image` + optional `lastFrame` for interpolation; `referenceImages` (up to 3, Veo 3.1 only) with `reference_type`.
- **Replicate video tracking** — first-frame prompt (point or box) for SAMURAI; video file for YOLO-World/SAM-2-video.

---

## Stage 3 — Image Understanding (Vision / Analysis)

This stage encompasses all capabilities that derive meaning from existing images. Two architectural approaches:

**Approach A — Generative multimodal vision (chat-based):** A mainline multimodal model analyzes the image and returns free-form or structured text. Used by OpenAI (Responses/Chat), Google (Interactions API), and implicitly by xAI (mainline Grok, out of Imagine scope).

**Approach B — Classical fixed-feature detection:** A hosted API runs many specialized detectors in one call. Used by Google Cloud Vision (`images:annotate`), Azure Computer Vision (`imageanalysis:analyze`), and Replicate (per-model predictions).

**Approach C — Train-your-own:** You train a classifier/detector on labeled images. Azure Custom Vision (deprecated 2025–2028).

### 3.1 Image Classification & Tagging
Assign textual categories to an image.

| Provider | Mechanism | Vocabulary |
|----------|-----------|------------|
| Google Cloud Vision | `LABEL_DETECTION` | Fixed |
| Azure Computer Vision | `features=Tags` | Fixed |
| Replicate Florence-2 | `<CAPTION>` task | Open (via task prompt) |
| Replicate YOLO-World | classification mode | Open (text-prompted) |
| Azure Custom Vision | Classification project | User-defined |
| Google Gemini | structured output (JSON schema) | Open (prompt-defined) |

### 3.2 Object Detection
Return bounding boxes with labels and confidence.

**Open-vocabulary (text→boxes):** Replicate Grounding DINO (`query`, `box_threshold`, `text_threshold`), YOLO-World (`class_names`, `score_thr`, `nms_thr`), OWL-ViT, Grounded SAM, Florence-2 `<OD>`.

**Fixed-vocabulary:** Google Cloud Vision `OBJECT_LOCALIZATION` (`NormalizedVertex` 0–1), Azure `Objects` (`boundingBox:{x,y,w,h}` pixels).

**Train-your-own:** Azure Custom Vision Object Detection project (regions drawn at training time).

**Gemini structured detection:** Returns `[ymin,xmin,ymax,xmax]` normalized 0–1000 + `mask` polygon + `label` via `response_format` JSON schema.

### 3.3 Semantic & Instance Segmentation
Per-pixel object extent.

- **Semantic segmentation** — class label per pixel (Replicate Semantic-Segment-Anything, Mask2Former on ADE20k).
- **Instance / promptable segmentation** — one mask per object instance; SAM-family is promptable (points/boxes) (Replicate `meta/sam-2`, `meta/sam-2-video`).
- **Grounded segmentation** — open-vocab detection (Grounding DINO) + mask generation (SAM) chained (Replicate `schananas/grounded_sam`).
- **Gemini segmentation** — `box_2d` + `mask` polygon (0–1000) + `label`.

**Mask encodings:** PNG image (SAM-2), COCO RLE (`{size:[h,w],counts}` — Semantic-Segment-Anything, SAMURAI), polygon (Gemini). Quality scores: `predicted_iou`, `stability_score`.

**Specialized segmentation models:** `ahmdyassr/mask-clothing` (clothing), `hadilq/hair-segment` (hair), `swook/inspyrenet` (salient foreground).

### 3.4 Image Captioning & Dense Captioning
- **Caption** — one sentence describing the whole image. Florence-2 `<CAPTION>`/`<DETAILED_CAPTION>`/`<MORE_DETAILED_CAPTION>`, Azure `Caption` (Florence-based, English-only, `gender-neutral-caption` option), Gemini (via prompt).
- **Dense captions** — many short captions each localized to a region (box + caption). Azure `denseCaptions` (up to 10), Florence-2 `<DENSE_REGION_CAPTION>`.
- **Caption-to-phrase grounding** — given a caption, locate each noun phrase. Florence-2 `<CAPTION_TO_PHRASE_GROUNDING>` (with `text_input`), Grounded SAM.
- **Structured describe (round-trip)** — Ideogram `/v1/ideogram-v4/describe` returns `V4JsonPrompt` (with optional `bbox` per element) reusable as `json_prompt` for generation; `include_bbox` controls layout preservation.
- **Layout extraction** — Reve `/v2/image/image_to_layout` returns a `Layout` of typed `Region`s (`coarse_detail`/`medium_detail`/`fine_detail`/`text`/`hand`/`face`) with parent/child hierarchy.

### 3.5 Optical Character Recognition (OCR)
- **Image OCR (in-the-wild)** — Google Cloud Vision `TEXT_DETECTION` (textAnnotations, full string + per-word boxes), Azure `Read` (synchronous, `blocks[].lines[].words[]` with `boundingPolygon` 4 points + confidence), Florence-2 `<OCR>` (text string).
- **Document OCR (dense/handwriting/PDF)** — Google `DOCUMENT_TEXT_DETECTION` (hierarchical pages→blocks→paragraphs→words→symbols, language hints via `imageContext.languageHints`), Azure Document Intelligence Read (async), Florence-2 `<OCR_WITH_REGION>` (quad boxes, 8 coords for tilted text).
- **Text layer extraction (editable)** — Ideogram `/v1/ideogram-v3/layerize-text` returns `base_image_url` (text erased) + `text_blocks[]` (geometry, font_name, font_size, color, alignment, role).

### 3.6 Face & Pose Detection
- **Google Cloud Vision `FACE_DETECTION`** — `faceAnnotations[]`: `boundingPoly`/`fdBoundingPoly`, `landmarks[]` (~30+ types with 3D `{x,y,z}`), `rollAngle`/`panAngle`/`tiltAngle`, `detectionConfidence`, joy/sorrow/anger/surprise/underExposed/blurred/headwear `Likelihood` enum.
- **Replicate MediaPipe Face** — `chigozienri/mediapipe-face`: full facial mesh / pose landmarks (richer keypoint set).
- **Azure** — faces removed from v4.0 (gated v3.2); use dedicated Face API service.
- **Facial recognition** (identifying *who*) — **not** supported by Google; Azure via separate Face API.

### 3.7 Object Tracking in Video
- **Tracking** = detection + temporal ID continuity across frames.
- **Zero-shot tracking** — prompt in first frame (point or box), model follows object. Replicate `zsxkib/samurai` (SAM 2 + motion-aware memory, COCO RLE per frame keyed by `object_id`).
- **YOLO-World video mode** — open-vocab detection+tracking across frames (`class_names`, annotated video or per-frame JSON).
- **SAM-2-video** — prompt object in one frame, get masks across all frames.
- **Hosted** — neither Google nor Azure offers video object tracking.

### 3.8 Specialized Detectors (Google Cloud Vision, mostly)
| Feature | Returns | Notes |
|---------|---------|-------|
| `LANDMARK_DETECTION` | `landmarkAnnotations[]` (name, score, `boundingPoly`, `locations[]` lat/long) | Natural/human-made monuments |
| `LOGO_DETECTION` | `logoAnnotations[]` (name, score, `boundingPoly`) | Company logos |
| `WEB_DETECTION` | `webEntities[]`, `visuallySimilarImages[]`, `fullMatchingImages[]`, `partialMatchingImages[]`, `pagesWithMatchingImages[]` | Reverse image / web entity lookup |
| `SAFE_SEARCH_DETECTION` | `safeSearchAnnotation` (adult, spoof, medical, violence, racy — `Likelihood` enum) | Content moderation |
| `IMAGE_PROPERTIES` | `dominantColors.colors[]` (color + pixel fraction + score) | Dominant-color palette |
| `CROP_HINTS` | `cropHints[].boundingPoly` | Suggested crop vertices |
| `PRODUCT_SEARCH` | Product matches in a product set | Retail product recognition |

**Replicate content moderation:** `falcons-ai/nsfw_image_detection` (fine-tuned ViT for NSFW classification).

**Azure v4.0 removed:** Brands, Landmarks, Celebrities, Adult, Color, Image type (exist in v3.2, now in other Azure services).

### 3.9 Multi-Feature Batch Annotation
- **Google Cloud Vision** — one `images:annotate` call runs a batch of `AnnotateImageRequest`s, each declaring a list of `features` (enum). A single call returns labels + faces + OCR + objects + safe-search simultaneously. Async `files:asyncBatchAnnotate` for up to 2000 files (PDF/TIFF/batch), output to Cloud Storage.
- **Azure Computer Vision v4.0** — `features` query param (`Caption`, `Tags`, `Read`, `Objects`, `People`, `SmartCrops`, `denseCaptions`); one result block per feature, synchronous.

### 3.10 Confidence / Likelihood Reporting
- **Float 0–1** — Replicate (`confidence`, `score`), Google (`score`, `detectionConfidence`), Azure (`confidence`). Caller chooses thresholds; platforms impose no single cutoff.
- **Likelihood enum** — Google `UNKNOWN(0)→VERY_LIKELY(5)` for face attributes and safe-search.
- **Threshold parameters** — Grounding DINO `box_threshold`/`text_threshold` (independent recall vs label-precision knobs); YOLO-World `score_thr`/`nms_thr`; SAM-2 `pred_iou_thresh`/`stability_score_thresh`.

---

## Stage 4 — Prompt Construction & Enhancement

### 4.1 Plain-Text Prompts
All platforms accept natural-language prompts. Length limits vary: Reve ≤2560 chars; Recraft V4/V4.1 ≤10,000 chars, V2/V3 ≤1,000 chars; Ideogram V3 prompt (no fixed limit stated); BFL (no fixed limit); OpenAI (no fixed limit).

### 4.2 Structured (JSON) Prompts
A machine-readable prompt contract for precise spatial/compositional control:

- **Ideogram `V4JsonPrompt`** — `high_level_description` + optional `style_description` (`aesthetics`, `art_style`, `lighting`, `medium`, `photo`) + required `compositional_deconstruction` (`background` + ordered `elements[]`). Each element: `type: "obj"|"text"`, `desc`/`text`, optional `bbox` `[y_min,x_min,y_max,x_max]` in `[0,1000]`. Disables magic-prompt.
- **Reve `Layout`** — `regions[]` each with `label`, `prompt`, `bbox` (normalized `[0,1]` top-left origin), optional `image_index`, `parent`, `region_type` (`coarse_detail`/`medium_detail`/`fine_detail`/`text`/`hand`/`face`). Optional `width`/`height` (multiples of 32, `width*height` between `3072×2560` and `4096×4096`).
- **BFL JSON prompting** — FLUX.2 accepts JSON-structured prompts (subject, background, lighting, style, camera_angle, etc.).
- **Recraft `text_layout`** — array of `{text, bbox}` placing individual words as 4-point polygons (relative coords 0,0 top-left to 1,1 bottom-right; coords can exceed [0,1]). V3/V3 Vector only; limited character set.

### 4.3 Prompt Enhancement / Rewriting
Automatic or on-demand rewriting for improved generation:

| Provider | Mechanism | Control | Returned? |
|----------|-----------|---------|-----------|
| OpenAI | `revised_prompt` (Responses `image_generation` tool) | automatic | Yes |
| BFL | Prompt Upsampling (PUP) | `disable_pup` (FLUX.2 pro/max) / `prompt_upsampling` (older, default false; flex default true) | No |
| Ideogram | Magic Prompt | `magic_prompt: AUTO/ON/OFF` (V3) / automatic when `text_prompt` (V4) / disabled when `json_prompt` (V4); dedicated `/v1/ideogram-v4/magic-prompt` endpoint | Yes (dedicated endpoint returns `V4JsonPrompt`) |
| Recraft | Enhance Prompt | `POST /v1/prompts/enhance` (dedicated, ≤2000 chars) | Yes (`enhanced_prompt`) |
| Reve | Auto-enhanced | automatic, silent | No (not surfaced) |
| Google | Thinking process | `generation_config.thinking_level: minimal|high` (default minimal); enabled by default, cannot disable | Yes (thought steps in `steps`, thought images not billed) |

### 4.4 Negative Prompts
Exclude undesired elements. Supported by Ideogram V3 (`negative_prompt`), Recraft V2/V3 (`negative_prompt`). **Not** supported by BFL FLUX (control via prompt structure, guidance, reference images).

### 4.5 Reference-Image Tagging in Prompts
Refer to specific reference images by index within the prompt:
- **Reve** — `<img>N</img>` (v1 docs/SDK also `<ref>N</ref>`), 0-indexed.
- **xAI reference-to-video** — `<IMAGE_1>`, `<IMAGE_2>`, `<IMAGE_3>` placeholders.
- **BFL FLUX.2** — describe roles ("person of image 1 wearing garments of image 2").
- **OpenAI Sora characters** — mention character name verbatim in prompt.

### 4.6 Grounding / Real-Time Information
- **Google** — `google_search` tool in Interactions API; `search_types: ["web_search","image_search"]` (3.1 Flash). Must render `search_suggestions` HTML. Not supported by Flash Lite.
- **BFL FLUX.2 [max]** — grounding search grounds generations in real-time info (trending products, events, scores, weather).

---

## Stage 5 — Image Generation

### 5.1 Text-to-Image Generation
Synthesize a new raster image from a text prompt (and optionally reference images).

**Alternative API patterns:**

| Pattern | Providers | Description |
|---------|-----------|-------------|
| **Dedicated Images endpoint** | OpenAI `/v1/images/generations`, xAI, BFL (per-model URL), Ideogram, Recraft | Pick model directly; single-shot |
| **Unified Interactions API** | Google | Same endpoint handles understanding + generation; pick model + `response_format` |
| **Built-in tool in chat** | OpenAI Responses (`image_generation` tool) | Mainline model hosts tool; conversational multi-turn editing |
| **Simple create endpoint** | Reve `/v1/image/create` | Text + aspect ratio + version + postprocessing |
| **Layout-aware create** | Reve v2 `create` | Returns image + echoed `layout` |

**Key generation parameters (union of all providers):**
- `prompt` / `text_prompt` / `input` (text)
- `json_prompt` (structured, Ideogram V4 / Reve v2 / BFL)
- `model` / `model_id`
- `size` / `aspect_ratio` / `resolution` (mutually exclusive in some providers)
- `quality` (OpenAI low/medium/high/auto) / `image_size` (Google 512px/1K/2K/4K) / `rendering_speed` (Ideogram FLASH/TURBO/DEFAULT/QUALITY) / `test_time_scaling` (Reve 1–15)
- `n` / `num_images` (number of images per request)
- `output_format` (png/jpeg/webp) / `response_format` (url/b64_json)
- `output_compression` (0–100%, OpenAI jpeg/webp)
- `background` (opaque/automatic/transparent, OpenAI; transparent unsupported on `gpt-image-2`)
- `seed` / `random_seed` (determinism)
- `negative_prompt` (Ideogram, Recraft)
- `style` / `style_id` / `style_type` / `style_preset` / `style_codes` / `style_reference_images` / `color_palette` (Ideogram, Recraft)
- `custom_model_uri` (Ideogram trained model)
- `character_reference_images` + `_mask` (Ideogram)
- `controls` (Recraft: colors, background_color, artistic_level 0–5 V3, no_text V3)
- `text_layout` (Recraft V3)
- `guidance` (BFL flex 1.5–10; dev 1.5–5.0; Fill 1.5–100) / `steps` (BFL flex 1–50; dev 1–50; Fill 15–50)
- `disable_pup` / `prompt_upsampling` (BFL)
- `moderation` (OpenAI auto/low) / `safety_tolerance` (BFL 0–5/0–6)
- `enable_copyright_detection` (Ideogram)
- `finetune_id` + `finetune_strength` (BFL LoRA)
- `raw` (BFL Ultra — candid aesthetic) / `image_prompt` + `image_prompt_strength` (BFL remix)
- `thinking_level` (Google minimal/high)
- `previous_response_id` / `image_generation_call` ids (OpenAI multi-turn)
- `action: auto|generate|edit` (OpenAI Responses tool)
- `input_fidelity` (OpenAI — omit for `gpt-image-2`, always high)
- `postprocessing` (Reve — ordered ops)
- `aspect_ratio: auto` (model selects — Reve v2, xAI, OpenAI `size: auto`)

### 5.2 Multi-Turn / Conversational Image Generation
Iteratively refine images across conversation turns:
- **OpenAI Responses** — `previous_response_id` chains turns; or pass prior `image_generation_call` ids in `input`. `action: auto|generate|edit` controls behavior.
- **Google Interactions** — `previous_interaction_id` continues the conversation; model retains prior image context.
- **Multi-turn editing loop** — feed each output as the next input (xAI, most providers support this implicitly).

### 5.3 Streaming / Partial Images
Stream partial images during generation (OpenAI only): `partial_images` parameter (0–3); 0 = final only, >0 = stream up to N partial images. Each partial adds 100 output-image tokens. Events: `response.image_generation_call.partial_image` (index + b64).

### 5.4 Interleaved Text & Image Output
Produce stories/guides with text blocks and illustrations in one response. **Google Nano Banana Pro** (`gemini-3-pro-image`) — iterate `steps` → `model_output` → `content[]` handling each `text`/`image` block (convenience properties don't capture the full sequence).

### 5.5 Transparent Background Generation
Generate images with native transparent background from a prompt:
- **Ideogram** — `/v1/ideogram-v3/generate-transparent` (die-cut stickers, logos, POD); `upscale_factor: X1/X2/X4`; no `FLASH` rendering speed; `aspect_ratio`.
- **Alternatives** — use `remove-background` on a generated image (Ideogram, Recraft), or Reve postprocessing `remove_background`.

### 5.6 Vector Image Generation
Produce SVG output directly:
- **Recraft** — vector model variants (`recraftv4_1_vector`, `recraftv3_vector`, etc.) with vector styles (`Vector art`, `Line art`, `Icon`); `/v1/images/generations/vector` rejects raster.
- **BFL / OpenAI / Google / xAI / Ideogram / Reve** — raster only (use vectorize as a post-step, Recraft only).

### 5.7 Determinism & Reproducibility
- **`seed`** — BFL, Ideogram, Recraft (`random_seed`), Google Veo, OpenAI. Improves consistency; not guaranteed (Google Veo "slightly improves determinism").
- **Pinned model versions** — Reve `reve-create@20250915`, BFL model versions, for reproducibility.
- **Temperature** — not exposed for most image models; Reve `test_time_scaling` (1–15, cost scales linearly, >5 rarely helps, does not increase latency).

---

## Stage 6 — Image Editing & Transformation

### 6.1 Prompt-Based Image Editing
Edit an existing image by describing the desired change in natural language. The model understands the image content and applies only requested changes.

**Single-image edit:** OpenAI `/v1/images/edits` or Responses tool; Google Interactions (text+image); xAI `/v1/images/edits`; Reve `/v1/image/edit` (`edit_instruction`); Recraft `imageToImage` (with `strength`); Ideogram `/v1/edit` or `/v1/ideogram-v4/remix` (`image_weight`); BFL same endpoint + `input_image`.

**Multi-image edit (compositing):** Combine multiple source images:
- **xAI** — up to 3 source images (mix url/file_id kinds); `aspect_ratio` override (defaults to first input).
- **Ideogram `/v1/edit`** — up to 10 images (files or URLs).
- **BFL FLUX.2** — up to 8 reference images via `input_image_1..8`; 9MP combined budget on [pro].
- **Reve remix** — 1–6 reference images with `<img>N</img>` tags.
- **Google 3.1 Flash** — up to 14 object + 4 character + 3 style references (roles).
- **Recraft imageToImage** — source image + style refs.

### 6.2 Inpainting (Masked Region Regeneration)
Regenerate masked regions while keeping the rest intact.

| Provider | Endpoint | Mask semantics |
|----------|----------|----------------|
| OpenAI | `/v1/images/edits` with `mask` (alpha) or Responses `input_image_mask.file_id` | Alpha channel; prompt-based (mask is guidance, may not follow exactly) |
| BFL | `/v1/flux-pro-1.0-fill` | `mask` B/W (white=inpaint) or alpha in PNG/WebP; `guidance` 1.5–100 |
| Ideogram | `/v1/ideogram-v3/inpaint` | `mask` B/W (black=edit region); full V3 style surface |
| Recraft | `/v1/images/inpaint` | `mask`/`mask_url` grayscale (white=modify); V3/V3 Vector only |

**Mask requirements:** same dimensions as image; OpenAI requires alpha channel; BFL accepts B/W mask or embedded alpha; Recraft pure black/white pixels.

### 6.3 Outpainting / Border Extension
Extend an image beyond its original borders.

**Approaches:**
- **Per-side pixel margins** — Recraft `expand_left/right/top/bottom` (0–4096, mutually exclusive with `size`); BFL FLUX.1 Expand `top/bottom/left/right` (0–2048, directed border expansion maintaining context).
- **Target size / canvas** — Recraft `size` (source placed inside); BFL outpainting `width`/`height` canvas with `reference_offset_x/y` (None=center) and `auto_crop`.
- **Zoom out** — Recraft `zoom_out_percentage` (0–100, scales source down before outpainting).
- **Reframe (aspect-ratio extension)** — Ideogram `/v1/ideogram-v3/reframe` (square inputs reframed to target `ResolutionV3`); designed to preserve focal point.
- **Outpainting mode** — BFL `mode: high|fast` (high = highest fidelity slower; fast = significantly faster, good for landscapes/backgrounds/products).

### 6.4 Background Operations
- **Remove background** — Ideogram `/v1/remove-background` (transparent cutout, no prompt); Recraft `/v1/images/removeBackground` (raster or SVG input → SVG output for SVG); Reve postprocessing `remove_background`.
- **Replace background (auto-detect)** — Ideogram `/v1/ideogram-v3/replace-background` (prompt + style); Recraft `/v1/images/replaceBackground` (auto-detects subject, no mask).
- **Generate background (masked)** — Recraft `/v1/images/generateBackground` (mask specifies regions to fill, white=fill).

### 6.5 Object Removal / Erase
Remove masked objects and fill background coherently:
- **BFL Erase** — `/v1/flux-tools/erase-v1` (`image` + `mask`, white=remove; `dilate_pixels` 0–25 expands mask to cover edges; powered by FLUX.2 Klein 9B).
- **Recraft Erase Region** — `/v1/images/eraseRegion` (`image` + `mask`, white=erase; content-aware fill without prompt; cheapest operation).
- **DaVinci editor** — "Object remove/replace" within `/app/edit`.

### 6.6 Deblur
Sharpen a blurry image while preserving scene/composition/lighting. **BFL only** — `/v1/flux-tools/deblur-v1` (no prompt, no mask; fixed FLUX.2 Klein 9B KV BF16 blur-removal LoRA with fixed prompt; caller controls only input image + seed/format/safety).

### 6.7 Virtual Try-On (VTO)
Generate try-on results from a person image + garment reference(s). **BFL only** — `/v1/flux-tools/vto-v1` (`prompt`, `person` → `input_image`, `garment` → `input_image_2`; prompt steers attribute transfer; low-latency; built on FLUX.2 Klein).

### 6.8 Restyle / Relight
Transform look/mood/lighting without changing the subject:
- **DaVinci editor** — "Restyle Image" and "Relight Image" tools at `/app/edit` (source image + style/mood/lighting prompt).
- **Other providers** — achieved via prompt-based editing (xAI style transfer, BFL style/material transfer, Google scene restyling).

### 6.9 Remix / Variate / Explore
Generate variations of an existing image:
- **Reve remix** — `/v1/image/remix` (1–6 refs + prompt with `<img>N</img>` tags; `aspect_ratio` model-chosen).
- **Ideogram remix** — `/v1/ideogram-v4/remix` or `/v1/ideogram-v3/remix` (`image` + `text_prompt` + `image_weight`; V3 cropped to aspect ratio, full style surface).
- **Recraft variateImage** — `/v1/images/variateImage` (no prompt; visual content of source; `size` required; for reformatting to different aspect ratios).
- **Recraft explore** — `/v1/images/explore` (grid of diverse images from a prompt; V4/V4.1 family).
- **Recraft explore similar** — `/v1/images/explore/similar` (`source_image_id` from prior explore + `similarity` 1–5).
- **BFL Ultra image remix** — `image_prompt` + `image_prompt_strength` (0–1 blend between text and image prompt).

### 6.10 Layout-Aware Composition (Reve v2)
Decouple "what to draw and where" from rendering:
- **`create_layout`** — text/refs → `Layout` (regions with label/prompt/bbox/region_type).
- **`edit_layout`** — layout + text + typed `LayoutCommand`s (`op: add/shift/remove/place/keep/change`; `at`/`to` as `Bbox`/`Point` normalized; `image_index` selects ref; `change` uses `new_description`).
- **`render`** — layout → image (layout2image); echoes layout.
- **`image_to_layout`** — image → `Layout` (image2layout; reverse-engineers structure; the only explicit image understanding in Reve).
- **`edit`** (v2) — image + text → image + echoed layout.
- **`create`** (v2) — text/refs → image + echoed layout.

**Region types:** `coarse_detail` (high-level object), `medium_detail` (sub-part of coarse), `fine_detail` (sub-part of medium), `text` (embedded text), `hand`, `face`. Parent/child hierarchy.

---

## Stage 7 — Image Format & Structure Conversion

### 7.1 Vectorization (Raster → SVG)
Convert a raster image (PNG/JPG/WEBP) into a scalable SVG. **Recraft only** — `/v1/images/vectorize` (deterministic, no model parameter; `<10MB; <16MP; max dim <4096px; min dim ≥256px`).

### 7.2 Text Layer Extraction
Analyze an image to detect text regions, return each text block with position/content/font/styling, plus a text-erased base image. **Ideogram only** — `/v1/ideogram-v3/layerize-text` (`base_image_url` + `text_blocks[]` with `role`, `text`, `x/y/width/height/angle`, `color`, `font_name`, `font_alternatives`, `font_size`, `line_height`, `alignment`, `formatting`).

### 7.3 Layout Extraction (Reverse Engineering)
Derive a structured layout from an existing image:
- **Reve** — `/v2/image/image_to_layout` → `Layout` of typed `Region`s.
- **Ideogram** — `/v1/ideogram-v4/describe` → `V4JsonPrompt` (with optional `bbox` per element; `include_bbox` controls layout preservation).

---

## Stage 8 — Video Generation

### 8.1 Text-to-Video
Generate a video clip from a text prompt. **Asynchronous** on all providers.

| Provider | Endpoint | Models | Duration | Resolution | Native audio |
|----------|----------|--------|----------|-----------|--------------|
| OpenAI | `POST /v1/videos` | `sora-2`, `sora-2-pro` | 16/20s | 480p/720p (sora-2), 1080p (sora-2-pro) | Yes |
| Google | `models.generate_videos` (predictLongRunning) | Veo 3.1/3.1 Fast/3.1 Lite, Veo 3/3 Fast, Veo 2 | 4/6/8s (8 required for 1080p/4k/refs/ext) | 720p/1080p/4k | Yes (Veo 3.x always on; Veo 2 silent) |
| xAI | `POST /v1/videos/generations` | `grok-imagine-video`, `grok-imagine-video-1.5` | 1–15s | 480p/720p (both), 1080p (1.5 I2V only) | — (not specified) |

### 8.2 Image-to-Video (First Frame)
Animate a still image; the image becomes the starting frame.
- **OpenAI Sora** — `input_reference` (multipart file or JSON `{file_id}`/`{image_url}`); must match `size`.
- **Google Veo** — `image` (initial frame); combine with Nano Banana for two-step pipeline (generate image → animate).
- **xAI** — `image` (`{url}` or `{file_id}`); `aspect_ratio` defaults to input image ratio (specifying stretches).

### 8.3 Last-Frame Interpolation
Provide both first (`image`) and last (`lastFrame`) frames; model generates a video transitioning between them. **Google Veo** (3.1/3.1 Fast/3/2).

### 8.4 Reference-to-Video
Use reference images to guide content without locking the first frame:
- **xAI** — `reference_images[]` + `<IMAGE_1>`, `<IMAGE_2>`, `<IMAGE_3>` placeholders in prompt; `grok-imagine-video` only (1.5 does not support).
- **Google Veo 3.1/3.1 Fast** — `referenceImages[]` (up to 3, `VideoGenerationReferenceImage` with `reference_type: "asset"`); requires `durationSeconds: "8"`.

### 8.5 Character Assets (Reusable Non-Human Subjects)
**OpenAI Sora** — `/v1/videos/characters` (upload short MP4 + `name`); use `characters:[{id}]` (up to 2 per video) + mention name verbatim in prompt. Best with 2–4s clips, 16:9 or 9:16, 720p–1080p. Can combine with `input_reference`. **Not** supported in extensions. Human-likeness blocked by default (eligibility via sales).

### 8.6 Native Audio Generation
Current-generation video models generate synchronized audio (dialogue, SFX, ambient, music) in the same pass:
- **Always on** — OpenAI Sora 2 (dialogue, SFX, music, lip-sync), Google Veo 3.x, DaVinci-listed Kling 2.6, Wan 2.6, Seedance 2.0.
- **Silent only** — Google Veo 2.
- **Prompt audio cues** — describe sound effects, ambient noise, dialogue; model generates synchronized soundtrack (Google Veo prompt guide).

### 8.7 Video Generation Parameters (Union)

| Parameter | OpenAI | Google | xAI |
|-----------|--------|--------|-----|
| `prompt` | ✔ | ✔ | ✔ |
| `model` | ✔ | ✔ | ✔ |
| `size` / `aspectRatio` | `size` (e.g. `1280x720`) | `aspectRatio` (`16:9` default, `9:16`) | `aspect_ratio` (`16:9` default) |
| `seconds` / `durationSeconds` / `duration` | `seconds` (16/20) | `durationSeconds` (4/6/8) | `duration` (1–15) |
| `resolution` | (via `size` + model) | `resolution` (`720p`/`1080p`/`4k`) | `resolution` (`480p`/`720p`/`1080p`*) |
| `seed` | — | ✔ (Veo 3+, slightly improves determinism) | — |
| `personGeneration` | — | `allow_all`/`allow_adult`/`dont_allow` (varies by mode/model) | — |
| `input_reference` / `image` | ✔ (first frame) | ✔ (first frame) | ✔ (first frame) |
| `lastFrame` | — | ✔ (with `image`) | — |
| `referenceImages` | — | ✔ (up to 3, Veo 3.1 only) | `reference_images` + `<IMAGE_N>` tags |
| `characters` | ✔ (up to 2, non-human) | — | — |
| `storage_options` | — | — | ✔ (persist output, optional public URL) |

\* `1080p` only on `grok-imagine-video-1.5` for image-to-video.

### 8.8 Effective Video Prompting
Recommended elements: **Subject and context** (main focus + background), **Action** (what subject does), **Style** (surreal, vintage, cinematic), **Camera motion & composition** (POV, aerial, tracking, wide/close-up, low angle), **Ambiance** (color palettes & lighting), **Audio cues** (sound effects, ambient, dialogue).

### 8.9 Guardrails & Restrictions (OpenAI Sora)
- Only content suitable for audiences under 18 (bypass setting coming).
- Copyrighted characters and copyrighted music rejected.
- Real people (including public figures) cannot be generated.
- Human-likeness characters blocked by default (eligibility via sales).
- Input images with human faces rejected.

---

## Stage 9 — Video Editing, Extension & Interpolation

### 9.1 Video Editing
Modify an existing video with a text prompt while preserving the rest of the scene.

| Provider | Endpoint | Input | Inherited/capped properties |
|----------|----------|-------|------------------------------|
| OpenAI | `/v1/videos/edits` | `video` (id or uploaded) + `prompt`; one focused change; model inferred from source for id, set explicitly for upload (uploaded-video editing gated) | — |
| xAI | `/v1/videos/edits` | `video` (`{url}` or `{file_id}`) + `prompt` | `duration`/`aspect_ratio`/`resolution` **ignored** — inherited from input; resolution capped at 720p (1080p downsized); duration capped at 8.7s |
| Google | — (use extension or new generation) | — | — |

### 9.2 Video Extension
Continue an existing video from its last frame; result is a single combined video.

| Provider | Endpoint | Extension length | Max total | Constraints |
|----------|----------|------------------|-----------|-------------|
| OpenAI | `/v1/videos/extensions` | up to 20s per extension | up to 120s (6× max) | No characters or image references; source video + prompt only |
| Google Veo 3.1/3/3 Fast | `generate_videos` with `video` param | +7s per extension | up to 148s (20× max) | 720p only; must be a prior Veo generation |
| xAI | `/v1/videos/extensions` | `duration` = extension portion only (total = input + extension) | — | `grok-imagine-video` only |

### 9.3 Video-to-Video & Audio-to-Video (Multimodal)
- **DaVinci/Seedance 2.0** — `@Image`/`@Video`/`@Audio` tag reference system; modes: text-to-video, image-to-video, video-to-video, audio-to-video; seamless scene extensions; structured refinement (swap characters, insert/remove scenes); auto audio-visual sync.

### 9.4 Batch Video Generation
**OpenAI Batch API** — `POST /v1/batches` targets `/v1/videos` (JSON only, no multipart). Upload assets ahead of time; reference via `input_reference` object with `file_id` or `image_url`. Use `custom_id` to map results to internal shot IDs. Batch-generated videos available for download 24h after batch completes.

---

## Stage 10 — Postprocessing & Effects

### 10.1 Postprocessing Pipeline (Reve)
An ordered array of operations applied **after** generation on all four image-producing endpoints (Create, Edit, Remix, v2 `render`). Cost scales with image megapixels (except `fit_image`, free).

| Process | Parameters | Cost | Notes |
|---------|-----------|------|-------|
| `upscale` | `upscale_factor` ∈ {2,3,4} | variable (≥2 credits), ~$0.002/MP | 4× output is very large |
| `remove_background` | — | variable (≥2 credits) | Keeps central subject; transparent output; poor on images without a clear subject |
| `fit_image` | `max_dim` and/or `max_width` and/or `max_height` (px, 1–4096) | **free** | Scale-down preserving aspect ratio; smaller images not enlarged |
| `effect` | `effect_name` (+ optional `effect_parameters` `{filterId:{uniformId:value}}`) | 3 credits, ~$0.004 | Apply named preset; missing params use saved defaults |

### 10.2 Effects System (Reve)
Named visual filter presets (built-in system + project-saved). Listed via `GET /v1/image/effect?source=all|project|preset`. Each effect: `name`, `source` (saved/builtin), `description`, `category`. Configure presets in the web app, save with a name, reference from API.

### 10.3 Upscaling (Provider Comparison)

| Provider | Endpoint | Type | Preserves content? |
|----------|----------|------|-------------------|
| Recraft | `/v1/images/crispUpscale` | Crisp | Yes (interpolation-based sharpening); min dim ≥32px; $0.004 |
| Recraft | `/v1/images/creativeUpscale` | Creative | No (regenerates finer details and faces); min dim ≥256px; $0.25 |
| Ideogram | `/upscale` | Guided | `resemblance`/`detail` controls; optional `prompt`; `magic_prompt_option` |
| Reve | postprocessing `upscale` | Factor | 2/3/4×; variable cost |

### 10.4 Smart Cropping
- **Google Cloud Vision** — `CROP_HINTS` returns `cropHints[].boundingPoly` (suggested crop vertices).
- **Azure Computer Vision v4.0** — `SmartCrops` feature returns suggested crop regions.

---

## Stage 11 — Output Formatting & Delivery

### 11.1 Image Output Formats

| Format | Providers | Notes |
|--------|-----------|-------|
| **PNG** | OpenAI (default), BFL (default for tools/Kontext), Google, Recraft, Reve (default JSON), Ideogram | Lossless |
| **JPEG** | OpenAI, BFL (default for generation), Google, Recraft, Reve, Ideogram | Faster than PNG; prefer when latency matters |
| **WebP** | OpenAI, BFL, Recraft, Reve (via `Accept`) | Modern, efficient |
| **SVG** | Recraft (vector models) | Scalable vector |
| **Transparent** | OpenAI (`background: transparent`, unsupported on `gpt-image-2`), Ideogram (`generate-transparent`), Recraft (removeBackground) | Die-cut stickers, logos |

### 11.2 Response Delivery Modes

| Mode | Description | Providers |
|------|-------------|-----------|
| **Synchronous (image bytes/URL)** | Complete image returned after generation | OpenAI Images, Google Interactions, xAI image, all Recraft, all Reve, Ideogram sync, BFL (poll), Classical vision |
| **Streaming (partial images)** | Partial images arrive progressively during generation | OpenAI (`stream` + `partial_images`) |
| **Asynchronous (job id → poll)** | Request returns job id; caller polls | OpenAI Videos, Google Veo (operation), xAI Videos, BFL (all endpoints), Replicate, Ideogram async, Azure Custom Vision training |
| **Webhook delivery** | Platform POSTs result to registered webhook | OpenAI Videos, BFL, Ideogram (Ed25519-signed), Replicate |

### 11.3 Response Shape

**Image URL (ephemeral):** all providers return temporary URLs that expire (BFL, Ideogram, Recraft ~24h, OpenAI video 1h / batch 24h, xAI, Reve). Download promptly or persist via Files API.

**Base64 inline:** OpenAI (`b64_json`), xAI (`response_format: b64_json`), Recraft (`response_format: b64_json`), Reve (JSON default returns base64 PNG), BFL (base64 or URL).

**Raw image bytes:** Reve via `Accept: image/png|jpeg|webp` headers; returns raw bytes + metadata headers.

### 11.4 Video Output

| Asset | Provider | Notes |
|-------|----------|-------|
| **MP4 video** | OpenAI (`variant=video`), Google (video.uri), xAI (video.url) | Download within URL TTL |
| **Thumbnail** | OpenAI (`variant=thumbnail` → `thumbnail.webp`) | Lightweight preview |
| **Spritesheet** | OpenAI (`variant=spritesheet` → `spritesheet.jpg`) | Scrubber/catalog asset |
| **Persisted file** | xAI (`storage_options` → `file_output.file_id` + optional `public_url`) | Permanent storage with optional shareable URL |

### 11.5 Files API Output Persistence (xAI)
`storage_options` on any Imagine request:
- `filename` (required) — filename; public URL path extension derived from it.
- `expires_after` (optional, 3600–2592000s = 1h–30d) — stored file auto-delete; omit for permanent.
- `public_url` (bool or `{expires_after}`) — create permanent/shareable URL; URL can never outlive file.
- `file_output` response block: `file_id`, `filename`, `expires_at` (if expiry), `public_url` (if requested & succeeded), `public_url_expires_at`, `public_url_error`.
- Multiple outputs (`n>1`): each gets independent `file_id` and `public_url`.
- Limit: 1,000 active public URLs per team.

### 11.6 Response Headers & Metadata

**Reve headers:** `X-Reve-Content-Violation`, `X-Reve-Request-Id`, `X-Reve-Version`, `X-Reve-Credits-Used`, `X-Reve-Credits-Remaining`, `X-Reve-Error-Code`.

**OpenAI moderation error:** `error.type: image_generation_user_error`, `error.code: moderation_blocked`, `moderation_details: { moderation_stage: input|output|unknown, categories: [...] }`.

**xAI response object:** `response.url`, `response.image` (base64), `response.model`, `response.respect_moderation`, `response.file_output`, `response.public_url`.

---

## Stage 12 — Async Job Lifecycle, Storage & Management

### 12.1 Async Lifecycle Patterns

**BFL (all endpoints):** `POST` → `AsyncResponse {id, polling_url, cost, input_mp, output_mp}` or `AsyncWebhookResponse` (when `webhook_url` supplied). Poll `GET polling_url` (or `GET /v1/get_result?id=`) every ~0.5–1s. Status transitions: `Pending` → `Reasoning` → `Generating` → `Ready` (or `Request Moderated`/`Content Moderated`/`Error`/`Task not found`). On `Ready`, read `result.sample` (image URL). Download immediately (ephemeral).

**OpenAI Videos:** `POST /videos` → job `id` (`status: queued`). Poll `GET /videos/{id}` (every 10–20s with exponential backoff) or register webhook (`video.completed`/`video.failed`). States: `queued`/`in_progress`/`completed`/`failed`. `GET /videos/{id}/content?variant=video|thumbnail|spritesheet` → streams binary MP4 (URL valid 1h).

**Google Veo:** `POST models/{model}:predictLongRunning` → `{name:"operations/..."}`. Poll `GET /v1beta/{name}` until `done:true`. Download `video.uri` (API key, follow redirects).

**xAI Videos:** `POST /v1/videos/generations` → `{request_id}`. Poll `GET /v1/videos/{request_id}` every few seconds. Status: `pending`/`done`/`expired`/`failed`. SDKs abstract polling with configurable `timeout` (default 10 min) and `interval` (default 100ms).

**Ideogram async:** `POST /v1/ideogram-v4/async/generate?webhook_url=` (HTTPS required, rejects private/loopback) → `{generation_id}`. Webhook POSTs result (Ed25519-signed). Fallback: `GET /v1/generations/{generation_id}` (`status: pending/completed/failed`; `data` only when completed).

**Replicate:** `POST /v1/predictions {version, input}` → poll `GET /v1/predictions/{id}` or use webhooks. Status: `starting`/`processing`/`succeeded`/`failed`/`canceled`.

**Azure Custom Vision:** Training is long-running; Prediction is synchronous.

### 12.2 Webhook Verification & Idempotency

**Ideogram (Ed25519):** Fetch public keys `GET /v1/.well-known/jwks.json` (JWK `x` base64url, cacheable). Signed message = `{Generation-Id}\n{User-Id}\n{Timestamp}\n{sha256_hex(raw body)}` (UTF-8, joined with `\n`). Verify hex signature against each public key; use `X-Ideogram-Webhook-Key-Id` as hint. Handlers must be idempotent (key on `generation_id`); delivery not guaranteed (fall back to polling).

**BFL:** `webhook_url` + optional `webhook_secret` for signature verification.

### 12.3 Video Job Object Fields (OpenAI)
`id`, `object`, `created_at`, `status`, `model`, `progress` (0–100), `seconds`, `size`, `error.message`.

### 12.4 SDK Polling Conveniences
- **OpenAI** — `client.videos.createAndPoll({...})` / `videos.create_and_poll(...)` blocks until terminal.
- **xAI** — `client.video.generate()` / `extend()` abstract polling; configurable `timeout`/`interval`; raises `VideoGenerationError` on failure (with `code`/`message`).
- **Google** — `client.operations.get(operation)`; `client.files.download(file=video.video)` then `video.video.save("name.mp4")`.

### 12.5 Error Handling

**xAI video error codes:** `invalid_argument` (fix params), `permission_denied` (confirm team access), `failed_precondition` (change model/mode/resolution), `service_unavailable` (retry later), `internal_error` (retry; contact support with `request_id`). Auth/rate-limit errors returned synchronously before job creation.

**Reve error hierarchy:** `ReveAPIError` (base) → `ReveAuthenticationError` (401), `ReveBudgetExhaustedError` (402), `ReveRateLimitError` (429, has `.retry_after`), `ReveValidationError` (400), `ReveContentViolationError`.

**Reve error codes:** `PROMPT_TOO_LONG`, `CONTENT_POLICY_VIOLATION`, `INDEX_OUT_OF_BOUNDS` (remix `<img>N</img>` tag refers to non-existent index), `MISSING_REQUIRED_PARAMETER`.

**OpenAI moderation:** `moderation_blocked` with `moderation_details.categories` (harassment, self-harm, sexual, violence). `moderation_stage: input|output|unknown`.

### 12.6 Library & Asset Management
- **OpenAI Videos** — `GET /v1/videos?limit=20&after=video_123&order=asc` (paginate + sort); `DELETE /v1/videos/{video_id}`.
- **xAI Files** — `GET/DELETE /v1/files/{id}`, `POST /v1/files/{id}/public-url` (create/recreate), `POST /v1/files/{id}/public-url/revoke`.
- **BFL finetunes** — `GET /v1/my_finetunes`, `GET /v1/finetune_details?finetune_id=`, `POST /v1/delete_finetune` (server `api.us1.bfl.ai`).
- **Ideogram models** — `GET /models` (`scope: owned|shared`, `status` filter), `GET /models/{model_id}`.
- **Recraft** — `GET /v1/users/me` (credits, email, id, name).

### 12.7 Rate Limits & Concurrency

| Provider | Limit |
|----------|-------|
| BFL | 24 concurrent (most endpoints); 6 concurrent (`flux-kontext-max`); exponential backoff on 429 |
| Recraft | 100 images/minute, 5 requests/second (per user) |
| Ideogram | 10 in-flight (default); Enterprise: contact for throughput |
| Reve | rate-limited (429 with `retry_after`) |
| OpenAI | tokens-per-minute (TPM) for vision; per-render for video |

### 12.8 Billing & Credits

| Provider | Unit | Notes |
|----------|------|-------|
| BFL | credits (1 credit = $0.01) | FLUX.2 megapixel-based; FLUX.1 flat per-image; tools per-task |
| Recraft | API units (1,000 = $1) | Per image (raster/vector); per request (transformations) |
| Reve | credits (750 ≈ $1) | Per request (create 18, edit/remix 30, fast 5); postprocessing adds cost (fit_image free) |
| Ideogram | credits | Per image; character-reference pricing; copyright detection adds latency |
| OpenAI | per-image tokens | Output tokens driven by quality+size; partial images +100 each; vision input via patch/tile |
| Google | per output-image tokens (flat per size tier) + thinking tokens | 0.5K=747, 1K/2K=1120, 4K=2000; video per render |
| xAI | flat per-image (generation); per-second (video) | Editing bills input + output image |
| DaVinci | credits (unified across all models) | Tier-based monthly (Pro 5,000 / Ultimate 15,000 / Creator 40,000) |

---

# Part III — The Unified API Specification

This part defines a **provider-agnostic API** that encompasses all features. It is written as a specification for a hypothetical "super complete" image & video AI platform.

## API Surface Overview

```
Authentication & Files:
  POST /files                          — Upload file (image/video) → file_id/uri
  GET  /files/{id}                     — Get file metadata/state
  DELETE /files/{id}                    — Delete file
  POST /files/{id}/public-url           — Create/recreate public URL
  POST /files/{id}/public-url/revoke    — Revoke public URL

Style & Asset Management:
  GET  /styles                         — List curated styles / presets / codes
  POST /styles                          — Create custom style (from reference images)
  GET  /styles/{id}                     — Get custom style details
  DELETE /styles/{id}                   — Delete custom style
  POST /finetunes                       — Train custom LoRA (BFL-style)
  GET  /finetunes                       — List finetunes
  GET  /finetunes/{id}                  — Get finetune details
  DELETE /finetunes/{id}                — Delete finetune
  POST /datasets                        — Create dataset (Ideogram-style training)
  POST /datasets/{id}/upload_assets     — Upload training assets
  POST /models/train                    — Train custom model (Ideogram-style)
  GET  /models                          — List custom models
  GET  /models/{id}                     — Get custom model details
  POST /characters                      — Create reusable character asset (video)
  GET  /effects                         — List effects (Reve-style presets)

Image Understanding (Vision):
  POST /vision/analyze                  — Multi-feature analysis (classical: labels, objects, OCR, faces, landmarks, safe-search)
  POST /vision/describe                 — Image → structured prompt (Ideogram describe / Reve image_to_layout)
  POST /vision/detect                   — Open-vocabulary object detection (boxes + labels + scores)
  POST /vision/segment                  — Segmentation (semantic / instance / promptable masks)
  POST /vision/caption                  — Captioning + dense captions + caption-to-phrase grounding
  POST /vision/ocr                      — OCR (image + document, hierarchical)
  POST /vision/track                    — Object tracking in video (first-frame prompt)
  POST /vision/face                     — Face & pose detection (landmarks, head pose, emotion likelihood)

Image Generation:
  POST /images/generations              — Text-to-image (raster)
  POST /images/generations/vector       — Text-to-image (vector/SVG)
  POST /images/generations/transparent  — Transparent-background generation
  POST /images/generations/async        — Async generation (webhook)
  GET  /images/generations/{id}         — Poll async generation
  POST /images/magic-prompt             — Prompt enhancement → structured prompt
  POST /images/enhance-prompt           — Prompt enhancement → enriched text

Image Editing & Transformation:
  POST /images/edit                     — Prompt-based editing (single/multi-image)
  POST /images/remix                    — Remix (1–6 refs + prompt with <img>N</img> tags)
  POST /images/image-to-image           — Image-to-image variation (with strength)
  POST /images/inpaint                  — Inpainting (image + mask + prompt)
  POST /images/outpaint                 — Outpainting (per-side margins or target size)
  POST /images/expand                   — Directed border expansion (per-side pixels)
  POST /images/reframe                  — Aspect-ratio extension (preserve focal point)
  POST /images/remove-background        — Background removal (transparent cutout)
  POST /images/replace-background       — Background replacement (auto-detect subject)
  POST /images/generate-background      — Background generation (masked)
  POST /images/erase                    — Object removal (mask-driven, content-aware fill)
  POST /images/deblur                   — Deblur (sharpen, no prompt)
  POST /images/vto                      — Virtual try-on (person + garment)
  POST /images/variate                  — Variate/remix (no prompt, visual content of source)
  POST /images/explore                  — Explore (grid of diverse images from prompt)
  POST /images/explore/similar          — Explore similar (by source_image_id + similarity)

Layout-Aware Composition:
  POST /images/layout/create            — Text/refs → Layout
  POST /images/layout/edit              — Layout + text + commands → Layout
  POST /images/layout/render            — Layout → image
  POST /images/layout/image-to-layout   — Image → Layout (reverse)

Format & Structure Conversion:
  POST /images/vectorize                — Raster → SVG
  POST /images/layerize-text            — Text layer extraction (base + text_blocks)
  POST /images/upscale/crisp             — Upscale preserving content
  POST /images/upscale/creative          — Upscale regenerating detail
  POST /images/upscale/guided            — Upscale with resemblance/detail/prompt controls

Video Generation:
  POST /videos/generations              — Text-to-video / image-to-video / reference-to-video
  POST /videos/interpolate              — First+last frame interpolation
  GET  /videos/{id}                     — Poll video job status
  GET  /videos/{id}/content             — Download MP4/thumbnail/spritesheet

Video Editing & Extension:
  POST /videos/edits                    — Edit existing video (prompt)
  POST /videos/extensions               — Extend video from last frame
  POST /videos/batch                     — Batch video render queue

Management:
  GET  /videos                          — List videos (paginate/sort)
  DELETE /videos/{id}                    — Delete video
  GET  /users/me                         — Get user info / credit balance
  GET  /credits                         — Get credit balance
```

---

## Unified Data Structures

### Image Input (Unified)

```json
{
  "type": "url | base64 | file_id | blob_path | project_ref",
  "url": "https://...",
  "base64": "data:image/jpeg;base64,...",
  "file_id": "file_...",
  "blob_path": "bfl-hosted/path",
  "project_ref": "id:<uuid> | reference:@<name>",
  "mime_type": "image/png | image/jpeg | image/webp | image/gif | image/heic | image/heif"
}
```

### Mask (Unified)

```json
{
  "type": "grayscale | alpha | polygon | rle",
  "grayscale": "base64 or URL (white=edit, black=keep)",
  "alpha_embedded": true,
  "polygon": [[x, y], ...],
  "rle": {"size": [h, w], "counts": "..."},
  "coordinate_space": "pixel | normalized_0_1 | normalized_0_1000",
  "dilate_pixels": 10
}
```

### Bounding Box (Unified)

```json
{
  "format": "xyxy | xywh | polygon | yxyx",
  "coordinates": [x1, y1, x2, y2],
  "coordinate_space": "pixel | normalized_0_1 | normalized_0_1000",
  "label": "person",
  "score": 0.95,
  "confidence": 0.95
}
```

### Style Specification (Unified)

```json
{
  "style": "Photorealism | Illustration | Vector art | ...",
  "style_id": "uuid-of-custom-style",
  "style_type": "AUTO | GENERAL | REALISTIC | DESIGN | FICTION",
  "style_preset": "ART_DECO | WATERCOLOR | BAUHAUS | ...",
  "style_codes": ["8charhex1", "8charhex2"],
  "style_reference_images": ["url or base64"],
  "color_palette": {
    "preset_name": "EMBER | FRESH | JUNGLE | MAGIC | MELON | MOSAIC | PASTEL | ULTRAMARINE",
    "members": [{"color_hex": "#FF0000", "color_weight": 0.5}]
  },
  "style_description": {
    "aesthetics": "warm, cozy, nostalgic",
    "art_style": "illustration",
    "lighting": "soft natural window light",
    "medium": "photograph",
    "photo": "50mm lens, film stock"
  },
  "custom_model_uri": "model/<name>/version/<version>",
  "finetune_id": "lora-name or org-id/lora-name",
  "finetune_strength": 1.2
}
```

### Controls (Unified, Recraft-style)

```json
{
  "colors": [{"rgb": [255, 0, 0], "weight": 0.5}],
  "background_color": {"rgb": [0, 0, 0]},
  "artistic_level": 3,
  "no_text": false,
  "text_layout": [
    {"text": "Hello", "bbox": [[0.3, 0.45], [0.6, 0.45], [0.6, 0.55], [0.3, 0.55]]}
  ]
}
```

### Layout (Unified, Reve v2-style)

```json
{
  "regions": [
    {
      "label": "armchair",
      "prompt": "a cozy reading armchair",
      "bbox": {"x0": 0.1, "y0": 0.5, "x1": 0.5, "y1": 0.9},
      "image_index": 0,
      "parent": null,
      "region_type": "coarse_detail"
    }
  ],
  "prompt": "A cozy reading nook",
  "width": 3072,
  "height": 2560
}
```

### LayoutCommand (Unified)

```json
{
  "op": "add | shift | remove | place | keep | change",
  "label": "mug",
  "description": "a small round side table",
  "image_index": 1,
  "at": {"x": 0.5, "y": 0.5},
  "to": {"x0": 0.6, "y0": 0.5, "x1": 0.8, "y1": 0.7},
  "new_description": "an open notebook"
}
```

### Structured Prompt (V4JsonPrompt, Ideogram-style)

```json
{
  "high_level_description": "A cozy reading nook with a cat.",
  "style_description": {
    "aesthetics": "warm, nostalgic",
    "art_style": "illustration",
    "lighting": "soft window light",
    "medium": "digital art"
  },
  "compositional_deconstruction": {
    "background": "A dim room with a bright window casting warm light.",
    "elements": [
      {"type": "obj", "bbox": [0, 0, 1000, 1000], "desc": "ginger cat sitting on a chair"},
      {"type": "text", "bbox": [120, 200, 300, 800], "text": "Creative Typography", "desc": "bold sans-serif headline"}
    ]
  }
}
```

### Image Generation Request (Unified)

```json
{
  "model": "string | null",
  "text_prompt": "string | null",
  "json_prompt": "StructuredPrompt | Layout | null",
  "aspect_ratio": "16:9 | 1:1 | auto | ...",
  "resolution": "1024x1024 | 2K | 4K | null",
  "quality": "low | medium | high | auto",
  "rendering_speed": "FLASH | TURBO | DEFAULT | QUALITY",
  "test_time_scaling": 1,
  "n": 1,
  "seed": null,
  "negative_prompt": null,
  "output_format": "png | jpeg | webp | svg",
  "output_compression": null,
  "background": "opaque | automatic | transparent",
  "response_format": "url | b64_json",
  "stream": false,
  "partial_images": 0,
  "style": { "...see Style Specification..." },
  "controls": { "...see Controls..." },
  "reference_images": ["ImageInput..."],
  "character_reference_images": ["ImageInput..."],
  "character_reference_images_mask": ["ImageInput..."],
  "moderation": "auto | low",
  "safety_tolerance": 2,
  "enable_copyright_detection": false,
  "thinking_level": "minimal | high",
  "prompt_upsampling": "auto | on | off | disable_pup",
  "guidance": null,
  "steps": null,
  "raw": false,
  "postprocessing": [{"process": "upscale", "upscale_factor": 2}],
  "storage_options": {"filename": "out.png", "public_url": true},
  "webhook_url": null,
  "webhook_secret": null,
  "previous_interaction_id": null,
  "action": "auto | generate | edit",
  "input_fidelity": null,
  "aspect_ratio_v2_set": false
}
```

### Image Editing Request (Unified)

```json
{
  "model": "string",
  "prompt": "edit instruction",
  "image": "ImageInput",
  "images": ["ImageInput..."],
  "mask": "Mask",
  "strength": 0.5,
  "image_weight": 50,
  "expand_left": 0, "expand_right": 0, "expand_top": 0, "expand_bottom": 0,
  "zoom_out_percentage": 0,
  "mode": "high | fast",
  "auto_crop": false,
  "reference_offset_x": null, "reference_offset_y": null,
  "dilate_pixels": 10,
  "person": "ImageInput",
  "garment": "ImageInput",
  "...all generation params (style, controls, seed, output_format, etc.)..."
}
```

### Image Understanding / Vision Request (Unified)

```json
{
  "model": "mainline multimodal | classical vision | detection model",
  "image": "ImageInput",
  "images": ["ImageInput..."],
  "prompt": "analysis instruction (e.g., 'what's in this image?')",
  "detail": "low | high | original | auto",
  "media_resolution": "low | ...",
  "features": ["LABEL_DETECTION", "OBJECT_LOCALIZATION", "TEXT_DETECTION", "FACE_DETECTION", "SAFE_SEARCH_DETECTION", "LANDMARK_DETECTION", "LOGO_DETECTION", "WEB_DETECTION", "IMAGE_PROPERTIES", "CROP_HINTS", "PRODUCT_SEARCH", "Caption", "Tags", "Read", "Objects", "People", "SmartCrops", "denseCaptions"],
  "response_format": {"type": "text | image | json", "schema": "..."},
  "language_hints": ["en"],
  "max_results": 10,
  "include_bbox": true,
  "thinking_level": "minimal",
  "task_input": "Caption | Object Detection | OCR | OCR with Region | Dense Region Caption | Caption to Phrase Grounding",
  "text_input": "caption for grounding",
  "query": "comma-separated object names",
  "class_names": "dog,cat,person",
  "box_threshold": 0.3, "text_threshold": 0.3,
  "score_thr": 0.5, "nms_thr": 0.5,
  "points_per_side": 32, "pred_iou_thresh": 0.88, "stability_score_thresh": 0.95,
  "use_m2m": true,
  "return_json": false
}
```

### Video Generation Request (Unified)

```json
{
  "model": "sora-2 | sora-2-pro | veo-3.1 | grok-imagine-video | ...",
  "prompt": "text description",
  "size": "1280x720 | 1920x1080",
  "aspect_ratio": "16:9 | 9:16 | 1:1",
  "seconds": 8,
  "duration": 10,
  "resolution": "480p | 720p | 1080p | 4k",
  "seed": null,
  "input_reference": "ImageInput (first frame)",
  "image": "ImageInput (first frame)",
  "lastFrame": "ImageInput (final frame)",
  "referenceImages": [{"image": "ImageInput", "reference_type": "asset"}],
  "reference_images": ["ImageInput..."],
  "characters": [{"id": "char_123"}],
  "personGeneration": "allow_all | allow_adult | dont_allow",
  "storage_options": {"filename": "out.mp4", "public_url": true},
  "webhook_url": null
}
```

### Video Edit/Extension Request (Unified)

```json
{
  "model": "string",
  "prompt": "edit/extension instruction",
  "video": {"id": "video_..."} or {"url": "..."} or {"file_id": "..."},
  "duration": 5,
  "seconds": 8
}
```

### Vision Analysis Response (Unified)

```json
{
  "model": "string",
  "output_text": "A green car parked in front of a yellow building.",
  "output_image": null,
  "steps": [
    {"type": "model_output", "content": [{"type": "text", "text": "..."}]},
    {"type": "thought", "summary": [{"type": "text", "text": "..."}]},
    {"type": "google_search_call", "url_citation": {...}}
  ],
  "labels": [{"description": "car", "score": 0.95, "boundingPoly": {...}}],
  "objects": [{"name": "car", "score": 0.9, "boundingBox": {"x": 10, "y": 20, "w": 100, "h": 50}}],
  "boxes": [{"box_2d": [100, 200, 300, 400], "mask": [[x,y]...], "label": "person"}],
  "masks": [{"segmentation": {"size": [h,w], "counts": "..."}, "bbox": [x,y,w,h], "class_name": "person", "predicted_iou": 0.95, "stability_score": 0.92}],
  "captions": [{"text": "a man pointing at a screen", "confidence": 0.49, "boundingBox": {...}}],
  "ocr": {"blocks": [{"lines": [{"text": "...", "boundingPolygon": [...], "words": [{"text": "You", "confidence": 0.99}]}]}]},
  "faces": [{"boundingPoly": {...}, "landmarks": [{"type": "LEFT_EYE", "position": {"x":0,"y":0,"z":0}}], "joyLikelihood": "LIKELY"}],
  "safeSearch": {"adult": "VERY_UNLIKELY", "violence": "LIKELY"},
  "landmarks": [{"description": "Eiffel Tower", "score": 0.9, "locations": [{"latLng": {"latitude": 48.8, "longitude": 2.3}}]}],
  "logos": [{"description": "ACME", "score": 0.9}],
  "webDetection": {"webEntities": [...], "visuallySimilarImages": [...]},
  "dominantColors": [{"color": {"red": 255}, "pixelFraction": 0.3, "score": 0.9}],
  "cropHints": [{"boundingPoly": {...}}],
  "json_prompt": "V4JsonPrompt (from describe)",
  "layout": {"regions": [...]}
}
```

### Image Generation Response (Unified)

```json
{
  "created": "ISO-8601",
  "data": [
    {
      "url": "ephemeral URL",
      "b64_json": "base64 (when response_format=b64_json)",
      "prompt": "possibly revised prompt",
      "revised_prompt": "OpenAI revised prompt",
      "resolution": "2048x2048",
      "upscaled_resolution": "4096x4096",
      "is_image_safe": true,
      "content_violation": false,
      "respect_moderation": true,
      "seed": 12345,
      "style_type": "REALISTIC",
      "file_output": {"file_id": "...", "filename": "...", "public_url": "...", "expires_at": null}
    }
  ],
  "layout": "Layout (v2 echo)",
  "version": "model version used",
  "request_id": "...",
  "credits_used": 18,
  "credits_remaining": 7482
}
```

### Async Response (Unified)

```json
{
  "id": "task or generation id",
  "polling_url": "https://...",
  "status": "queued | pending | in_progress | processing | Reasoning | Generating | Ready | completed | done | failed | expired | Request Moderated | Content Moderated | Error",
  "progress": 0,
  "webhook_url": "https://...",
  "cost": 3.0,
  "input_mp": 1.0,
  "output_mp": 1.17,
  "model": "string",
  "seconds": 8,
  "size": "1280x720",
  "error": {"code": "invalid_argument", "message": "..."}
}
```

### Video Response (Unified)

```json
{
  "status": "done",
  "video": {
    "url": "https://vidgen.../video.mp4",
    "duration": 8,
    "respect_moderation": true
  },
  "model": "grok-imagine-video",
  "file_output": {"file_id": "...", "public_url": "..."},
  "thumbnail_url": "...",
  "spritesheet_url": "..."
}
```

---

## Capability Decision Matrix

The definitive guide to choosing the right capability for any need:

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Text → raster image | Image generation | `text_prompt`, `model`, `aspect_ratio`/`size`, `quality`, `n` |
| Text → vector image (SVG) | Vector generation (Recraft) | `model: recraftv4_1_vector`, vector `style` |
| Text → transparent-background image | Transparent generation (Ideogram) | `prompt`, `upscale_factor`, `aspect_ratio` |
| Structured/spatial control over generation | JSON prompt / Layout | `json_prompt` (Ideogram V4) or `Layout` (Reve v2) with `bbox` per element |
| Text → image with web-grounded info | Grounding search | FLUX.2 [max] grounding, Google `google_search` tool |
| Interleaved text + images in one response | Pro model interleaved output (Google) | `gemini-3-pro-image`, iterate `steps` |
| Stream partial images during generation | Streaming (OpenAI) | `stream: true`, `partial_images: 0–3` |
| Multi-turn conversational image editing | Responses/Interactions with `previous_*_id` | `previous_response_id` (OpenAI), `previous_interaction_id` (Google) |
| Edit image by prompt (single) | Image editing | `prompt`, `image`/`reference_image` |
| Edit with multiple reference images | Multi-image editing | `images[]` (up to 10 Ideogram / 8 BFL / 6 Reve / 3 xAI / 14 Google) |
| Edit a specific region with a mask | Inpainting | `image`, `mask` (B/W or alpha), `prompt` |
| Extend image borders | Outpainting / Expand / Reframe | `expand_*` (Recraft/BFL) or `size`/`top/right/bottom/left` (BFL) or `resolution` (Ideogram reframe) |
| Remove background → transparent cutout | Background removal | `image` only |
| Replace background with prompt | Background replace | `image`, `prompt` (auto-detect subject) |
| Generate background with mask | Background generate (Recraft) | `image`, `mask`, `prompt` |
| Remove object content-aware | Erase | `image`, `mask`, `dilate_pixels` (BFL) |
| Sharpen blurry image | Deblur (BFL) | `image` only |
| Virtual try-on | VTO (BFL) | `person`, `garment`, `prompt` |
| Variations of an image (no prompt) | Variate / Remix | `image`, `size` (Recraft variate) or `image` + `prompt` + `image_weight` (Ideogram remix) |
| Explore diverse images from a prompt | Explore (Recraft) | `prompt`, `model` (V4/V4.1) |
| Find similar images to a prior explore result | Explore similar (Recraft) | `source_image_id`, `similarity` 1–5 |
| Convert raster → SVG | Vectorize (Recraft) | `file`/`image_url` |
| Extract editable text layers | Layerize text (Ideogram) | `image`, optional `prompt` → `base_image_url` + `text_blocks[]` |
| Reverse-engineer layout from image | image_to_layout (Reve) / describe (Ideogram) | `image` → `Layout` or `V4JsonPrompt` |
| Plan composition then render | Layout-aware pipeline (Reve v2) | `create_layout` → `edit_layout` → `render` |
| Upscale preserving content | Crisp upscale | `file`/`image_url` (Recraft crisp, Reve postprocessing) |
| Upscale regenerating detail/faces | Creative upscale | `file`/`image_url` (Recraft creative, Ideogram guided) |
| Apply named visual filter | Effects (Reve) | postprocessing `effect` with `effect_name` |
| Scale image down preserving aspect | Fit image (Reve) | postprocessing `fit_image` with `max_dim`/`max_width`/`max_height` (free) |
| Create reusable custom style | Custom style creation | Recraft `/v1/styles` (up to 5 refs), Ideogram custom model (15–100 imgs), BFL LoRA |
| Keep character consistent | Character reference | Ideogram `character_reference_images` (+mask), Google character refs, OpenAI Sora `characters` |
| Control colors | Color palette / controls | Ideogram `color_palette`, Recraft `controls.colors`, BFL hex in prompt |
| Place individual words precisely | Text layout (Recraft V3) | `text_layout` array of `{text, bbox}` (4-point polygon, normalized 0–1) |
| Negative prompt (exclude elements) | Negative prompt | `negative_prompt` (Ideogram V3, Recraft V2/V3) |
| Deterministic generation | Seed | `seed`/`random_seed` |
| Text → video | Video generation | `prompt`, `model`, `size`/`aspect_ratio`, `seconds`/`duration` |
| Image → video (first frame) | Image-to-video | `image`/`input_reference` (must match size) |
| First + last frame → video | Interpolation (Google Veo) | `image` + `lastFrame` |
| Reference images guide video | Reference-to-video | `reference_images[]` (xAI up to 3 + `<IMAGE_N>` tags; Google Veo 3.1 up to 3) |
| Reusable character in video | Character assets (OpenAI Sora) | `POST /videos/characters` → `characters:[{id}]` + name in prompt |
| Extend a video | Video extension | `video.id`/`video.url`, `prompt`, `seconds`/`duration` |
| Edit an existing video | Video editing | `video` + `prompt` (one focused change) |
| Batch video rendering | Batch API (OpenAI) | `POST /batches` targeting `/videos`, JSON only, `custom_id` |
| Async results via webhook | Webhooks | `webhook_url` (+ `webhook_secret` for BFL; Ed25519 for Ideogram) |
| Persist generated asset | Files API output (xAI) | `storage_options` with `filename`, `expires_after`, `public_url` |
| Classify/tag an image | Classification (classical vision) | GCV `LABEL_DETECTION`, Azure `Tags`, Florence-2, Custom Vision |
| Detect objects by text | Open-vocabulary detection (Replicate) | Grounding DINO (`query`), YOLO-World (`class_names`) |
| Pixel-precise masks | Segmentation (Replicate) | SAM-2 (promptable), Grounded SAM (text→mask), Semantic-Segment-Anything |
| Read text in image/document | OCR | GCV `TEXT_DETECTION`/`DOCUMENT_TEXT_DETECTION`, Azure `Read`, Florence-2 `<OCR>` |
| Detect faces & landmarks | Face detection (GCV/MediaPipe) | `FACE_DETECTION` (~30 landmarks, head pose, emotion likelihood) |
| Track object across video frames | Video tracking (Replicate) | SAMURAI (zero-shot, COCO RLE per frame), YOLO-World (open-vocab), SAM-2-video |
| Recognize landmarks/logos | Specialized detectors (GCV) | `LANDMARK_DETECTION`, `LOGO_DETECTION` |
| Reverse image / web entity lookup | Web detection (GCV) | `WEB_DETECTION` |
| Content moderation / safe search | Safe search (GCV/Replicate) | `SAFE_SEARCH_DETECTION` (likelihood enum), NSFW ViT |
| Smart crop suggestions | Crop hints (GCV/Azure) | `CROP_HINTS`, `SmartCrops` |
| Dominant colors | Image properties (GCV) | `IMAGE_PROPERTIES` |
| Train your own classifier/detector | Custom Vision (Azure, deprecated) | Create project → add images/tags → train → publish → predict; export ONNX/TF/CoreML |
| Multiple features in one call | Multi-feature annotation (GCV/Azure) | `features=[...]` in one request |

---

## Coordinate System Reference

| Provider / model | Box encoding | Coordinate space |
|------------------|-------------|------------------|
| Replicate Grounding DINO | `[x1,y1,x2,y2]` (XYXY) | pixels |
| Replicate YOLO-World (JSON) | `x0,y0,x1,y1` (XYXY) | pixels |
| Replicate Florence-2 | `[x1,y1,x2,y2]` (XYXY); OCR quad = 8 coords (4 pts) | pixels |
| Replicate Semantic-Segment-Anything | `bbox:[x,y,w,h]` (XYWH) + COCO RLE | pixels |
| Replicate SAM-2 | mask PNG URLs | image-sized |
| Replicate SAMURAI | COCO RLE per frame | frame-sized |
| Google Cloud Vision (labels/faces) | `BoundingPoly.vertices:[{x,y}]` | pixels |
| Google Cloud Vision (objects) | `NormalizedVertex` polygon | normalized 0–1 |
| Google Gemini (detection/segmentation) | `[ymin,xmin,ymax,xmax]` + polygon `[x,y]` | normalized 0–1000 |
| Azure Computer Vision v4.0 (OCR) | `boundingPolygon:[{x,y}×4]` | pixels (relative to metadata.width/height) |
| Azure Computer Vision v4.0 (objects/dense captions/smart crops) | `boundingBox:{x,y,w,h}` (XYWH) | pixels |
| Ideogram V4JsonPrompt bbox | `[y_min,x_min,y_max,x_max]` | normalized 0–1000 |
| Recraft text_layout bbox | 4-point polygon `[[x,y]×4]` | normalized 0–1 (top-left origin; can exceed [0,1]) |
| Reve v2 Layout bbox | `{x0,y0,x1,y1}` (Bbox) | normalized 0–1 (top-left origin) |

---

## Sync vs Async Reference

| Provider | Pattern |
|----------|---------|
| OpenAI Images / Responses | Synchronous (with optional streaming) |
| OpenAI Videos | Asynchronous (poll or webhook) |
| Google Interactions (vision + image gen) | Synchronous |
| Google Veo (video) | Asynchronous (operation poll) |
| Google Cloud Vision `images:annotate` | Synchronous (batch up to N) |
| Google Cloud Vision `files:asyncBatchAnnotate` | Asynchronous (up to 2000 files, output to GCS) |
| Azure Computer Vision v4.0 | Synchronous |
| Azure Document Intelligence Read | Asynchronous (submit→poll→get) |
| Azure Custom Vision Training | Long-running; Prediction synchronous |
| xAI Images | Synchronous |
| xAI Videos | Asynchronous (start + poll; SDK abstracts) |
| BFL (all endpoints) | Asynchronous (poll or webhook) |
| Ideogram (most) | Synchronous |
| Ideogram async generate | Asynchronous (webhook + poll fallback) |
| Recraft (all) | Synchronous |
| Reve (all) | Synchronous |
| Replicate (all models) | Asynchronous (poll or webhook) |
| DaVinci | Consumer SaaS (no public API) |

---

## Open- vs Fixed-Vocabulary Reference

| | Open-vocabulary (text prompt) | Fixed vocabulary | Custom-trained |
|---|---|---|---|
| Replicate | Grounding DINO, YOLO-World, OWL-ViT, Grounded SAM, Florence-2 (via task prompts) | YOLOX, Mask2Former (ADE20k classes) | Self-host via Cog |
| Google Cloud Vision | — | `LABEL_DETECTION`, `OBJECT_LOCALIZATION`, `LOGO_DETECTION` | — |
| Azure Computer Vision | — | `Tags`, `Objects` | — |
| Azure Custom Vision | — | — | Classification + Object Detection (user-defined tags) |
| Google Gemini | ✔ (via prompt + structured output schema) | — | — |

---

## Summary of Unique Provider Capabilities

Each provider contributes unique features that no other provider offers:

| Provider | Unique capabilities not found elsewhere |
|----------|----------------------------------------|
| **OpenAI** | Conversational multi-turn image editing via Responses `image_generation` tool (`previous_response_id`, `image_generation_call` ids, `action: auto/generate/edit`), streaming partial images (`partial_images` 0–3, +100 tokens each), `revised_prompt` returned, `detail: original` for large/dense/computer-use images, DALL·E variations (legacy), WebRTC/SIP not applicable here but video `characters` assets (non-human, reusable), video `spritesheet`/`thumbnail` variants, Batch API for video, Org Verification gate |
| **Google Gemini** | Interactions API unifying understanding + generation in one call, thinking process with visible thought images (billed), `thinking_level` control, interleaved text+image output (Pro), up to 14 reference images with roles (object/character/style), Google Search grounding (`web_search` + `image_search`), video-to-image generation (3.1 Flash), SynthID watermarking, Veo first+last frame interpolation, Veo video extension up to 20× (148s), `personGeneration` controls, `media_resolution` |
| **xAI Grok Imagine** | Files API bidirectional integration (`file_id` input substitution + `storage_options` output persistence with permanent public URLs and independent expiry), `<IMAGE_N>` placeholder tagging in reference-to-video prompts, video extension where `duration` = extension portion only (not total), video editing inherits duration/aspect/resolution (capped 720p/8.7s), `grok-imagine-video-1.5` 1080p I2V-only, two video models with distinct mode support |
| **Black Forest Labs (FLUX)** | Megapixel-based pricing (FLUX.2), FLUX.2 [max] grounding search, hex color control in prompts, JSON-structured prompting, FLUX.2 [flex] typography/text-rendering specialization, up to 8 reference images (9MP budget), directed border expansion (`top/bottom/left/right` 0–2048), deblur (fixed LoRA, no prompt/mask), virtual try-on (VTO), `safety_tolerance` 0–6, `finetune_strength` 0–2, `raw` mode (candid aesthetic), `image_prompt`+`image_prompt_strength` remix (Ultra), FLUX.1 Kontext context-aware editing, `disable_pup` / `prompt_upsampling` toggle |
| **Ideogram** | Crystal-clear text rendering (differentiator), `V4JsonPrompt` structured prompt with `compositional_deconstruction` (elements discriminated `obj`/`text` with normalized 0–1000 bbox), `describe` round-trip (image → V4JsonPrompt → image), `magic_prompt` (AUTO/ON/OFF) + dedicated `/magic-prompt` endpoint, ~60 `style_preset` named artistic presets, `style_codes` (8-char hex, shareable), `color_palette` (preset or hex+weight members), `character_reference_images` + `_mask`, `layerize-text` (editable text layer extraction with font/color/geometry), `reframe` (aspect-ratio extension preserving focal point), `generate-transparent` (native transparent background with upscale), `enable_copyright_detection` (Hive likeness + logo checks), custom model training (15–100 images, `custom_model_uri`), Ed25519-signed webhooks with JWKS verification, model registry (`scope: owned/shared`), open weights on GitHub/HuggingFace |
| **Recraft** | Raster **and** vector (SVG) generation model variants, `style_id` custom style creation from up to 5 reference images (`POST /v1/styles`), `text_layout` per-word placement as 4-point polygons (normalized 0–1, V3/V3 Vector only, limited character set), `controls` (`colors`, `background_color`, `artistic_level` 0–5 V3, `no_text` V3), `imageToImage` with `strength` [0,1], `variateImage` (no prompt, visual content), `explore` grid + `explore/similar` (similarity 1–5), `vectorize` (raster→SVG), `eraseRegion` (content-aware fill, cheapest), `crispUpscale` (preserve) vs `creativeUpscale` (regenerate detail/faces), `generateBackground` (masked), OpenAI Python library compatibility (`client.images.generate`), API units billing (1,000 = $1), synchronous only (no polling/webhooks) |
| **Reve** | `Layout`-aware composition (`regions` with `label`/`prompt`/`bbox`/`region_type`/`parent` hierarchy; `coarse_detail`/`medium_detail`/`fine_detail`/`text`/`hand`/`face`), typed `LayoutCommand`s (`op: add/shift/remove/place/keep/change`), `image_to_layout` reverse engineering, `render` (layout→image), `<img>N</img>` tag prompt referencing, `test_time_scaling` (1–15, cost scales linearly, no latency increase), `postprocessing` ordered pipeline (`upscale`/`remove_background`/`fit_image` free/`effect`), Effects system (built-in + project-saved presets with `effect_parameters` overrides), `image_weight` control on remix, auto-enhanced prompts (silent), `Accept` header response format negotiation (raw image bytes + metadata headers), project references (`id:<uuid>` / `reference:@<name>`), SKILL.md for agent integration, content violation surfaced on every response |
| **DaVinci.ai** | Model-aggregator platform (50+ models, 15+ image, 14+ video in one subscription), unified billing via credits across all models, no tool-switching, mobile parity (iOS/Android), always-current models (new models added at launch), privacy/ownership posture (outputs private, never used for training, user owns outputs, commercial use by default, no attribution), templates tab for model-specific quick starts, Seedance 2.0 multimodal director (`@Image`/`@Video`/`@Audio` tag system; text/image/video/audio-to-video modes; structured refinement swap characters/insert-remove scenes), Spotlight/Reshoot tool, channels/auto-resize for social/ads/web |
| **Classical Vision (Replicate / GCV / Azure / Custom Vision)** | Open-vocabulary detection (Grounding DINO, YOLO-World, OWL-ViT), promptable segmentation (SAM-2), grounded segmentation (text→mask), video object tracking (SAMURAI, SAM-2-video, YOLO-World video), multi-feature batch annotation in one call (GCV `images:annotate`, Azure `features`), specialized detectors (landmarks, logos, web entities/reverse image, product search, dominant colors, crop hints), hierarchical document OCR (pages→blocks→paragraphs→words→symbols with confidence), face landmarks (~30+ types with 3D positions) + head pose + emotion likelihood enum, Azure Custom Vision train-your-own classifier/detector with ONNX/TF/CoreML/Docker export, Replicate uniform prediction lifecycle across open-source models |

---

## Language & Text-Rendering Notes

- **Text rendering in images** — Ideogram (crystal-clear, differentiator), BFL FLUX.2 [flex] (typography/text specialization), OpenAI GPT Image (improved but struggles with precise placement), Google Nano Banana 2 (reliable text rendering), Recraft V3 `text_layout` (per-word 4-point polygon placement, limited character set).
- **Non-Latin text** — OpenAI vision reduced performance for Japanese/Korean; small text should be enlarged (`original` detail helps).
- **Image captioning language** — Azure Caption/DenseCaptions are **English-only** and **region-restricted** (specific Azure datacenters); `gender-neutral-caption=true` option replaces gendered terms with "person".

---

*This specification aggregates the complete capabilities of OpenAI, Google Gemini, xAI Grok Imagine, Black Forest Labs (FLUX), Ideogram, Recraft, Reve, DaVinci.ai, and the classical vision APIs (Replicate, Google Cloud Vision, Azure Computer Vision, Azure Custom Vision) as documented in their respective platform study files. It is intended as a reference for building or evaluating a comprehensive image & video AI platform.*