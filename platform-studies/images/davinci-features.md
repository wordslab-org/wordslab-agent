# DaVinci.ai — Image & Video Creation/Editing Features Study

> Source: https://davinci.ai/ (site crawl + per-model pages, analyzed in July 2026)
> Publisher: DaVinci Company S.L.U. (parent: hubx.co)
> Positioning: "All-in-one generative media platform" aggregating 50+ third-party and proprietary AI models behind a single subscription.

---

## Table of Contents

1. [Platform Concepts](#1-platform-concepts)
2. [Transformation Steps & Pipelines](#2-transformation-steps--pipelines)
3. [Features Description & Parameters](#3-features-description--parameters)
   - 3.1 [Image Generation Models](#31-image-generation-models)
   - 3.2 [Video Generation Models](#32-video-generation-models)
   - 3.3 [Editing Tools](#33-editing-tools-ai-editor--reshoot)
   - 3.4 [Generation Modes](#34-generation-modes)
4. [UI Design](#4-ui-design)
5. [Cross-Cutting Notes](#5-cross-cutting-notes)

---

## 1. Platform Concepts

DaVinci is a **model-aggregator platform**: instead of training its own foundation models, it wraps multiple leading third-party image/video models (OpenAI Sora, Google Veo & Nano Banana, ByteDance Seedance & Seedream, Kuaishou Kling, xAI Grok, Black Forest Labs Flux, etc.) behind one unified UI, billing system, and ownership framework.

### Core concepts

| Concept | Description |
|---|---|
| **AI Toolkit** | The single web/mobile workspace that hosts every generation and editing flow. All models and tools are reached without leaving the app. |
| **Credits** | Unified internal currency consumed per generation; cost depends on model + output type. Credits reset per billing cycle; extra credits purchasable; subscription required. |
| **Model selection** | Each generation starts by picking a model from a catalog; availability scales with plan tier (Basic → Creator → Pro → Ultimate). |
| **Reference images** | Users can upload one or several images to guide style, identity, composition, framing — used by image-to-image, image-to-video, and editing tools. |
| **Start/end frames** | Video models accept a starting frame (and sometimes an ending frame) to control scene entry/exit and transitions. |
| **Native audio** | Most current-generation video models (Sora 2, Veo 3.1, Kling 2.6, Wan 2.6, Seedance 2.0) generate synchronized audio (dialogue, SFX, ambient, music) in the same pass. |
| **Consistency / style locking** | Multiple models expose "locked character consistency" and "style locking" so the same subject/style persists across variations and edits. |
| **Ownership & commercial use** | By default users own outputs and may use them commercially under the Terms of Use; no attribution required; prompts/outputs are private and never used for training. |
| **Channels / templates** | Outputs can be auto-resized for social/ads/web; a templates tab (`/app/templates`, `/app/video?tab=templates`) drives model-specific quick starts. |
| **Mobile parity** | iOS & Android apps expose the same models and editing tools as the web app. |

### Architectural analogy
DaVinci ≈ a **creative SaaS gateway / router**: request → model picker → generation → in-app refinement loop → export. The platform differentiates on breadth of models, unified billing, and privacy/ownership posture rather than on proprietary model quality.

---

## 2. Transformation Steps & Pipelines

All flows follow the same 3-step skeleton advertised across landing pages:

```
Step 1 — Open Toolkit        →  Step 2 — Choose model + write prompt (+ refs)  →  Step 3 — Generate & refine
```

### 2.1 Generic image generation pipeline
```
Prompt (text) ──┐
                ├─► Model selector ─► Generation ─► Preview ─► [Refine/Remix/Upscale/Edit] ─► Download
Reference image(s) ─┘        ▲                                                    │
                             └────────────── iterate (loop back) ─────────────────┘
```
Parameters at Step 2: model, prompt text, aspect ratio, resolution, number of outputs, reference images (up to 3 for Grok Imagine; multi-image supported elsewhere).

### 2.2 Image-to-image transformation pipeline
```
Existing image(s) ─► Image-to-Image tool ─► Model + edit prompt (natural language)
                ─► Controlled transform (background/style/elements/layout)
                ─► Refine loop ─► Export
```
Edit vocabulary (model-dependent): "remove elements", "replace objects", "adjust lighting", "change background", "restyle", "relight", "resize", "upscale".

### 2.3 Text-to-video pipeline
```
Prompt (subject, action, environment, lighting, camera, style)
   ─► Video model ─► (optional) start frame / end frame
   ─► Generation (motion + physics) + native audio sync
   ─► Preview ─► [extend / continue from last frame / refine] ─► Download
```

### 2.4 Image-to-video (animation) pipeline
```
Upload image (or pick from session) ─► add prompt ─► adjust settings ─► Generate
   ─► Animated clip with camera movement & natural motion
```

### 2.5 Seedance 2.0 multimodal director pipeline (richest flow)
```
Text prompt with @-tags ──┐
@Image / @Video / @Audio ─┼─► Seedance 2.0 ─► Cinematic multi-shot sequence
                          │                  ├─ audio-visual sync generation
                          │                  ├─ scene extension (continue from last frame)
                          │                  └─ structured refinement (swap characters, insert/remove scenes)
                          └─► Download
```
Modes supported: text-to-video, image-to-video, video-to-video, audio-to-video.

### 2.6 Editing pipeline (AI Editor at `/app/edit`)
```
Upload image ─► Edit tool (Spotlight / Change Background / Resize / Restyle / Relight / Upscale / Object remove/replace)
            ─► AI processing ─► Preview ─► Iterate ─► Export
```

### 2.7 Product-photo → commercial-video pipeline (advertised on homepage)
```
Still product shot ─► Image-to-video ─► Short-form commercial video ─► Auto-resize per platform
```

---

## 3. Features Description & Parameters

### 3.1 Image Generation Models

> ~17 models listed. All support text-to-image; most also support image-to-image with reference uploads. Common parameters: prompt, aspect ratio, resolution (up to 4K on premium models), number of outputs, reference images.

| Model | Provider | Strengths / Key Parameters |
|---|---|---|
| **Nano Banana 2** | Google (Gemini 3.1 Flash) | High-volume, cost-effective; natural-language prompts; seamless multi-image blending; subject consistency across angles; targeted edits (smart object replacement, style locking, scene restyling). Up to 4K. |
| **Nano Banana Pro** | Google | Ultra-realistic 4K; multi-language text rendering; consistent chars/objects across scenes; integrated search (reflects real-time weather/locations/events); enhanced creative controls (camera angles, DoF, lighting, color via text). |
| **DaVinci Ultra** | Google (proprietary label) | Fast rendering; prompt-the-way-you-think; seamless element blending; consistency across scenes; fine-tune detail (local edits without affecting whole scene); smart object replacement, style locking, scene restyling. |
| **Kling 3.0 / O1 Image** | Kuaishou | Director-level control; locked character consistency (faces, proportions, logos); stack multiple actions per prompt; high-fidelity detail; 2K/4K; multi-image sequences; unified text↔image workflow. |
| **Grok Imagine** | xAI | Fast expressive generation; accurate text rendering; aspect ratios 1:1, 16:9, 9:16, widescreen; up to 3 reference images per prompt. |
| **Seedream 4.5** | ByteDance | Cinematic realism; natural lighting & textures (skin/fabric/glass); reliable consistency across generations; targeted edits (background, layout, clothing, style); clear small detail/text (packaging mockups). |
| **Flux.2 Pro / Pro Edit** | Black Forest Labs | Production-ready quality; precise AI editing & refinement; strong prompt control. Pro Edit specialized for controlled edits. |
| **Flux.2 Dev / Flash / Turbo** | Black Forest Labs | Experimentation / rapid iteration / high-throughput variants. |
| **GPT Image 1.5 / 1 Mini** | OpenAI | Text-to-image + image-to-image; precise text rendering; multi-input composition; Mini = speed/efficiency. |
| **Ideogram V3 / V3 Character** | Ideogram | Typography-aware generation; V3 Character for consistent expressive characters. |
| **Imagen 4 / Imagen 4 Ultra** | Google | Realistic lighting/textures/detail; Ultra = premium fidelity for high-end commercial use. |
| **LTX 2.0 Pro / Z-Image Turbo** | Lightricks / others | High-fidelity production-ready; Turbo = speed-focused for previews/concepts. |

**Common image parameters (inferred from docs):** prompt text, model, aspect ratio (incl. 1:1, 16:9, 9:16, widescreen), resolution (up to 4K), output count, reference image(s), edit instructions (natural language for object replacement, restyle, relight, background change).

---

### 3.2 Video Generation Models

> ~14 models. Common parameters: prompt (subject, action, environment, lighting, camera movement, style, pacing, mood), model, aspect ratio, start frame, end frame, native audio toggle. Most current models generate synchronized audio in-pass.

| Model | Provider | Duration/Quality | Signature Features |
|---|---|---|---|
| **Sora 2** | OpenAI | High-end | Multi-modal (text/image) references; built-in audio sync (dialogue, SFX, music, lip-sync); physics-based motion (gravity, momentum, collisions); consistent multi-shot scenes; prompt-level camera/framing/pacing/mood control. |
| **Sora 2 Pro** | OpenAI | Premium, production-ready | All Sora 2 features + higher visual fidelity, stronger consistency, refined motion, **advanced scene control** (deeper control over timing, camera behavior, scene structure, layered sequences). |
| **Veo 3.1** | Google | Cinematic, 4K-capable | Cinematic prompt interpretation (time-lapse, over-the-shoulder, dolly zoom); define opening & closing frames; built-in audio (ambience, dialogue, music); real-world physics; HD output for large screens. |
| **Veo 3.1 Fast** | Google | Faster iteration | Same quality as Veo 3.1 with reduced generation time; all Veo 3.1 cinematic features. |
| **Seedance 2.0** | ByteDance | Multimodal cinematic | **Universal @-tag reference system** (@Image/@Video/@Audio) for opening frames, camera movement, music cues; text/image/video/audio-to-video modes; seamless scene extensions (continue from last frame); auto audio-visual sync; structured refinement (swap chars, insert/remove scenes). |
| **Seedance 1.5 Pro / 1.0 Pro Fast** | ByteDance | Expressive/choreographed motion | Performance-driven clips; Fast variant for rapid testing. |
| **Kling 2.6 Pro** | Kuaishou | Cinematic, native audio | Built-in audio (SFX, ambient, dialogue); precise prompt control (pacing, camera, emotion); cinematic realism (lighting/depth); expressive character performance; reliable visual consistency across shots. |
| **Kling 2.5 Turbo Pro** | Kuaishou | Speed-optimized | Rapid generation for drafts/iterations; motion stability. |
| **Kling O3** | Kuaishou | High-end | Cinematic realism, precise prompt adherence, detailed motion. |
| **Kling 1.6 Standard / Pro** | Kuaishou | Everyday / professional | Balanced quality & stable motion; Pro = sharper visuals + better motion control. |
| **Wan 2.6** | (Alibaba Wan) | Cinematic, native audio | Built-in audio; strong prompt adherence; cinematic realism; natural storytelling & expression; advanced motion & camera control; consistent visual continuity. |
| **Grok Imagine Video** | xAI | Stylized | Creative-first, bold concepts, expressive distinctive visuals. |

**Common video parameters (inferred):** prompt, model, aspect ratio, start frame image, end frame image (Veo 3.1 / Sora 2 Pro), camera language, pacing, mood, audio generation, scene structure/continuity controls.

---

### 3.3 Editing Tools (AI Editor & Reshoot)

Located at `/app/edit` and `/app/reshoot`. These are tool-mode features (largely model-agnostic, powered by the underlying image models) surfaced as distinct functions in the footer:

| Tool | Route | What it does | Parameters (inferred) |
|---|---|---|---|
| **Image Upscale** | `/app/edit` | Increase resolution while preserving clarity/detail for large-format/premium ad placements. | Source image, target resolution. |
| **Change Background** | `/app/edit` | Replace background while keeping subject, lighting, depth, context intact. | Source image, new background prompt. |
| **Resize Image** | `/app/edit` | Auto-resize designs for each platform (social/ads/web) without rebuilding layouts or losing visual balance. | Source image, target aspect/format. |
| **Restyle Image** | `/app/edit` | Transform look/mood of a scene without changing the subject; adjust setting/environment/tone. | Source image, style/mood prompt. |
| **Relight Image** | `/app/edit` | Adjust lighting on an existing image. | Source image, lighting instructions. |
| **Object remove/replace** | (within editor) | Remove unwanted elements or swap objects naturally, preserving lighting/depth/context. | Source image, mask/selection, replacement prompt. |
| **Spotlight (Reshoot)** | `/app/reshoot?tab=spotlight` | Re-shoot / reframe existing footage or images with new lighting/composition. (Detailed page not publicly crawlable.) | Source asset, reshoot direction. |
| **Retouch details** | (within editor) | Fine-tune small details / retouch. | Source image, edit instructions. |

The **AI Editor** umbrella is described as: "Edit and enhance images instantly with AI powered tools. Remove backgrounds, retouch details, adjust lighting, replace elements, or upscale visuals with precision."

---

### 3.4 Generation Modes

| Mode | Inputs | Available across |
|---|---|---|
| **Text-to-Image** | Text prompt | All image models. |
| **Image-to-Image** | Reference image(s) + text prompt/edit instructions | All image models (most support multi-ref). |
| **Text-to-Video** | Text prompt (+ optional start/end frame) | All video models. |
| **Image-to-Video** | Image + prompt → animated clip | All video models (Sora 2, Veo 3.1, Kling, Wan, Seedance). |
| **Video-to-Video** | Video sample + prompt (refine existing footage) | Seedance 2.0 (explicit); others via extend/refine. |
| **Audio-to-Video** | Audio reference drives generation | Seedance 2.0 (explicit via @Audio). |
| **Multimodal combined** | Text + @Image + @Video + @Audio with structured tags | Seedance 2.0 (@-tag system). |

---

## 4. UI Design

DaVinci's product is a **single-page web app** (`/app/...`) supplemented by marketing landing pages per model/feature. Inferred UI architecture from routes, page structure, and descriptions:

### 4.1 Navigation / Top-level sections
Primary nav (from footer + headers):
- **AI Image** → image generation hub (`/image-generator`, `/app/explore`)
- **AI Video** → video generation hub (`/video-generator`)
- **AI Editor** → editing workspace (`/app/edit`)
- **Explore** → gallery/inspiration of generated images (`/app/explore`)
- **Templates** → template-driven quick starts (`/app/templates`, `/app/video?tab=templates`)
- **Pricing** / **Sign in** / **Start Free Now** → `/app`

### 4.2 App routes (the actual product UI)
| Route | Purpose |
|---|---|
| `/app/explore` | Image generation workspace + gallery; model picker exposed via `?model=` query (e.g. `?model=nano-banana-pro`, `?model=seedream-4-5`). |
| `/app/video?tab=templates` | Video generation workspace; preselects model via `?model=` (e.g. `kling-26`, `sora-2`, `sora-2-pro`, `veo-31`, `veo-31-fast`, `seedance-2`, `wan-26`). Tabs include `templates`. |
| `/app/edit` | AI Editor: upscale, change background, resize, restyle, relight, object remove/replace, retouch. |
| `/app/reshoot?tab=spotlight` | Reshoot/Spotlight tool. |
| `/app/templates` | Template library across image & video. |

### 4.3 Generation flow UI pattern (3-step wizard)
Every landing page documents the identical 3-step wizard, suggesting a consistent left-to-right panel layout:
1. **Toolkit entry** — choose image vs video tool.
2. **Model + prompt panel** — model selector (dropdown/grid), prompt textarea, reference image upload zone, parameter controls (aspect ratio, resolution, output count).
3. **Generate + refine** — result preview grid, refine/remix/upscale/edit actions, download.

### 4.4 Model selection UX
- **Marketing pages**: a "Featured models" / "Model showcase" section presents each model as a card with name, one-line description, and `More on` + `Try` CTAs.
- **In-app**: a model picker (likely a dropdown or carousel) with `?model=` deep-linking; models gated by subscription tier.
- Each model has its own **model landing page** (`/image-generator/<model>`, `/video-generator/<model>`) with: hero example, "Why creators choose" feature grid, "How to create" accordion (3 steps), "Who is it for" persona cards, "Try it" interactive block, "Model capabilities/Powerful features" grid, footer.

### 4.5 Common page blocks (per model page)
- Hero with example image/video + "Try [Model]" CTA.
- "Why creators choose" — 3-4 feature highlight cards (repeated twice for responsive/duplicate markup).
- "How to create" — accordion: (1) Open Toolkit, (2) Add references/enter prompt, (3) Generate, refine, download.
- "Who is it for" — 3 persona cards with imagery (Content creators/YouTubers, Brands & agencies, Filmmakers/storytellers).
- "Try it" — interactive prompt-or-upload block with example output.
- "Powerful features / Model capabilities" — 4 feature cards (duplicated markup).
- Universal footer: Models list, Features list, About, Socials, copyright (DaVinci Company S.L.U., © 2026).

### 4.6 Visual / brand design language (inferred)
- Dark, cinematic hero backgrounds (`.webp`/`.webm` media) showcasing generated output.
- Card-based layouts with large imagery, short titles, one-paragraph descriptions.
- Strong CTAs: "Start Free Now", "Start Creating", "Try [Model]", "Learn more".
- Persona-driven marketing (consistent 3 personas across model pages using shared `/_img/landing/model-landing/shared/who-is-*.webp` assets).
- Step-numbered how-to sections (01, 02, 03).
- Mobile parity emphasized; app store badges present on every page.

### 4.7 Pricing UI
Pricing shown on homepage with 4 tiers (note: contact-page FAQ lists Basic/Creator/Pro/Ultimate; homepage shows Pro/Ultimate/Creator with explicit monthly credits):

| Plan | Price (annual) | Credits/mo | Images/mo | Videos/mo | Notes |
|---|---|---|---|---|---|
| Pro | $39.99 | 5,000 | ~2,500 | ~125 | Priority speed/queue, core tools |
| Ultimate | $99.99 | 15,000 | ~7,500 | ~375 | Max performance, highest priority |
| Creator | $199.99 | 40,000 | ~20,000 | ~1,000 | High-volume, full core tools |
| Basic | (FAQ) | — | — | — | Entry-level, limited models/limits |

### 4.8 Editor UI (inferred from `/app/edit` feature set)
A tool-tabbed editor where each tab (Upscale, Background, Resize, Restyle, Relight, Remove/Replace objects, Retouch) presents: upload/drop zone → parameter/prompt controls → preview → iterate/export. Reshoot (`/app/reshoot`) appears as a related workspace with a `spotlight` tab.

---

## 5. Cross-Cutting Notes

### 5.1 Differentiators (per marketing)
- **Breadth**: 50+ models, 15+ image models, 14+ video models in one subscription; no tool-switching.
- **Privacy/ownership**: outputs private by default; never used for training; user owns outputs; commercial use by default; no attribution required.
- **Always-current models**: new industry models added at launch, no upgrade needed.
- **Mobile parity**: full model access on iOS/Android.
- **Unified billing via credits** across all tools/models.

### 5.2 Limitations / gaps observed
- App pages (`/app/*`) are gated/JS-rendered — detailed parameter panels, exact controls, and precise numeric parameters (resolution presets, exact aspect ratios per model, duration limits, credit costs per model) are not exposed on the public marketing site. The parameters documented above are inferred from descriptive copy, not from an API spec or in-app UI capture.
- `text-to-video/editor` route returned 404; some model pages (Kling O3, Kling 1.6 variants, Grok Imagine Video, Seedance 1.x, Flux variants, GPT Image variants, Ideogram, Imagen, LTX, Z-Image) redirect to `/app` rather than having dedicated public feature pages, so their detailed parameters are summarized from list descriptions only.
- No public API documentation found; DaVinci is positioned as a consumer/creator SaaS, not a developer platform.

### 5.3 Model provider map (who powers what)
| Provider | Image models | Video models |
|---|---|---|
| Google | Nano Banana 2, Nano Banana Pro, DaVinci Ultra, Imagen 4 / 4 Ultra | Veo 3.1, Veo 3.1 Fast |
| OpenAI | GPT Image 1.5 / 1 Mini | Sora 2, Sora 2 Pro |
| ByteDance | Seedream 4.5 (5.0 mentioned in FAQ) | Seedance 2.0, 1.5 Pro, 1.0 Pro Fast |
| Kuaishou | Kling 3.0 / O1 Image | Kling 2.6 Pro, 2.5 Turbo Pro, O3, 1.6 Std/Pro |
| xAI | Grok Imagine | Grok Imagine Video |
| Black Forest Labs | Flux.2 Pro/Pro Edit/Dev/Flash/Turbo | — |
| Ideogram | Ideogram V3, V3 Character | — |
| Lightricks/others | LTX 2.0 Pro, Z-Image Turbo | — |
| Alibaba | — | Wan 2.6 |

### 5.4 Sources crawled
- Homepage: https://davinci.ai/
- Hubs: /video-generator, /image-generator, /image-generator/image-to-image, /ai-art-generator, /video-generator/text-to-video
- Image model pages: nano-banana, nano-banana-pro, seedream-4-5, davinci-ultra, grok-imagine, kling-o1-image
- Video model pages: seedance-2-0, sora-2-0, sora-2-pro, veo-3-1, veo-3-1-fast, kling-2-6, wan-2-6
- Support: /contact (FAQ), footer feature links (/app/edit, /app/reshoot?tab=spotlight)