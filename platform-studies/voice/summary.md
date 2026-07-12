# Unified Voice AI API Specification

> **Aggregated from:** ElevenLabs, OpenAI, Google Gemini, Cartesia, Deepgram, and Gradium
> **Purpose:** A single, exhaustive specification that encompasses every voice capability, concept, parameter, and processing step described across the six platform studies in `./platform-studies/voice/`.

---

## How to Read This Document

This document is written for end users — developers, product managers, and architects who want to understand the **full landscape** of voice AI capabilities before choosing a provider or building a system. It is organized as follows:

1. **Part I — Concepts & Vocabulary** — An approachable introduction to every concept you will encounter, with plain-language explanations and a glossary that maps the many different names providers use for the *same* idea.
2. **Part II — The Processing Pipeline** — Every voice capability ordered into a single exhaustive end-to-end pipeline, from authentication through delivery. Each stage lists the alternative approaches available and the synonyms used by each platform.
3. **Part III — The Unified API Specification** — A detailed, provider-agnostic API reference that describes every endpoint, parameter, and data structure needed to implement a "super complete" voice AI platform.

---

# Part I — Concepts & Vocabulary

## 1. What Is Voice AI?

Voice AI is the family of technologies that let machines **understand spoken audio**, **generate spoken audio**, and **converse with humans in real time**. A complete voice AI platform touches six broad domains:

| Domain | One-line description |
|--------|----------------------|
| **Speech-to-Text (STT)** | Convert recorded or live speech into written text. |
| **Text-to-Speech (TTS)** | Convert written text into natural-sounding spoken audio. |
| **Speech-to-Speech (STS)** | Convert spoken audio in one voice/language into spoken audio in another, preserving or changing the voice. |
| **Voice Design & Cloning** | Create new synthetic voices from samples, text descriptions, or by transforming existing voices. |
| **Audio Intelligence** | Extract meaning from audio: speakers, emotions, topics, intents, entities, summaries. |
| **Conversational Agents** | Stateful, low-latency voice sessions where the system listens, reasons, and speaks back — optionally calling tools. |

A single product may span several domains. For example, a "voice agent" combines STT (Listen), an LLM (Think), and TTS (Speak) into one real-time session.

## 2. Core Concepts (Provider-Agnostic)

### 2.1 Audio Modalities
Voice systems deal with four kinds of data flowing in and out:
- **Audio input** — sound captured from a microphone, file, or phone line.
- **Audio output** — generated speech or sound effects played back.
- **Text transcript** — the written form of spoken words (input or output).
- **Text prompt / instructions** — text that controls *how* the model speaks or what it does.

### 2.2 Voice
A **voice** is a distinct speech identity. It can come from:
- A **prebuilt library** of curated voices (every provider has one, ranging from 5 to 10,000+ voices).
- An **instant clone** created from a short audio sample (seconds to 2 minutes).
- A **professional clone** fine-tuned on extended audio data.
- A **consent-based custom voice** where a voice actor reads a consent phrase plus a sample.
- A **designed voice** generated from a text description ("a raspy middle-aged male with a Southern drawl").
- A **remixed voice** created by transforming an existing voice's attributes.
- A **localized voice** — an existing voice adapted to speak a new language.

### 2.3 Model
The **model** is the underlying AI. Providers expose multiple models tuned for different trade-offs:
- **Quality vs latency** — high-fidelity models vs ultra-fast models (~75–300 ms to first audio byte).
- **Language coverage** — from English-only to 90+ languages.
- **Domain specialization** — medical, meeting, finance, phonecall, drivethru, automotive, conversational.
- **Versioning** — pinned/dated snapshots for reproducibility vs rolling aliases that always point to the latest stable version.

### 2.4 Streaming vs Buffered
Audio can be delivered in three modes:
- **Buffered** — the entire audio file is returned after generation completes. Best for pre-generated content.
- **HTTP streaming** — audio bytes arrive progressively via chunked transfer encoding. The player can start before generation finishes.
- **WebSocket (bidirectional)** — a persistent connection where the client sends text/audio chunks and receives audio/transcript chunks. Best for real-time agents and lowest latency.
- **SSE (Server-Sent Events)** — one-way HTTP streaming with JSON-wrapped events (used by Cartesia for timestamps).
- **WebRTC** — peer-to-peer media transport for browser/mobile clients (used by OpenAI and Google partner integrations).

### 2.5 Session
A **session** is a stateful, persistent connection (usually WebSocket) that keeps context across multiple turns. Sessions enable:
- Continuous audio streaming.
- Conversation history accumulation.
- Turn-taking and interruption handling.
- Mid-session configuration updates.
- Resumption across reconnections.
- Context-window compression for indefinite conversations.

### 2.6 Turn Detection (VAD)
**Voice Activity Detection (VAD)** decides when a user starts and stops speaking. Approaches:
- **Silence-based VAD** — detects pauses in audio.
- **Semantic VAD** — a classifier scores whether the user has finished based on what they said.
- **Model-native turn detection** — the STT model itself signals end-of-turn (Deepgram Flux, Cartesia Ink-2).
- **Manual / push-to-talk** — the client explicitly commits audio and triggers a response.
- **Adaptive delay** — controls how much context the model buffers before emitting text (latency vs accuracy).

### 2.7 Interruption (Barge-in)
When a user speaks while the agent is responding, the system must **cancel the in-flight response** and **truncate unplayed audio**. Platforms differ in whether the server or client manages this (WebRTC/SIP = server-side auto-truncation; WebSocket = client must stop playback and send a truncate command).

### 2.8 Function Calling / Tools
In conversational sessions, the model can call **external functions** (weather, database lookups, API calls). Variants:
- **Synchronous** — model waits for the result before continuing.
- **Asynchronous / non-blocking** — model continues interacting while the function executes (Google 2.5).
- **Client-side** — client executes the function and returns the result.
- **Server-side** — the platform calls an HTTP endpoint directly.
- **HTTP server tools** — define tools from JSON schemas without writing function code (Cartesia Line).

### 2.9 Pronunciation Control
Mechanisms to override how specific words are pronounced:
- **Pronunciation dictionaries** — versioned sets of text-to-pronunciation mappings (ElevenLabs, Cartesia, Gradium).
- **Keyterm prompting** — bias recognition toward specific terms (ElevenLabs, Deepgram, Cartesia Flux).
- **Keyword boosting** — weighted terms with boost scores (Deepgram legacy).
- **Prompt-based context** — pass correct spellings as a prompt (OpenAI Whisper — first 224 tokens).
- **Rewrite rules** — language-specific text rewriting before synthesis (Gradium).

### 2.10 Emotion & Style Control
How to control *how* the voice sounds (not just *what* it says):
- **Voice settings** — stability, similarity_boost, style, use_speaker_boost, speed (ElevenLabs).
- **Generation config** — volume, speed, emotion enum with 60+ values (Cartesia).
- **json_config** — temperature, cfg_coef (voice similarity), padding_bonus (speed) (Gradium).
- **Instructions** — natural-language prompt controlling accent, emotion, intonation, speed, tone, whispering (OpenAI gpt-4o-mini-tts).
- **Director prompts** — Audio Profile + Scene + Director's Notes + Transcript (Google).
- **Audio tags** — inline `[laughs]`, `[whispering]`, `[excitedly]` (ElevenLabs v3, Google).
- **SSML tags** — `<speed>`, `<volume>`, `<emotion>`, `<break>`, `<spell>` (Cartesia).
- **In-text tags** — `<flush>`, `<break time="Ns"/>` (Gradium).
- **Affective dialog** — model adapts response style to match input expression (Google 2.5).

### 2.11 Diarization
**Diarization** identifies *who* is speaking. Capabilities:
- Speaker count up to 32 (ElevenLabs), 4 named speakers (OpenAI), unlimited via structured output (Google).
- Speaker roles (agent vs customer) — ElevenLabs.
- Speaker library matching — ElevenLabs.
- Known speaker names + audio references — OpenAI.
- Diarization models (v1, v2, latest) — Deepgram.
- Utterance segmentation — Deepgram.

### 2.12 Audio Intelligence
Extracting meaning beyond the transcript:
- **Summarization** — concise summary of content.
- **Sentiment analysis** — positive/negative/neutral with scores.
- **Topic detection** — automatic + custom topics.
- **Intent recognition** — automatic + custom intents.
- **Entity detection** — names, orgs, dates, phone numbers, emails, PII, PHI, PCI.
- **Emotion detection** — happy/sad/angry/neutral per segment (Google structured output).
- **Redaction** — remove sensitive info from transcripts.

### 2.13 Privacy & Data Retention
- **Zero Retention / Zero Data Retention** — prevent storage of request data (ElevenLabs, Cartesia).
- **Model Improvement Program opt-out** — exclude requests from model training (Deepgram).
- **Data residency** — regional servers (ElevenLabs: US, EU, India, Singapore).
- **Safety identifiers** — privacy-preserving user tracking (OpenAI).

### 2.14 Billing Units
- **Characters** — per character of text synthesized (ElevenLabs, Gradium, Deepgram TTS).
- **Credits** — platform-wide billing unit (Cartesia, ElevenLabs, Gradium).
- **Per second of audio** — STT and voice changing (Cartesia, Deepgram, Gradium).
- **Per minute** — dubbing, agent calls (ElevenLabs, Cartesia Line).
- **Per generation** — music, sound effects (ElevenLabs).
- **Surcharges** — entity detection +30%, keyterm prompting +20%, speaker roles +10% (ElevenLabs).

## 3. Cross-Provider Synonym Glossary

The table below maps the **many different names** providers use for the **same concept**. When reading any provider's docs, use this to translate to the unified vocabulary.

| Unified concept | ElevenLabs | OpenAI | Google Gemini | Cartesia | Deepgram | Gradium |
|----------------|-----------|--------|---------------|----------|----------|---------|
| API key header | `xi-api-key` | `Authorization: Bearer` | `x-goog-api-key` | `Authorization: Bearer` | `Authorization: Token` | `x-api-key` |
| Ephemeral client token | Single Use Token | Ephemeral Client Secret (`ek_...`) | Ephemeral Token (`access_token`) | Access Token (JWT) | Temporary API Token (JWT) | Browser Token (`?token=`) |
| Prebuilt voice library | Voice Library (10,000+) | Built-in Voices (13) | Prebuilt Voices (30) | Voice Library | Aura Voice Catalog (40+) | Flagship Voices (60+) |
| Voice identifier | `voice_id` | `voice` (name) or `{id}` | voice name (e.g. `Kore`) | `voice_id` (UUID) | `model` (`aura-2-{voice}-{lang}`) | `voice_id` |
| Instant voice clone | Instant Voice Cloning (IVC) | Custom Voice (consent + sample) | — (not available) | Instant Voice Clone | — (not available) | Custom Voice (≥10s sample) |
| Professional voice clone | Professional Voice Cloning (PVC) | — | — | Pro Voice Clone (fine-tune) | — | — |
| Voice from text description | Voice Design (`text-to-voice`) | — | — | — | — | — |
| Voice transformation | Voice Remix | — | — | — | — | — |
| Voice to new language | — (uses Dubbing API) | — | — | Voice Localization | — | — |
| Speech-to-Text | Speech to Text (STT) | Speech to Text / Transcription | Audio Understanding | Speech-to-Text (STT) | Speech-to-Text / Listen | Speech-to-Text (STT) |
| Batch STT | Batch STT (`/speech-to-text`) | File transcription (`/audio/transcriptions`) | Audio Understanding (Interactions API) | Batch STT (`/stt`) | Pre-Recorded STT (`/v1/listen`) | STT REST (`/post/speech/asr`) |
| Streaming STT | Realtime STT (WebSocket) | Realtime Transcription (session) | — (use Cloud Speech-to-Text) | Realtime STT (WebSocket) | Live Streaming STT (`WSS /v1/listen`) | STT WebSocket (`/speech/asr`) |
| Turn detection | — (separate VAD needed) | `server_vad` / `semantic_vad` | Automatic Activity Detection | Native turn detection (Ink-2) | End-of-Turn (Flux v2) | Semantic VAD (`step` messages) |
| Eager/speculative turn end | — | — | — | `turn.eager_end` | `EagerEndOfTurn` | — (use `inactivity_prob` horizons) |
| Manual turn control | — | `turn_detection: null` + manual commit | Manual VAD (`activityStart`/`activityEnd`) | STT Manual (`/stt/websocket` + `finalize`) | — (use v1 streaming + endpointing) | `flush` message |
| Diarization | Diarization (up to 32) | `gpt-4o-transcribe-diarize` (diarized_json) | Structured output (speaker field) | — | `diarize_model` (v1/v2/latest) + `utterances` | — |
| Speaker roles | `detect_speaker_roles` (agent/customer) | `known_speaker_names[]` | — | — | — | — |
| Speaker library matching | `use_speaker_library` | — | — | — | — | — |
| Vocabulary boosting | `keyterms` (up to 1000) | `prompt` (Whisper, 224 tokens) | — | `keyterm` (Flux) | `keyterm` (Nova-3) / `keywords` (legacy, weighted) | — |
| Pronunciation override | Pronunciation Dictionaries | — | — | Pronunciation Dictionaries | — | Pronunciation Dictionaries (rewrite rules) |
| Text-to-Speech | Text to Speech (TTS) | Speech Generation (`/audio/speech`) | Text-to-Speech (Interactions API) | TTS (Sonic) | TTS (`/v1/speak`) | TTS (`/speech/tts`) |
| Multi-speaker TTS | Text to Dialogue (v3 only) | — | Multi-speaker TTS (up to 2, speech_config) | — | — | — |
| Streaming TTS | `/stream`, `/stream-input` (WS) | chunk transfer encoding | `stream: true` | `/tts/bytes`, `/tts/sse`, `/tts/websocket` | `WSS /v1/speak` (continuous) | WebSocket (`/speech/tts`) |
| Buffered TTS | POST (returns audio bytes) | POST (returns audio) | POST (buffered) | `/tts/bytes` (one POST) | POST `/v1/speak` | REST POST (`/post/speech/tts`) |
| Word timestamps | `with-timestamps` (character-level) | `verbose_json` + `timestamp_granularities` | — (structured output timestamps) | `add_timestamps` (word-level) | `words[]` with start/end | `text` messages with `start_s`/`stop_s` |
| Phoneme timestamps | — | — | — | `add_phoneme_timestamps` | — | — |
| Speed control | `voice_settings.speed` (0.7–1.2) | — (via `instructions`) | — (via Director's Notes pacing) | `generation_config.speed` (0.6–1.5) | `speed` (multiplier) | `padding_bonus` (-4 to 4) |
| Volume control | — | — | — | `generation_config.volume` (0.5–2.0) | — | — |
| Emotion control | Audio tags `[laughs]` + `style` | `instructions` (emotion range) | Audio tags + Director's Notes | `generation_config.emotion` (60+ values) | — | — |
| Stability/similarity | `stability`, `similarity_boost`, `style` | — | — | — | — | `cfg_coef` (voice similarity) |
| Temperature | `seed` (determinism) | — | `temperature` | — | — | `temp` (0.0–1.4) |
| Inline audio directives | Audio Tags `[laughs]` | — | Audio Tags `[whispers]` | SSML Tags `<speed>`, `<emotion>` | — | In-text tags `<flush>`, `<break>` |
| Context stitching | `previous_text`/`next_text`/`request_ids` | — | — | Contexts + continuations (`continue` flag) | — | — |
| Determinism | `seed` (0–4294967295) | — | — | — | — | `temp: 0.0` |
| Text normalization | `apply_text_normalization` | — (via prompt) | — (model handles) | — | `smart_format`, `numerals`, `dictation` | `rewrite_rules` |
| Voice changer (no translation) | Voice Changer (`/speech-to-speech`) | — | — | Voice Changer (`/voice-changer`) | — | — |
| Noise removal | Voice Isolator (`/audio-isolation`) | — | — | — | — | — |
| Sound effects | Sound Effects (`/text-to-sound-effect`) | — | — | — | — | — |
| Music generation | Music (`/music/compose`) | — | — | — | — | — |
| Dubbing (batch translation) | Dubbing (`/dubbing`) | Audio Translation (`/audio/translations`, English only) | — | — | — | — |
| Live translation | — (not realtime) | Realtime Translation (`/realtime/translations`) | Live Translation (`gemini-3.5-live-translate`) | — | — | S2S Live Translation (`/speech/s2s`) |
| Translating transcription | — | — | — | — | — | `stt-translate` model |
| Forced alignment | Forced Alignment (`/forced-alignment`) | — | — | — | — | — |
| Conversational agent | Speech Engine (BYO LLM) | Realtime Voice Agent (`/realtime`) | Live API (`BidiGenerateContent`) | Line (managed platform) | Voice Agent API (`/agent/converse`) | — (integrates with LiveKit/Pipecat) |
| Function calling | Via custom LLM (Speech Engine) | `tools` + `tool_choice` | `tools` (function declarations) | Tools (loopback/passthrough/handoff) | `think.functions[]` | — |
| Async function calling | — | — | `NON_BLOCKING` + `scheduling` (Gemini 2.5) | — | — | — |
| Image input in voice | — | `input_image` content part (gpt-realtime-2+) | Video/image frames via `send_realtime_input` | — | — | — |
| Video input | — | — | `send_realtime_input(video=...)` (max 1 FPS) | — | — | — |
| Session resumption | — | — | `sessionResumption` (handle, 2h validity) | — | — | — |
| Context compression | — | — | `contextWindowCompression` (sliding window) | — | — | — |
| Proactive audio | — | — | `proactive_audio` (model decides not to respond) | — | — | — |
| Affective dialog | — | — | `enable_affective_dialog` (emotion-adaptive) | — | — | — |
| Thinking control | — | `reasoning.effort` (low/medium/high) | `thinkingLevel` / `thinkingBudget` | — | `reasoning_mode` (Think provider) | — |
| Multiplexing | Multi-context WebSocket (`multi-stream-input`) | — | — | Multiple `context_id` on one WS | — | `client_req_id` multiplexing |
| Stem separation | Music Stem Separation | — | — | — | — | — |
| Audio bridging/infill | — | — | — | Infill (`/infill/bytes`) | — | — |
| Webhooks | Webhooks (STT, Dubbing, Agents) | — | — | Webhooks (Line call events) | `callback` URL parameter | — |
| Zero retention | `enable_logging=false` (Enterprise) | — | — | Zero Data Retention (Enterprise) | `mip_opt_out` | — |
| Data residency | US, EU, India, Singapore servers | — | — | — | — | Europe, US servers |
| API versioning | — (no explicit versioning) | — (beta→GA migration) | `v1beta` / `v1alpha` | `Cartesia-Version` header | — | — |
| Latency optimization | `optimize_streaming_latency` (0–4) | — | `thinkingLevel: minimal` | `max_buffer_delay_ms: 0` | — | `delay_in_frames` (0–80) |
| Output formats | mp3, pcm, wav, opus, μ-law, a-law | mp3, opus, aac, flac, wav, pcm | raw PCM (24kHz) | raw, wav, mp3 (pcm_s16le/f32le/mulaw/alaw) | mp3, linear16, flac, mulaw, alaw, opus, aac | wav, pcm, opus, ulaw_8000, alaw_8000 |

---

# Part II — The Exhaustive Processing Pipeline

This section orders **every** capability into a single end-to-end pipeline. The pipeline represents the full lifecycle of a voice AI system, from setup through delivery. Each stage notes the **alternative approaches** available and the **synonyms** used across providers.

```
Stage 0:  Authentication & Access Control
Stage 1:  Voice Asset Management
Stage 2:  Audio Input Preprocessing
Stage 3:  Speech-to-Text / Audio Understanding
Stage 4:  Translation & Dubbing
Stage 5:  Text-to-Speech Generation
Stage 6:  Voice Transformation (Changer / Isolator / Infill)
Stage 7:  Sound Effects & Music Generation
Stage 8:  Conversational Voice Agent Orchestration
Stage 9:  Output Formatting & Delivery
Stage 10: Observability, Management & Billing
```

---

## Stage 0 — Authentication & Access Control

### 0.1 API Key Authentication
Every request is authenticated with an API key. The header name and format differ by provider (see glossary §3).

**Key management capabilities:**
- **Scoped keys** — restrict which endpoints a key can access (ElevenLabs, Deepgram).
- **Credit quotas** — per-key credit limits (ElevenLabs).
- **IP allowlisting** — restrict to specific IPs/CIDR ranges (ElevenLabs).
- **Admin keys** — separate admin keys for management endpoints (Cartesia `sk_car_admin_`).
- **Project-scoped keys** — keys tied to a project (Deepgram).

### 0.2 Ephemeral / Client-Side Tokens
For browser/mobile clients that must not expose the API key, the server generates a short-lived, single-use token. Synonyms: *Single Use Token* (ElevenLabs), *Ephemeral Client Secret* (OpenAI), *Ephemeral Token* (Google), *Access Token / JWT* (Cartesia), *Temporary API Token* (Deepgram), *Browser Token* (Gradium).

**Token configuration options (Google — most advanced):**
- `uses` — number of sessions the token can start (0 = unlimited).
- `expire_time` — max 20 hours.
- `new_session_expire_time` — time after which new sessions are rejected.
- `field_mask` — controls which setup fields the token overrides.
- `live_connect_constraints` — lock model and configuration.
- `safety_identifier` — bind a privacy-preserving user ID to the session (OpenAI).

### 0.3 Privacy & Data Retention
- **Zero Retention Mode** — `enable_logging=false` (ElevenLabs, Enterprise only).
- **Zero Data Retention (ZDR)** — Cartesia Enterprise.
- **Model Improvement Program opt-out** — `mip_opt_out=true` (Deepgram).
- **Data residency** — regional server selection (ElevenLabs: US/EU/India/Singapore; Gradium: Europe/US).

### 0.4 API Versioning
- **Dated version header** — `Cartesia-Version: 2026-03-01` (Cartesia).
- **API version in URL** — `v1beta` / `v1alpha` (Google).
- **Beta → GA migration** — header removal, endpoint changes (OpenAI).
- **No explicit versioning** — ElevenLabs, Deepgram, Gradium (use model ID aliases for reproducibility).

---

## Stage 1 — Voice Asset Management

### 1.1 Prebuilt Voice Library
Every provider offers a library of curated voices with metadata (name, gender, language, country, personality/description). Sizes range from 5 voices (Gradium, 5 languages) to 10,000+ (ElevenLabs Voice Library, community-shared).

**Voice selection parameters:**
- `voice_id` / `voice` (name) — identifies the voice.
- Filtering by language, gender, country (Cartesia `/voices` query params).
- Search by name/description/ID (Cartesia `q` parameter).
- Preview audio URLs (Cartesia `expand[]=preview_file_url`).
- "Find similar" voices (ElevenLabs `/voices/find-similar`).

### 1.2 Instant Voice Cloning (IVC)
Create a voice from a short audio sample (seconds to 2 minutes). No training step required.

| Provider | Min sample | Max sample | Format | Training |
|----------|-----------|-----------|--------|----------|
| ElevenLabs | — | <2 min | audio/video | None |
| OpenAI | — | ≤30s | mpeg, wav, ogg, aac, flac, webm, mp4 | None (requires consent recording) |
| Cartesia | — | ≤10s (recommended) | flac, mp3, mpeg, mpga, oga, ogg, wav, webm | Free |
| Gradium | 10s (recommended) | — | WAV, MP3, etc. | None |

**OpenAI consent requirement:** Two separate recordings — (1) consent phrase in one of 16 languages, (2) the actual audio sample (≤30s). Returns a `consent_id` used to create the voice.

**Best practices (Cartesia):** Choose a script matching target energy, speak clearly, avoid background noise, trim long pauses, speak in the target language.

### 1.3 Professional Voice Cloning (PVC)
Fine-tune a model on extended audio data for highest fidelity.

- **ElevenLabs PVC** — create → add samples → train → verify captcha (ensures clone is from your own voice). Supports speaker separation on samples. Verification via voice-captcha.
- **Cartesia Pro Voice Clone** — `/fine-tunes/create`, costs 1,000,000 credits per fine-tune, ~1.5 credits/char for TTS (50% more than standard). Supports datasets API for file management.
- **OpenAI** — no separate PVC; custom voice is instant from ≤30s.

### 1.4 Voice Design from Text Description
Generate a voice from a natural-language description (20–1000 characters). **ElevenLabs only** (`/text-to-voice/design`). Returns 3 preview voices; create from a preview; remix existing voices with attribute transformations (gender, accent, speaking style, pacing, audio quality). `prompt_strength`: low/medium/high/max.

### 1.5 Voice Remixing
Transform an existing voice's attributes via natural language while maintaining recognizable characteristics. **ElevenLabs only** (`/text-to-voice/remix`). Iteratively remixable. Output is backward-compatible with older models.

### 1.6 Voice Localization
Adapt an existing voice to speak a new language/dialect. **Cartesia only** (`/voices/localize`). Supports 20 target languages with dialect options (e.g., English: Australian, Indian, Southern, British, American; Spanish: Latin American, Peninsular; Portuguese: Brazilian, European; French: Standard, Canadian).

### 1.7 Voice Metadata Management
- **List/search voices** — pagination, filtering by language/gender/ownership.
- **Get voice details** — metadata, settings, preview.
- **Update voice** — name, description, gender, settings.
- **Delete voice**.
- **Default voice settings** — get/set defaults (ElevenLabs).
- **Share to library** — make a voice public (ElevenLabs).

### 1.8 Pronunciation Dictionaries
Versioned sets of text-to-pronunciation mappings. Supported by ElevenLabs, Cartesia, and Gradium.

| Feature | ElevenLabs | Cartesia | Gradium |
|---------|-----------|----------|---------|
| Creation | from-file, from-rules | POST with items | POST with rewrite rules |
| Versioning | Yes (version_id) | No explicit | No explicit |
| Attachment | `pronunciation_dictionary_locators` (up to 3) | `pronunciation_dict_id` | `pronunciation_id` |
| Per-request | Yes | Yes | Yes |
| Access control | — | private/public | — |
| Management API | Full CRUD | Full CRUD | Full CRUD |
| Item format | phonetic rules | `{text, pronunciation}` | language-specific rewrite rules |

### 1.9 Datasets & Fine-Tuning (Music)
- **Cartesia** — datasets API for organizing fine-tune training files.
- **ElevenLabs** — music model fine-tunes on your own non-copyrighted tracks (5–10 min, copyright screening, captures instrumentation/tempo/production style). Curated finetunes available (Afro House, Reggaeton, Arabic Groove, etc.).

---

## Stage 2 — Audio Input Preprocessing

### 2.1 Audio Format Detection & Encoding
Input audio can arrive in many formats. The system must detect or be told the encoding:

| Category | Examples |
|----------|---------|
| **Compressed containers** | MP3, AAC, OGG, OPUS, M4A, WebM, FLAC |
| **Uncompressed** | WAV, AIFF, raw PCM (16-bit, 32-bit float, signed LE) |
| **Telephony** | μ-law (US), A-law (Europe), G.729, AMR-narrowband, AMR-wideband |
| **Video** | MP4, AVI, MKV, MOV, WMV, FLV, MPEG, 3GPP |
| **Raw PCM** | requires `encoding` + `sample_rate` parameters |

### 2.2 Sample Rate Configuration
- 8 kHz — telephony (μ-law/A-law)
- 16 kHz — low-latency streaming, speech recognition input
- 22.05 kHz — standard quality
- 24 kHz — voice applications, Gemini Live input
- 32 kHz — medium quality
- 44.1 kHz — CD quality
- 48 kHz — professional audio, Gemini Live output, Gradium TTS output

### 2.3 Multichannel Handling
- **Independent channel transcription** — each channel transcribed separately (ElevenLabs `use_multi_channel` up to 5; Deepgram `multichannel`).
- **Output style** — `separate` (one transcript per channel) or `combined` (merged by time) — ElevenLabs.
- **Channel count** — specified in streaming setup (Deepgram `channels`).

### 2.4 Voice Isolation / Noise Removal (Pre-processing)
Extract clean speech from audio with background noise, music, or ambient sounds. Produces studio-quality isolated speech.

- **ElevenLabs Voice Isolator** — `/audio-isolation` (buffered + streamed). Max 500 MB, 1 hour. Not specifically optimized for vocal isolation from music.
- **Voice changer option** — `remove_background_noise` boolean (ElevenLabs STS).
- **Speaker separation** — ElevenLabs PVC sample speaker separation; ElevenLabs Music stem separation.

---

## Stage 3 — Speech-to-Text / Audio Understanding

This stage encompasses all capabilities that derive meaning from audio.

### 3.1 Batch Transcription (File-Based)
Transcribe a complete recorded file. Input via file upload (multipart) or URL (including YouTube, TikTok, hosted files).

**Synonyms:** Batch STT (ElevenLabs, Cartesia), Pre-Recorded STT (Deepgram), File transcription (OpenAI), Audio Understanding (Google).

| Constraint | ElevenLabs | OpenAI | Deepgram | Cartesia | Gradium |
|-----------|-----------|--------|----------|----------|---------|
| Max file size | 3 GB | 25 MB | 2 GB | — | — |
| Max duration | 10 hours | — | 10 min (Nova) / 20 min (Whisper) | — | — |
| URL input | Yes (YouTube, TikTok) | No | Yes | No | No |
| Video input | Yes (MP4, AVI, etc.) | Yes (mp4) | — | No | No |

### 3.2 Real-Time Streaming Transcription
Transcribe live audio as it arrives. WebSocket-based.

**Synonyms:** Realtime STT (ElevenLabs), Realtime Transcription (OpenAI), Live Streaming STT (Deepgram), STT WebSocket (Cartesia, Gradium).

**Key features:**
- Interim results (may evolve as more audio arrives) vs final results (committed text).
- `is_final` vs `speech_final` distinction (Deepgram) — Deepgram-level finality vs natural speech pause.
- Voice Activity Detection (VAD) events — `SpeechStarted` (Deepgram), `speech_started`/`speech_stopped` (OpenAI).
- KeepAlive messages — pause connection without closing (Deepgram).
- Manual commit — client explicitly commits audio buffer (OpenAI, Cartesia manual mode).

### 3.3 Language Detection & Hinting
- **Smart language detection** — auto-detect with confidence score (ElevenLabs `language_probability`).
- **Language hint** — ISO 639-1 or BCP-47 code to guide recognition (all providers).
- **Code-switching / multilingual** — `multi` value for Nova-3/Nova-2 (Deepgram), `flux-general-multi` (Deepgram Flux, 10 languages), `language_hints` array (Flux).
- **Auto-detect with candidate list** — Deepgram `detect_language` can accept an array of candidate languages.

### 3.4 Diarization (Speaker Identification)
- **ElevenLabs** — up to 32 speakers, `diarize` + `diarization_threshold`, `num_speakers` hint, `detect_speaker_roles` (agent/customer, +10% surcharge), `use_speaker_library` (match against registered speakers).
- **OpenAI** — `gpt-4o-transcribe-diarize` model, `diarized_json` format, `known_speaker_names[]` (up to 4), `known_speaker_references[]` (2–10s audio data URLs).
- **Google** — via structured output schema with `speaker` field.
- **Deepgram** — `diarize_model` (latest/v1/v2), `utterances=true` + `utt_split` (pause threshold).

### 3.5 Vocabulary Boosting / Keyterm Prompting
Bias transcription toward specific terms.

| Provider | Parameter | Limit | Format | Surcharge |
|----------|-----------|-------|--------|-----------|
| ElevenLabs | `keyterms` | 1000 (batch), 50 (realtime) | Plain terms, ≤50 chars | +20% |
| OpenAI | `prompt` | 224 tokens (Whisper) | Context text | — |
| Deepgram | `keyterm` (Nova-3) | — | Plain terms | — |
| Deepgram | `keywords` (legacy) | — | `term:boost` (0–100) | — |
| Deepgram | `search` | — | Search for terms, returns hits | — |
| Cartesia | `keyterm` (Flux) | — | Plain terms | — |

### 3.6 Text Normalization & Formatting
- **Smart format** — currency, phone numbers, email addresses, dates (Deepgram `smart_format`).
- **Punctuation & capitalization** — Deepgram `punctuate`.
- **Paragraph splitting** — Deepgram `paragraphs`.
- **Numerals** — convert written numbers to numerical format (Deepgram `numerals`).
- **Dictation mode** — format dictated speech (Deepgram `dictation`).
- **Measurements** — convert spoken measurements to abbreviations (Deepgram `measurements`).
- **Profanity filter** — Deepgram `profanity_filter`.
- **Filler words** — include/exclude "uh", "um" (Deepgram `filler_words`).
- **No verbatim mode** — remove filler words, false starts, disfluencies (ElevenLabs `no_verbatim`).
- **Text normalization** — `apply_text_normalization` auto/on/off (ElevenLabs), `apply_language_text_normalization` (Japanese-specific).
- **Rewrite rules** — language-specific text rewriting before synthesis (Gradium `rewrite_rules`).
- **Search & replace** — Deepgram `search` + `replace`.

### 3.7 Entity Detection & Redaction
- **Entity detection** — PII, PHI, PCI, offensive language, person names, orgs, dates (ElevenLabs `entity_detection`, Deepgram `detect_entities`).
- **Entity redaction** — modes: `redacted`, `entity_type`, `enumerated_entity_type` (ElevenLabs).
- **Redaction** — `pci`, `pii`, `numbers`, `ssn`, `aggressive_numbers` (Deepgram `redact`).
- **Surcharges** — entity detection +30%, entity redaction +30% (ElevenLabs).

### 3.8 Timestamps
- **Word-level** — start/end per word (all providers).
- **Character-level** — ElevenLabs `timestamps_granularity: character`.
- **Phoneme-level** — Cartesia `add_phoneme_timestamps`.
- **Segment-level** — OpenAI `timestamp_granularities: segment` (Whisper).
- **MM:SS format** — Google (prompt-based referencing).

### 3.9 Audio Event Tagging
Tag non-speech sounds (laughter, applause, footsteps). ElevenLabs `tag_audio_events` (default true). Word types: `word`, `spacing`, `audio_event`.

### 3.10 Audio Intelligence (Post-Transcription Analysis)
Available as STT add-ons or separate Text Intelligence API.

| Feature | Parameter | Description |
|---------|-----------|-------------|
| **Summarization** | `summarize` | Concise summary (`true`/`false`/`"v2"`) |
| **Sentiment** | `sentiment` | positive/negative/neutral with score per segment + average |
| **Topics** | `topics` | Auto-detected + `custom_topic` (up to 100), `custom_topic_mode: strict/extended` |
| **Intents** | `intents` | Auto-detected + `custom_intent`, `custom_intent_mode: strict/extended` |
| **Entities** | `detect_entities` | NAME, PHONE_NUMBER, EMAIL_ADDRESS, ORGANIZATION, CARDINAL, etc. |
| **Emotion** | structured output | happy/sad/angry/neutral per segment (Google) |

**Deepgram Read API** (`/v1/read`) — same intelligence features applied to text content (not just audio).

### 3.11 Forced Alignment
Align a text transcript to audio to produce exact word/phrase timestamps. **ElevenLabs only** (`/forced-alignment`). Input: audio file + plain string transcript. Not supported with diarized text. Max 3 GB, 10 hours, 675,000 chars. 29 languages.

### 3.12 Turn Detection (VAD) — STT Layer
How the system decides a user has finished speaking.

**Alternative approaches:**

| Approach | Providers | How it works |
|----------|-----------|-------------|
| **Silence-based VAD** | OpenAI (`server_vad`), Google (automatic VAD), Deepgram v1 (`endpointing`) | Detects pauses in audio. Configurable threshold, padding, silence duration. |
| **Semantic VAD** | OpenAI (`semantic_vad`), Gradium (`step` messages) | Classifier scores probability user is done based on words uttered. `eagerness` controls how quick to interrupt. |
| **Model-native turn detection** | Deepgram (Flux v2), Cartesia (Ink-2) | The STT model itself emits turn lifecycle events. |
| **Manual / push-to-talk** | OpenAI (`turn_detection: null`), Google (manual VAD), Cartesia (`/stt/websocket` + `finalize`) | Client explicitly commits audio and triggers response. |

**Turn lifecycle events (Deepgram Flux / Cartesia Ink-2):**
- `StartOfTurn` / `turn.start` — user begins speaking.
- `Update` / `turn.update` — additional audio transcribed (cumulative, not delta).
- `EagerEndOfTurn` / `turn.eager_end` — moderate confidence user finished (start preparing reply early).
- `TurnResumed` / `turn.resume` — speech continued after eager end (false alarm, cancel preparation).
- `EndOfTurn` / `turn.end` — definitive turn end.

**Gradium semantic VAD `step` messages** — every 80ms, contains `vad` array with horizon predictions (`horizon_s`, `inactivity_prob`). Client decides threshold to flush.

**Adaptive delay** — `delay_in_frames` (Gradium, each frame = 80ms, range 0–80). Lower = faster/more reactive; higher = better quality. OpenAI `delay` setting: minimal/low/medium/high/xhigh.

**Turn detection configuration parameters:**
- `threshold` — activation threshold 0–1 (OpenAI server_vad).
- `prefix_padding_ms` — audio before speech start (OpenAI 300, Google 20).
- `silence_duration_ms` — silence to detect stop (OpenAI 500, Google 100).
- `eagerness` — auto/low/medium/high (OpenAI semantic_vad).
- `start_of_speech_sensitivity` / `end_of_speech_sensitivity` — HIGH/LOW (Google).
- `turn_start_threshold` / `turn_eager_end_threshold` / `turn_end_threshold` / `turn_end_timeout_ms` (Cartesia, must be ordered: start > eager_end > end).
- `eot_threshold` / `eager_eot_threshold` / `eot_timeout_ms` (Deepgram Flux).
- `activity_handling` — START_OF_ACTIVITY_INTERRUPTS / NO_INTERRUPTION (Google).
- `turn_coverage` — ONLY_ACTIVITY / ALL_INPUT / AUDIO_ACTIVITY_AND_ALL_VIDEO (Google).

### 3.13 Export Formats
Transcript export beyond JSON:
- **SRT** — subtitle format with configurable `max_characters_per_line`, `segment_on_silence_longer_than_s`, `max_segment_duration_s`, `max_segment_chars`, `include_speakers` (ElevenLabs).
- **TXT** — plain text with optional speakers/timestamps.
- **DOCX** — Word document.
- **HTML** — web format.
- **PDF** — document format.
- **Segmented JSON** — segmented structure.
- **verbose_json** — detailed with timestamps (OpenAI Whisper).
- **VTT** — WebVTT subtitles (OpenAI Whisper).
- **diarized_json** — speaker-segmented JSON (OpenAI).
- **NDJSON** — newline-delimited JSON streaming (Gradium REST).

### 3.14 Webhooks / Async Callbacks
- **ElevenLabs** — `webhook=true` in STT request returns early with `request_id`/`transcription_id`, results delivered to configured webhooks. `webhook_id` for specific webhook, `webhook_metadata` for correlation (max 16KB).
- **Deepgram** — `callback` URL parameter on STT/TTS, `callback_method` POST/PUT.
- **Cartesia Line** — webhook endpoints for call events.

---

## Stage 4 — Translation & Dubbing

### 4.1 File-Based Audio Translation
- **OpenAI** — `/audio/translations` with `whisper-1` only. Output is always **English text**. Same 25 MB limit as transcriptions.
- **Deepgram** — no native audio translation.
- **ElevenLabs** — no file-based translation (uses Dubbing).

### 4.2 Translating Transcription (STT with translation output)
- **Gradium** — `stt-translate` model. Set `language`/`target_language` for output language. Input audio in any supported language → transcript in target language. 5 languages (en, fr, de, es, pt).

### 4.3 Batch Dubbing (Audio/Video Translation)
**ElevenLabs only** (`/dubbing`). Translates audio/video across 90+ languages while preserving emotion, timing, tone, and speaker characteristics.

**Features:**
- Speaker separation — auto-detects multiple speakers, even overlapping speech.
- Preserve original voices — retains speaker identity and emotional tone.
- Keep background audio — avoids re-mixing music, effects, ambient sounds.
- Cloning strength — configurable (default 7). Higher = more voice similarity, less natural cross-language. Lower = more natural, less resemblance.
- Source inputs — YouTube, X, TikTok, Vimeo, direct URLs, file uploads.
- No realtime dubbing.
- Watermark on free tier; none on paid.

**Limits:** 2 GB / 180 min (v2); 1 GB / 45 min (V1 Studio). Up to 9 speakers. 5 concurrent (self-serve), 100 (Enterprise).

### 4.4 Live Speech-to-Speech Translation (Real-Time)
Real-time translation of spoken audio into another language, producing translated audio (and optionally transcripts).

| Provider | Model | Languages | Architecture |
|----------|-------|-----------|-------------|
| OpenAI | `gpt-realtime-translate` | — | Continuous WebSocket session, no turn lifecycle |
| Google | `gemini-3.5-live-translate-preview` | 70+ | Live API WebSocket, continuous stream |
| Gradium | `s2s-translate` | 5 (en, fr, de, es, pt) | Duplex WebSocket, STT-translate + TTS-default pipeline |

**OpenAI translation sessions:**
- No `response.create` — don't wait for turn commit; keep appending audio.
- No conversation lifecycle — model acts as interpreter, not assistant.
- Events: `session.output_audio.delta` (translated audio), `session.output_transcript.delta` (translated text), `session.input_transcript.delta` (source text).
- One session per target language. `session.close` to flush and close.
- Architecture patterns: listen-along (one source, many listeners) or conversational (two participants, one session per direction).

**Google Live Translation:**
- `translationConfig: {targetLanguageCode, echoTargetLanguage}`.
- Audio-only input (no video/text) for strict real-time latency.
- No tools or instructions — translation only.
- `echoTargetLanguage: true` echoes target language audio back.
- Ephemeral tokens with `live_connect_constraints` to lock translation config.

**Gradium S2S:**
- Pipeline: input audio → STT (`stt-translate`) → translated text → TTS (`default`) → output audio.
- `voice_id` must match `target_language`.
- Input: 24 kHz PCM; Output: 48 kHz PCM.
- Omit `target_language` for transcription + re-synthesis without translation.

---

## Stage 5 — Text-to-Speech Generation

### 5.1 Single-Speaker TTS
Convert text to spoken audio with one voice. All providers support this.

**Request parameters (union of all providers):**
- `text` / `transcript` / `input` — the text to synthesize.
- `model` / `model_id` — the TTS model.
- `voice` / `voice_id` — the voice identifier.
- `language` / `language_code` — enforce a language.
- `output_format` — codec, sample rate, bitrate, container.
- `voice_settings` / `generation_config` / `json_config` / `instructions` — voice behavior controls.
- `pronunciation_dictionary_locators` / `pronunciation_dict_id` / `pronunciation_id` — pronunciation overrides.
- `seed` — deterministic sampling (ElevenLabs 0–4294967295; Gradium `temp: 0.0`).
- `previous_text` / `next_text` / `previous_request_ids` / `next_request_ids` — context stitching (ElevenLabs).
- `apply_text_normalization` — auto/on/off (ElevenLabs).
- `apply_language_text_normalization` — Japanese-specific (ElevenLabs).
- `optimize_streaming_latency` — 0–4 (ElevenLabs).
- `enable_logging` — false for zero retention (ElevenLabs Enterprise).
- `speed` — speaking rate multiplier (Deepgram, Cartesia via generation_config).
- `stream` — boolean to enable streaming (Google, OpenAI file-based).

### 5.2 Multi-Speaker TTS / Dialogue
Generate audio with multiple speakers from text.

| Provider | Mechanism | Max speakers | Model |
|----------|-----------|-------------|-------|
| ElevenLabs | Text to Dialogue (`inputs[]` with text + voice_id per turn) | Unlimited | `eleven_v3` only |
| Google | Multi-speaker TTS (`speech_config` with speaker + voice pairs) | 2 | `gemini-3.1-flash-tts-preview` |

**ElevenLabs Text to Dialogue constraints:** ≤2,000 chars total, `eleven_v3` only, supports audio tags per turn, punctuation for flow (interruptions via `"Hello, is this seat-"`, trailing ellipsis). Up to 2 free regenerations (dashboard only).

**Google multi-speaker:** Speaker names in `speech_config` must match the prompt transcript. Director-style prompting with Audio Profile + Scene + Director's Notes + Transcript.

### 5.3 Voice Behavior Control
The union of all voice control mechanisms:

| Control | ElevenLabs | OpenAI | Google | Cartesia | Gradium |
|---------|-----------|--------|--------|----------|---------|
| Stability | `stability` (0–1) | — | — | — | — |
| Similarity | `similarity_boost` (0–1) | — | — | — | `cfg_coef` (1–4) |
| Style | `style` (0–1) | — | — | — | — |
| Speaker boost | `use_speaker_boost` | — | — | — | — |
| Speed | `speed` (0.7–1.2) | via `instructions` | via Director's Notes | `generation_config.speed` (0.6–1.5) | `padding_bonus` (-4 to 4) |
| Volume | — | — | — | `generation_config.volume` (0.5–2.0) | — |
| Emotion | Audio tags + `style` | `instructions` | Audio tags + Director's Notes | `generation_config.emotion` (60+ values) | — |
| Temperature | `seed` | — | `temperature` | — | `temp` (0.0–1.4) |
| Instructions | — | `instructions` (accent, emotion, intonation, speed, tone, whispering) | Director's Notes (style, accent, pacing) | — | — |
| Accent | — | via `instructions` | Director's Notes accent field | — | — |
| Whispering | Audio tags `[whispering]` | via `instructions` | Audio tags `[whispers]` | — | — |

### 5.4 Inline Audio Directives (Tags)
Three different syntaxes for inline control:

**Audio Tags (ElevenLabs v3, Google):** Square brackets within text.
- Emotions: `[sad]`, `[laughing]`, `[whispering]`, `[cautiously]`, `[elated]`, `[excitedly]`, `[bored]`, `[amazed]`, `[curious]`, `[crying]`, `[panicked]`, `[sarcastic]`, `[serious]`
- Delivery: `[whispers]`, `[shouting]`, `[tired]`, `[trembling]`, `[reluctantly]`, `[mischievously]`
- Non-verbal: `[laughs]`, `[giggles]`, `[sighs]`, `[gasp]`, `[cough]`
- Pace: `[very fast]`, `[very slow]`
- Audio events: `[leaves rustling]`, `[gentle footsteps]`, `[applause]`
- Overall direction: `[football]`, `[wrestling match]`, `[auctioneer]`
- Creative: `[like a cartoon dog]`, `[like dracula]`
- Use English tags even for non-English transcripts (Google).

**SSML Tags (Cartesia):** XML-like tags within transcript.
- `<speed ratio="1.5"/>` — speed 0.6–1.5
- `<volume ratio="0.5"/>` — volume 0.5–2.0
- `<emotion value="angry"/>` — emotion (beta)
- `<break time="1s"/>` — explicit pause
- `<spell>ABC123</spell>` — spell out characters
- Nonverbalism: `[laughter]` in transcript to make model laugh.

**In-Text Tags (Gradium):**
- `<flush>` — force audio emission for all buffered text (don't flush after every token; small fragments reduce prosody).
- `<break time="Ns"/>` — pause 0.1–2.0s (must be preceded/followed by space).

### 5.5 Context Stitching & Continuations
Maintain prosody continuity across chunked generations.

- **ElevenLabs** — `previous_text`/`next_text` (strings) or `previous_request_ids`/`next_request_ids` (up to 3 each). For content longer than per-request char limits.
- **Cartesia** — WebSocket contexts + continuations. `context_id` identifies a context; `continue: true` on partial chunks, `continue: false` on final. All fields except `transcript`, `continue`, `duration` must be identical across inputs. Transcripts concatenated verbatim (include spaces at boundaries). Contexts auto-expire 1s after last audio output.
- **Gradium** — WebSocket text streaming; server inserts single whitespace between consecutive text messages. Never split inside a word.

### 5.6 Buffering Control
- **Managed buffering** (Cartesia `max_buffer_delay_ms` >0, default 3000) — server decides when to start generating. Stream LLM tokens directly.
- **Custom buffering** (`max_buffer_delay_ms: 0`) — server generates immediately; you aggregate sentences.
- **Rule:** Don't aggregate client-side AND use default `max_buffer_delay_ms` — adds unnecessary latency.

### 5.7 Multiplexing
Multiple independent requests on a single WebSocket connection.
- **ElevenLabs** — multi-context WebSocket (`multi-stream-input`). Multiple conversation contexts, only generation time counts toward concurrency.
- **Cartesia** — multiple `context_id` values active simultaneously; audio tagged with `context_id`. Only unique active contexts count toward concurrency.
- **Gradium** — `close_ws_on_eos: false` + unique `client_req_id` on every message. Server stamps all responses with matching `client_req_id`.

### 5.8 Timestamps in TTS Output
- **ElevenLabs** — `with-timestamps` endpoints (character-level timestamps).
- **Cartesia** — `add_timestamps` (word-level), `add_phoneme_timestamps` (phoneme-level), `use_normalized_timestamps`. Available on SSE and WebSocket.
- **Gradium** — `text` messages with `start_s`/`stop_s` per text segment (word-level).
- **Deepgram** — metadata headers: `dg-char-count`, `dg-request-id`.

### 5.9 Determinism
- **ElevenLabs** — `seed` (0–4294967295). Determinism not guaranteed but improves consistency. Up to 2 free regenerations with identical params.
- **Gradium** — `temp: 0.0` for deterministic generation.
- **Cartesia** — pinned model IDs (`sonic-3.5-2026-05-04`) for reproducibility.

### 5.10 Free Regenerations
- **ElevenLabs** — up to 2 free regenerations per generation when identical text and parameters. Dashboard-only for Text to Dialogue.

---

## Stage 6 — Voice Transformation

### 6.1 Voice Changer (Speech-to-Speech, No Translation)
Transform source audio into a different voice while preserving emotional delivery, nuances, whispers, laughs, and accents.

| Provider | Endpoint | Billing | Max segment | Key params |
|----------|----------|---------|-------------|------------|
| ElevenLabs | `/speech-to-speech/{voice_id}` + `/stream` | 1,000 chars/min | 5 min | `model_id`, `remove_background_noise`, `audio` |
| Cartesia | `/voice-changer/bytes` + `/sse` | 15 credits/sec | — | `clip`, `voice[id]`, `output_format` |

### 6.2 Voice Isolator (Audio Isolation / Noise Removal)
Extract clean speech from noisy audio. **ElevenLabs only** (`/audio-isolation` + `/stream` + history management). Max 500 MB, 1 hour. Not optimized for vocal isolation from music.

### 6.3 Audio Infill / Bridging
Generate audio to bridge between two existing audio clips: `left_audio` → `transcript` → `right_audio`. **Cartesia only** (`/infill/bytes`). At least one of left/right must be provided. Longer transcripts give more adaptation flexibility. Cost: 300 credits + ~1 credit/char.

### 6.4 Stem Separation
Separate a song into individual instrument/vocal stems. **ElevenLabs only** (`/music/separate-stems`).

---

## Stage 7 — Sound Effects & Music Generation

### 7.1 Sound Effects (Text-to-Sound)
Generate audio effects from text descriptions. **ElevenLabs only** (`/text-to-sound-effect`, model `eleven_text_to_sound_v2`).

**Parameters:**
- `text` — description of the sound.
- `duration_seconds` — 0.1–30s (auto if not specified; 40 credits/sec when specified).
- `prompt_influence` — high = literal, low = creative variations.
- `loop` — seamless looping for effects >30s.

**Prompting categories:** Simple effects, complex sequences, musical elements (drum loops, bass lines), audio terminology (impact, whoosh, ambience, one-shot, loop, stem, braam, glitch, drone).

**Output:** MP3 (all effects), WAV at 48 kHz (non-looping).

### 7.2 Music Generation
**ElevenLabs only** (`/music/...`). Models: `music_v2` (next-gen), `music_v1` (previous).

**Endpoints:**
- `/music/compose` — simple prompt
- `/music/compose/stream` — stream music
- `/music/compose-detailed` — detailed parameters
- `/music/compose-detailed/stream` — stream with details
- `/music/create-composition-plan` — structured JSON for precise control (sections, genres, lyrics, transitions)
- `/music/video-to-music` — video to music
- `/music/upload` — upload music file
- `/music/separate-stems` — stem separation

**Key features:**
- Prompt — natural language (genre, style, mood, instruments).
- Composition Plan — structured JSON for precise control.
- Inpainting — edit and combine sections of existing songs.
- Stem separation — separate into instrument/vocal stems.
- Finetunes — fine-tune on your own audio (captures instrumentation, tempo, production style, timbre, vocal character).
- Multilingual — English, Spanish, German, Japanese, more.
- Vocals or instrumental control.
- Min 3s, max 5 min. MP3 (44.1kHz, 128–192kbps), WAV.
- Commercial use cleared for nearly all uses.
- Mid-track genre transitions, fast rap, complex vocals (v2).

---

## Stage 8 — Conversational Voice Agent Orchestration

### 8.1 Architecture Choices

| Architecture | Best for | Providers |
|-------------|----------|-----------|
| **End-to-end voice agent** (single session, model handles STT+reasoning+TTS) | Natural, low-latency conversations | OpenAI Realtime, Google Live API, Deepgram Voice Agent |
| **BYO LLM** (platform handles STT+TTS, you provide reasoning) | Extending existing text agents, custom logic | ElevenLabs Speech Engine, Cartesia Line |
| **Chained pipeline** (explicit STT → text reasoning → TTS) | Predictable workflows | OpenAI (Python VoicePipeline), any provider combination |
| **Managed platform** (deployment, scaling, telephony handled) | Fastest time to production | Cartesia Line |

### 8.2 Connection Methods

| Transport | Use when | Providers |
|-----------|----------|-----------|
| **WebSocket** | Server receives raw audio from media pipeline, call system, or worker | All |
| **WebRTC** | Browser/mobile clients that capture/play audio directly | OpenAI, Google (partner integrations: LiveKit, Pipecat, Fishjam, Agora, Voximplant) |
| **SIP** | Telephony voice agents | OpenAI |

### 8.3 Session Configuration
The initial setup message/configures the session for its duration.

**Unified session configuration fields:**
- `model` — the realtime/conversational model.
- `voice` — output voice (name or `{id}`).
- `instructions` / `systemInstruction` / `prompt` — system instructions.
- `output_modalities` / `responseModalities` — `["audio"]`, `["text"]`, or both.
- `audio.input.format` — input encoding (PCM 24kHz for OpenAI, 16kHz for Google).
- `audio.output.format` — output encoding.
- `turn_detection` / `realtimeInputConfig` — VAD configuration or null for manual.
- `tools` — function declarations.
- `tool_choice` — auto/none/specific.
- `temperature` — sampling temperature.
- `thinking` / `reasoning` — reasoning effort control.
- `prompt` — server-stored prompt with `{id, version, variables}` (OpenAI).
- `context` — conversation history seeding.
- `sessionResumption` — resumption handle (Google).
- `contextWindowCompression` — sliding window config (Google).
- `inputAudioTranscription` / `outputAudioTranscription` — enable transcriptions (Google).
- `proactivity` — proactive audio (Google 2.5).
- `enable_affective_dialog` — emotion-adaptive responses (Google 2.5).
- `historyConfig` — initial history exchange config (Google).
- `mediaResolution` — input media resolution (Google).

**OpenAI response-level overrides (`response.create`):**
- `output_modalities`, `instructions`, `input` (custom context), `conversation: "none"` (out-of-band), `metadata`, `tools`, `tool_choice`, `input_audio_format`, `audio.output.format`.

### 8.4 Turn-Taking & VAD in Sessions
See Stage 3.12 for VAD approaches. In conversational sessions:
- `create_response` — whether to auto-create response when turn ends (OpenAI, Google).
- `interrupt_response` — whether user speech interrupts in-progress response (OpenAI, Google).
- VAD can be disabled for push-to-talk (`turn_detection: null`).
- VAD without auto-response — keep VAD but trigger responses manually.

### 8.5 Interruption / Barge-in Handling
- **WebRTC/SIP** — server manages output audio buffer, auto-truncates unplayed audio (OpenAI).
- **WebSocket** — client manages playback, must stop and send `conversation.item.truncate` with `item_id`, `content_index`, `audio_end_ms` (OpenAI).
- **Google** — `serverContent.interrupted` flag; ongoing generation cancelled and discarded; only already-sent content retained in history.
- **Cartesia** — `UserTurnStarted` event interrupts agent; `cancel_filter: [UserTurnStarted]`.
- **Deepgram** — `UserStartedSpeaking` event.

**Transcript truncation caveat (OpenAI):** The model cannot precisely align transcript and audio. Truncation cuts audio and removes text for unplayed portion but does not provide a truncated transcript.

### 8.6 Function Calling in Sessions
**Synchronous (all providers with tools):**
1. Register tools in session config or response.
2. User speaks → model decides to call a function.
3. Model emits function call arguments (streamed).
4. Client/server executes function.
5. Result returned to model.
6. Model responds using the result.

**Asynchronous (Google Gemini 2.5 only):**
- `behavior: "NON_BLOCKING"` — model continues interacting while function executes.
- `scheduling` in FunctionResponse:
  - `INTERRUPT` — immediately interrupt model to deliver response.
  - `WHEN_IDLE` — deliver when model is idle.
  - `SILENT` — use knowledge later, no immediate delivery.

**Execution location:**
- **Client-side** — client executes and returns result (OpenAI `SendFunctionCallResponse`, Deepgram `FunctionCallRequest` with `client_side: true`, Cartesia loopback tool).
- **Server-side** — platform calls HTTP endpoint directly (Deepgram `endpoint` in function definition, Cartesia `http_server_tool`).
- **Tool cancellation** — on interruption, server sends `toolCallCancellation` with IDs (Google); undo side effects if necessary.

**Tool types (Cartesia Line):**
- **Loopback** — result goes back to LLM for continued generation (default).
- **Passthrough** — output goes directly to user, bypassing LLM.
- **Handoff** — transfers control to another handler.

**Built-in tools (Cartesia Line):** `end_call`, `transfer_call` (E.164), `web_search`, `knowledge_base` (with metadata filters, top_k), `send_dtmf`.

**HTTP server tools (Cartesia Line):** Define tools from JSON schemas without writing function code. Parameters: `url` (with `{param}` templating), `method`, `path_params_schema`, `request_body_schema`, `query_params_schema`, `auth` (headers with `${ENV_VAR}`), `content_type`, `is_background`.

**Google Search grounding** — `tools: [{google_search: {}}]`. Model generates/executes Python code to use Search. Results include `groundingMetadata`.

### 8.7 Image & Video Input (Multimodal)
- **OpenAI** — `gpt-realtime-2+` supports `input_image` content part with `image_url` (base64 data URI). Images ride on `conversation.item.create` mechanism.
- **Google** — `send_realtime_input(video=Blob(...))` with `image/jpeg` or `image/png`, max 1 FPS. Turn coverage: `TURN_INCLUDES_AUDIO_ACTIVITY_AND_ALL_VIDEO` (3.1 default) includes all video frames.

### 8.8 Session Management
- **Session resumption (Google)** — resume across WebSocket reconnections via resumption handle. Token valid 2 hours after session termination. `sessionResumptionUpdate.newHandle` — save for next reconnection. `resumable` is false during function execution or generation.
- **Context window compression (Google)** — sliding window discards older context to keep sessions running indefinitely. `triggerTokens` (default 80% of model limit), `targetTokens` (default trigger/2). Context always begins at start of a USER turn. System instructions and prefix turns preserved.
- **GoAway message (Google)** — server sends before connection termination with `timeLeft`.
- **Generation complete vs turn complete (Google)** — `generationComplete` signals model finished; `turnComplete` comes after playback delay.
- **Session duration limits:** 60 min (OpenAI), 15 min audio / 2 min audio+video without compression (Google), ~10 min connection lifetime (Google).

### 8.9 Advanced Session Features
- **Thinking control** — `reasoning.effort: low` recommended for production voice agents (OpenAI). `thinkingLevel: minimal/low/medium/high` (Google 3.1), `thinkingBudget: 0-N` (Google 2.5). `reasoning_mode: none/minimal/low/medium/high` (Deepgram Think provider). `include_thoughts: true` for thought summaries (Google).
- **Proactive audio (Google 2.5)** — model decides not to respond if input is not relevant. `proactivity: {proactive_audio: true}` (v1alpha).
- **Affective dialog (Google 2.5)** — model adapts response style to match input expression and tone. `enable_affective_dialog: true` (v1alpha).
- **Out-of-band responses (OpenAI)** — `response.create` with `conversation: "none"` for responses not added to the default conversation.
- **Server-stored prompts (OpenAI)** — `prompt: {id, version, variables}`. Direct session fields override prompt fields if they overlap.
- **Latency metrics (Deepgram)** — `total_latency`, `tts_latency`, `ttt_latency` (text-to-text/LLM) on `AgentStartedSpeaking`.
- **Pre-call handler (Cartesia Line)** — configure voice, language, pronunciation, or reject calls before agent starts. Return `None` to reject with 403.
- **Multi-agent handoffs (Cartesia Line)** — `agent_as_handoff` transfers to specialized agent with `UpdateCallConfig` (e.g., change `voice_id` for language switch).
- **Knowledge base / RAG (Cartesia Line)** — upload documents (up to 100 bulk), folders, metadata filters, top_k.

### 8.10 Telephony Integration
- **Cartesia Line** — provision US phone numbers via API, import Twilio numbers, outbound calling, batch calling, call management (list/get/cancel/delete), call audio download, runtime logs, webhooks, provider linking (Twilio).
- **OpenAI** — SIP support for telephony voice agents. Telephony formats: μ-law, A-law.
- **Deepgram** — telephony integration for inbound/outbound calls.

### 8.11 LLM Provider Support
| Provider | LLMs supported |
|----------|---------------|
| OpenAI Realtime | Built-in (`gpt-realtime-2.1`) |
| Google Live API | Built-in (`gemini-3.1/2.5-flash-live`) |
| Deepgram Voice Agent | OpenAI, Anthropic, Google, Groq, AWS Bedrock, custom endpoint |
| Cartesia Line | 100+ via LiteLLM (Anthropic, OpenAI, Google, etc.) |
| ElevenLabs Speech Engine | Any LLM (you bring your own) |

**LLM config options (Cartesia Line LlmConfig):** `system_prompt`, `introduction`, `temperature`, `max_tokens`, `top_p`, `stop`, `seed`, `presence_penalty`, `frequency_penalty`, `num_retries`, `fallbacks`, `timeout`, `reasoning_effort`, `extra` (provider-specific via LiteLLM).

### 8.12 Event Systems
**Deepgram Voice Agent events:**
- Client → Server: `Settings`, `UpdateListen`, `UpdateThink`, `UpdateSpeak`, `UpdatePrompt`, `InjectUserMessage`, `InjectAgentMessage`, `SendFunctionCallResponse`, `KeepAlive`, `Media` (binary audio).
- Server → Client: `Welcome`, `SettingsApplied`, `ConversationText`, `UserStartedSpeaking`, `AgentThinking`, `AgentStartedSpeaking` (with latency metrics), `AgentAudioDone`, `Audio` (binary), `FunctionCallRequest`, `FunctionCallResponse`, `History`, `*Updated` acknowledgements, `InjectionRefused`, `Error`, `Warning`.

**Cartesia Line events:** `CallStarted`, `UserTurnStarted`, `UserTurnEnded`, `UserTextSent` (partial transcription), `CallEnded`, `AgentSendText`, `AgentEndCall`, `AgentHandedOff`. Event filters: `run_filter` (default `[CallStarted, UserTurnEnded, CallEnded]`), `cancel_filter` (default `[UserTurnStarted]`).

---

## Stage 9 — Output Formatting & Delivery

### 9.1 Audio Output Formats

| Format | Providers | Notes |
|--------|-----------|-------|
| **MP3** | ElevenLabs (22k–44.1k, 32–192kbps), OpenAI, Deepgram (22.05k fixed, 32k/48k), Cartesia (8k–48k, 32k–192k) | General playback |
| **PCM (raw)** | ElevenLabs (8k–48k, S16LE), OpenAI (24k S16LE), Google (24k), Cartesia (pcm_s16le/f32le, 8k–48k), Deepgram (linear16, 8k–48k), Gradium (48k) | Low latency, no decode |
| **WAV** | ElevenLabs (8k–48k), OpenAI, Cartesia (inherits raw encoding), Deepgram (container), Gradium (48k) | Lossless, file output |
| **Opus** | ElevenLabs (48k, 32–192kbps), OpenAI, Deepgram (48k, ogg container), Gradium (ogg/opus) | Bandwidth-efficient streaming |
| **μ-law (ulaw)** | ElevenLabs (8k), Cartesia (pcm_mulaw), Deepgram (mulaw, 8k/16k), Gradium (ulaw_8000) | US telephony |
| **A-law (alaw)** | ElevenLabs (8k), Cartesia (pcm_alaw), Deepgram (alaw, 8k/16k), Gradium (alaw_8000) | European telephony |
| **FLAC** | OpenAI, Deepgram | Lossless compression |
| **AAC** | OpenAI, Deepgram | YouTube, Android, iOS |
| **PCM f32le** | Cartesia | 32-bit float, Web Audio API |

### 9.2 Format Selection Guide

| Use case | Recommended format |
|----------|-------------------|
| General playback | MP3 44.1k 128kbps |
| Highest MP3 quality | MP3 44.1k 192kbps (Creator+) |
| Telephony / Twilio | μ-law 8k or A-law 8k |
| Low latency streaming | PCM 16k or 24k |
| High quality PCM | PCM 44.1k (Pro+) or 48k |
| Efficient streaming | Opus 48k 64kbps |
| WAV (lossless) | WAV 44.1k (Pro+) |
| Browser playback | PCM f32le 24k (Web Audio API) |
| File storage | MP3 44.1k 128kbps |
| Audio archiving | FLAC |

### 9.3 Streaming Delivery Modes

| Mode | Description | Providers |
|------|-------------|-----------|
| **Buffered HTTP** | Complete audio returned after generation | All |
| **HTTP chunked streaming** | Audio bytes arrive progressively | ElevenLabs, OpenAI, Google, Deepgram |
| **WebSocket bidirectional** | Persistent connection, send text/audio, receive audio/transcripts | All |
| **SSE** | Server-Sent Events with JSON-wrapped chunks + timestamps | Cartesia |
| **WebRTC** | Peer-to-peer media for browser/mobile | OpenAI, Google (via partners) |

### 9.4 Concurrency Management
- **HTTP** — each request counts individually toward concurrency (ElevenLabs).
- **WebSocket** — only model generation time counts; open connections mostly don't count (ElevenLabs, Cartesia).
- **TTS and STT** have **separate** concurrency limits (Cartesia).
- **Multiplexing** reduces connection overhead — multiple contexts on one connection (ElevenLabs, Cartesia, Gradium).
- **Heuristic:** concurrency limit of 5 can support ~100 simultaneous audio broadcasts (ElevenLabs, because generation is faster than playback).

### 9.5 Webhooks & Async Delivery
- **STT async** — `webhook=true` returns early, results delivered to configured webhook (ElevenLabs). `callback` URL (Deepgram).
- **Dubbing** — job completion webhook (ElevenLabs).
- **Post-call analysis** — call end + analysis completion (ElevenLabs Agents).
- **Line call events** — webhook endpoints (Cartesia).

### 9.6 Cost Tracking Headers
- `character-cost` — character cost for generation (ElevenLabs).
- `request-id` — unique request identifier (ElevenLabs).
- `x-trace-id` — trace ID for debugging (ElevenLabs).
- `current-concurrent-requests` / `maximum-concurrent-requests` (ElevenLabs).
- `dg-char-count`, `dg-request-id`, `dg-model-name`, `dg-model-uuid` (Deepgram).

---

## Stage 10 — Observability, Management & Billing

### 10.1 Usage & Billing
| Capability | Billing unit |
|-----------|-------------|
| TTS | Characters (ElevenLabs, Gradium, Deepgram); credits (Cartesia) |
| STT | Per hour of audio (ElevenLabs); per second (Cartesia, Gradium, Deepgram) |
| Voice Changer | 1,000 chars/min (ElevenLabs); 15 credits/sec (Cartesia) |
| Voice Isolator | 1,000 chars/min (ElevenLabs) |
| Sound Effects | 40 credits/sec (ElevenLabs, when duration specified) |
| Music | Per generation (ElevenLabs) |
| Dubbing | Per minute USD (ElevenLabs) |
| Forced Alignment | Same as STT (ElevenLabs) |
| Agent calls | USD per minute (Cartesia Line $0.06 + $0.014 telephony) |
| Entity Detection | +30% surcharge (ElevenLabs) |
| Keyterm Prompting | +20% surcharge (ElevenLabs) |
| Speaker Roles | +10% surcharge (ElevenLabs) |
| Pro Voice Clone | 1M credits per fine-tune (Cartesia); 1.5 credits/char TTS (Cartesia) |
| Infill | 300 credits + ~1 credit/char (Cartesia) |

### 10.2 Credit Monitoring
- **Cartesia** — `/usage/credits` and `/usage/agents` (admin API key required).
- **Gradium** — `GET /usages/credits`.
- **Deepgram** — `/v1/manage/projects/{id}/balances`, `/v1/manage/projects/{id}/usage`.

### 10.3 Management APIs
- **Deepgram** — projects, members, invitations, scopes, billing, usage, requests, API keys, models, model metadata, self-hosted credentials.
- **Cartesia** — API keys (admin), usage, agent management, phone numbers, providers, webhooks, documents, folders, metrics, deployments.
- **ElevenLabs** — voice management, pronunciation dictionaries, models listing.

### 10.4 Observability (Cartesia Line)
- **Call logs** — debug conversations and monitor performance.
- **Evaluations** — custom metrics (LLM-as-a-Judge).
- **Deployments** — track versions and roll back.
- **Metrics** — create, list, export metric results as CSV.

### 10.5 Self-Hosted Deployment
- **Deepgram** — self-hosted solution for enterprises (performance, latency, compliance, data residency, on-premise).
- **Cartesia** — Docker, Kubernetes, SageMaker self-hosting.

### 10.6 Partner Integrations
- **Google** — LiveKit, Pipecat, Fishjam, Stream Vision Agents, Voximplant, Agora, Firebase AI SDK.
- **Gradium** — LiveKit, Pipecat, OpenClaw, Gradbot; web search via Tavily, Linkup, Keenable.

### 10.7 Migration Guides
- **Gradium** — migration guides from Cartesia, Deepgram, ElevenLabs.

---

# Part III — The Unified API Specification

This part defines a **provider-agnostic API** that encompasses all features. It is written as a specification for a hypothetical "super complete" voice AI platform.

## API Surface Overview

```
Authentication:
  POST /auth/tokens                    — Create ephemeral client token

Voice Assets:
  GET  /voices                         — List/search voices
  GET  /voices/{id}                    — Get voice details
  POST /voices/clone                   — Instant voice clone
  POST /voices/clone/professional      — Create professional voice clone
  POST /voices/{id}/train              — Train professional clone
  POST /voices/{id}/verify             — Verify voice ownership (captcha)
  POST /voices/design                  — Design voice from text description
  POST /voices/remix                   — Remix existing voice
  POST /voices/localize                — Localize voice to new language
  PATCH /voices/{id}                   — Update voice metadata
  DELETE /voices/{id}                  — Delete voice
  GET  /voices/find-similar            — Find similar voices
  POST /voices/consent                — Create voice consent recording (OpenAI-style)
  GET  /voices/settings/default        — Get default voice settings

Pronunciation:
  POST /pronunciation-dictionaries     — Create dictionary
  GET  /pronunciation-dictionaries     — List dictionaries
  GET  /pronunciation-dictionaries/{id}— Get dictionary
  PATCH /pronunciation-dictionaries/{id}— Update dictionary
  DELETE /pronunciation-dictionaries/{id}— Delete dictionary
  POST /pronunciation-dictionaries/{id}/rules — Set rules
  POST /pronunciation-dictionaries/{id}/rules/add — Add rules
  POST /pronunciation-dictionaries/{id}/rules/remove — Remove rules

Datasets & Fine-tunes:
  POST /datasets                       — Create dataset
  GET  /datasets/{id}                  — Get dataset
  POST /datasets/{id}/files            — Upload file to dataset
  POST /fine-tunes/create              — Create fine-tune (Pro Voice Clone / Music)

Text-to-Speech:
  POST /tts                            — Buffered TTS (returns audio)
  POST /tts/stream                     — HTTP streaming TTS (chunked)
  WS   /tts/websocket                  — WebSocket TTS (contexts, continuations, multiplexing)
  POST /tts/sse                        — SSE TTS (with timestamps)
  POST /tts/with-timestamps            — TTS with character/word/phoneme timestamps
  POST /tts/dialogue                   — Multi-speaker dialogue TTS
  POST /tts/dialogue/stream            — Stream dialogue

Speech-to-Text:
  POST /stt                            — Batch transcription (file upload or URL)
  WS   /stt/realtime                   — Realtime streaming STT
  WS   /stt/turns                      — Realtime STT with native turn detection (Auto)
  WS   /stt/manual                     — Realtime STT with manual finalization
  GET  /stt/{transcript_id}            — Get transcript (async)
  DELETE /stt/{transcript_id}          — Delete transcript

Translation & Dubbing:
  POST /dubbing                        — Batch dubbing (audio/video)
  GET  /dubbing/{id}                    — Get dubbing details
  GET  /dubbing/{id}/audio              — Get dubbed audio
  GET  /dubbing/{id}/transcripts        — Retrieve dubbing transcript
  DELETE /dubbing/{id}                  — Delete dubbing
  POST /audio/translate                 — File-based audio translation (to English text)
  WS   /translation/realtime           — Live speech-to-speech translation
  WS   /s2s                            — Speech-to-speech live translation (duplex)

Voice Transformation:
  POST /voice-changer                  — Voice changer (buffered)
  POST /voice-changer/stream           — Voice changer (streamed)
  POST /audio-isolation                — Voice isolator (buffered)
  POST /audio-isolation/stream         — Voice isolator (streamed)
  POST /infill                         — Audio bridging/infill

Sound & Music:
  POST /sound-effects                  — Text-to-sound effects
  POST /music/compose                  — Music generation (simple prompt)
  POST /music/compose/stream           — Stream music
  POST /music/compose-detailed         — Music with detailed parameters
  POST /music/composition-plan         — Create composition plan (JSON)
  POST /music/video-to-music           — Video to music
  POST /music/upload                   — Upload music file
  POST /music/separate-stems           — Stem separation

Forced Alignment:
  POST /forced-alignment               — Align transcript to audio

Text Intelligence:
  POST /read                           — Analyze text content (summarize, sentiment, topics, intents)

Conversational Agent:
  WS   /agent/converse                  — End-to-end voice agent (single WebSocket)
  WS   /speech-engine                   — BYO LLM speech engine
  POST /agent/configs                  — Create agent configuration
  GET  /agent/configs/{id}              — Get agent config
  POST /agents                         — Create managed agent (Line-style)
  POST /agents/calls/create-outbound    — Outbound call
  POST /agents/call-batches/create-call-batch — Batch calling
  POST /agents/documents               — Upload knowledge base document
  POST /agents/webhooks                 — Register webhook

Realtime (OpenAI-style):
  WS   /realtime                        — Voice agent realtime session
  POST /realtime/calls                  — WebRTC call establishment (SDP)
  POST /realtime/client_secrets         — Create ephemeral client secret
  WS   /realtime/translations           — Translation session
  POST /realtime/translations/calls     — Translation WebRTC
  POST /realtime/translations/client_secrets — Translation client secret

Management:
  GET  /usage/credits                  — Credit balance
  GET  /usage/agents                    — Agent usage
  GET  /manage/projects                 — List projects
  GET  /manage/projects/{id}/keys       — List API keys
  POST /manage/projects/{id}/keys       — Create API key
  DELETE /manage/projects/{id}/keys/{key_id} — Delete API key
  GET  /models                          — List models
```

---

## Unified Data Structures

### Voice Object

```json
{
  "id": "uuid-or-string",
  "name": "Skylar - Friendly Guide",
  "description": "Approachable American female ideal for customer care.",
  "gender": "feminine | masculine | gender_neutral",
  "language": "en",
  "country": "US",
  "age_group": "young_adult | adult | mature",
  "is_owner": false,
  "is_pro": false,
  "access": { "type": "private | public", "visibility": "owner | all" },
  "preview_file_url": "https://...",
  "created_at": "ISO-8601"
}
```

### Voice Settings (Unified)

```json
{
  "stability": 0.5,
  "similarity_boost": 0.75,
  "style": 0,
  "use_speaker_boost": true,
  "speed": 1.0,
  "volume": 1.0,
  "emotion": "neutral",
  "temperature": 0.7,
  "cfg_coef": 2.0,
  "padding_bonus": 0.0,
  "instructions": "Speak cheerfully with a slight Southern accent.",
  "seed": 42
}
```

| Field | Type | Range/Values | Description | Source providers |
|-------|------|-------------|-------------|-------------------|
| `stability` | float | 0–1 | Lower = broader emotional range; higher = monotonous | ElevenLabs |
| `similarity_boost` | float | 0–1 | How closely to adhere to original voice | ElevenLabs |
| `style` | float | 0–1 | Style exaggeration amplification | ElevenLabs |
| `use_speaker_boost` | boolean | — | Boost similarity to original speaker (increases latency) | ElevenLabs |
| `speed` | float | 0.6–1.5 | Speech speed multiplier | ElevenLabs (0.7–1.2), Cartesia (0.6–1.5), Deepgram |
| `volume` | float | 0.5–2.0 | Volume multiplier | Cartesia |
| `emotion` | enum | 60+ values | Emotional guidance (neutral, calm, angry, content, sad, scared, happy, excited, ...) | Cartesia |
| `temperature` | float | 0.0–1.5 | Sampling temperature; 0.0 = deterministic | Gradium (0.0–1.4 TTS, 0.0–1.5 STT), Google |
| `cfg_coef` | float | 1.0–4.0 | Voice similarity; higher = closer to target voice | Gradium |
| `padding_bonus` | float | -4.0 to 4.0 | Speed; negative = faster, positive = slower | Gradium |
| `instructions` | string | — | Natural-language style control (accent, emotion, intonation, speed, tone, whispering) | OpenAI (gpt-4o-mini-tts), Google (Director's Notes) |
| `seed` | integer | 0–4294967295 | Deterministic sampling seed | ElevenLabs |

### Output Format Object (Unified)

```json
{
  "container": "raw | wav | mp3 | ogg | none",
  "encoding": "pcm_s16le | pcm_f32le | pcm_mulaw | pcm_alaw | linear16 | linear32 | flac | mp3 | opus | aac",
  "sample_rate": 8000 | 16000 | 22050 | 24000 | 32000 | 44100 | 48000,
  "bit_rate": 32000 | 48000 | 64000 | 96000 | 128000 | 192000
}
```

### Pronunciation Dictionary Locator

```json
{
  "pronunciation_dictionary_id": "dict_id",
  "version_id": "version_id_or_null_for_latest"
}
```

### Pronunciation Dictionary Item

```json
{
  "text": "acme",
  "pronunciation": "<<ˈ|æ|k|m|i>>",
  "alias": "deprecated-use-pronunciation"
}
```

### STT Request Parameters (Unified)

```json
{
  "model_id": "string",
  "file": "binary or null",
  "source_url": "string or null",
  "language_code": "ISO-639-1 or null",
  "tag_audio_events": true,
  "num_speakers": "int or null",
  "timestamps_granularity": "none | word | character | phoneme",
  "diarize": true,
  "diarization_threshold": "float or null",
  "diarize_model": "latest | v1 | v2",
  "use_multi_channel": false,
  "multichannel_output_style": "separate | combined",
  "file_format": "pcm_s16le_16 | other",
  "additional_formats": ["srt", "txt", "docx", "html", "pdf", "segmented_json"],
  "entity_detection": "all | pii | phi | pci | other | offensive_language | [entity_types]",
  "entity_redaction": "string | array | null",
  "entity_redaction_mode": "redacted | entity_type | enumerated_entity_type",
  "keyterms": ["term1", "term2"],
  "keywords": ["term:boost"],
  "search": ["term"],
  "replace": ["old:new"],
  "redact": ["pii", "pci", "numbers", "ssn"],
  "no_verbatim": false,
  "use_speaker_library": false,
  "detect_speaker_roles": false,
  "known_speaker_names": ["Alice", "Bob"],
  "known_speaker_references": ["data:audio/wav;base64,..."],
  "smart_format": false,
  "punctuate": false,
  "paragraphs": false,
  "numerals": false,
  "dictation": false,
  "profanity_filter": false,
  "filler_words": false,
  "measurements": false,
  "summarize": "true | false | v2",
  "sentiment": false,
  "topics": false,
  "custom_topic": ["topic1"],
  "custom_topic_mode": "strict | extended",
  "intents": false,
  "custom_intent": ["intent1"],
  "custom_intent_mode": "strict | extended",
  "detect_entities": false,
  "detect_language": "boolean | array",
  "temperature": "float or null",
  "seed": "int or null",
  "chunking_strategy": "auto | vad_config",
  "stream": false,
  "include": ["logprobs"],
  "utterances": false,
  "utt_split": 0.8,
  "webhook": false,
  "webhook_id": "string or null",
  "webhook_metadata": "JSON string or null",
  "callback": "URL or null",
  "callback_method": "POST | PUT",
  "tag": ["label"],
  "mip_opt_out": false,
  "enable_logging": true,
  "delay": "minimal | low | medium | high | xhigh",
  "delay_in_frames": 16
}
```

### STT Response Structure (Unified)

```json
{
  "language_code": "en",
  "language_probability": 0.98,
  "text": "Hello world!",
  "words": [
    {
      "text": "Hello",
      "start": 0.0,
      "end": 0.5,
      "type": "word | spacing | audio_event",
      "speaker_id": "speaker_0",
      "logprob": -0.124,
      "confidence": 0.999,
      "characters": [{"text": "H", "start": 0.0, "end": 0.1}]
    }
  ],
  "transcription_id": "abc123",
  "entities": [
    {"text": "John Smith", "entity_type": "person_name", "start_char": 0, "end_char": 10}
  ],
  "audio_duration_secs": 15.2,
  "utterances": [
    {"start": 0, "end": 6, "confidence": 0.95, "transcript": "...", "speaker": 0}
  ],
  "summary": {"result": "success", "short": "..."},
  "sentiments": {
    "segments": [{"text": "...", "sentiment": "positive", "sentiment_score": 0.58}],
    "average": {"sentiment": "positive", "sentiment_score": 0.58}
  },
  "topics": {"results": {"topics": {"segments": [{"text": "...", "topics": [{"topic": "...", "confidence_score": 0.9}]}]}}},
  "intents": {"results": {"intents": {"segments": [{"text": "...", "intents": [{"intent": "...", "confidence_score": 0.9}]}]}}},
  "detected_channels": 1
}
```

### TTS Request (Unified)

```json
{
  "model_id": "string",
  "text": "Text to synthesize",
  "voice_id": "string",
  "voice_settings": { "...see Voice Settings..." },
  "output_format": { "...see Output Format..." },
  "language_code": "ISO-639-1 or null",
  "pronunciation_dictionary_locators": [
    {"pronunciation_dictionary_id": "...", "version_id": null}
  ],
  "pronunciation_id": "string or null",
  "seed": "int or null",
  "previous_text": "string or null",
  "next_text": "string or null",
  "previous_request_ids": ["array or null"],
  "next_request_ids": ["array or null"],
  "use_pvc_as_ivc": false,
  "apply_text_normalization": "auto | on | off",
  "apply_language_text_normalization": false,
  "enable_logging": true,
  "optimize_streaming_latency": "int or null",
  "add_timestamps": false,
  "add_phoneme_timestamps": false,
  "use_normalized_timestamps": null,
  "context_id": "string",
  "continue": true,
  "max_buffer_delay_ms": 3000,
  "flush": false,
  "cancel": false,
  "instructions": "string or null",
  "rewrite_rules": "en | fr | de | es | pt or null"
}
```

### Multi-Speaker Dialogue Request (Unified)

```json
{
  "inputs": [
    {"text": "[giggling] That's funny!", "voice_id": "voice_1", "speaker": "Joe"},
    {"text": "[groaning] That was awful.", "voice_id": "voice_2", "speaker": "Jane"}
  ],
  "speech_config": [
    {"speaker": "Joe", "voice": "Kore"},
    {"speaker": "Jane", "voice": "Puck"}
  ],
  "model_id": "string",
  "output_format": "mp3_44100_128",
  "seed": 42
}
```

### Voice Changer Request (Unified)

```json
{
  "audio": "binary file",
  "voice_id": "target_voice_id",
  "model_id": "string",
  "remove_background_noise": false,
  "output_format": { "...see Output Format..." }
}
```

### Dubbing Request (Unified)

```json
{
  "file": "binary or null",
  "source_url": "string or null",
  "source_language_code": "ISO-639-1 or null",
  "target_language_code": "ISO-639-1",
  "num_speakers": "int or null",
  "cloning_strength": 7,
  "preserve_original_voices": true,
  "keep_background_audio": true,
  "watermark": false
}
```

### Forced Alignment Request (Unified)

```json
{
  "audio_file": "binary",
  "text": "plain string transcript (NOT JSON-wrapped)",
  "model_id": "alignment_model"
}
```

### Sound Effects Request (Unified)

```json
{
  "text": "Glass shattering on concrete",
  "duration_seconds": 2.5,
  "prompt_influence": 0.7,
  "loop": false,
  "output_format": "mp3 | wav_48000"
}
```

### Infill Request (Unified)

```json
{
  "left_audio": "binary or null",
  "right_audio": "binary or null",
  "transcript": "Text to generate between clips",
  "model_id": "string",
  "language": "ISO-639-1",
  "voice_id": "string",
  "output_format": { "...see Output Format..." }
}
```

### Realtime Session Configuration (Unified)

```json
{
  "type": "realtime | transcription | translation",
  "model": "string",
  "output_modalities": ["audio", "text"],
  "audio": {
    "input": {
      "format": {"type": "audio/pcm", "rate": 24000},
      "turn_detection": {
        "type": "server_vad | semantic_vad | null",
        "threshold": 0.5,
        "prefix_padding_ms": 300,
        "silence_duration_ms": 500,
        "eagerness": "auto | low | medium | high",
        "create_response": true,
        "interrupt_response": true
      }
    },
    "output": {
      "format": {"type": "audio/pcm"},
      "voice": "string or {id: string}",
      "language": "ISO-639-1"
    }
  },
  "instructions": "System instructions for the model",
  "prompt": {"id": "pmpt_123", "version": "89", "variables": {"city": "Paris"}},
  "tools": ["function_definitions"],
  "tool_choice": "auto",
  "thinking": {"level": "minimal | low | medium | high", "budget": 0},
  "enable_affective_dialog": false,
  "proactivity": {"proactive_audio": false},
  "input_audio_transcription": {},
  "output_audio_transcription": {},
  "session_resumption": {"handle": "previous_handle"},
  "context_window_compression": {
    "sliding_window": {"target_tokens": 5000},
    "trigger_tokens": 8000
  },
  "translation_config": {
    "target_language_code": "es",
    "echo_target_language": true
  },
  "history_config": {"initial_history_in_client_content": true},
  "context": {
    "messages": [
      {"type": "History", "role": "user", "content": "..."},
      {"type": "History", "role": "assistant", "content": "..."}
    ]
  }
}
```

### Realtime Event Reference (Unified)

**Client → Server events:**

| Event | Description | Sessions |
|-------|-------------|---------|
| `session.update` | Update session configuration | All |
| `conversation.item.create` | Add conversation item (message, function_call_output) | Voice agent |
| `conversation.item.truncate` | Truncate unplayed audio after interruption (WebSocket) | Voice agent |
| `input_audio_buffer.append` | Append base64 audio chunk | All |
| `input_audio_buffer.commit` | Commit audio buffer (VAD disabled) | Voice agent, transcription |
| `input_audio_buffer.clear` | Clear audio buffer | Voice agent |
| `output_audio_buffer.clear` | Clear unplayed output audio (WebRTC/SIP) | Voice agent |
| `response.create` | Request model response | Voice agent |
| `response.cancel` | Cancel in-progress response | Voice agent (push-to-talk) |
| `session.close` | Close translation session gracefully | Translation |
| `send_realtime_input` | Send audio/video/text stream (Google) | Live API |
| `send_client_content` | Send incremental conversation update (Google) | Live API |
| `send_tool_response` | Return function execution result (Google) | Live API |
| `finalize` | Emit transcripts for buffered audio (Cartesia manual STT) | STT manual |
| `close` | Close stream (Cartesia/Deepgram) | STT, TTS |
| `flush` | Force processing of buffered input (Gradio STT) | STT |
| `KeepAlive` | Keep connection alive (Deepgram) | All |

**Server → Client events:**

| Event | Description | Sessions |
|-------|-------------|---------|
| `session.created` / `setupComplete` / `ready` / `Welcome` | Session is ready | All |
| `session.updated` / `SettingsApplied` | Session config updated | All |
| `session.closed` | Translation session fully closed | Translation |
| `conversation.item.added` / `done` | Conversation item added/complete | Voice agent |
| `input_audio_buffer.speech_started` / `SpeechStarted` / `turn.start` / `StartOfTurn` / `UserStartedSpeaking` | User started speaking | All with VAD |
| `input_audio_buffer.speech_stopped` | User stopped speaking | Voice agent, transcription |
| `input_audio_buffer.committed` | Audio buffer committed | Voice agent |
| `response.created` | Response started | Voice agent |
| `response.output_audio.delta` / `Audio` (binary) / `chunk` / `audio` | Audio streaming delta | Voice agent, TTS, translation |
| `response.output_audio.done` | Audio output complete | Voice agent |
| `response.output_audio_transcript.delta` / `outputTranscription` | Audio transcript streaming delta | Voice agent |
| `response.output_text.delta` | Text streaming delta | Voice agent |
| `response.output_text.done` | Text output complete | Voice agent |
| `response.function_call_arguments.delta` | Function call args streaming | Voice agent |
| `response.done` | Full response complete | Voice agent |
| `response.cancelled` | Response cancelled (barge-in) | Voice agent |
| `turn.update` / `Update` | Additional audio transcribed (cumulative) | STT turn-based |
| `turn.eager_end` / `EagerEndOfTurn` | Moderate confidence user finished | STT turn-based |
| `turn.resume` / `TurnResumed` | Speech continued after eager end | STT turn-based |
| `turn.end` / `EndOfTurn` | Definitive turn end | STT turn-based |
| `step` | Semantic VAD predictions (Gradium, every 80ms) | STT |
| `flushed` | Flush operation completed | STT, TTS |
| `transcript.text.delta` / `Results` | File/streaming transcription delta | STT |
| `transcript.text.done` | File transcription complete | STT |
| `transcript.text.segment` | Diarized segment complete | STT |
| `session.output_audio.delta` | Translated audio delta (translation) | Translation |
| `session.output_transcript.delta` | Translated text delta | Translation |
| `session.input_transcript.delta` | Source transcript delta | Translation |
| `interrupted` | User interrupted model generation (Google) | Live API |
| `turnComplete` / `generationComplete` | Turn/generation complete (Google) | Live API |
| `toolCall` / `FunctionCallRequest` | Server requests function execution | Voice agent |
| `toolCallCancellation` | Function call cancelled (interruption) | Live API |
| `goAway` | Server signaling connection termination (Google) | Live API |
| `sessionResumptionUpdate` | New resumable handle (Google) | Live API |
| `rate_limits.updated` | Rate limit info updated | Voice agent |
| `AgentThinking` | Agent's thought process text (Deepgram) | Voice agent |
| `AgentStartedSpeaking` | Agent begins speaking, includes latency metrics (Deepgram) | Voice agent |
| `AgentAudioDone` | Agent finished sending audio (Deepgram) | Voice agent |
| `ConversationText` | Transcript of what was said (Deepgram) | Voice agent |
| `InjectionRefused` | Injection refused with reason (Deepgram) | Voice agent |
| `UsageMetadata` | Token usage metadata (Google) | Live API |
| `error` | Error event | All |

---

## Capability Decision Matrix

The definitive guide to choosing the right capability for any need:

| If you need... | Use this capability | Key parameters |
|----------------|---------------------|---------------|
| Text → single-voice audio | TTS (buffered/streaming/WebSocket) | `text`, `model_id`, `voice_id`, `voice_settings`, `output_format` |
| Text → multi-speaker dialogue audio | Dialogue TTS | `inputs[]` (text + voice per turn) or `speech_config[]` (speaker + voice) |
| Control how the voice sounds | Voice settings / instructions / tags | `stability`, `similarity_boost`, `style`, `speed`, `emotion`, `instructions`, audio tags, SSML tags |
| Stream LLM tokens → speech (real-time) | WebSocket TTS with continuations | `context_id`, `continue`, `max_buffer_delay_ms` |
| Stitch TTS chunks for prosody continuity | Context stitching | `previous_text`/`next_text` or `previous_request_ids`/`next_request_ids` |
| Deterministic TTS output | Seed / temperature | `seed` (0–4294967295) or `temp: 0.0` |
| Control pronunciation of specific words | Pronunciation dictionaries | Create dict → attach via locator/id in TTS request |
| Audio → text transcript (file) | Batch STT | `file`/`source_url`, `model_id`, `diarize`, `keyterms` |
| Live audio → text | Realtime STT (WebSocket) | `encoding`, `sample_rate`, `interim_results`, VAD/turn detection |
| Live audio → text with auto turn detection | STT with native turn detection | Turn lifecycle events (start/update/eager_end/resume/end) |
| Identify who is speaking | Diarization | `diarize`, `diarize_model`, `num_speakers`, `known_speaker_names` |
| Improve recognition of specific terms | Keyterm/keyword prompting | `keyterms` (plain terms) or `keywords` (weighted) or `prompt` (Whisper) |
| Detect PII/PHI/PCI in transcripts | Entity detection + redaction | `entity_detection`, `entity_redaction`, `redact` |
| Summarize audio content | Audio intelligence | `summarize`, `sentiment`, `topics`, `intents`, `detect_entities` |
| Remove filler words | No verbatim mode | `no_verbatim` (ElevenLabs), `filler_words` (Deepgram) |
| Format transcripts (currency, phones) | Smart format | `smart_format`, `numerals`, `dictation`, `measurements` |
| Export to SRT/TXT/DOCX/HTML/PDF | Additional formats | `additional_formats` array |
| Change voice in audio (preserve emotion) | Voice changer | `audio`, `voice_id`, `remove_background_noise` |
| Remove background noise from audio | Voice isolator | `audio` file |
| Bridge two audio clips with generated audio | Infill | `left_audio`, `right_audio`, `transcript`, `voice_id` |
| Text → sound effect | Sound effects | `text`, `duration_seconds`, `prompt_influence`, `loop` |
| Text → music | Music generation | `text` prompt, `composition_plan`, `model_id` |
| Edit/combine music sections | Music inpainting | Existing song + edit instructions |
| Separate song into stems | Stem separation | Song file |
| Translate audio/video to another language (batch) | Dubbing | `file`/URL, source/target language, `cloning_strength` |
| Translate audio to English text (file) | Audio translation | `file`, `model` (whisper) |
| Real-time speech-to-speech translation | Live translation session | `target_language_code`, continuous audio stream |
| Transcribe + translate in one STT call | Translating transcription | `model_name: stt-translate`, `target_language` |
| Align transcript to audio timestamps | Forced alignment | `audio_file`, `text` (plain string) |
| Clone a voice quickly | Instant voice clone | Audio sample (≤10s to 2min), `name`, `language` |
| Clone a voice professionally | Professional voice clone | Extended audio data, train, verify |
| Design a new voice from text | Voice design | `voice_description` (20–1000 chars) |
| Transform an existing voice | Voice remix | `voice_id`, `prompt_strength` |
| Adapt voice to new language | Voice localization | `voice_id`, `target_language`, `dialect` |
| Build a low-latency voice agent | Realtime/conversational session | `model`, `voice`, `instructions`, `turn_detection`, `tools` |
| Build a voice agent with your own LLM | BYO LLM speech engine | WebSocket, any LLM |
| Build a managed voice agent platform | Managed agent platform (Line-style) | `LlmAgent`, `tools`, deploy via CLI |
| Handle user interruptions | Barge-in handling | VAD enabled, `interrupt_response: true`, truncate unplayed audio |
| Push-to-talk voice agent | Manual turn control | `turn_detection: null`, manual `commit` + `response.create` |
| Call external APIs from voice agent | Function calling | `tools[]` with `endpoint` (server-side) or client-side execution |
| Async function calling (non-blocking) | Async tools (Google 2.5) | `behavior: NON_BLOCKING`, `scheduling: INTERRUPT/WHEN_IDLE/SILENT` |
| Send image to voice agent | Multimodal input | `input_image` content part (OpenAI), `send_realtime_input(video)` (Google) |
| Resume session after reconnection | Session resumption (Google) | `sessionResumption.handle`, save `newHandle` |
| Run session indefinitely | Context compression (Google) | `contextWindowCompression.slidingWindow` |
| Model decides not to respond | Proactive audio (Google 2.5) | `proactivity: {proactive_audio: true}` |
| Emotion-adaptive responses | Affective dialog (Google 2.5) | `enable_affective_dialog: true` |
| Control reasoning depth | Thinking control | `reasoning.effort`, `thinkingLevel`, `thinkingBudget` |
| Multi-agent handoff | Handoff tools | `agent_as_handoff` with `UpdateCallConfig` |
| Connect agent to phone | Telephony integration | Provision numbers, import Twilio, SIP support |
| Track agent latency | Latency metrics | `total_latency`, `tts_latency`, `ttt_latency` |
| Multiplex multiple TTS/STT requests | Multiplexing | `context_id` (Cartesia/ElevenLabs), `client_req_id` (Gradium) |
| Cancel in-flight TTS generation | WebSocket cancel | `{"context_id": "...", "cancel": true}` (Cartesia) |
| Pin model version for production | Dated model IDs | `sonic-3.5-2026-05-04` (Cartesia) |
| Lowest latency TTS | Custom buffering + WebSocket | `max_buffer_delay_ms: 0`, full sentences (Cartesia) |
| Lowest latency STT turn detection | Native turn detection | Ink-2 (Cartesia) or Flux (Deepgram) with eager end |
| Secure client-side connection | Ephemeral tokens | Server generates token → client uses token |
| Zero data retention | Privacy mode | `enable_logging=false` (ElevenLabs), ZDR (Cartesia), `mip_opt_out` (Deepgram) |
| Async STT results | Webhooks | `webhook=true`, `callback` URL |
| Track costs | Response headers | `character-cost`, `dg-char-count` |
| Monitor credit balance | Usage API | `GET /usage/credits`, `GET /manage/projects/{id}/balances` |

---

## Language Support Summary

| Capability | Max languages | Providers with widest coverage |
|-----------|--------------|-------------------------------|
| TTS | 95+ | Google (95+ auto-detected), Cartesia (42), ElevenLabs (70+ v3, 32 Flash) |
| STT batch | 90+ | ElevenLabs Scribe (90+), Cartesia ink-whisper (90+), OpenAI Whisper (57 listed, 98 trained) |
| STT realtime | 90+ | ElevenLabs Scribe Realtime (90+), Cartesia ink-whisper (90+) |
| Live translation | 70+ | Google (70+), ElevenLabs Dubbing (90+ batch), Gradium S2S (5) |
| Conversational | 97 | Google Live API (97), ElevenLabs (70+), Cartesia (42 TTS + English STT) |
| Turn detection (native) | 10 | Deepgram Flux Multi (10), Cartesia Ink-2 (English) |

---

## Summary of Unique Provider Capabilities

Each provider contributes unique features that no other provider offers:

| Provider | Unique capabilities not found elsewhere |
|----------|----------------------------------------|
| **ElevenLabs** | Voice Design from text, Voice Remixing, Text to Dialogue (multi-speaker, unlimited), Sound Effects, Music Generation (compose, inpainting, stems, finetunes, video-to-music), Dubbing (batch, 90+), Forced Alignment, Voice Isolator, Audio Tags (v3), Speaker Roles (agent/customer), Speaker Library matching, PVC captcha verification |
| **OpenAI** | WebRTC realtime sessions, Ephemeral Client Secrets, Image input in voice agent, Server-stored prompts in Realtime, Out-of-band responses (`conversation: "none"`), Multimodal Audio Chat (gpt-audio-1.5 in Chat Completions), Instruction-based TTS control (`gpt-4o-mini-tts`), Consent-based custom voice creation (16-language consent phrases), `known_speaker_references` (audio data URLs for diarization) |
| **Google Gemini** | Director-style TTS prompting (Audio Profile + Scene + Director's Notes + Transcript), Session Resumption, Context Window Compression (indefinite sessions), GoAway messages, Proactive Audio, Affective Dialog, Async function calling (`NON_BLOCKING` + `scheduling`), Video input in Live API (max 1 FPS), Thinking control (`thinkingLevel`/`thinkingBudget`), `turnCoverage` options, `mediaResolution`, Ephemeral token constraints (`live_connect_constraints`), YouTube URL as audio input, 30 prebuilt voices with personality traits |
| **Cartesia** | Native semantic turn detection (Ink-2, no separate VAD), Voice Localization (20 languages + dialects), Audio Infill/bridging, SSML tags (`<speed>`, `<volume>`, `<emotion>`, `<break>`, `<spell>`), 60+ emotion values in generation config, `max_buffer_delay_ms` buffering control, WebSocket multiplexing with `context_id`, Managed Line platform (CLI deploy <30s, telephony, knowledge base, evaluations, deployments, metrics), HTTP server tools (JSON schema-based), `cartesia-line` SDK with LiteLLM (100+ LLM providers), Pre-call handler, Pro Voice Clone fine-tunes, Datasets API |
| **Deepgram** | Flux model-native turn detection (EagerEndOfTurn/TurnResumed), Audio Intelligence as STT add-ons (summarize, sentiment, topics, intents, entities, custom topics/intents with strict/extended modes), Text Intelligence Read API, Smart Format (currency, phone, email, dates), Paragraph splitting, Numerals, Dictation mode, Measurements, Profanity filter, Filler words, Search & Replace in transcripts, Redaction (pii/pci/numbers/ssn/aggressive_numbers), Domain-specific STT models (medical, meeting, finance, phonecall, voicemail, video, drivethru, automotive, conversationalai), Self-hosted deployment, `is_final` vs `speech_final` distinction, Endpointing, Whisper Cloud, Multi-provider Think (LLM) support in Voice Agent |
| **Gradium** | Semantic VAD `step` messages every 80ms with multi-horizon `inactivity_prob` predictions, Adaptive delay control (`delay_in_frames` 0–80, each frame = 80ms), `<flush>` in-text tag for forced audio emission, `cfg_coef` voice similarity control, `padding_bonus` speed control (-4 to 4), `rewrite_rules` (language-specific text rewriting), Unified WebSocket lifecycle (TTS/STT/S2S share same protocol), S2S with explicit inner model selection (`stt_model_name`/`tts_model_name`), `close_ws_on_eos: false` for multiplexing, Dual transport (WebSocket + REST) with same models/config, Low-latency (<300ms TTFT), 5 languages with full TTS+STT+S2S coverage |

---

*This specification aggregates the complete capabilities of ElevenLabs, OpenAI, Google Gemini, Cartesia, Deepgram, and Gradium as documented in their respective platform study files. It is intended as a reference for building or evaluating a comprehensive voice AI platform.*