# Classical AI Vision APIs Analysis — Image Classification, Detection, Segmentation, Captioning, OCR & Tracking

> **Platforms studied:**
> - **Replicate** — Object detection & segmentation model collection (`https://replicate.com/collections/ai-detect-objects`)
> - **Google Cloud Vision API** (`https://docs.cloud.google.com/vision/docs`)
> - **Azure AI Vision / Computer Vision** (`https://learn.microsoft.com/en-us/azure/ai-services/computer-vision/`)
> - **Azure Custom Vision** (`https://learn.microsoft.com/en-us/azure/ai-services/custom-vision-service/`)
>
> **Scope:** "Classical" (non-generative) computer-vision capabilities — recognizing and structuring *existing* image content. This contrasts with image *generation* models (see `google-api.md`, `blackforest-api.md`, etc.). Generative multimodal vision (Gemini, GPT-4o) is out of scope here.
>
> **Description:** This document maps the classical vision capability surface across four providers. Replicate offers a marketplace of task-specific open models invoked through a uniform prediction API; Google Cloud Vision and Azure Computer Vision expose hosted REST APIs that run many fixed feature detectors in a single call; Azure Custom Vision lets you train your own classifier/detector on labeled images and deploy it as a prediction endpoint. For each capability, we extract the core concepts and analyze the API functions and parameters.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Capability Matrix](#2-capability-matrix)
3. [Image Classification & Tagging](#3-image-classification--tagging)
4. [Object Detection](#4-object-detection)
5. [Semantic & Instance Segmentation](#5-semantic--instance-segmentation)
6. [Image Captioning & Dense Captioning](#6-image-captioning--dense-captioning)
7. [Optical Character Recognition (OCR)](#7-optical-character-recognition-ocr)
8. [Face & Pose Detection](#8-face--pose-detection)
9. [Object Tracking in Video](#9-object-tracking-in-video)
10. [Landmark, Logo, Web & Safe-Search Detection](#10-landmark-logo-web--safe-search-detection)
11. [Azure Custom Vision — Train-Your-Own Models](#11-azure-custom-vision--train-your-own-models)
12. [Cross-Platform Comparison & Concepts](#12-cross-platform-comparison--concepts)

---

## 1. Platform Overview & Core Concepts

### Architectural models

| Provider | Surface | Model | What it is |
|---|---|---|---|
| **Replicate** | `POST /v1/predictions` (+ SDK `replicate.run`) | Per-model, GPU-hosted | A registry of open-source models packaged with Cog. Each model has its own input/output schema; the platform provides a uniform prediction lifecycle (create → poll → output). You pick a model per task. |
| **Google Cloud Vision** | `POST /v1/images:annotate` (+ async `files:asyncBatchAnnotate`) | Hosted, multi-feature | One endpoint runs a batch of `AnnotateImageRequest`s, each declaring a list of `features` (enum) to run on one image. A single call can return labels + faces + OCR + objects simultaneously. |
| **Azure Computer Vision** | `POST /computervision/imageanalysis:analyze` (v4.0) | Hosted, multi-feature | One endpoint with a `features` query param selecting visual features (`Caption`, `Tags`, `Read`, `Objects`, `People`, `SmartCrops`, `denseCaptions`). Synchronous; returns one result block per feature. |
| **Azure Custom Vision** | Training API + Prediction API | Train-your-own | You create a project, upload/tag images, train an iteration, publish it to a named prediction endpoint, then call that endpoint at runtime. Two project types: classification and object detection. |

### Core concepts shared across platforms

- **Label / Tag / Class** — A textual category assigned to an image (classification) or to a region (detection/segmentation). Fixed vocabulary in Google/Azure hosted APIs; *open-vocabulary* (text-prompted) in Replicate models like Grounding DINO and YOLO-World.
- **Bounding box** — A rectangle locating an object. Two encodings appear:
  - **XYXY** — `[x1, y1, x2, y2]` (top-left + bottom-right corners). Used by Replicate models (Grounding DINO, Florence-2, YOLO-World JSON).
  - **XYWH** — `{x, y, w, h}` (top-left + width/height). Used by Azure (`boundingBox`), Google `NormalizedVertex`-derived boxes, and Semantic-Segment-Anything mask metadata.
- **Bounding polygon** — A list of vertices (often 4). Google uses `BoundingPoly { vertices: [{x,y}] }` for faces/text; Azure v4.0 uses `boundingPolygon: [{x,y}×4]` for OCR lines/words; Florence-2 uses `quad_boxes` (8 coords = 4 points) for OCR-with-region.
- **Normalized vs pixel coordinates** — Google object localization returns `NormalizedVertex` (floats 0–1); most other surfaces return pixel coordinates relative to the image dimensions in `metadata`.
- **Confidence / score** — A float 0.0–1.0 per detection. Caller chooses thresholds; platforms do not impose a single cutoff. Google also uses a `Likelihood` enum (VERY_UNLIKELY → VERY_LIKELY) for face attributes and safe-search.
- **Segmentation mask** — Per-pixel object extent. Encoded as PNG (Replicate SAM-2 returns mask image URLs), COCO RLE (`{size:[h,w], counts:"..."}` — used by Semantic-Segment-Anything and SAMURAI), or a class label per pixel (semantic segmentation).
- **Open-vocabulary / zero-shot detection** — Detect arbitrary objects described by free-form text at inference time, with no retraining. Grounding DINO, YOLO-World, OWL-ViT, and Grounded SAM offer this; Google/Azure hosted object detection use a fixed label set.
- **Region caption / dense caption** — A short text description *localized to a region* (a bounding box plus a caption). Azure `denseCaptions` and Florence-2 `<DENSE_REGION_CAPTION>` both produce this; it bridges detection and captioning.

### End-to-end flows

```
Hosted multi-feature (Google / Azure Computer Vision)
   image (URL | base64 | GCS URI) ──▶ single POST with features=[...] ──▶ one response block per feature

Replicate per-model
   image (URL | local file | data URI) ──▶ POST /v1/predictions {version, input} ──▶ poll /v1/predictions/{id}
                                              └─ (or webhook) ──▶ output: file URL(s) | JSON

Custom Vision (train-then-predict)
   create project ──▶ add images + tags (+regions for OD) ──▶ train iteration ──▶ publish ──▶
   POST prediction endpoint {image} ──▶ predicted tags / bounding boxes with probabilities
```

---

## 2. Capability Matrix

| Capability | Replicate (model) | Google Cloud Vision | Azure Computer Vision (v4.0) | Azure Custom Vision |
|---|---|---|---|---|
| Image classification / tagging | Florence-2 (`<CAPTION>`), YOLO-World (cls), NSFW ViT | `LABEL_DETECTION` | `Tags` | Classification project |
| Open-vocabulary object detection (text→boxes) | `adirik/grounding-dino`, `zsxkib/yolo-world`, `adirik/owlvit-base-patch32` | — | — | — |
| Fixed-vocabulary object detection | YOLOX, YOLO-World | `OBJECT_LOCALIZATION` | `Objects` | Object-detection project |
| Semantic segmentation (per-pixel class) | `cjwbw/semantic-segment-anything`, Mask2Former | — | — | — |
| Instance / promptable segmentation (masks) | `meta/sam-2`, `schananas/grounded_sam`, `ahmdyassr/mask-clothing`, `hadilq/hair-segment`, `swook/inspyrenet` | — | — | — |
| Image captioning | Florence-2 (`<CAPTION>`, `<DETAILED_CAPTION>`) | — | `Caption` | — |
| Dense / region captioning | Florence-2 (`<DENSE_REGION_CAPTION>`) | — | `denseCaptions` | — |
| Caption-to-phrase grounding | Florence-2 (`<CAPTION_TO_PHRASE_GROUNDING>`), Grounded SAM | — | — | — |
| OCR (text in images) | Florence-2 (`<OCR>`, `<OCR_WITH_REGION>`) | `TEXT_DETECTION`, `DOCUMENT_TEXT_DETECTION` | `Read` | — |
| Face detection | `chigozienri/mediapipe-face` | `FACE_DETECTION` | (v3.2 only; removed in v4.0) | — |
| Pose / facial landmarks | MediaPipe Face | `FACE_DETECTION` (landmarks + head pose) | — | — |
| Object tracking in video | `zsxkib/yolo-world`, `zsxkib/samurai`, `meta/sam-2-video` | — | — | — |
| Landmark recognition | — | `LANDMARK_DETECTION` | (v3.2 Landmarks) | — |
| Logo recognition | — | `LOGO_DETECTION` | (v3.2 Brands) | — |
| Web entity / reverse image | — | `WEB_DETECTION` | — | — |
| Safe-search / content moderation | `falcons-ai/nsfw_image_detection` | `SAFE_SEARCH_DETECTION` | (v3.2 Adult) | — |
| Smart cropping | — | `CROP_HINTS` | `SmartCrops` | — |
| Image properties (colors) | — | `IMAGE_PROPERTIES` | (v3.2 Color) | — |
| Product search | — | `PRODUCT_SEARCH` | (v3.2 Product) | — |
| Custom-trained models | Self-host via Cog | — | — | Classification + Object Detection |

---

## 3. Image Classification & Tagging

### Concepts
- **Tagging** assigns a flat list of `{name, confidence}` pairs to an image — no hierarchy, no taxonomy (Azure `Tags` explicitly "not a taxonomy"). Distinct from **classification** (which assigns one of N trained classes) and from **object detection** (which adds boxes).
- **Open-vocabulary** tagging/classification accepts arbitrary text labels at inference (Replicate). **Fixed-vocabulary** tagging uses a provider-trained label set (Google `LABEL_DETECTION`, Azure `Tags`).
- Florence-2 unifies tagging/detection/captioning behind task-prompt tokens; "classification" is expressed as captioning or region captioning rather than a discrete classifier head.

### Replicate — `lucataco/florence-2-large`
Task-prompt-driven; `task_input` selects the operation.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `image` | string (URI) | yes | Input image |
| `task_input` | string | no | Task selector (e.g. `"Caption"`, `"Object Detection"`, `"OCR"`) |
| `text_input` | string | no | Auxiliary text for tasks like caption-to-phrase grounding |

Output: `{ img: <uri>, text: "<python-dict-string>" }`. For `<CAPTION>` the text is `{'<CAPTION>': 'A green car parked in front of a yellow building.'}`.

### Google Cloud Vision — `LABEL_DETECTION`
One of the `Feature.Type` enum values passed in `features[]`. Returns `labelAnnotations[]` of `EntityAnnotation` objects, each with `description` (label), `score`, `topicality`, and `boundingPoly` (when a label localizes to a region).

### Azure Computer Vision (v4.0) — `Tags`
Selected via `features=Tags`. Response section `tagsResult.values[]`:

```jsonc
"tagsResult": {
  "values": [
    { "name": "grass",      "confidence": 0.9960 },
    { "name": "outdoor",     "confidence": 0.9956 },
    { "name": "building",    "confidence": 0.9893 }
  ]
}
```
No fixed threshold; caller decides cutoff. Ambiguous tags may include contextual hint metadata.

### Azure Custom Vision — Classification project
A trainable classifier. Each iteration is trained on labeled images; the published endpoint returns predicted tags with probabilities for a new image. Tags are user-defined at training time (not a provider vocabulary).

---

## 4. Object Detection

### Concepts
- **Bounding-box detection** returns `{box, label, score}` per object. Open-vocabulary detectors take a text `query`/`class_names` list at inference; closed detectors return a fixed class set.
- **Two thresholds** in Grounding DINO: `box_threshold` (keep boxes above this) and `text_threshold` (keep label words above this) — independent knobs for recall vs label precision.
- **NMS** (non-maximum suppression) deduplicates overlapping boxes; YOLO-World exposes `nms_thr`.
- **Open-vocabulary detection** lets the same model find "dog", "the vision superhero", or "vr goggles" without retraining. Hosted APIs (Google `OBJECT_LOCALIZATION`, Azure `Objects`) use a fixed COCO-style label set.

### Replicate — `adirik/grounding-dino` (open-vocabulary, image)

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `image` | string (URI) | optional | — | Input image |
| `query` | string | optional | — | Comma-separated object names/referring expressions to detect |
| `box_threshold` | number | optional | — | Min box-confidence to keep |
| `text_threshold` | number | optional | — | Min text-similarity to keep a predicted label |
| `show_visualisation` | boolean | optional | — | Draw boxes on `result_image` |

Output:
```jsonc
{
  "detections": [
    { "bbox": [19, 204, 408, 563], "label": "pink mug", "confidence": 0.8077 }   // XYXY
  ],
  "result_image": "https://.../result.png"                                        // optional annotated image
}
```

### Replicate — `zsxkib/yolo-world` (open-vocabulary, image **and video**, real-time)

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `input_media` | string (URI) | yes | — | Path to input image or video |
| `class_names` | string | optional | — | Comma-separated classes to detect |
| `max_num_boxes` | integer | optional | — | Cap on returned boxes |
| `score_thr` | number | optional | — | Score threshold for display |
| `nms_thr` | number | optional | — | NMS IoU threshold |
| `return_json` | boolean | optional | false | Return JSON detections instead of just the annotated media |

Two output modes:
- `return_json=false` → `{ "media_path": "<output.png|.mp4>" }`
- `return_json=true`  → `{ "json_str": "{\"Det-0\": {\"x0\":332.8, \"y0\":458.4, \"x1\":471.6, \"y1\":565.4, \"score\":0.7637, \"cls\":\"tongue\"}, ...}" }` (per-detection `Det-N` keyed JSON with XYXY coords, score, class)

### Replicate — Florence-2 `<OD>`
`task_input="Object Detection"` → `{'<OD>': {'bboxes': [[x1,y1,x2,y2],...], 'labels': ['car','door',...]}}`. Fixed COCO vocabulary from the Florence-2 pretraining.

### Google Cloud Vision — `OBJECT_LOCALIZATION`
Returns `localizedObjectAnnotations[]`; each has `name` (label), `score`, and `boundingPoly` using **`NormalizedVertex`** (float 0–1) rather than pixel coords. Fixed label set.

### Azure Computer Vision (v4.0) — `Objects`
`features=Objects` → result block with detected objects, each `{name, confidence, boundingBox:{x,y,w,h}}` in pixels. Supports multiple instances of the same tag.

### Azure Custom Vision — Object Detection project
Train with images where each labeled object has a **region** (bounding box). Published prediction endpoint returns each detected tag with a bounding box and probability.

---

## 5. Semantic & Instance Segmentation

### Concepts
- **Semantic segmentation** assigns a class label to *every pixel* (one mask per class across the image). Used for scene labeling / training-data generation.
- **Instance segmentation** produces one mask per object instance. SAM-family models are **promptable**: you give points or boxes to indicate which object to segment.
- **Grounded segmentation** = open-vocabulary detection (Grounding DINO) + mask generation (SAM) chained so a *text prompt* yields a mask. Grounded SAM does this in one call.
- **Mask encodings:**
  - **PNG image** — SAM-2 returns `combined_mask` and `individual_masks` as image URLs.
  - **COCO RLE** — `{size:[h,w], counts:"<encoded>"}`; used by Semantic-Segment-Anything and SAMURAI for compact per-mask JSON.
  - **Quality scores** — `predicted_iou` (model's own mask-quality estimate) and `stability_score` (mask stability under threshold perturbation) appear on SAM-derived outputs.

### Replicate — `meta/sam-2` (promptable, image; video variant is `meta/sam-2-video`)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `image` | uri | — | Input image |
| `use_m2m` | boolean | `true` | Use M2M (mask-to-mask) refinement |
| `points_per_side` | integer | `32` | Grid density of auto-generated prompt points |
| `pred_iou_thresh` | number | `0.88` | Predicted-IoU threshold for keeping masks |
| `stability_score_thresh` | number | `0.95` | Stability-score threshold for keeping masks |

Output: `{ "combined_mask": <uri>, "individual_masks": [<uri>, ...] }`.

### Replicate — `schananas/grounded_sam` (text-prompted masks)
Combines Grounding DINO + SAM. Inputs: image, text prompt(s) describing objects. Allows **multiple masks combined into one** and a **negative mask subtraction** for fine-grained control. Designed for inpainting pipelines (e.g. clothing segmentation for virtual try-on). Output: mask image(s).

### Replicate — `cjwbw/semantic-segment-anything` (semantic labels per SAM mask)

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `image` | file | yes | — | Input image |
| `output_json` | boolean | optional | `true` | Return raw JSON output |

Output wrapper: `{ "img_out": <uri>, "json_out": <uri> }`. The JSON is an array of mask objects:
```jsonc
{
  "segmentation":    { /* COCO RLE */ },
  "bbox":            [x, y, w, h],     // XYWH
  "area":            12345,            // pixels
  "predicted_iou":   0.95,
  "stability_score": 0.92,
  "crop_box":        [x, y, w, h],
  "point_coords":    [[x, y]],
  "class_name":      "person",         // chosen category
  "class_proposals": ["person", "man", ...]   // top-k candidates
}
```
Engine: close-set semantic seg (OneFormer) + open-vocab classification + CLIP-based decision/CLIPSeg, with BLIP captioning to generate candidate categories.

### Other Replicate segmentation models
- `ahmdyassr/mask-clothing` — clothing/face segmentation with erosion/dilation.
- `hadilq/hair-segment` — hair segmentation.
- `swook/inspyrenet` — foreground salient-object matting (InSPyReNet).
- `hassamdevsy/mask2former` — Mask2Former on ADE20k (semantic).
- `idea-research/ram-grounded-sam` — strong image tagging + SAM.

### Google / Azure hosted
Neither offers pixel-level segmentation as a hosted feature. Segmentation is only available via Replicate-style open models or Custom Vision (which is box-level, not mask-level).

---

## 6. Image Captioning & Dense Captioning

### Concepts
- **Caption** — one sentence describing the whole image (no location).
- **Dense captions** — many short captions, each *localized* to a region (box + caption). Bridges captioning and detection.
- **Caption-to-phrase grounding** — given a caption, locate each noun phrase in the image (boxes per phrase). Florence-2 and Grounded SAM offer this.
- Azure Caption/DenseCaptions are **Florence-based**, **English-only**, and **region-restricted** (specific Azure datacenters). An optional `gender-neutral-caption=true` replaces gendered terms with "person".

### Replicate — Florence-2 tasks

| `task_input` | Output shape |
|---|---|
| `"Caption"` | `{'<CAPTION>': 'A green car parked in front of a yellow building.'}` |
| `"Detailed Caption"` | `{'<DETAILED_CAPTION>': '...'}` |
| `"More Detailed Caption"` | `{'<MORE_DETAILED_CAPTION>': '...'}` |
| `"Dense Region Caption"` | `{'<DENSE_REGION_CAPTION>': {'bboxes': [[x1,y1,x2,y2],...], 'labels': ['turquoise Volkswagen Beetle','wheel',...]}}` |
| `"Caption to Phrase Grounding"` (with `text_input=<caption>`) | `{'<CAPTION_TO_PHRASE_GROUNDING>': {'bboxes': [...], 'labels': ['A green car','a yellow building']}}` |

Output also includes an optional annotated `img` URL.

### Azure Computer Vision (v4.0)

`features=Caption` → `captionResult`:
```jsonc
"captionResult": { "captions": [ { "text": "a man pointing at a screen", "confidence": 0.4891 } ] }
```

`features=denseCaptions` → `denseCaptionsResult.values[]`, up to **10** entries — one for the whole image plus per-region:
```jsonc
{ "text": "a man driving a tractor in a field", "confidence": 0.5428, "boundingBox": {"x":132,"y":266,"w":209,"h":219} }
```

### Google Cloud Vision
No captioning feature in the classical Vision API (captioning is available via Gemini multimodal vision, out of scope here).

---

## 7. Optical Character Recognition (OCR)

### Concepts
- **Image OCR** — text "in the wild" (signs, posters, labels). Synchronous, near-real-time.
- **Document OCR** — dense text, multi-page, handwriting, PDF/TIFF. Often asynchronous with submit→poll→fetch.
- **Hierarchy:** pages → blocks → paragraphs → words → symbols (Google) or blocks → lines → words (Azure). Each level carries a bounding polygon and (word/symbol-level) confidence.
- **Quad boxes** — Florence-2 `<OCR_WITH_REGION>` returns 4-point polygons (8 coords) for tilted text.

### Google Cloud Vision — `TEXT_DETECTION` vs `DOCUMENT_TEXT_DETECTION`

| Aspect | `TEXT_DETECTION` | `DOCUMENT_TEXT_DETECTION` |
|---|---|---|
| Use case | Any image with text | Dense documents, handwriting, PDF/TIFF |
| Response | `textAnnotations[]` (full string + per-word boxes) | `fullTextAnnotation` hierarchical |
| `maxResults` | Ignored (returns all) | Ignored (returns all) |

`TextAnnotation` hierarchy:
```
TextAnnotation
 └─ pages[]
     ├─ width, height
     └─ blocks[]  (blockType: TEXT/PICTURE/SEPARATOR/...)
         ├─ boundingBox  (polygon vertices)
         └─ paragraphs[]
             └─ words[]
                 └─ symbols[]  (each with confidence)
```
Language hints via `imageContext.languageHints` (BCP47, e.g. `"en-t-i0-handwrit"`). Python client uses `FeatureType` enum: `PAGE=1, BLOCK=2, PARA=3, WORD=4, SYMBOL=5`.

### Azure Computer Vision (v4.0) — `Read` (synchronous image OCR)
`features=Read` → `readResult.blocks[].lines[].words[]`; each line and word has a `boundingPolygon` (4 `{x,y}` points) and each word a `confidence`:
```jsonc
"readResult": {
  "blocks": [{
    "lines": [{
      "text": "You must be the change you",
      "boundingPolygon": [{"x":251,"y":265},{"x":673,"y":260},{"x":674,"y":308},{"x":252,"y":318}],
      "words": [
        { "text": "You", "boundingPolygon": [...], "confidence": 0.996 }
      ]
    }]
  }]
}
```
For PDFs/Office docs use the separate **Document Intelligence Read** model (asynchronous). The legacy v3.2 Read API was also asynchronous (submit→poll→get); v4.0 is synchronous.

### Replicate — Florence-2

| `task_input` | Output |
|---|---|
| `"OCR"` | `{'<OCR>': '<recognized text string>'}` |
| `"OCR with Region"` | `{'<OCR_WITH_REGION>': {'quad_boxes': [[x1,y1,x2,y2,x3,y3,x4,y4],...], 'labels': ['text1',...]}}` |

---

## 8. Face & Pose Detection

### Concepts
- **Face detection** returns a bounding box per face. **Facial recognition** (identifying *who*) is explicitly **not** supported by Google; Azure removed faces from v4.0 entirely (gated/limited v3.2 surface).
- **Landmarks** — named facial points (eyes, nose, mouth, eyebrows, chin, cheeks) with 3D positions `{x,y,z}` (z = depth). Google exposes ~30+ landmark types.
- **Head pose** — `rollAngle`, `panAngle`, `tiltAngle` (degrees).
- **Emotion likelihood** — Google uses a `Likelihood` enum (UNKNOWN/VERY_UNLIKELY/UNLIKELY/POSSIBLE/LIKELY/VERY_LIKELY) for joy/sorrow/anger/surprise, plus `underExposedLikelihood`, `blurredLikelihood`, `headwearLikelihood`.
- **Pose detection** (full-body keypoints) is offered by MediaPipe-based models on Replicate, not by the hosted Google/Azure vision APIs.

### Google Cloud Vision — `FACE_DETECTION`
`faceAnnotations[]`, each `FaceAnnotation`:

| Field | Type | Description |
|---|---|---|
| `boundingPoly` / `fdBoundingPoly` | `BoundingPoly` | Face box (skin-tight variant for `fd`) |
| `landmarks[]` | array | `{type, position:{x,y,z}}` — e.g. `LEFT_EYE`, `NOSE_TIP`, `MOUTH_CENTER`, `CHIN_GNATHION` |
| `rollAngle`, `panAngle`, `tiltAngle` | float | Head pose in degrees |
| `detectionConfidence`, `landmarkingConfidence` | float | 0.0–1.0 |
| `joy/sorrow/anger/surprise/underExposed/blurred/headwearLikelihood` | `Likelihood` enum | Attribute ratings |

### Replicate — `chigozienri/mediapipe-face`
Batch/individual face detection + keypoints using Google's MediaPipe. Provides full facial mesh / pose landmarks (richer keypoint set than the hosted API).

### Azure Computer Vision
Faces are **not** in v4.0 `features`. The v3.2 API exposed face rectangles + age/gender, now gated; Microsoft recommends the dedicated **Face API** service (separate from Computer Vision) for face detection/identification.

---

## 9. Object Tracking in Video

### Concepts
- **Tracking** = detection + temporal ID continuity across frames. The same object keeps one ID/mask as it moves.
- **Zero-shot tracking** — no per-video training; you prompt in the first frame (a point or box) and the model follows the object.
- **Motion-aware memory** — SAMURAI extends SAM 2 with a memory module that models object motion for robust tracking under occlusion/blur.
- **Per-frame output** — tracking models return masks (COCO RLE per frame) or bounding boxes keyed by frame number, plus an `object_id`.

### Replicate — `zsxkib/samurai` (zero-shot video tracker, built on SAM 2)

| Input | Description |
|---|---|
| `video` / frames | Input video or frame sequence |
| starting position | `(x, y)` top-left of the object in the first frame |
| size | `(width, height)` of the object's bounding box in the first frame |

Outputs:
- A tracking video (object highlighted in red).
- Per-frame masks in **COCO RLE**, keyed by frame number:
```jsonc
{
  "<frame_number>": [{
    "size": [height, width],
    "counts": "<encoded_string>",   // RLE mask
    "object_id": 0
  }]
}
```

### Replicate — `zsxkib/yolo-world` (video mode)
Accepts a video as `input_media`; returns an annotated output video (`output.mp4`) or per-frame JSON detections (`return_json=true`). Open-vocabulary — you pass `class_names` and it detects+tracks those classes across frames.

### Replicate — `meta/sam-2-video`
SAM 2 applied to video: prompt an object in one frame, get masks across all frames. Same family as `meta/sam-2` (image) but for video.

### Google / Azure hosted
Neither offers video object tracking as a hosted feature.

---

## 10. Landmark, Logo, Web & Safe-Search Detection

These are Google-hosted-only specialized detectors (with partial Azure v3.2 equivalents now deprecated in v4.0).

### Google Cloud Vision feature types

| Feature | Returns | Notes |
|---|---|---|
| `LANDMARK_DETECTION` | `landmarkAnnotations[]` — `description` (landmark name), `score`, `boundingPoly`, `locations[]` (lat/long) | Natural/human-made monuments |
| `LOGO_DETECTION` | `logoAnnotations[]` — `description`, `score`, `boundingPoly` | Company logos |
| `WEB_DETECTION` | `webDetection` — `webEntities[]`, `visuallySimilarImages[]`, `fullMatchingImages[]`, `partialMatchingImages[]`, `pagesWithMatchingImages[]` | Reverse image / web entity lookup |
| `SAFE_SEARCH_DETECTION` | `safeSearchAnnotation` — `adult`, `spoof`, `medical`, `violence`, `racy` (each a `Likelihood` enum) | Content moderation |
| `IMAGE_PROPERTIES` | `imagePropertiesAnnotation.dominantColors.colors[]` (color + pixel fraction + score) | Dominant-color palette |
| `CROP_HINTS` | `cropHintsAnnotation.cropHints[].boundingPoly` | Suggested crop vertices |
| `PRODUCT_SEARCH` | Product matches in a product set | Retail product recognition |

### Replicate
- `falcons-ai/nsfw_image_detection` — fine-tuned ViT for NSFW image classification (content moderation equivalent).
- No hosted landmark/logo/web-reverse-image models.

### Azure Computer Vision v4.0
Removed `Brands`, `Landmarks`, `Celebrities`, `Adult`, `Color`, `Image type` from v4.0 (these existed in v3.2). Equivalent specialized detectors are now in other Azure AI services.

---

## 11. Azure Custom Vision — Train-Your-Own Models

### Concepts
- **Project types:** Image **Classification** (labels per whole image) vs **Object Detection** (labels + bounding boxes per region).
- **Tags / labels** — user-defined at training time. For object detection you draw **regions** (bounding boxes) around tagged objects.
- **Iteration** — a trained version of the model; multiple can coexist; you choose which to **publish**.
- **Published endpoint** — a named prediction URL tied to a published iteration; apps call it at runtime.
- **Training API vs Prediction API** — two separate APIs (and often two separate Azure resources): Training builds/evaluates/publishes; Prediction serves inference.
- **Domains** — algorithm variants optimized for subject matter (landmarks, retail, general).
- **Export** — trained models can be exported for offline/mobile use: **ONNX**, **TensorFlow**, **CoreML**, **DockerFile**.
- **Deprecation note:** Custom Vision is deprecated (retirement 2025-09-25 → 2028-09-25). Microsoft recommends migrating to **Azure ML AutoML** or **Foundry model catalog** generative models, or **Azure Content Understanding** (preview) for custom classification.

### Lifecycle
```
1. Add images        (positive + negative examples)
2. Tag / label       (regions for object detection)
3. Train             (creates an iteration; self-tests accuracy)
4. Iterate           (test, retrain, improve)
5. Publish           (iteration → named prediction endpoint)
6. Predict           (POST image to published endpoint → tags + boxes + probabilities)
7. Export (optional) (ONNX / TF / CoreML / Docker)
```

### Optimization
- Good for **major** visual differences; ~50 images/label is enough to prototype.
- Not optimal for subtle defects (minor cracks/dents) — choose a different domain/service.
- Smart Labeler can auto-suggest tags/regions to speed labeling.

---

## 12. Cross-Platform Comparison & Concepts

### Coordinate systems at a glance

| Provider / model | Box encoding | Coordinate space |
|---|---|---|
| Replicate Grounding DINO | `[x1,y1,x2,y2]` (XYXY) | pixels |
| Replicate YOLO-World (JSON) | `x0,y0,x1,y1` (XYXY) | pixels |
| Replicate Florence-2 | `[x1,y1,x2,y2]` (XYXY); OCR quad = 8 coords (4 pts) | pixels |
| Replicate Semantic-Segment-Anything | `bbox:[x,y,w,h]` (XYWH) + COCO RLE `segmentation` | pixels |
| Replicate SAM-2 | mask PNG URLs | image-sized |
| Replicate SAMURAI | COCO RLE per frame | frame-sized |
| Google Cloud Vision (labels/faces) | `BoundingPoly.vertices:[{x,y}]` | pixels |
| Google Cloud Vision (objects) | `NormalizedVertex` polygon | normalized 0–1 |
| Azure Computer Vision v4.0 (OCR) | `boundingPolygon:[{x,y}×4]` | pixels (relative to `metadata.width/height`) |
| Azure Computer Vision v4.0 (objects/dense captions/smart crops) | `boundingBox:{x,y,w,h}` (XYWH) | pixels |

### Confidence / likelihood

| Provider | Form | Notes |
|---|---|---|
| Replicate | float 0–1 (`confidence`, `score`) | Thresholds exposed as inputs (e.g. `box_threshold`, `score_thr`, `pred_iou_thresh`) |
| Google | float 0–1 (`score`, `detectionConfidence`) + `Likelihood` enum | `Likelihood`: UNKNOWN(0)→VERY_LIKELY(5) for faces/safe-search |
| Azure | float 0–1 (`confidence`) | No API-mandated threshold; caller decides |

### Open- vs fixed-vocabulary

| | Open-vocabulary (text prompt) | Fixed vocabulary |
|---|---|---|
| Replicate | Grounding DINO, YOLO-World, OWL-ViT, Grounded SAM, Florence-2 (via task prompts) | YOLOX, Mask2Former (ADE20k classes) |
| Google | — | `LABEL_DETECTION`, `OBJECT_LOCALIZATION`, `LOGO_DETECTION` |
| Azure | — | `Tags`, `Objects` |
| Custom Vision | — (but you *define* the vocabulary at training time) | Your trained tags |

### Sync vs async

| Provider | Pattern |
|---|---|
| Replicate | Always async-by-default: `POST /v1/predictions` returns `starting`/`processing`; poll `predictions.get` or use webhooks. Long video models can take minutes. |
| Google Cloud Vision | `images:annotate` is synchronous (batch up to N images). `files:asyncBatchAnnotate` is asynchronous for up to 2000 files (PDF/TIFF/batch), output written to Cloud Storage. |
| Azure Computer Vision v4.0 | Synchronous single-call `Analyze`. Legacy v3.2 Read and Document Intelligence Read are async (submit→poll→get). |
| Azure Custom Vision | Training is a long-running operation; Prediction is synchronous. |

### API request shapes (summary)

**Replicate (uniform across all models):**
```http
POST https://api.replicate.com/v1/predictions
Authorization: Bearer $REPLICATE_API_TOKEN
{ "version": "<model-version-hash>", "input": { /* model-specific schema */ } }
```
Then `GET /v1/predictions/{id}` until `status` ∈ {`succeeded`,`failed`,`canceled`}.

**Google Cloud Vision:**
```http
POST https://vision.googleapis.com/v1/images:annotate
Authorization: Bearer $(gcloud auth print-access-token)
{
  "requests": [{
    "image":   { "content": "<base64>" } | { "source": { "imageUri": "gs://..." } },
    "features": [ { "type": "LABEL_DETECTION", "maxResults": 10 }, { "type": "FACE_DETECTION" } ],
    "imageContext": { "languageHints": ["en"] }
  }]
}
```

**Azure Computer Vision (v4.0):**
```http
POST {endpoint}/computervision/imageanalysis:analyze?features=Caption,Tags,Read,Objects&api-version=2024-02-01
Ocp-Apim-Subscription-Key: <key>
Content-Type: application/octet-stream
<binary image bytes>
```
Response: `{ "modelVersion":"2024-02-01", "metadata":{width,height}, "captionResult":..., "tagsResult":..., "readResult":..., ... }` — one block per requested feature.

**Azure Custom Vision Prediction:**
```http
POST {prediction-endpoint}/customvision/v3.0/prediction/detect/{projectId}/iterations/{publishedName}/image
Prediction-Key: <key>
Content-Type: application/octet-stream
<binary image bytes>
```
Returns predicted tags (classification) or tags + bounding boxes (object detection) with probabilities.

### When to choose which

- **Need a single hosted call returning many features at once** (labels + faces + OCR + objects + safe-search) → **Google Cloud Vision** `images:annotate`.
- **Need open-vocabulary detection ("find my custom objects by text")** → **Replicate** `adirik/grounding-dino` (image) or `zsxkib/yolo-world` (image/video).
- **Need pixel-precise masks / inpainting-ready cutouts** → **Replicate** `meta/sam-2` (promptable) or `schananas/grounded_sam` (text-prompted).
- **Need region captions + OCR + tags in one Florence-based call** → **Azure Computer Vision v4.0** `Analyze`.
- **Need a unified model that does captioning + detection + OCR + grounding via task prompts** → **Replicate** `lucataco/florence-2-large`.
- **Need to train a custom classifier/detector on your own labeled images and export to mobile (ONNX/TF/CoreML)** → **Azure Custom Vision** (mind the 2025–2028 retirement; plan migration to Azure ML AutoML / Foundry).
- **Need video object tracking with a first-frame prompt** → **Replicate** `zsxkib/samurai` (mask-based) or `zsxkib/yolo-world` (box-based, open-vocab).