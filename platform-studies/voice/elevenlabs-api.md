# ElevenLabs API Analysis — Text to Speech, Speech to Text, Voice Cloning, Music & Conversational AI

> **Base URL:** `https://api.elevenlabs.io/v1` (REST) | **WebSocket:** `wss://api.elevenlabs.io/v1` | **Servers:** US, EU, India, Singapore (data residency)
> **Docs:** `https://elevenlabs.io/docs` | **Product:** `https://elevenlabs.io` | **Auth:** API key (`xi-api-key` header)
> **SDKs:** `elevenlabs` (Python), `@elevenlabs/elevenlabs-js` (TypeScript/Node), Scribe (JS/React), Speech Engine (Python/JS)
> **Description:** ElevenLabs is a voice AI platform providing APIs and SDKs for text-to-speech, speech-to-text, voice cloning, voice changing, voice isolation, sound effects generation, music generation, dubbing, forced alignment, text-to-dialogue, voice design/remixing, pronunciation dictionaries, and conversational AI agents. Its models span 70+ languages and support ultra-low-latency streaming for real-time applications.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Authentication & Access Control](#2-authentication--access-control)
3. [Models — Catalog & Selection Guide](#3-models--catalog--selection-guide)
4. [Text to Speech (TTS)](#4-text-to-speech-tts)
5. [Speech to Text (STT) — Batch & Realtime](#5-speech-to-text-stt--batch--realtime)
6. [Text to Dialogue](#6-text-to-dialogue)
7. [Voices — Cloning, Design, Remixing & Library](#7-voices--cloning-design-remixing--library)
8. [Voice Changer (Speech to Speech)](#8-voice-changer-speech-to-speech)
9. [Voice Isolator (Audio Isolation)](#9-voice-isolator-audio-isolation)
10. [Sound Effects](#10-sound-effects)
11. [Music Generation](#11-music-generation)
12. [Dubbing](#12-dubbing)
13. [Forced Alignment](#13-forced-alignment)
14. [Speech Engine — Conversational Voice with Custom LLM](#14-speech-engine--conversational-voice-with-custom-llm)
15. [Pronunciation Dictionaries](#15-pronunciation-dictionaries)
16. [Audio Streaming Architecture](#16-audio-streaming-architecture)
17. [Concurrency, Limits & Pricing](#17-concurrency-limits--pricing)
18. [Webhooks](#18-webhooks)
19. [Audio Output Formats Reference](#19-audio-output-formats-reference)
20. [Capability Summary & Cross-Reference](#20-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

ElevenLabs' platform is organized around these core abstractions:

- **Voice** — A distinct speech identity identified by a `voice_id`. Voices can come from the Voice Library (community-shared), be cloned from audio samples (Instant/Professional Voice Cloning), or be generated from text descriptions (Voice Design). Each voice has configurable settings (stability, similarity, style, speed).
- **Model** — The underlying AI model used for generation, identified by a `model_id` (e.g. `eleven_v3`, `eleven_multilingual_v2`, `eleven_flash_v2_5`, `scribe_v2`, `scribe_v2_realtime`, `music_v2`, `eleven_text_to_sound_v2`). Models are optimized for different quality/latency tradeoffs and language coverage.
- **Character** — The billing unit for text-to-speech and voice-changing operations. Each model has a per-request character limit. Response headers include `character-cost` for tracking.
- **Credit** — The platform-wide billing unit. Different operations consume credits at different rates (per character, per second of audio, per minute of dubbing).
- **Voice Settings** — Per-request overrides for voice behavior: `stability` (0–1), `similarity_boost` (0–1), `style` (0–1), `use_speaker_boost` (boolean), `speed` (0.7–1.2).
- **Audio Tags** — Inline non-speech directives in text, wrapped in square brackets (e.g. `[laughs]`, `[whispering]`, `[applause]`), supported by the Eleven v3 model to influence emotional delivery and ambient sounds.
- **Pronunciation Dictionary** — A versioned set of phonetic rules that override how specific words or phrases are pronounced. Attachable to TTS requests via locators.
- **Zero Retention Mode** — A privacy mode that prevents storage of request data; available to Enterprise customers via `enable_logging=false`.

### Platform Architecture

```
Text Input ──▶ TTS Model ──▶ Audio Output (buffered or streamed)
                  │
    Voice Selection ──▶ Voice Library / Cloned / Designed
    Voice Settings ──▶ Stability, Similarity, Style, Speed
    Pronunciation Dict ──▶ Override pronunciation
    Context Stitching ──▶ previous_text / next_text / request_ids

Audio Input ──▶ STT Model ──▶ Transcript JSON (words, timestamps, speakers)
                  │
    Diarization ──▶ Speaker identification (up to 32)
    Entity Detection ──▶ PII/PHI/PCI entities with positions
    Keyterm Prompting ──▶ Bias toward domain terms
    Additional Formats ──▶ SRT, TXT, DOCX, HTML, PDF, JSON

Audio Input ──▶ Voice Changer ──▶ New voice audio (preserves emotion)
Audio Input ──▶ Voice Isolator ──▶ Clean speech (removes background noise)
```

### End-to-End Flows

**Text to Speech pipeline:**
```
Select voice_id ──▶ Choose model_id ──▶ POST /v1/text-to-speech/{voice_id}
                     │                         │
        optional: voice_settings        returns: audio bytes (MP3/PCM/WAV/Opus/μ-law)
                 pronunciation_dictionary
                 previous_text / next_text (stitching)
                 seed (determinism)
```

**Real-time conversational agent (Speech Engine):**
```
Browser ──▶ ElevenLabs (audio capture + STT) ──WebSocket──▶ Your Server (LLM logic)
                                                              │
                    Browser ◀──ElevenLabs (TTS)◀──WebSocket──  ┘ (streamed text response)
```

**Voice cloning pipeline:**
```
Upload audio samples ──▶ Create IVC/PVC voice ──▶ (PVC: train + verify captcha) ──▶ Use voice_id in TTS
```

---

## 2. Authentication & Access Control

### API Keys

All API requests require an `xi-api-key` HTTP header:

```
xi-api-key: ELEVENLABS_API_KEY
```

API keys can be configured with:
- **Scope restriction** — Limit which API endpoints the key can access
- **Credit quota** — Define custom credit limits per key
- **IP allowlisting** — Restrict to specific IPs/CIDR ranges (non-allowlisted IPs get `403`)

### Single Use Tokens

For client-side connections without exposing the API key, single-use tokens can be created (time-limited, one-time use). Useful for browser-based WebSocket connections.

### SDK Authentication

```python
from elevenlabs.client import ElevenLabs
client = ElevenLabs(api_key=os.getenv("ELEVENLABS_API_KEY"))
```

```typescript
import { ElevenLabsClient } from "@elevenlabs/elevenlabs-js";
const client = new ElevenLabsClient(); // reads ELEVENLABS_API_KEY env var
```

### Response Headers for Cost Tracking

| Header | Description |
|--------|-------------|
| `character-cost` | Character cost for the generation |
| `request-id` | Unique request identifier (used for stitching) |
| `x-trace-id` | Trace ID for debugging |
| `current-concurrent-requests` | Current concurrent request count |
| `maximum-concurrent-requests` | Plan's concurrency limit |

### Data Residency

Available server regions:
- `https://api.elevenlabs.io` — Default (US)
- `https://api.us.elevenlabs.io` — US
- `https://api.eu.residency.elevenlabs.io` — EU
- `https://api.in.residency.elevenlabs.io` — India
- `https://api.sg.residency.elevenlabs.io` — Singapore

---

## 3. Models — Catalog & Selection Guide

### Model Catalog

| Model ID | Type | Description | Languages | Char Limit | Latency |
|----------|------|-------------|-----------|------------|---------|
| `eleven_v3` | TTS / Dialogue | Most expressive, emotional range, multi-speaker dialogue | 70+ | 5,000 | Higher |
| `eleven_multilingual_v2` | TTS | Lifelike, emotionally-aware, stable on long-form | 29 | 10,000 | Medium |
| `eleven_flash_v2_5` | TTS | Ultra-fast, cost-effective, real-time | 32 | 40,000 | ~75ms |
| `eleven_flash_v2` | TTS | Ultra-fast, English-only | 1 (en) | 30,000 | ~75ms |
| `eleven_ttv_v3` | Voice Design | Text-to-voice design model | 70+ | — | — |
| `eleven_multilingual_ttv_v2` | Voice Design | Multilingual voice designer | 29 | — | — |
| `eleven_multilingual_sts_v2` | Voice Changer | Multilingual speech-to-speech | 29 | 10,000 | — |
| `eleven_english_sts_v2` | Voice Changer | English-only speech-to-speech | 1 (en) | 10,000 | — |
| `scribe_v2` | STT (Batch) | State-of-the-art speech recognition | 90+ | — | — |
| `scribe_v2_realtime` | STT (Realtime) | Real-time streaming recognition | 90+ | — | ~150ms |
| `eleven_text_to_sound_v2` | Sound Effects | Text-to-sound generation | N/A | — | — |
| `music_v2` | Music | Studio-grade music generation (next-gen) | en, es, de, ja + | — | — |
| `music_v1` | Music | Studio-grade music (outclassed by v2) | en, es, de, ja + | — | — |

### Deprecated Models

| Model ID | Replacement |
|----------|-------------|
| `eleven_turbo_v2_5` | `eleven_flash_v2_5` |
| `eleven_turbo_v2` | `eleven_flash_v2` |
| `scribe_v1` | `scribe_v2` |

### Selection Guide

| Use Case | Recommended Model | Rationale |
|----------|-------------------|-----------|
| Highest fidelity, emotional content | `eleven_v3` | Most expressive, multi-speaker |
| Professional narration, audiobooks | `eleven_multilingual_v2` | Stable, high quality, 10k char |
| Real-time / conversational agents | `eleven_flash_v2_5` | ~75ms latency, 40k char, 32 languages |
| English-only real-time | `eleven_flash_v2` | Lowest cost, English-focused |
| Multilingual voice changing | `eleven_multilingual_sts_v2` | Preserves emotion across voices |
| Batch transcription | `scribe_v2` | 90+ languages, diarization, entities |
| Live transcription | `scribe_v2_realtime` | ~150ms latency, streaming |
| Music generation | `music_v2` | Improved prompt adherence, inpainting |

### Character Limits per Request

| Model | Limit | Approx. Duration |
|-------|-------|-----------------|
| `eleven_v3` | 5,000 | ~5 minutes |
| `eleven_flash_v2_5` | 40,000 | ~40 minutes |
| `eleven_flash_v2` | 30,000 | ~30 minutes |
| `eleven_multilingual_v2` | 10,000 | ~10 minutes |
| `eleven_english_sts_v2` | 10,000 | ~10 minutes |

For longer content, split into segments and use `previous_text`/`next_text`/`previous_request_ids`/`next_request_ids` for prosody continuity.

---

## 4. Text to Speech (TTS)

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/text-to-speech/{voice_id}` | POST | Create speech (buffered) |
| `/v1/text-to-speech/{voice_id}/stream` | POST | Stream speech (chunked transfer) |
| `/v1/text-to-speech/{voice_id}/stream-input` | WebSocket | WebSocket streaming TTS |
| `/v1/text-to-speech/{voice_id}/multi-stream-input` | WebSocket | Multi-context WebSocket (multiple contexts on one connection) |
| `/v1/text-to-speech/{voice_id}/with-timestamps` | POST | Create speech with character-level timestamps |
| `/v1/text-to-speech/{voice_id}/stream/with-timestamps` | POST | Stream speech with timestamps |

### Main Concepts

- **Voice ID** — Path parameter identifying which voice to use. Browse at `elevenlabs.io/app/voice-library` or list via the Voices API.
- **Output Format** — Query parameter `output_format`, formatted as `codec_sample_rate_bitrate` (e.g. `mp3_44100_128`).
- **Voice Settings** — Per-request overrides for stability, similarity, style, speed, speaker boost.
- **Request Stitching** — Use `previous_text`/`next_text` or `previous_request_ids`/`next_request_ids` to maintain prosody across chunked generations. Max 3 request IDs.
- **Seed** — Integer (0–4294967295) for deterministic sampling. Determinism is not guaranteed but improves consistency.
- **Text Normalization** — `apply_text_normalization`: `auto` (default), `on`, `off`. Controls number/date spelling. Enterprise-only for Flash v2.5.
- **Language Code** — ISO 639-1 code to enforce a language (not supported for `eleven_multilingual_v2`).

### Request Body Parameters (POST `/v1/text-to-speech/{voice_id}`)

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `text` | string | **Yes** | — | Text to convert to speech |
| `model_id` | string | No | `eleven_multilingual_v2` | Model identifier |
| `language_code` | string\|null | No | null | ISO 639-1 language enforcement |
| `voice_settings` | object\|null | No | voice defaults | Stability, similarity, style, speed overrides |
| `pronunciation_dictionary_locators` | array\|null | No | null | Up to 3 pronunciation dictionary locators |
| `seed` | integer\|null | No | null | Deterministic sampling seed (0–4294967295) |
| `previous_text` | string\|null | No | null | Text before current (for stitching) |
| `next_text` | string\|null | No | null | Text after current (for stitching) |
| `previous_request_ids` | array\|null | No | null | Up to 3 prior request IDs |
| `next_request_ids` | array\|null | No | null | Up to 3 following request IDs |
| `use_pvc_as_ivc` | boolean | No | false | Use IVC version instead of PVC (latency workaround) |
| `apply_text_normalization` | enum | No | `auto` | `auto` / `on` / `off` |
| `apply_language_text_normalization` | boolean | No | false | Language-specific normalization (currently Japanese only; increases latency) |

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enable_logging` | boolean | true | Set `false` for Zero Retention Mode (Enterprise only) |
| `optimize_streaming_latency` | integer\|null | null | 0=default, 1=50% improvement, 2=75%, 3=max, 4=max+text-normalizer-off |
| `output_format` | enum | `mp3_44100_128` | Audio output format (see [§19](#19-audio-output-formats-reference)) |

### Voice Settings Object

```json
{
  "stability": 0.5,
  "similarity_boost": 0.75,
  "style": 0,
  "use_speaker_boost": true,
  "speed": 1.0
}
```

| Field | Type | Default | Range | Description |
|-------|------|---------|-------|-------------|
| `stability` | float | 0.5 | 0–1 | Lower = broader emotional range; higher = monotonous |
| `similarity_boost` | float | 0.75 | 0–1 | How closely to adhere to original voice |
| `style` | float | 0 | 0–1 | Style exaggeration amplification |
| `use_speaker_boost` | boolean | true | — | Boost similarity to original speaker (increases latency) |
| `speed` | float | 1.0 | 0.7–1.2 | Speech speed multiplier |

### WebSocket Streaming (stream-input)

The WebSocket TTS endpoint enables real-time generation by sending text chunks and receiving audio chunks:

```
Connection: wss://api.elevenlabs.io/v1/text-to-speech/{voice_id}/stream-input
            ?output_format=mp3_44100_128&auto_mode=true

1. Send initialization message: {"text": " "}
2. Send text chunks: {"text": "Hello world"}
3. Send flush: {"text": ""}
4. Receive: {"audio": "<base64>", "isFinal": true/false, ...}
```

### Prompting & Emotion

The TTS models interpret emotional context directly from text:
- Descriptive text like `"she said excitedly"` influences delivery
- Exclamation marks affect emotion
- Audio tags (v3 only): `[giggling]`, `[whispering]`, `[sad]`, `[applause]` — wrapped in square brackets within text

### Free Regenerations

Up to 2 free regenerations per generation when the exact same text and parameters are used. Useful for fixing audio distortion.

---

## 5. Speech to Text (STT) — Batch & Realtime

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/speech-to-text` | POST | Create transcript (batch, synchronous or async via webhook) |
| `/v1/speech-to-text/{transcript_id}` | GET | Get transcript (for async results) |
| `/v1/speech-to-text/{transcript_id}` | DELETE | Delete transcript |
| `/v1/speech-to-text/realtime` | WebSocket | Real-time streaming transcription |

### Main Concepts

- **Transcript** — The full text output with word-level detail: text, start/end timestamps, type (`word`/`spacing`/`audio_event`), speaker_id, logprob, characters.
- **Diarization** — Identifies which speaker is talking. Supports up to 32 speakers. Configurable via `diarize` and `diarization_threshold`.
- **Entity Detection** — Detects PII, PHI, PCI, and other entity types with character positions. Incurs 30% surcharge.
- **Entity Redaction** — Redacts detected entities from transcript text. Modes: `redacted`, `entity_type`, `enumerated_entity_type`.
- **Keyterm Prompting** — Bias transcription toward specific terms. Batch: up to 1000 keyterms (50 chars each). Realtime: up to 50 (20 chars each). Incurs 20% surcharge.
- **Multichannel** — Process each channel independently (up to 5 channels). Output style: `separate` (one transcript per channel) or `combined` (merged by time).
- **Additional Formats** — Export transcript to SRT, TXT, DOCX, HTML, PDF, or segmented JSON alongside the JSON response.
- **Speaker Roles** — Detect `agent` vs `customer` roles (requires diarize=true, 10% surcharge).
- **Speaker Library** — Match detected speakers against workspace's registered speakers.
- **No Verbatim Mode** — Removes filler words, false starts, and disfluencies for cleaner output.
- **Smart Language Detection** — Automatically detects language with a confidence score.

### Request Parameters (POST `/v1/speech-to-text`)

Content-Type: `multipart/form-data`

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `model_id` | enum | **Yes** | — | `scribe_v2` or `scribe_v1` |
| `file` | binary | One of file/source_url | — | Audio/video file to transcribe (<5GB) |
| `source_url` | string | One of file/source_url | — | URL to audio/video (supports YouTube, TikTok, hosted files) |
| `language_code` | string\|null | No | null | ISO-639-1 or ISO-639-3 language hint |
| `tag_audio_events` | boolean | No | true | Tag non-speech sounds (laughter, footsteps) |
| `num_speakers` | int\|null | No | null | Max speakers (up to 32). Helps diarization. |
| `timestamps_granularity` | enum | No | `word` | `none` / `word` / `character` |
| `diarize` | boolean | No | false | Annotate speaker turns |
| `diarization_threshold` | float\|null | No | null (model-dependent, ~0.22) | Higher = fewer speakers; lower = more speakers |
| `use_multi_channel` | boolean | No | false | Transcribe each channel independently (max 5) |
| `multichannel_output_style` | enum | No | `separate` | `separate` / `combined` |
| `file_format` | enum | No | `other` | `pcm_s16le_16` (16-bit PCM @ 16kHz mono LE) or `other` |
| `additional_formats` | array | No | — | Export to SRT/TXT/DOCX/HTML/PDF/JSON |
| `entity_detection` | string\|array\|null | No | null | `all`, category (`pii`, `phi`, `pci`, `other`, `offensive_language`), or entity types |
| `entity_redaction` | string\|array\|null | No | null | Redact entities (subset of entity_detection) |
| `entity_redaction_mode` | enum | No | `enumerated_entity_type` | `redacted` / `entity_type` / `enumerated_entity_type` |
| `keyterms` | array | No | [] | Up to 1000 terms to bias transcription |
| `no_verbatim` | boolean | No | false | Remove filler words (scribe_v2 only) |
| `use_speaker_library` | boolean | No | false | Match against workspace speaker library |
| `detect_speaker_roles` | boolean | No | false | Detect agent/customer roles (10% surcharge) |
| `temperature` | float\|null | No | null (model-dependent) | Randomness 0.0–2.0 |
| `seed` | int\|null | No | null | Deterministic sampling (0–2147483647) |
| `webhook` | boolean | No | false | Send results to configured webhooks (async) |
| `webhook_id` | string\|null | No | null | Specific webhook to send to |
| `webhook_metadata` | JSON string\|null | No | null | Custom metadata (max 16KB, 2 levels deep) |

### Response Structure (Single Channel)

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
      "type": "word",
      "speaker_id": "speaker_0",
      "logprob": -0.124,
      "characters": [{ "text": "H", "start": 0.0, "end": 0.1 }, ...]
    }
  ],
  "transcription_id": "abc123",
  "entities": [
    { "text": "John Smith", "entity_type": "person_name", "start_char": 0, "end_char": 10 }
  ],
  "audio_duration_secs": 15.2
}
```

### Word Types

| Type | Description |
|------|-------------|
| `word` | A spoken word |
| `spacing` | Space between words (not applicable for Japanese, Mandarin, Thai, Lao, Burmese, Cantonese) |
| `audio_event` | Non-speech sounds (laughter, applause, footsteps) |

### Additional Export Formats

| Format | Key Options |
|--------|-------------|
| `srt` | `max_characters_per_line` (42), `segment_on_silence_longer_than_s` (0.8), `max_segment_duration_s` (4), `max_segment_chars` (84), `include_speakers` (false) |
| `txt` | `max_characters_per_line` (100), `include_speakers` (true), `include_timestamps` (true) |
| `docx` | `include_speakers`, `include_timestamps`, segmentation options |
| `html` | `include_speakers`, `include_timestamps`, segmentation options |
| `pdf` | `include_speakers`, `include_timestamps`, segmentation options |
| `segmented_json` | `include_speakers`, `include_timestamps`, segmentation options |

### Realtime STT (WebSocket)

```
Connection: wss://api.elevenlabs.io/v1/speech-to-text/realtime

Features:
- Ultra-low latency: ~150ms for partial transcriptions
- Streaming: send audio chunks, receive transcripts in real-time
- Audio formats: PCM (8kHz–48kHz), μ-law encoding
- Voice Activity Detection (VAD): automatic speech segmentation
- Manual commit control: finalize transcript segments on demand
- Smart language detection: auto-detect language
- Keyterm prompting: up to 50 keyterms (20 chars each)
- No verbatim mode available
```

### Concurrency for STT

Files over 8 minutes are transcribed in parallel internally (chunked into 4 segments):

```
Concurrency = min(4, round_up(audio_duration_secs / 480))
```

### Supported Input Formats

**Audio:** AAC, AIFF, OGG, MP3, OPUS, WAV, FLAC, M4A, WebM
**Video:** MP4, AVI, MKV, MOV, WMV, FLV, WebM, MPEG, 3GPP

| Limit | Value |
|-------|-------|
| Max file size | 3 GB (batch), 500 MB (realtime upload) |
| Max duration | 10 hours (standard), <10 hours combined (multichannel) |
| Max text length | 675,000 characters |
| Min audio length | 100ms |

---

## 6. Text to Dialogue

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/text-to-dialogue` | POST | Create dialogue (buffered) |
| `/v1/text-to-dialogue/stream` | POST | Stream dialogue |
| `/v1/text-to-dialogue/with-timestamps` | POST | Create dialogue with timestamps |
| `/v1/text-to-dialogue/stream/with-timestamps` | POST | Stream dialogue with timestamps |

### Main Concepts

- **Dialogue Turn** — Each entry in the `inputs` array has its own `text` and `voice_id`, representing one speaker's turn.
- **Multi-speaker** — No limit on the number of speakers. Each turn can use a different `voice_id`.
- **Audio Tags** — Inline directives in square brackets (e.g. `[giggling]`, `[whispering]`) influence delivery per turn.
- **Eleven v3 only** — Text to Dialogue is exclusively available with the `eleven_v3` model.
- **Not for real-time** — Not intended for conversational agents. Multiple generations may be needed.

### Request Structure

```json
{
  "inputs": [
    { "text": "[giggling] That's really funny!", "voice_id": "JBFqnCBsd6RMkjVDRZzb" },
    { "text": "[groaning] That was awful.", "voice_id": "Xb7hHDs7RMkjVDRZzb" }
  ],
  "model_id": "eleven_v3",
  "output_format": "mp3_44100_128",
  "seed": 42
}
```

### Key Constraints

| Constraint | Value |
|------------|-------|
| Model | `eleven_v3` only |
| Total text length | ≤ 2,000 characters across all `inputs[].text` |
| Speakers | Unlimited |
| Determinism | Use `seed` parameter (not guaranteed) |
| Free regenerations | Up to 2 (dashboard only, same params) |

### Audio Tag Categories

| Category | Examples |
|----------|---------|
| Emotions/delivery | `[sad]`, `[laughing]`, `[whispering]`, `[cautiously]`, `[elated]` |
| Audio events | `[leaves rustling]`, `[gentle footsteps]`, `[applause]` |
| Overall direction | `[football]`, `[wrestling match]`, `[auctioneer]` |

### Punctuation for Dialogue Flow

- Interruptions: `"Hello, is this seat-"` / `"[jumping in] Free?"`
- Trailing sentences (ellipsis): `"Hi, can I get uhhh..."`

---

## 7. Voices — Cloning, Design, Remixing & Library

### Voice Management Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/voices` | GET | List/search voices |
| `/v1/voices/{voice_id}` | GET | Get voice details |
| `/v1/voices/{voice_id}` | DELETE | Delete a voice |
| `/v1/voices/{voice_id}` | POST | Edit voice metadata |
| `/v1/voices/{voice_id}/settings` | GET | Get voice settings |
| `/v1/voices/{voice_id}/settings` | POST | Edit voice settings |
| `/v1/voices/settings/default` | GET | Get default voice settings |
| `/v1/voices/find-similar` | GET | List similar voices |
| `/v1/voices/voice-library/shared` | GET | List shared voices (Voice Library) |
| `/v1/voices/voice-library/share` | POST | Add shared voice to library |

### Voice Types

| Type | Description | Requirements |
|------|-------------|--------------|
| **Community** | Voices from the Voice Library (10,000+) | Not available to free tier via API |
| **Cloned (IVC)** | Instant Voice Clone from short audio (<2 min) | Most plans |
| **Cloned (PVC)** | Professional Voice Clone from extended audio | Creator plan+ |
| **Voice Design** | Generated from text description (20–1000 chars) | — |
| **Voice Remix** | Transform existing voice attributes | Own voices + infinite-notice Voice Library voices |

### Instant Voice Cloning (IVC)

**Endpoint:** `POST /v1/voices/ivc/create`

Quickly clones a voice from short audio samples. No training step required.

### Professional Voice Cloning (PVC)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/voices/pvc/create` | POST | Create PVC voice |
| `/v1/voices/pvc/{voice_id}` | POST | Update PVC voice |
| `/v1/voices/pvc/{voice_id}/train` | POST | Train PVC voice |
| `/v1/voices/pvc/{voice_id}/samples` | POST | Add samples |
| `/v1/voices/pvc/{voice_id}/samples/{sample_id}` | POST | Update sample |
| `/v1/voices/pvc/{voice_id}/samples/{sample_id}` | DELETE | Delete sample |
| `/v1/voices/pvc/{voice_id}/samples/{sample_id}/audio` | GET | Get sample audio |
| `/v1/voices/pvc/{voice_id}/samples/{sample_id}/waveform` | GET | Get sample waveform |
| `/v1/voices/pvc/{voice_id}/verification/request` | POST | Request manual verification |
| `/v1/voices/pvc/{voice_id}/verification/captcha` | GET | Get verification captcha |
| `/v1/voices/pvc/{voice_id}/verification/captcha/verify` | POST | Verify captcha |
| `/v1/voices/pvc/{voice_id}/samples/speaker-separation` | POST | Start speaker separation |
| `/v1/voices/pvc/{voice_id}/samples/speaker-separation/status` | GET | Get separation status |
| `/v1/voices/pvc/{voice_id}/samples/separated-speaker-audio` | GET | Get separated speaker audio |

**PVC Workflow:**
```
1. Create PVC voice ──▶ 2. Add audio samples ──▶ 3. Train ──▶ 4. Verify captcha (voice-captcha)
                                                                  └─ Ensures clone is from your own voice
```

### Voice Design

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/text-to-voice/design` | POST | Design a voice (returns 3 previews) |
| `/v1/text-to-voice/create` | POST | Create a voice from a designed preview |
| `/v1/text-to-voice/remix` | POST | Remix an existing voice |
| `/v1/text-to-voice/stream` | POST | Stream voice preview |

**Design Input:**
- `voice_description`: 20–1000 characters (e.g. "A raspy middle-aged male with a slight Southern drawl")
- Optional `text`: 100–1000 characters for preview
- Model: `eleven_ttv_v3` (v3, 70+ languages) or `eleven_multilingual_ttv_v2` (29 languages)

### Voice Remixing

**Endpoint:** `POST /v1/text-to-voice/remix`

Transforms existing voices by modifying attributes through natural language prompts while maintaining recognizable characteristics.

| Parameter | Description |
|-----------|-------------|
| `prompt_strength` | `low` / `medium` / `high` / `max` — controls transformation intensity |
| Script options | Default scripts or custom text with v3 audio tags (`[laughs]`, `[whispers]`) |
| Input voices | Any owned cloned voice, Voice Design voice, or Voice Library voice with infinite notice period |
| Output | Full-quality v3 voice (backward compatible with v2 models), iteratively remixable |

**Remixing attributes:** gender, accent, speaking style, pacing, audio quality.

---

## 8. Voice Changer (Speech to Speech)

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/speech-to-speech/{voice_id}` | POST | Voice changer (buffered) |
| `/v1/speech-to-speech/{voice_id}/stream` | POST | Voice changer (streamed) |

### Main Concepts

Transforms source audio into a different voice while preserving emotional delivery, nuances, whispers, laughs, and accents. Billing: 1,000 characters per minute of processed audio.

### Key Parameters

| Parameter | Description |
|-----------|-------------|
| `voice_id` | Target voice (path parameter) — any cloned/designed/library voice |
| `model_id` | `eleven_multilingual_sts_v2` (recommended) or `eleven_english_sts_v2` |
| `remove_background_noise` | Boolean — minimize environmental sounds in output |
| `audio` | Source audio file (multipart upload) |

### Constraints

| Constraint | Value |
|------------|-------|
| Max segment length | 5 minutes (split longer recordings) |
| Models | `eleven_multilingual_sts_v2` (29 langs), `eleven_english_sts_v2` (English) |
| Billing | 1,000 characters per minute of audio |

---

## 9. Voice Isolator (Audio Isolation)

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/audio-isolation` | POST | Audio isolation (buffered) |
| `/v1/audio-isolation/stream` | POST | Audio isolation (streamed) |
| `/v1/audio-isolation` | GET | Get isolation history |
| `/v1/audio-isolation/{id}` | DELETE | Delete history item |

### Main Concepts

Extracts clean speech from audio with background noise, music, or ambient sounds. Produces studio-quality isolated speech. Billing: 1,000 characters per minute of audio.

### Constraints

| Constraint | Value |
|------------|-------|
| Max file size | 500 MB |
| Max duration | 1 hour |
| Billing | 1,000 characters per minute |
| Music vocals | Not specifically optimized for vocal isolation from music |
| Input formats | Audio: AAC, AIFF, OGG, MP3, OPUS, WAV, FLAC, M4A. Video: MP4, AVI, MKV, MOV, WMV, FLV, WEBM, MPEG, 3GPP |

---

## 10. Sound Effects

### Endpoint

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/text-to-sound-effects` | POST | Create sound effect |

**Model:** `eleven_text_to_sound_v2`

### Main Concepts

Generates high-quality audio effects from text descriptions. The model understands natural language and audio terminology.

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `text` | string | — | Text description of the sound effect |
| `duration_seconds` | float\|null | auto | 0.1–30 seconds. Cost: 40 credits/second when specified. |
| `prompt_influence` | float | — | High = literal; Low = creative variations |
| `loop` | boolean | false | Seamless looping for effects >30s |

### Constraints

| Constraint | Value |
|------------|-------|
| Max duration | 30 seconds per generation |
| Output formats | MP3 (all effects); WAV at 48 kHz (non-looping) |
| Looping | Seamless repeat playback — no audible start/end |
| Musical elements | Drum loops, bass lines, melodic samples supported |

### Prompting Categories

| Category | Examples |
|----------|---------|
| Simple effects | "Glass shattering on concrete", "Thunder rumbling in the distance" |
| Complex sequences | "Footsteps on gravel, then a metallic door opens" |
| Musical elements | "90s hip-hop drum loop, 90 BPM", "Vintage brass stabs in F minor" |
| Audio terminology | Impact, Whoosh, Ambience, One-shot, Loop, Stem, Braam, Glitch, Drone |

---

## 11. Music Generation

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/music/compose` | POST | Compose music (simple prompt) |
| `/v1/music/compose/stream` | POST | Stream music |
| `/v1/music/compose-detailed` | POST | Compose music with detailed parameters |
| `/v1/music/compose-detailed/stream` | POST | Stream music with details |
| `/v1/music/create-composition-plan` | POST | Create structured composition plan (JSON) |
| `/v1/music/video-to-music` | POST | Video to music |
| `/v1/music/upload` | POST | Upload music file |
| `/v1/music/separate-stems` | POST | Stem separation |

### Main Concepts

- **Prompt** — Natural language description of desired music (genre, style, mood, instruments)
- **Composition Plan** — Structured JSON for precise control over sections, genres, lyrics, and transitions
- **Inpainting** — Edit and combine sections of existing songs
- **Stem Separation** — Separate a song into individual instrument/vocal stems
- **Finetunes** — Fine-tune the music model on your own audio for consistent style
- **Multilingual** — English, Spanish, German, Japanese, and more
- **Vocals or instrumental** — Control whether vocals are included

### Models

| Model | Description |
|-------|-------------|
| `music_v2` | Next-gen: improved prompt adherence, composition, long-form sections, mid-track genre transitions, fast rap, complex vocals, improved inpainting, embedded sound effects |
| `music_v1` | Previous generation (outclassed by v2, still available during transition) |

### Key Constraints

| Constraint | Value |
|------------|-------|
| Min duration | 3 seconds |
| Max duration | 5 minutes |
| Output formats | MP3 (44.1kHz, 128–192kbps), WAV |
| Commercial use | Cleared for nearly all commercial uses (see music terms) |

### Music Finetunes

Fine-tune the model on your own non-copyrighted tracks:
- Upload tracks → automatic copyright screening → ready in 5–10 minutes
- Captures instrumentation, tempo, production style, timbre, vocal character
- Curated finetunes available (Afro House, Reggaeton, Arabic Groove, 70s Cambodian Rock, 80s Nu-Disco, Mozart Symphony)
- Currently only available for Music v1

---

## 12. Dubbing

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/dubbing` | POST | Dub a video or audio file |
| `/v1/dubbing` | GET | List dubs |
| `/v1/dubbing/{dubbing_id}` | GET | Get dubbing details |
| `/v1/dubbing/{dubbing_id}` | DELETE | Delete dubbing |
| `/v1/dubbing/{dubbing_id}/audio` | GET | Get dubbed audio |
| `/v1/dubbing/{dubbing_id}/transcripts` | GET | Retrieve transcript |

### Main Concepts

Translates audio/video across 90+ languages while preserving emotion, timing, tone, and speaker characteristics.

- **Speaker separation** — Automatically detects multiple speakers, even with overlapping speech
- **Multi-language output** — 90+ languages
- **Preserve original voices** — Retains speaker identity and emotional tone
- **Keep background audio** — Avoids re-mixing music, effects, ambient sounds
- **Cloning strength** — Configurable (default 7). Higher = more voice similarity, less natural cross-language. Lower = more natural delivery, less resemblance.
- **Source inputs** — YouTube, X, TikTok, Vimeo, direct URLs, or file uploads

### Constraints

| Constraint | Value |
|------------|-------|
| Dubbing v2 upload limit | 2 GB, 180 minutes |
| Dubbing Studio (V1) limit | 1 GB, 45 minutes (maintenance mode) |
| Recommended speakers | Up to 9 per file |
| Concurrency (self-serve) | 5 concurrent jobs |
| Concurrency (Enterprise) | 100 concurrent jobs |
| Watermark | Free tier auto-watermarked; paid tier no watermark |
| Realtime dubbing | Not available |

---

## 13. Forced Alignment

### Endpoint

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/forced-alignment` | POST | Create forced alignment |

### Main Concepts

Turns spoken audio + text transcript into a time-aligned transcript with exact word/phrase timestamps. Useful for subtitle timing and audiobook chapter alignment.

### Parameters

| Parameter | Description |
|-----------|-------------|
| `audio_file` | Audio file (multipart) |
| `text` | Plain string transcript (NOT JSON-wrapped) |
| `model_id` | Alignment model |

### Constraints

| Constraint | Value |
|------------|-------|
| Input text format | Plain string only — no JSON wrapping |
| Diarization | Not supported — diarized text produces unexpected results |
| Max file size | 3 GB |
| Max audio duration | 10 hours |
| Max text length | 675,000 characters |
| Pricing | Same as Speech to Text API |
| Languages | 29 (same as `eleven_multilingual_v2`) |

---

## 14. Speech Engine — Conversational Voice with Custom LLM

### Endpoints

| Endpoint | Protocol | Description |
|----------|----------|-------------|
| `/v1/speech-engine` | WebSocket | Bidirectional audio + transcript stream |
| `/v1/speech-engines` | POST | Create Speech Engine configuration |
| `/v1/speech-engines/{id}` | GET | Get Speech Engine |
| `/v1/speech-engines/{id}` | POST | Update Speech Engine |
| `/v1/speech-engines/{id}` | DELETE | Delete Speech Engine |
| `/v1/speech-engines` | GET | List Speech Engines |

### Main Concepts

Speech Engine adds voice capabilities to any chat agent where **you bring your own LLM**. ElevenLabs handles STT and TTS; your server provides the LLM logic. The SDK manages connection lifecycle, turn-taking, and interruption detection.

### Architecture

```
Browser ──▶ ElevenLabs (captures audio, transcribes)
                │
                ▼ WebSocket
          Your Server (Speech Engine SDK)
                │
                ▼
            Your LLM (OpenAI, Anthropic, Gemini, or any text model)
                │
                ▼ (streamed response)
          Speech Engine SDK
                │
                ▼ WebSocket
          ElevenLabs (TTS) ──▶ Browser (audio playback)
```

### Key Features

- **Any LLM** — OpenAI, Anthropic, Google Gemini, or any text-producing model. SDK auto-extracts text from OpenAI/Anthropic/Gemini stream formats.
- **Interruption handling** — User speaking mid-response cancels in-flight LLM request via `AbortSignal` (TS) / task cancellation (Python).
- **Streaming** — Pass a string, async iterable, or native LLM stream object.
- **Turn-taking** — SDK manages conversation turns; server only responds to transcripts.
- **IP allowlisting** — Static egress IPs available for firewall configuration.

### Static Egress IPs

| Region | IP Address |
|--------|-----------|
| US (Default) | 34.67.146.145, 34.59.11.47 |
| EU | 35.204.38.71, 34.147.113.54 |
| Asia | 35.185.187.110, 35.247.157.189 |
| EU Residency | 34.77.234.246, 34.140.184.144 |
| India Residency | 34.93.26.174, 34.93.252.69 |
| Singapore Residency | 34.87.23.17, 34.126.179.103 |

### SDKs

| SDK | Package |
|-----|---------|
| Python | `elevenlabs` (Speech Engine module) |
| JavaScript | `@elevenlabs/elevenlabs-js` (Speech Engine module) |

**Python:**
```python
# Standalone server
engine.serve()

# Or integrate with ASGI framework
engine.create_session()  # FastAPI, Starlette, etc.
```

**TypeScript:**
```typescript
// Attach to any Node.js HTTP server (Express, Fastify, http.createServer())
// Or run standalone WebSocket server
```

---

## 15. Pronunciation Dictionaries

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/pronunciation-dictionaries/from-file` | POST | Create from file |
| `/v1/pronunciation-dictionaries/from-rules` | POST | Create from rules |
| `/v1/pronunciation-dictionaries/{id}` | GET | Get dictionary |
| `/v1/pronunciation-dictionaries/{id}` | POST | Update dictionary |
| `/v1/pronunciation-dictionaries/{id}/download` | GET | Get by version |
| `/v1/pronunciation-dictionaries` | GET | List dictionaries |
| `/v1/pronunciation-dictionaries/{id}/rules` | POST | Set rules |
| `/v1/pronunciation-dictionaries/{id}/rules/add` | POST | Add rules |
| `/v1/pronunciation-dictionaries/{id}/rules/remove` | POST | Remove rules |

### Main Concepts

Versioned sets of phonetic rules that override how specific words/phrases are pronounced. Attached to TTS requests via `pronunciation_dictionary_locators` (up to 3 per request).

**Locator structure:**
```json
{
  "pronunciation_dictionary_id": "dict_id",
  "version_id": "version_id_or_null_for_latest"
}
```

---

## 16. Audio Streaming Architecture

### Streaming vs. Buffered

| Mode | Description | Use Case |
|------|-------------|----------|
| **Buffered** | Returns complete audio file after generation completes | Pre-generated content, files |
| **HTTP Streaming** | Chunked transfer encoding — audio bytes arrive as generated | Low-latency playback, progressive delivery |
| **WebSocket** | Bidirectional — send text chunks, receive audio chunks | Real-time conversational agents, interactive TTS |

### Streaming Endpoints

| Capability | HTTP Stream | WebSocket |
|-----------|-------------|-----------|
| Text to Speech | `/v1/text-to-speech/{voice_id}/stream` | `/v1/text-to-speech/{voice_id}/stream-input` |
| Text to Dialogue | `/v1/text-to-dialogue/stream` | — |
| Voice Changer | `/v1/speech-to-speech/{voice_id}/stream` | — |
| Audio Isolation | `/v1/audio-isolation/stream` | — |
| Speech to Text | — | `/v1/speech-to-text/realtime` |
| Music | `/v1/music/compose/stream` | — |
| Speech Engine | — | `/v1/speech-engine` (upstream) |

### Multi-Context WebSocket

The multi-context WebSocket (`/v1/text-to-speech/{voice_id}/multi-stream-input`) allows building real-time voice agents by maintaining multiple conversation contexts on a single WebSocket connection. Only the time where the model is generating audio counts toward concurrency limits.

### Concurrency Model: HTTP vs WebSocket

- **HTTP:** Each request counts individually toward concurrency limit
- **WebSocket:** Only model generation time counts; open connections don't count toward concurrency for most of the time

---

## 17. Concurrency, Limits & Pricing

### Concurrency Limits by Plan

| Plan | Multilingual v2 | Flash | STT | Realtime STT | Music | Priority |
|------|-----------------|-------|-----|-------------|-------|----------|
| Free | 2 | 4 | 8 | 6 | 0 | 3 |
| Starter | 3 | 6 | 12 | 9 | 2 | 4 |
| Creator | 5 | 10 | 20 | 15 | 2 | 5 |
| Pro | 10 | 20 | 40 | 30 | 2 | 5 |
| Scale | 15 | 30 | 60 | 45 | 5 | 5 |
| Business | 15 | 30 | 60 | 45 | 5 | 5 |
| Enterprise | Elevated | Elevated | Elevated | Elevated | Highest | 6 |

### Concurrency Heuristic

A concurrency limit of 5 can typically support ~100 simultaneous audio broadcasts, because audio generation is much faster than audio playback.

### Dubbing Concurrency

| Plan Type | Concurrent Jobs |
|-----------|----------------|
| Self-serve (Free–Business) | 5 |
| Enterprise | 100 |

### Billing Units

| Capability | Billing Unit |
|-----------|-------------|
| Text to Speech | Characters (varies by model) |
| Voice Changer | 1,000 characters per minute of audio |
| Voice Isolator | 1,000 characters per minute of audio |
| Sound Effects | 40 credits per second (when duration specified) |
| Speech to Text | Per hour of audio |
| Music | Per generation (varies by settings/duration) |
| Dubbing | Per minute (USD) |
| Forced Alignment | Same as Speech to Text |
| Entity Detection | +30% surcharge on base STT cost |
| Keyterm Prompting | +20% surcharge (min 20s billable when >100 keyterms) |
| Speaker Role Detection | +10% surcharge |
| Entity Redaction | +30% surcharge |

### Free Regenerations

Up to 2 free regenerations per generation when identical text and parameters are used. Dashboard-only for Text to Dialogue.

---

## 18. Webhooks

### Available Webhook Events

- **Speech to Text** — Asynchronous transcription completion (set `webhook=true` in STT request)
- **Dubbing** — Dubbing job completion
- **Post-call (Agents)** — Call end + analysis completion

### STT Webhook Configuration

When `webhook=true` in the STT request:
1. API returns early with `{ "message": "...", "request_id": "...", "transcription_id": "..." }`
2. Transcription result delivered to configured webhooks
3. Optional `webhook_id` to target a specific webhook
4. Optional `webhook_metadata` for correlation/tracking (max 16KB, 2 levels deep)

---

## 19. Audio Output Formats Reference

### Format Enum Values

| Format | Sample Rate | Bitrate | Notes |
|--------|-------------|---------|-------|
| `mp3_22050_32` | 22.05kHz | 32kbps | |
| `mp3_24000_48` | 24kHz | 48kbps | |
| `mp3_44100_32` | 44.1kHz | 32kbps | |
| `mp3_44100_64` | 44.1kHz | 64kbps | |
| `mp3_44100_96` | 44.1kHz | 96kbps | |
| `mp3_44100_128` | 44.1kHz | 128kbps | **Default** |
| `mp3_44100_192` | 44.1kHz | 192kbps | Creator tier+ |
| `pcm_8000` | 8kHz | — | 16-bit S16LE |
| `pcm_16000` | 16kHz | — | 16-bit S16LE |
| `pcm_22050` | 22.05kHz | — | 16-bit S16LE |
| `pcm_24000` | 24kHz | — | 16-bit S16LE |
| `pcm_32000` | 32kHz | — | 16-bit S16LE |
| `pcm_44100` | 44.1kHz | — | 16-bit S16LE, Pro tier+ |
| `pcm_48000` | 48kHz | — | 16-bit S16LE, Pro tier+ |
| `wav_8000` | 8kHz | — | |
| `wav_16000` | 16kHz | — | |
| `wav_22050` | 22.05kHz | — | |
| `wav_24000` | 24kHz | — | |
| `wav_32000` | 32kHz | — | |
| `wav_44100` | 44.1kHz | — | Pro tier+ |
| `wav_48000` | 48kHz | — | Pro tier+ |
| `ulaw_8000` | 8kHz | — | μ-law, telephony (Twilio) |
| `alaw_8000` | 8kHz | — | A-law, telephony |
| `opus_48000_32` | 48kHz | 32kbps | |
| `opus_48000_64` | 48kHz | 64kbps | |
| `opus_48000_96` | 48kHz | 96kbps | |
| `opus_48000_128` | 48kHz | 128kbps | |
| `opus_48000_192` | 48kHz | 192kbps | |

### Format Selection Guide

| Use Case | Recommended Format |
|----------|-------------------|
| General playback | `mp3_44100_128` (default) |
| Highest MP3 quality | `mp3_44100_192` (Creator+) |
| Telephony / Twilio | `ulaw_8000` or `alaw_8000` |
| Low latency streaming | `pcm_16000` or `pcm_24000` |
| High quality PCM | `pcm_44100` (Pro+) |
| Efficient streaming | `opus_48000_64` |
| WAV (lossless) | `wav_44100` (Pro+) |

---

## 20. Capability Summary & Cross-Reference

### Complete API Endpoint Map

| Capability | Endpoint Base | Methods | Protocol |
|-----------|--------------|---------|----------|
| **Text to Speech** | `/v1/text-to-speech/{voice_id}` | POST, Stream, WS, WS-multi | HTTP, WebSocket |
| **Text to Dialogue** | `/v1/text-to-dialogue` | POST, Stream | HTTP |
| **Speech to Text (Batch)** | `/v1/speech-to-text` | POST, GET, DELETE | HTTP |
| **Speech to Text (Realtime)** | `/v1/speech-to-text/realtime` | — | WebSocket |
| **Voice Changer** | `/v1/speech-to-speech/{voice_id}` | POST, Stream | HTTP |
| **Audio Isolation** | `/v1/audio-isolation` | POST, Stream, GET, DELETE | HTTP |
| **Sound Effects** | `/v1/text-to-sound-effects` | POST | HTTP |
| **Music** | `/v1/music/...` | POST (compose, stream, plan, upload, stems) | HTTP |
| **Dubbing** | `/v1/dubbing` | POST, GET, DELETE | HTTP |
| **Forced Alignment** | `/v1/forced-alignment` | POST | HTTP |
| **Speech Engine** | `/v1/speech-engine` + `/v1/speech-engines` | WS (stream), CRUD (config) | WebSocket, HTTP |
| **Voices** | `/v1/voices/...` | GET, POST, DELETE | HTTP |
| **IVC** | `/v1/voices/ivc/create` | POST | HTTP |
| **PVC** | `/v1/voices/pvc/...` | POST, GET, DELETE | HTTP |
| **Voice Design** | `/v1/text-to-voice/...` | POST (design, create, remix, stream) | HTTP |
| **Pronunciation Dictionaries** | `/v1/pronunciation-dictionaries/...` | GET, POST | HTTP |
| **Audio Native** | `/v1/audio-native/...` | POST, GET | HTTP |
| **Models** | `/v1/models` | GET | HTTP |

### Capability Decision Matrix

| If you need... | Use... | Key Parameters |
|----------------|--------|----------------|
| Text → spoken audio | TTS (`/v1/text-to-speech/{voice_id}`) | `text`, `model_id`, `voice_settings`, `output_format` |
| Multi-speaker dialogue from text | Text to Dialogue (`/v1/text-to-dialogue`) | `inputs[]` (text + voice_id per turn), `eleven_v3` |
| Audio → text transcript | STT Batch (`/v1/speech-to-text`) | `file`/`source_url`, `model_id`, `diarize`, `keyterms` |
| Live audio → text | STT Realtime (`/v1/speech-to-text/realtime`) | WebSocket, PCM/μ-law, VAD |
| Change voice in audio | Voice Changer (`/v1/speech-to-speech/{voice_id}`) | `audio`, `model_id`, `remove_background_noise` |
| Remove background noise | Voice Isolator (`/v1/audio-isolation`) | `audio` file |
| Text → sound effect | Sound Effects (`/v1/text-to-sound-effects`) | `text`, `duration_seconds`, `loop` |
| Text → music | Music (`/v1/music/compose`) | `text` prompt, `model_id` |
| Translate audio/video to another language | Dubbing (`/v1/dubbing`) | `file`/URL, source/target language |
| Align transcript to audio timestamps | Forced Alignment (`/v1/forced-alignment`) | `audio_file`, `text` (plain string) |
| Clone a voice quickly | IVC (`/v1/voices/ivc/create`) | Audio samples (<2 min) |
| Clone a voice professionally | PVC (`/v1/voices/pvc/create` → train → verify) | Extended audio, Creator+ |
| Design a new voice from text | Voice Design (`/v1/text-to-voice/design`) | `voice_description` (20–1000 chars) |
| Transform an existing voice | Voice Remix (`/v1/text-to-voice/remix`) | `voice_id`, `prompt_strength` |
| Build a conversational voice agent with own LLM | Speech Engine (`/v1/speech-engine`) | WebSocket, any LLM |
| Control pronunciation of specific words | Pronunciation Dictionaries | Create dict → attach via locators in TTS |
| Low-latency real-time TTS | WebSocket TTS (`stream-input`) | `auto_mode=true`, Flash models |
| Stitch multiple TTS chunks | `previous_text`/`next_text` or request IDs | Max 3 request IDs per direction |
| Deterministic TTS output | `seed` parameter | 0–4294967295 (not guaranteed) |
| Track generation costs | Response headers | `character-cost`, `request-id`, `x-trace-id` |