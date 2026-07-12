# Cartesia API Analysis — Text-to-Speech, Speech-to-Text, Voice Cloning, Voice Changing, Infill & Voice Agent Platform

> **Base URL:** `https://api.cartesia.ai` (REST/SSE) | **WebSocket:** `wss://api.cartesia.ai` | **API Version:** `2026-03-01`
> **Docs:** `https://docs.cartesia.ai` | **Playground:** `https://play.cartesia.ai` | **Auth:** API key (`sk_car_...`, Bearer) or short-lived JWT access token
> **SDKs:** `cartesia` (Python), `@cartesia/cartesia-js` (TypeScript/Node), `cartesia-line` (Python — Line agent SDK)
> **Description:** Cartesia is a voice AI platform providing real-time, ultra-low-latency APIs for text-to-speech (Sonic models), speech-to-text (Ink models), voice cloning, voice changing, audio infill, pronunciation dictionaries, and a managed voice agent platform called Line. Its flagship TTS model (Sonic 3.5) streams first audio byte in ~90ms across 42 languages, and its STT model (Ink 2) provides native turn detection with no separate VAD required. The platform is purpose-built for conversational AI, dubbing, narration, and real-time agent workloads.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Authentication & Access Control](#2-authentication--access-control)
3. [Models — Catalog & Selection Guide](#3-models--catalog--selection-guide)
4. [Text-to-Speech (TTS)](#4-text-to-speech-tts)
5. [Speech-to-Text (STT) — Batch & Realtime](#5-speech-to-text-stt--batch--realtime)
6. [Voices — Cloning, Localization & Library](#6-voices--cloning-localization--library)
7. [Voice Changer (Speech to Speech)](#7-voice-changer-speech-to-speech)
8. [Infill — Audio Bridging](#8-infill--audio-bridging)
9. [Pronunciation Dictionaries](#9-pronunciation-dictionaries)
10. [Line — Voice Agent Platform](#10-line--voice-agent-platform)
11. [Audio Streaming Architecture](#11-audio-streaming-architecture)
12. [Concurrency, Limits & Pricing](#12-concurrency-limits--pricing)
13. [Error Handling](#13-error-handling)
14. [Audio Output Formats Reference](#14-audio-output-formats-reference)
15. [Capability Summary & Cross-Reference](#15-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Cartesia's platform is organized around these core abstractions:

- **Voice** — A distinct speech identity identified by a `voice_id` (UUID). Voices come from the Voice Library (community-shared, public), Instant Voice Clones (from ≤10s audio clips), Pro Voice Clones (fine-tuned models from extended data), or voice localization (adapting an existing voice to a new language). Each voice has metadata: name, description, gender, language, country, and access controls (private/public).
- **Model** — The underlying AI model identified by a `model_id`. Two model families: **Sonic** (TTS: `sonic-3.5`, `sonic-3`, `sonic-latest`) and **Ink** (STT: `ink-2`, `ink-whisper`). Model IDs can be pinned to dated snapshots (e.g. `sonic-3.5-2026-05-04`) or use rolling aliases.
- **Credit** — The billing unit for TTS, STT, infill, and voice changing operations. Each subscription plan includes a monthly credit allotment. Agent (Line) usage is billed separately in USD per minute.
- **Context** — A unique identifier (`context_id`) for a speech generation session on the WebSocket TTS endpoint. Contexts maintain prosody across multiple input chunks (continuations) and support multiplexing multiple utterances on one connection.
- **Continuation** — A WebSocket TTS feature that allows streaming partial transcript chunks (e.g. LLM token-by-token output) into a single context. Set `continue: true` on partials and `continue: false` on the final chunk.
- **Turn** — A user speech segment in the STT (Auto) endpoint. Ink 2 detects turns natively via semantic + acoustic signals, emitting a lifecycle of events: `turn.start`, `turn.update`, `turn.eager_end`, `turn.resume`, `turn.end`.
- **Generation Config** — Per-request controls for TTS speech attributes: `volume` (0.5–2.0), `speed` (0.6–1.5), and `emotion` (60+ emotion values). Available on `sonic-3` and `sonic-3.5`.
- **SSML Tags** — Inline XML-like tags in transcripts: `<speed>`, `<volume>`, `<emotion>`, `<break>`, `<spell>` for mid-transcript control of speech delivery.
- **Pronunciation Dictionary** — A versioned set of text-to-pronunciation mappings that override how specific words are spoken. Referenced in TTS requests via `pronunciation_dict_id`.
- **Cartesia-Version Header** — A required header (`Cartesia-Version: 2026-03-01`) on all requests that pins the API behavior version. Enables backwards compatibility and timely deprecation notices.

### Platform Architecture

```
Text Input ──▶ Sonic Model ──▶ Audio Output (streamed as bytes, SSE events, or WebSocket messages)
                 │
     Voice Selection ──▶ Voice Library / Instant Clone / Pro Clone / Localized
     Generation Config ──▶ Speed, Volume, Emotion
     Pronunciation Dict ──▶ Override pronunciation
     SSML Tags ──▶ Inline mid-transcript controls (<speed>, <break>, <spell>, etc.)
     Context + Continuations ──▶ Stream partial transcripts (LLM token streaming)

Audio Input ──▶ Ink Model ──▶ Transcript (batch JSON or realtime turn events)
                 │
     Turn Detection (Auto) ──▶ turn.start / turn.update / turn.eager_end / turn.resume / turn.end
     Manual Finalization ──▶ send "finalize" command to emit transcripts
     Batch Upload ──▶ Single complete file transcription

Audio Input ──▶ Voice Changer ──▶ Same speech with a different voice (preserves intonation)
Two Audio Clips ──▶ Infill ──▶ Generated audio bridging between them
```

### End-to-End Flows

**Text to Speech pipeline (Bytes — simplest):**
```
Select voice_id ──▶ Choose model_id ──▶ POST /tts/bytes
                     │                         │
          optional: generation_config    returns: streaming audio bytes (raw/wav/mp3)
                   pronunciation_dict_id
                   language
                   output_format
```

**Real-time conversational agent (WebSocket TTS + WebSocket STT Auto):**
```
Browser ──▶ Cartesia STT (Auto) ──WebSocket──▶ turn.end event with transcript
                                                │
                          Your LLM processes transcript
                                                │
Browser ◀── Cartesia TTS (WebSocket) ◀──WebSocket── Your text response (streamed token-by-token)
```

**Line voice agent (managed platform):**
```
Phone/Web ──▶ Cartesia Line (managed runtime) ──▶ Your Agent code (LlmAgent)
                  │                                     │
           Ink 2 (STT) + Sonic 3.5 (TTS)         Any LLM via LiteLLM (100+ providers)
                  │                                     │
           Audio orchestration                    Tools, handoffs, knowledge base
           Turn detection                         Deploy via CLI in <30s
```

**Voice cloning pipeline:**
```
Upload audio clip (≤10s) ──▶ POST /voices/clone ──▶ Get voice_id ──▶ Use in TTS requests
```

---

## 2. Authentication & Access Control

### API Keys

Server-side requests use an API key in the `Authorization` header:

```
Authorization: Bearer sk_car_...
Cartesia-Version: 2026-03-01
```

API keys are created at [play.cartesia.ai/keys](https://play.cartesia.ai/keys). For management endpoints (credit usage, agent usage, API key metadata), an **admin API key** (`sk_car_admin_...`) is required, created at [play.cartesia.ai/keys/admin](https://play.cartesia.ai/keys/admin) (organization admins only).

### Access Tokens (Client-Side)

For browser/client apps, never use API keys directly. Generate a short-lived JWT access token via your server:

```
POST /access-token
{
  "grants": { "tts": true, "stt": true, "agent": true },
  "expires_in": 3600
}
```

| Grant | Description |
|-------|-------------|
| `tts` | Access to all TTS endpoints |
| `stt` | Access to all STT endpoints |
| `agent` | Access to Agent WebSocket calling endpoint |

Token is valid for up to 1 hour (3600 seconds). Use as `Authorization: Bearer <token>` for HTTP, or `?access_token=<token>` query parameter for WebSocket (browsers can't set headers on WebSocket handshakes).

### SDK Authentication

```python
from cartesia import Cartesia
client = Cartesia(api_key=os.getenv("CARTESIA_API_KEY"))
```

```typescript
import { Cartesia } from "@cartesia/cartesia-js";
const client = new Cartesia({ apiKey: process.env.CARTESIA_API_KEY });
```

### API Versioning

All requests require a `Cartesia-Version` header (e.g. `2026-03-01`). For a given version, Cartesia preserves existing input/output fields but may make non-breaking changes (add optional fields, add response fields, add enum variants). Inspired by Anthropic's versioning scheme.

### Zero Data Retention (ZDR)

Available to Enterprise customers for TTS and STT APIs. Prevents storage of request data.

---

## 3. Models — Catalog & Selection Guide

### TTS Models (Sonic Family)

| Model ID | Type | Languages | Latency | Status | Notes |
|----------|------|----------|---------|--------|-------|
| `sonic-3.5` | TTS | 42 | ~90ms TTFB | Stable | **Recommended.** Fastest, most emotive, context-aware heteronym resolution |
| `sonic-3.5-YYYY-MM-DD` | TTS | 42 | ~90ms | Snapshotted | Pinned version for reproducibility (e.g. `sonic-3.5-2026-05-04`) |
| `sonic-latest` | TTS | 42 | ~90ms | Beta | Latest features, may change without notice |
| `sonic-3` | TTS | 32+ | Higher | Older | Previous generation |
| `sonic-turbo` | TTS | — | — | Deprecated | Replaced by sonic-3 |
| `sonic` | TTS | — | — | Deprecated | Original model |

### STT Models (Ink Family)

| Model ID | Type | Languages | Turn Detection | Status | Notes |
|----------|------|-----------|----------------|--------|-------|
| `ink-2` | STT (Realtime) | English only | **Native** | Stable | **Recommended for voice agents.** Best WER, semantic turn detection |
| `ink-whisper` | STT (Batch + Realtime) | 90+ | No (manual finalize) | Older | Whisper-based, multilingual |

### Model ID Strategy

| `model_id` | Behavior | Recommended For |
|------------|----------|-----------------|
| `sonic-3.5-YYYY-MM-DD` | Snapshotted, never changes | Production with internal evals before updates |
| `sonic-3.5` | Updated to latest stable snapshot | Most users — stable + up-to-date |
| `sonic-latest` | Always latest beta | Testing new features only |

### Selection Guide

| Use Case | Recommended Model | Rationale |
|----------|-------------------|----------|
| Real-time / conversational agents | `sonic-3.5` + `ink-2` | ~90ms TTFB, native turn detection |
| Multilingual TTS (non-English) | `sonic-3.5` | 42 languages at native quality |
| Multilingual STT (non-English) | `ink-whisper` | 90+ languages (batch or manual realtime) |
| Batch transcription | `ink-whisper` via `/stt` | File upload, any length |
| Voice agents with turn detection | `ink-2` via `/stt/turns/websocket` | Semantic turn detection, no VAD needed |
| Push-to-talk / manual control | `ink-2` via `/stt/websocket` | You control finalization |
| Production stability | `sonic-3.5-2026-05-04` | Pinned snapshot |

### Sonic 3.5 Key Features

- **42 languages** out of the box — English, Hindi, Spanish, French, German, Japanese, Hebrew, and 35 more
- **Expressive, conversational delivery** — strong pacing and emotional range
- **Clean audio** — no artifacts to edit out across every language and voice
- **Alphanumerics** — order numbers, phone numbers, IDs, emails spoken naturally without preprocessing
- **Context-aware English pronunciation** — heteronyms like *read*, *bass*, *bow* resolved from surrounding words

### Sonic 3.5 Supported Languages

`en`, `fr`, `de`, `es`, `pt`, `zh`, `ja`, `hi`, `it`, `ko`, `nl`, `pl`, `ru`, `sv`, `tr`, `tl`, `bg`, `ro`, `ar`, `cs`, `el`, `fi`, `hr`, `ms`, `sk`, `da`, `ta`, `uk`, `hu`, `no`, `vi`, `bn`, `th`, `he`, `ka`, `id`, `te`, `gu`, `kn`, `ml`, `mr`, `pa`

### Recommended Voices for Voice Agents

| Voice | Language | Gender | Voice ID |
|-------|----------|--------|----------|
| Katie | en-US | Female | `f786b574-daa5-4673-aa0c-cbe3e8534c02` |
| Skylar | en-US | Female | `db6b0ed5-d5d3-463d-ae85-518a07d3c2b4` |
| Jameson | en-US | Male | `a5136bf9-224c-4d76-b823-52bd5efcffcc` |
| Gemma | en-GB | Female | `62ae83ad-4f6a-430b-af41-a9bede9286ca` |
| Archie | en-GB | Male | `ef191366-f52f-447a-a398-ed8c0f2943a1` |

Browse all voices at [play.cartesia.ai/voices](https://play.cartesia.ai/voices).

---

## 4. Text-to-Speech (TTS)

### Endpoints

| Endpoint | Method | Protocol | Description |
|----------|--------|----------|-------------|
| `/tts/bytes` | POST | HTTP | Stream audio from a complete transcript (streaming response body) |
| `/tts/sse` | POST | HTTP (SSE) | Stream audio with metadata (timestamps, phoneme timestamps) via Server-Sent Events |
| `/tts/websocket` | WebSocket | WS | Generate audio in real-time with contexts, continuations, multiplexing |

### Endpoint Comparison

| Feature | WebSocket | Bytes | SSE |
|---------|-----------|-------|-----|
| Multiple generations per connection | Yes | No (one POST per generation) | No |
| Timestamps (word + phoneme) | Yes | No | Yes |
| Continuations (streaming input) | Yes | No | No |
| Multiplexing (multiple contexts) | Yes | No | No |
| Wire format | JSON messages + base64 audio | Raw/file bytes | JSON-wrapped SSE events |
| Best for | Real-time agents, LLM token streaming | Batch jobs, caching, notifications | HTTP with timestamps |

### Main Concepts

- **Voice Specifier** — Object with `mode: "id"` and `id: <voice_uuid>`. Must stay the same across continuations on a context.
- **Output Format** — Object specifying `container` (raw/wav/mp3), `encoding` (pcm_s16le/pcm_f32le/pcm_mulaw/pcm_alaw), `sample_rate` (8000–48000), and `bit_rate` (mp3 only: 32k–192k).
- **Generation Config** — `volume` (0.5–2.0), `speed` (0.6–1.5), `emotion` (60+ values). Available on sonic-3 and sonic-3.5.
- **Language** — ISO 639-1 code. Must match the voice's supported language. Must stay the same across continuations.
- **Context ID** — Unique identifier for a speech context (any string, e.g. UUID). Used for continuations and multiplexing on WebSocket.
- **Continue Flag** — `true` if more transcript chunks will follow on this context; `false` on the final chunk to minimize latency.
- **Buffering** — `max_buffer_delay_ms` (0–5000, default 3000). Controls how long the server buffers text before generating. Set >0 for managed buffering (stream LLM tokens directly); set to 0 for custom buffering (you aggregate sentences).
- **Timestamps** — `add_timestamps` (word-level), `add_phoneme_timestamps` (phoneme-level), `use_normalized_timestamps`. Available on SSE and WebSocket.
- **Pronunciation Dictionary** — `pronunciation_dict_id` references a dictionary of text-to-pronunciation overrides.

### Request Body Parameters (POST `/tts/bytes`)

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `model_id` | enum | **Yes** | `sonic-3.5` | `sonic-3.5`, `sonic-3`, `sonic-latest` |
| `transcript` | string | **Yes** | — | Text to convert to speech |
| `voice` | object | **Yes** | — | `{ mode: "id", id: "<voice_uuid>" }` |
| `output_format` | object | **Yes** | — | Container/encoding/sample_rate/bit_rate |
| `language` | enum\|null | No | null | ISO 639-1 language code (42 languages) |
| `pronunciation_dict_id` | string\|null | No | null | Pronunciation dictionary ID |
| `generation_config` | object\|null | No | null | `{ volume, speed, emotion }` |
| `speed` | enum | No | `normal` | **Deprecated.** Use `generation_config.speed` instead. `slow`/`normal`/`fast` |

### Generation Config Object

```json
{
  "volume": 1.0,
  "speed": 1.0,
  "emotion": "neutral"
}
```

| Field | Type | Default | Range | Description |
|-------|------|---------|-------|-------------|
| `volume` | float | 1.0 | 0.5–2.0 | Volume multiplier |
| `speed` | float | 1.0 | 0.6–1.5 | Speed multiplier |
| `emotion` | enum | — | 60+ values | Emotional guidance (see below) |

### Emotion Values

**Primary emotions** (best results): `neutral`, `calm`, `angry`, `content`, `sad`, `scared`

**Complete list**: `neutral`, `happy`, `excited`, `enthusiastic`, `elated`, `euphoric`, `triumphant`, `amazed`, `surprised`, `flirtatious`, `curious`, `content`, `peaceful`, `serene`, `calm`, `grateful`, `affectionate`, `trust`, `sympathetic`, `anticipation`, `mysterious`, `angry`, `mad`, `outraged`, `frustrated`, `agitated`, `threatened`, `disgusted`, `contempt`, `envious`, `sarcastic`, `ironic`, `sad`, `dejected`, `melancholic`, `disappointed`, `hurt`, `guilty`, `bored`, `tired`, `rejected`, `nostalgic`, `wistful`, `apologetic`, `hesitant`, `insecure`, `confused`, `resigned`, `anxious`, `panicked`, `alarmed`, `scared`, `proud`, `confident`, `distant`, `skeptical`, `contemplative`, `determined`

### SSML Tags (Inline Transcript Controls)

| Tag | Syntax | Description | Model Support |
|-----|--------|-------------|---------------|
| `<speed>` | `<speed ratio="1.5"/>` | Speed multiplier 0.6–1.5 | sonic-3, sonic-3.5 |
| `<volume>` | `<volume ratio="0.5"/>` | Volume multiplier 0.5–2.0 | sonic-3, sonic-3.5 |
| `<emotion>` | `<emotion value="angry"/>` | Emotion guidance (beta) | sonic-3, sonic-3.5 |
| `<break>` | `<break time="1s"/>` | Explicit pause (seconds/ms) | All |
| `<spell>` | `<spell>ABC123</spell>` | Spell out characters | All |

**Nonverbalism:** Insert `[laughter]` in transcript to make the model laugh.

### WebSocket TTS Protocol

**Connection:** `wss://api.cartesia.ai/tts/websocket?cartesia_version=2026-03-01`

**Send messages (JSON):**

| Message Type | Key Fields | Description |
|--------------|------------|-------------|
| Generation Request | `model_id`, `transcript`, `voice`, `output_format`, `context_id`, `continue`, `language`, `max_buffer_delay_ms`, `flush`, `add_timestamps`, `add_phoneme_timestamps`, `pronunciation_dict_id`, `generation_config` | Generate speech for a transcript chunk |
| Cancel Request | `context_id`, `cancel: true` | Cancel a context (halts pending requests) |

**Receive messages (JSON):**

| Message Type | Key Fields | Description |
|--------------|------------|-------------|
| `chunk` | `data` (base64 audio), `done`, `context_id`, `step_time`, `status_code` | Audio data chunk |
| `timestamps` | `word_timestamps` (words[], start[], end[]), `context_id` | Word-level timing |
| `phoneme_timestamps` | `phoneme_timestamps` (phonemes[], start[], end[]), `context_id` | Phoneme-level timing |
| `flush_done` | `flush_id`, `context_id` | Acknowledgment of flush command |
| `done` | `context_id` | Generation complete for this context |
| `error` | `title`, `message`, `error_code`, `status_code`, `request_id`, `context_id` | Error event |

**Generation Request Example:**
```json
{
  "model_id": "sonic-3.5",
  "transcript": "Hello, world! I'm generating audio on Cartesia!",
  "voice": { "mode": "id", "id": "a0e99841-438c-4a64-b679-ae501e7d6091" },
  "language": "en",
  "context_id": "ab977222-f9e0-4563-a1c0-5a934ae8fdd6",
  "output_format": { "container": "raw", "encoding": "pcm_s16le", "sample_rate": 8000 },
  "add_timestamps": true,
  "continue": false
}
```

### Contexts & Continuations

Contexts maintain prosody across multiple input chunks. Rules:

1. All fields except `transcript`, `continue`, and `duration` must be identical across inputs on the same context.
2. Transcripts are concatenated **verbatim** — include spaces between words at chunk boundaries.
3. `continue: true` on partial chunks; `continue: false` on the final chunk.
4. If the last transcript is unknown, send an empty transcript with `continue: false`.
5. Contexts auto-expire 1 second after the last audio output is streamed.

**Continuation example:**
```json
{"transcript": "Hello, Sonic!", "continue": true, "context_id": "happy-monkeys-fly"}
{"transcript": " I'm streaming ", "continue": true, "context_id": "happy-monkeys-fly"}
{"transcript": "inputs.", "continue": false, "context_id": "happy-monkeys-fly"}
```

### Buffering Modes

| Mode | `max_buffer_delay_ms` | Description | Best For |
|------|-----------------------|-------------|----------|
| **Managed buffering** | >0 (default 3000) | Server decides when to start generating. Stream LLM tokens directly. | Most users — natural speech, minimal integration |
| **Custom buffering** | 0 | Server generates immediately from whatever you provide. You aggregate sentences. | Precise control over latency/prosody tradeoff |

**Key rule:** Don't aggregate client-side AND use default `max_buffer_delay_ms` — this adds unnecessary latency. Pick one approach.

### SSE TTS Protocol

**Connection:** `POST /tts/sse` with `Cartesia-Version` header

Response is a Server-Sent Events stream. Each event is `data: <json>\n\n`:

| Event Type | Description |
|------------|-------------|
| `chunk` | Audio data (base64), `done: false` |
| `timestamps` | Word-level timing (words, start, end arrays) |
| `phoneme_timestamps` | Phoneme-level timing |
| `done` | Generation complete (`done: true`) |
| `error` | Error with `error_code`, `title`, `message`, `request_id` |

**SSE-specific parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `add_timestamps` | boolean | false | Word-level timestamps |
| `add_phoneme_timestamps` | boolean | false | Phoneme-level timestamps |
| `use_normalized_timestamps` | boolean | null | Normalized vs original timestamps |
| `context_id` | string\|null | null | Echoed back in events (no continuation support) |

**SSE output format:** Only `raw` container (no wav/mp3 on SSE).

### Prompting Tips

- Punctuation is the first tool for pausing — commas/periods produce natural pauses.
- Descriptive text influences delivery (e.g. emotional subtext is interpreted from the transcript).
- Use `<spell>` for confirmation codes, order IDs, serial numbers.
- For phone numbers and credit card numbers, write as plain strings — text normalization handles grouping.
- Emotion tags work best when consistent with transcript content.

---

## 5. Speech-to-Text (STT) — Batch & Realtime

### Endpoints

| Endpoint | Method | Protocol | Model(s) | Description |
|----------|--------|----------|----------|-------------|
| `/stt` | POST | HTTP (multipart) | `ink-whisper` | Batch transcription of pre-recorded audio |
| `/stt/turns/websocket` | WebSocket | WS | `ink-2` | Realtime STT with native turn detection (Auto) |
| `/stt/websocket` | WebSocket | WS | `ink-2`, `ink-whisper` | Realtime STT with manual finalization |

### Endpoint Comparison

| Feature | `/stt/turns/websocket` (Auto) | `/stt/websocket` (Manual) | `/stt` (Batch) |
|---------|-------------------------------|--------------------------|----------------|
| Transport | WebSocket | WebSocket | HTTP file upload |
| Best for | Natural back-and-forth voice agents | Explicit turn control (push-to-talk) | Pre-recorded files, offline jobs |
| Supported models | `ink-2` only | `ink-2`, `ink-whisper` | `ink-whisper` only |
| Who handles VAD? | Cartesia | Your app | N/A |
| Who decides turn completion? | Cartesia | Your app (send `finalize`) | N/A |
| Audio input | Chunked stream | Chunked stream | Complete file |
| Output | Turn events with cumulative transcripts | Transcript deltas | One complete transcript |

### Batch STT (POST `/stt`)

**Content-Type:** `multipart/form-data`

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `file` | binary | **Yes** | — | Audio file (flac, m4a, mp3, mp4, mpeg, mpga, oga, ogg, wav, webm) |
| `model` | enum | **Yes** | `ink-whisper` | Must be `ink-whisper` family |
| `language` | enum | No | `en` | ISO 639-1 (90+ languages supported) |
| `timestamp_granularities[]` | array | No | — | `word` — word-level start/end timestamps |
| `encoding` (query) | enum | No | null | Required for raw PCM: pcm_s16le, pcm_s32le, pcm_f16le, pcm_f32le, pcm_mulaw, pcm_alaw |
| `sample_rate` (query) | integer | No | null | Sample rate in Hz (for raw PCM) |

**Response:**
```json
{
  "type": "transcript",
  "request_id": "...",
  "text": "Hello world!",
  "language": "en",
  "duration": 15.2,
  "words": [
    { "word": "Hello", "start": 0.0, "end": 0.5 },
    { "word": "world!", "start": 0.5, "end": 0.9 }
  ]
}
```

Long files are intelligently chunked by the server — no need to break up audio.

### Realtime STT (Auto) — `/stt/turns/websocket`

**Connection:** `wss://api.cartesia.ai/stt/turns/websocket?model=ink-2&encoding=pcm_s16le&sample_rate=16000&cartesia_version=2026-03-01`

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `model` | string | **Yes** | — | `ink-2` |
| `encoding` | enum | **Yes** | — | pcm_s16le, pcm_s32le, pcm_f16le, pcm_f32le, pcm_mulaw, pcm_alaw |
| `sample_rate` | integer | **Yes** | — | Audio sample rate in Hz |
| `cartesia_version` | enum | **Yes** | — | `2026-03-01` |
| `turn_start_threshold` | float | No | 0.8 | Range 0.5–0.9. Likelihood above which turn starts |
| `turn_eager_end_threshold` | float | No | 0.4 | Range 0.3–0.6. Likelihood below which eager end fires |
| `turn_end_threshold` | float | No | 0.2 | Range 0.05–0.5. Likelihood below which turn ends |
| `turn_end_timeout_ms` | integer | No | 5600 | Range 640–11200. Max wait after user stops before ending turn |

**Send messages:**

| Type | Format | Description |
|------|--------|-------------|
| Audio data | Binary WebSocket message | Raw audio chunks (~100ms each) |
| Config command | JSON text: `{"type": "config", "turn": {...}}` | Update turn detection thresholds mid-session |
| Close command | JSON text: `{"type": "close"}` | Flush buffered audio, close session cleanly |

**Receive events (turn lifecycle):**

| Event | Fires When | Carries Transcript? | Description |
|-------|------------|-------------------|-------------|
| `connected` | WebSocket established | No | Connection ready (can send audio immediately) |
| `turn.start` | User begins speaking | No | Can interrupt agent to avoid talking over user |
| `turn.update` | Repeatedly during transcription | Yes (cumulative) | Full text so far in this turn (not a delta) |
| `turn.eager_end` | Model predicts user might be done | Yes (cumulative) | Early signal — start generating reply, cancel if `turn.resume` |
| `turn.resume` | User continues after eager_end | No | Previous eager_end should be ignored |
| `turn.end` | User turn definitively complete | Yes (final) | Definitive transcript for the turn |
| `error` | Error occurs | No | `error_code`, `title`, `message`, `status_code` |

**Key guarantees:**
- First event in every turn is `turn.start`
- `turn.eager_end` is always followed by `turn.end` or `turn.resume`
- `turn.resume` only fires after a preceding `turn.eager_end`
- All emitted text is **final** — the model never revises previous output
- Transcript is **cumulative within a turn** — not a delta

**Turn detection state machine:**
```
Idle ──turn.start──▶ Speaking ──turn.eager_end──▶ EagerEnded
                       │                              │
                  turn.end                    turn.resume → Speaking
                  (→ Idle)                    turn.end (→ Idle)
```

**Turn detection configuration:**

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| `start_threshold` | 0.8 | 0.5–0.9 | Raise = fewer false starts, slower reaction. Lower = faster, more false starts |
| `eager_end_threshold` | 0.4 | 0.3–0.6 | Lower = conservative, fewer false eager ends. Higher = faster, more false eager ends |
| `end_threshold` | 0.2 | 0.05–0.5 | Lower = wait longer, fewer mid-thought endings. Higher = faster, risk of early ending |
| `end_timeout_ms` | 5600 | 640–11200 | Max wait after user stops before ending turn |

Thresholds must be strictly ordered: `start_threshold > eager_end_threshold > end_threshold`.

**Low-latency pattern (with eager end):**
```python
pending_reply = None
async for message in websocket:
    event = json.loads(message)
    match event["type"]:
        case "turn.start": tts.interrupt()
        case "turn.eager_end": pending_reply = llm.generate_async(event["transcript"])
        case "turn.resume": pending_reply.cancel(); pending_reply = None
        case "turn.end":
            if pending_reply: tts.speak(pending_reply); pending_reply = None
            else: tts.speak(llm.generate(event["transcript"]))
```

### Realtime STT (Manual) — `/stt/websocket`

**Connection:** `wss://api.cartesia.ai/stt/websocket?model=ink-2&encoding=pcm_s16le&sample_rate=16000&cartesia_version=2026-03-01`

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `model` | enum | **Yes** | — | `ink-2` or `ink-whisper` |
| `encoding` | enum | **Yes** | — | pcm_s16le, pcm_s32le, pcm_f16le, pcm_f32le, pcm_mulaw, pcm_alaw |
| `sample_rate` | integer | **Yes** | — | Sample rate in Hz |
| `cartesia_version` | enum | **Yes** | — | `2026-03-01` |
| `language` | enum | No | `en` | `en` only |
| `min_volume` | float | No | — | ink-whisper only. Silence threshold 0.0–1.0 |
| `max_silence_duration_secs` | float | No | — | ink-whisper only. Max silence before auto-finalize |

**Send messages:**

| Type | Format | Description |
|------|--------|-------------|
| Audio data | Binary WebSocket message | Raw audio chunks (~100ms) |
| `finalize` | Text message: `"finalize"` | Emit transcripts for buffered audio (crucial for low latency) |
| `close` | Text message: `"close"` | Flush remaining audio, close session |

**Receive messages:**

| Type | Fields | Description |
|------|--------|-------------|
| `transcript` | `is_final`, `text` (delta from last final), `request_id`, `words[]` (optional) | Transcript chunk — concatenate `is_final: true` chunks for full transcript |
| `flush_done` | `request_id` | Acknowledgment of `finalize` command |
| `done` | `request_id` | Acknowledgment of `close` command |
| `error` | `error_code`, `title`, `message`, `status_code`, `request_id` | Error |

**Key difference from Auto:** Text is a **delta** (not cumulative). To assemble the full transcript, concatenate all chunks where `is_final: true`. Do not strip whitespace or add separators.

### Audio Input Requirements (Realtime)

- Send audio in small chunks (~100ms)
- Format must match `encoding` and `sample_rate` parameters
- Send silence (all zeros) when audio input is muted — the server expects a continuous stream
- Supported encodings: `pcm_s16le`, `pcm_s32le`, `pcm_f16le`, `pcm_f32le`, `pcm_mulaw`, `pcm_alaw`
- Audio must be sent "in real time" (1 second of audio per second of wall clock) — not faster

### Batch STT Input Formats

**Audio:** flac, m4a, mp3, mp4, mpeg, mpga, oga, ogg, wav, webm
**Raw PCM:** Requires `encoding` and `sample_rate` query parameters
**Languages:** 90+ (ink-whisper): en, zh, de, es, ru, ko, fr, ja, pt, tr, pl, ca, nl, ar, sv, it, id, hi, fi, vi, he, uk, el, ms, cs, ro, da, hu, ta, no, th, ur, hr, bg, lt, la, mi, ml, cy, sk, te, fa, lv, bn, sr, az, sl, kn, et, mk, br, eu, is, hy, ne, mn, bs, kk, sq, sw, gl, mr, pa, si, km, sn, yo, so, af, oc, ka, be, tg, sd, gu, am, yi, lo, uz, fo, ht, ps, tk, nn, mt, sa, lb, my, bo, tl, mg, as, tt, haw, ln, ha, ba, jw, su, yue

---

## 6. Voices — Cloning, Localization & Library

### Voice Management Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/voices` | GET | List/search voices (paginated, filterable) |
| `/voices/{id}` | GET | Get voice details |
| `/voices/{id}` | DELETE | Delete a voice |
| `/voices/{id}` | POST | Update voice metadata (name, description, gender) |
| `/voices/clone` | POST | Instant Voice Clone from audio clip |
| `/voices/localize` | POST | Localize a voice to a new language/dialect |

### Voice Types

| Type | Description | Requirements |
|------|-------------|--------------|
| **Voice Library** | Community-shared, public voices | Browse at play.cartesia.ai/voices |
| **Instant Voice Clone (IVC)** | Clone from ≤10s audio clip | Fast, free training |
| **Pro Voice Clone (PVC)** | Fine-tuned model from extended data | 1M credits per fine-tune, bespoke model |
| **Localized Voice** | Adapt existing voice to new language | 20 target languages, dialect support |

### List Voices Parameters (GET `/voices`)

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | integer | 1–100 voices per page |
| `starting_after` | string | Pagination cursor (Voice ID) |
| `ending_before` | string | Pagination cursor (Voice ID) |
| `q` | string | Search by name, description, or Voice ID |
| `is_owner` | boolean | Only return voices owned by your org |
| `gender` | enum | `masculine`, `feminine`, `gender_neutral` |
| `language` | string | Filter by language or language-locale (e.g. `en`, `en-GB`) |
| `expand[]` | array | `preview_file_url` — include preview audio URL |

### Voice Object

```json
{
  "id": "db6b0ed5-d5d3-463d-ae85-518a07d3c2b4",
  "is_owner": false,
  "access": { "type": "public", "visibility": "all" },
  "name": "Skylar - Friendly Guide",
  "description": "Approachable American female ideal for customer care and support.",
  "gender": "feminine",
  "language": "en",
  "country": "US",
  "created_at": "2026-03-31T17:37:05.961874Z",
  "is_public": true,
  "is_pro": false
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string (UUID) | Voice identifier |
| `is_owner` | boolean | Whether your org owns the voice |
| `access.type` | enum | `private` / `public` |
| `access.visibility` | enum | `owner` / `all` — who sees it in list |
| `name` | string | Voice name |
| `description` | string | Voice description |
| `gender` | enum\|null | `masculine`, `feminine`, `gender_neutral` |
| `language` | string | ISO 639-1 language code |
| `country` | string\|null | ISO 3166-1 alpha-2 country code |
| `is_pro` | boolean | Whether this is a Pro Voice Clone |
| `preview_file_url` | string\|null | Preview audio URL (requires same auth header, don't store permanently) |

### Instant Voice Cloning (POST `/voices/clone`)

**Content-Type:** `multipart/form-data`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `clip` | binary | **Yes** | Audio clip (flac, mp3, mpeg, mpga, oga, ogg, wav, webm). Recommended ≤10s |
| `name` | string | **Yes** | Voice name |
| `description` | string | No | Voice description |
| `language` | enum | **Yes** | ISO 639-1 language (42 supported languages) |
| `base_voice_id` | string\|null | No | Optional base voice ID the clone is derived from |
| `access[type]` | enum | No | `private` (default) / `public` |

**Best practices for cloning:**
1. Choose a script matching your target use case's energy
2. Speak clearly, avoid background noise, use a high-quality microphone
3. Avoid long pauses (they'll be mimicked by the clone)
4. Trim the recording — speech from start to finish, no cut-offs or excessive silence
5. Speak in the target language (or use localization afterward)

**Billing:** Instant clones are billed at standard TTS rate (~1 credit/char). Training is free.

### Pro Voice Cloning (PVC)

Pro Voice Clones fine-tune a model on your extended audio data via `/fine-tunes/create`.

| Aspect | Detail |
|--------|--------|
| Cost | 1,000,000 credits per fine-tune |
| Billing | Only charged when training succeeds |
| TTS rate | ~1.5 credits/char (50% more than standard) |
| Retraining | Costs another 1M credits (new base model or new data) |
| Fine-tune endpoints | `/fine-tunes/create`, `/fine-tunes/get`, `/fine-tunes/list`, `/fine-tunes/delete` |
| Fine-tune voices | `/fine-tunes/list-voices` — list voices created from a fine-tune |

### Voice Localization (POST `/voices/localize`)

Creates a new voice from an existing voice, localized to a new language and optional dialect.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `voice_id` | string | **Yes** | Source voice ID to localize |
| `name` | string | **Yes** | New voice name |
| `description` | string | **Yes** | New voice description |
| `language` | enum | **Yes** | Target language (20 supported) |
| `original_speaker_gender` | enum | **Yes** | `male` / `female` |
| `dialect` | enum\|null | No | Dialect (language-specific) |

**Supported target languages:** `en`, `de`, `es`, `fr`, `ja`, `pt`, `zh`, `hi`, `it`, `ko`, `nl`, `pl`, `ru`, `sv`, `tr`, `ar`, `he`, `ta`, `te`, `th`

**Dialect options:**

| Language | Dialects |
|----------|----------|
| English (`en`) | `au` (Australian), `in` (Indian), `so` (Southern), `uk` (British), `us` (American) |
| Spanish (`es`) | `mx` (Latin American), `pe` (Peninsular) |
| Portuguese (`pt`) | `br` (Brazilian), `eu` (European) |
| French (`fr`) | `eu` (Standard Parisian/Metropolitan), `ca` (Canadian) |

### Datasets (for Fine-Tuning)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/datasets` | POST | Create a dataset |
| `/datasets/{id}` | GET | Get dataset |
| `/datasets` | GET | List datasets |
| `/datasets/{id}` | POST | Update dataset |
| `/datasets/{id}` | DELETE | Delete dataset |
| `/datasets/{id}/files` | GET | List files in dataset |
| `/datasets/{id}/files` | POST | Upload file to dataset |
| `/datasets/{id}/files/{file_id}` | DELETE | Remove file from dataset |

---

## 7. Voice Changer (Speech to Speech)

### Endpoints

| Endpoint | Method | Protocol | Description |
|----------|--------|----------|-------------|
| `/voice-changer/bytes` | POST | HTTP | Voice changer (streaming audio bytes) |
| `/voice-changer/sse` | POST | HTTP (SSE) | Voice changer with SSE events (timestamps, metadata) |

### Main Concepts

Transforms source audio into a different voice while preserving the same intonation, emotion, and speech patterns. Billing: 15 credits per second of input audio.

### Request Parameters (POST `/voice-changer/bytes`)

**Content-Type:** `multipart/form-data`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `clip` | binary | **Yes** | Source audio (flac, mp3, mpeg, mpga, oga, ogg, wav, webm) |
| `voice[id]` | string | **Yes** | Target voice ID |
| `output_format[container]` | enum | **Yes** | `raw`, `wav`, `mp3` |
| `output_format[sample_rate]` | integer | **Yes** | 8000, 16000, 22050, 24000, 44100, 48000 |
| `output_format[encoding]` | enum | For raw/wav | pcm_f32le, pcm_s16le, pcm_mulaw, pcm_alaw |
| `output_format[bit_rate]` | integer | For mp3 | MP3 bit rate |

### SSE Response Events

| Event | Fields | Description |
|-------|--------|-------------|
| Chunk | `data` (base64), `done: false`, `status_code: 206`, `sample_rate`, `step_time` | Audio data chunk |
| Done | `done: true`, `status_code: 200` | Generation complete |
| Error | `type: "error"`, `error_code`, `title`, `message`, `status_code`, `request_id` | Error event |

### Constraints

| Constraint | Value |
|------------|-------|
| Billing | 15 credits per second of input audio |
| Input formats | flac, mp3, mpeg, mpga, oga, ogg, wav, webm |
| Auth | API key only (no access token) |

---

## 8. Infill — Audio Bridging

### Endpoint

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/infill/bytes` | POST | Generate audio that smoothly connects two existing audio segments |

### Main Concepts

Generates audio to bridge between two existing audio clips: `left_audio` → `transcript` → `right_audio`. Useful for fixing gaps, inserting words, or creating smooth transitions in pre-recorded audio.

### Request Parameters (POST `/infill/bytes`)

**Content-Type:** `multipart/form-data`

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `left_audio` | binary | One of left/right | Audio clip before the infill transcript |
| `right_audio` | binary | One of left/right | Audio clip after the infill transcript |
| `transcript` | string | **Yes** | Infill text to generate (longer = better adaptation) |
| `model_id` | enum | **Yes** | `sonic-3`, `sonic-3-2026-01-12`, `sonic-3-2025-10-27` |
| `language` | enum | **Yes** | Supported language (42 languages) |
| `voice_id` | string | **Yes** | Voice ID for generated audio |
| `output_format[container]` | enum | **Yes** | `raw`, `wav`, `mp3` |
| `output_format[sample_rate]` | integer | **Yes** | 8000–48000 |
| `output_format[encoding]` | enum | For raw/wav | pcm_f32le, pcm_s16le, pcm_mulaw, pcm_alaw |
| `output_format[bit_rate]` | integer | For mp3 | MP3 bit rate |

### Best Practices

- Target natural pauses in the audio and clip tightly
- Use longer transcripts to give the model more flexibility to adapt to surrounding audio
- At least one of `left_audio` or `right_audio` must be provided

### Pricing

| Component | Cost |
|-----------|------|
| Fixed per request | 300 credits |
| Per character of transcript | ~1 credit (standard TTS rate) |

---

## 9. Pronunciation Dictionaries

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/pronunciation-dicts/` | POST | Create a new pronunciation dictionary |
| `/pronunciation-dicts/{id}` | GET | Get a pronunciation dictionary |
| `/pronunciation-dicts` | GET | List all pronunciation dictionaries |
| `/pronunciation-dicts/{id}` | POST | Update a pronunciation dictionary |
| `/pronunciation-dicts/{id}` | DELETE | Delete a pronunciation dictionary |

### Main Concepts

Versioned sets of text-to-pronunciation mappings that override how specific words or phrases are spoken. Supported by `sonic-3` models and newer. Referenced in TTS requests via `pronunciation_dict_id`.

### Create Request (POST `/pronunciation-dicts/`)

```json
{
  "name": "Acme",
  "description": "An example dictionary",
  "items": [
    { "text": "acme", "pronunciation": "<<ˈ|æ|k|m|i>>" }
  ],
  "access": { "type": "private" }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | **Yes** | Dictionary name |
| `description` | string | No | Dictionary description (default: "") |
| `items` | array\|null | No | Initial pronunciation mappings |
| `access.type` | enum | No | `private` (default) / `public` |

### Pronunciation Dictionary Item

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `text` | string | **Yes** | Original text to be replaced |
| `pronunciation` | string | **Yes** | Phonetic representation or text to say instead |
| `alias` | string | Deprecated | Use `pronunciation` instead |

### Pronunciation Dictionary Object

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier |
| `name` | string | Dictionary name |
| `description` | string | Dictionary description |
| `is_owner` | boolean | Whether your org owns it |
| `access` | object | `{ type: private/public, visibility: owner/all }` |
| `pinned` | boolean | Whether pinned for user |
| `items` | array | List of pronunciation mappings |
| `created_at` | string | ISO 8601 timestamp |

### Usage in TTS

Pass `pronunciation_dict_id` in TTS requests (bytes, SSE, WebSocket). The dictionary applies custom pronunciations to matching text in the transcript.

---

## 10. Line — Voice Agent Platform

### Overview

Line is Cartesia's managed voice agent platform. It handles audio orchestration (STT with Ink, TTS with Sonic), deployment, auto-scaling, and observability, so you focus on agent reasoning. Agents are deployed in Cartesia's managed runtime in under 30 seconds.

### Architecture

```
Phone/Web ──▶ Cartesia Line (managed runtime)
                  │
           Audio orchestration (Ink 2 STT + Sonic 3.5 TTS)
           Turn detection, interruption handling
                  │
           Your Agent code (Python SDK)
                  │
           Any LLM via LiteLLM (100+ providers)
           Tools, handoffs, knowledge base
                  │
           Deploy via CLI (cartesia deploy)
```

### SDK

**Package:** `cartesia-line` (Python)

```bash
uv add cartesia-line
```

### Core Concepts

| Component | Purpose |
|-----------|---------|
| `Agent` | Controls the input/output event loop via a `process` method |
| `LlmAgent` | Built-in agent wrapping 100+ LLM providers via LiteLLM |
| `Tools` | Functions your agent can call — database lookups, handoffs, web search |
| `VoiceAgentApp` | HTTP server connecting your agent to Cartesia's audio infrastructure |

### Agent Types

**LlmAgent (recommended):**
```python
from line.llm_agent import LlmAgent, LlmConfig, end_call
from line.voice_agent_app import VoiceAgentApp

async def get_agent(env, call_request):
    return LlmAgent(
        model="anthropic/claude-haiku-4-5-20251001",
        api_key=os.getenv("ANTHROPIC_API_KEY"),
        tools=[end_call],
        config=LlmConfig(
            system_prompt="You are a helpful assistant.",
            introduction="Hello! How can I help you today?",
        ),
    )

app = VoiceAgentApp(get_agent=get_agent)
```

**Custom Agent (class or function):**
```python
class HelloAgent:
    async def process(self, env, event):
        if isinstance(event, CallStarted):
            yield AgentSendText(text="Hello!")
        elif isinstance(event, UserTurnEnded):
            yield AgentSendText(text="I heard you!")
```

### LlmConfig Options

| Option | Type | Description |
|--------|------|-------------|
| `system_prompt` | str | System prompt defining agent behavior |
| `introduction` | Optional[str] | Message on call start. None/"" to wait for user |
| `temperature` | Optional[float] | Sampling temperature |
| `max_tokens` | Optional[int] | Max tokens per response |
| `top_p` | Optional[float] | Nucleus sampling threshold |
| `stop` | Optional[List[str]] | Stop sequences |
| `seed` | Optional[int] | Random seed |
| `presence_penalty` | Optional[float] | Presence penalty |
| `frequency_penalty` | Optional[float] | Frequency penalty |
| `num_retries` | int | Retries on failure (default: 2) |
| `fallbacks` | Optional[List[str]] | Fallback models |
| `timeout` | Optional[float] | Request timeout in seconds |
| `reasoning_effort` | Optional[str] | none/low/medium/high (provider-dependent) |
| `extra` | Dict[str, Any] | Provider-specific options via LiteLLM |

### Supported LLM Providers

| Provider | Model Examples |
|----------|---------------|
| Anthropic | `anthropic/claude-haiku-4-5-20251001`, `anthropic/claude-sonnet-4-5` |
| OpenAI | `gpt-5.4`, `gpt-5.2`, `gpt-5-nano` |
| Google | `gemini/gemini-2.5-flash-preview-09-2025`, `gemini/gemini-3.0-preview` |
| 100+ more | Via [LiteLLM](https://docs.litellm.ai/docs/providers) |

### Event System

| Event | Description |
|-------|-------------|
| `CallStarted` | Call has started — agent should greet |
| `UserTurnStarted` | User began speaking — can interrupt agent |
| `UserTurnEnded` | User finished speaking — agent should respond |
| `UserTextSent` | Partial transcription (for responsive agents) |
| `CallEnded` | Call has ended |
| `AgentSendText` | Agent sends text to be spoken (output event) |
| `AgentEndCall` | Agent ends the call (output event) |
| `AgentHandedOff` | Control transferred to another handler |

### Event Filters (Controlling the Loop)

| Filter | Default | Description |
|--------|---------|-------------|
| `run_filter` | `[CallStarted, UserTurnEnded, CallEnded]` | Events that trigger `process()` |
| `cancel_filter` | `[UserTurnStarted]` | Events that interrupt the agent |

### Tool Types

| Tool Type | Decorator | Description |
|-----------|-----------|-------------|
| **Loopback** | `@loopback_tool` | Result goes back to LLM for continued generation (default) |
| **Passthrough** | `@passthrough_tool` | Output goes directly to user, bypassing LLM |
| **Handoff** | `@handoff_tool` | Transfers control to another handler |

### Built-in Tools

| Tool | Description |
|------|-------------|
| `end_call` | Ends the call (custom description supported) |
| `transfer_call` | Transfers to another number (E.164 format) |
| `web_search` | Real-time web search |
| `knowledge_base` | Looks up attached documents (with metadata filters, top_k) |
| `send_dtmf` | Sends DTMF tones |

### HTTP Server Tools

Define tools that call HTTP APIs from JSON schemas without writing tool functions:

```python
create_ticket = http_server_tool(
    name="create_ticket",
    description="Creates a support ticket.",
    url="https://api.example.com/v1/{tenant_id}/tickets",
    method="POST",
    request_body_schema={...},
    auth={"Authorization": "Bearer ${SUPPORT_API_KEY}"},
    timeout=5.0,
)
```

| Parameter | Description |
|-----------|-------------|
| `url` | Request URL with `{param}` templating |
| `method` | GET/HEAD/POST/PUT/PATCH/DELETE/OPTIONS |
| `path_params_schema` | Maps `{param}` names to type/description |
| `request_body_schema` | JSON schema for request body |
| `query_params_schema` | JSON schema for query string |
| `auth` | Headers with `${ENV_VAR}` placeholders |
| `content_type` | `application/json` (default) or `application/x-www-form-urlencoded` |
| `is_background` | Run in background task (default: True) |

### Multi-Agent Handoffs

```python
spanish_agent = LlmAgent(model="gpt-5-nano", ...)

main_agent = LlmAgent(
    tools=[
        end_call,
        agent_as_handoff(
            spanish_agent,
            name="transfer_to_spanish",
            description="Transfer when user requests Spanish.",
            update_call=UpdateCallConfig(voice_id="spanish-voice-id"),
        ),
    ],
    ...
)
```

### Pre-Call Handler

Configure voice, language, pronunciation, or reject calls before the agent starts:

```python
async def pre_call_handler(call_request: CallRequest):
    return PreCallResult(
        metadata={"tier": "premium"},
        config={
            "tts": {
                "voice_id": "a0e99841-438c-4a64-b679-ae501e7d6091",
                "model": "sonic-3.5",
                "language": "en",
                "pronunciation_dict_id": "your-dict-id",
            },
            "stt": { "language": "en" }
        }
    )
```

Return `None` to reject with 403.

### CallRequest Fields

| Field | Type | Description |
|-------|------|-------------|
| `call_id` | str | Unique call identifier |
| `from_` | str | Caller identifier (phone number or client ID) |
| `to` | str | Called number or agent ID |
| `agent_call_id` | str | Agent call ID for logging/correlation |
| `metadata` | Optional[dict] | Custom data from client application |
| `agent` | AgentConfig | Prompts from Playground or API |

### CLI

```bash
cartesia auth login        # Authenticate
cartesia init              # Link project
cartesia deploy            # Deploy agent (<30s)
cartesia chat 8000         # Test locally
cartesia call +1XXXXXXXXXX # Call deployed agent
cartesia env set KEY=VAL   # Set environment variables
```

### Agent Management API (REST)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/agents` | GET | List agents |
| `/agents/{id}` | GET | Get agent |
| `/agents/{id}` | POST | Update agent |
| `/agents/{id}` | DELETE | Delete agent |
| `/agents/templates` | GET | List public agent templates |
| `/agents/phone-numbers` | GET | Agent phone numbers |

### Telephony

| Feature | Description |
|---------|-------------|
| Cartesia numbers | Provision US phone numbers via API |
| Import numbers | Bring your own Twilio numbers |
| Outbound calling | `POST /agents/calls/create-outbound-call` |
| Batch calling | `POST /agents/call-batches/create-call-batch` |
| Call management | List, get, cancel, delete calls |
| Call audio | Download call recordings |
| Runtime logs | Get call runtime logs |
| Webhooks | Register webhook endpoints for call events |
| Providers | Link telephony provider accounts (Twilio, etc.) |

### Observability

| Feature | Description |
|---------|-------------|
| Call logs | Debug conversations and monitor performance |
| Evaluations | Custom metrics (LLM-as-a-Judge) |
| Deployments | Track versions and roll back |
| Metrics | Create, list, export metric results as CSV |

### Knowledge Base

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/agents/documents` | POST | Upload document |
| `/agents/documents/bulk` | POST | Upload up to 100 documents |
| `/agents/documents` | GET | List documents |
| `/agents/documents/{id}` | GET/DELETE | Get/delete document |
| `/agents/folders` | POST/GET | Create/list folders |
| `/agents/folders/{id}` | GET/POST/DELETE | Manage folder |

---

## 11. Audio Streaming Architecture

### Streaming Modes

| Mode | Endpoints | Description | Use Case |
|------|-----------|-------------|----------|
| **HTTP Streaming (Bytes)** | `/tts/bytes`, `/voice-changer/bytes` | Streaming response body (raw/file bytes) | Batch jobs, caching, file generation |
| **SSE** | `/tts/sse`, `/voice-changer/sse` | Server-Sent Events with JSON-wrapped chunks | HTTP with timestamps, SSE-native stacks |
| **WebSocket** | `/tts/websocket`, `/stt/turns/websocket`, `/stt/websocket` | Bidirectional — send text/audio, receive audio/transcripts | Real-time agents, LLM token streaming, lowest latency |

### TTS Streaming Architecture

```
                        ┌─────────────────┐
                        │  Complete Text   │
                        └────────┬────────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
              POST /tts/bytes  POST /tts/sse  WebSocket
              (raw bytes)    (SSE events)   (contexts)
                    │            │            │
                    │            │     ┌──────┴──────┐
                    │            │     │ Context 1   │ Context 2
                    │            │     │ (continue)  │ (continue)
                    │            │     └──────┬──────┘
                    └────────────┴──────────┘
                                 │
                          Streaming Audio
                    (base64 chunks or raw bytes)
```

### WebSocket Connection Lifecycle

**TTS WebSocket:**
- Open connection → send generation requests with `context_id` → receive audio chunks → receive `done` → close or send more requests
- Idle connections closed after **5 minutes**
- Max parallel WebSocket connections = 10× concurrency limit

**STT WebSocket:**
- Open connection → send audio chunks → receive transcripts/turn events → send `close` → receive final events
- Idle connections closed after **3 minutes**
- Each connection counts toward STT concurrency (including idle)

### Multiplexing (WebSocket TTS)

Multiple `context_id` values can be active on a single WebSocket connection simultaneously. Each context generates independently and audio is tagged with its `context_id`. Only unique active contexts count toward concurrency.

### Endpoint Selection Decision Tree

```
Are you streaming text from an LLM or partial input?
├── Yes → WebSocket (/tts/websocket)
│         (continuations, lowest latency across turns)
└── No → Do you need timestamps without WebSocket?
    ├── Yes → SSE (/tts/sse)
    └── No → Will you speak often enough that connect/TLS cost hurts?
        ├── Yes → WebSocket
        └── No → Bytes (/tts/bytes)
```

---

## 12. Concurrency, Limits & Pricing

### Concurrency Limits by Plan

| Plan | TTS Concurrent Requests | STT Concurrent Requests |
|------|------------------------|------------------------|
| Free | 2 | 8 |
| Pro | 3 | 12 |
| Startup | 5 | 20 |
| Scale | 15 | 60 |
| Enterprise | Custom | Custom |

**Key points:**
- TTS and STT have **separate** concurrency limits
- TTS concurrency = number of unique active contexts (same `context_id` = one context)
- HTTP endpoints: each request is a separate context
- WebSocket: same `context_id` = one context (sequential processing)
- STT: each active stream (HTTP or WebSocket) counts, including idle connections
- Exceeding limits returns `429 Too Many Requests`

### TTS WebSocket Limits

- Max parallel WebSocket connections = **10× concurrency limit** (e.g. 15 concurrency → 150 connections)
- Idle TTS WebSocket connections closed after **5 minutes**

### STT WebSocket Limits

- Idle STT WebSocket connections closed after **3 minutes**
- Idle connections still count toward concurrency

### Concurrency Interpretation

| Use Case | Rule of Thumb |
|----------|---------------|
| Conversational (voice agents) | ~4× concurrency limit in parallel conversations |
| Non-conversational (batch) | ~1× concurrency limit in parallel generations |

### Pricing

**Credits** are used for TTS, STT, infill, and voice changing. **Agent dollars (USD)** are used for Line agent calls.

| Capability | Cost |
|------------|------|
| TTS (standard voice) | ~1 credit per character |
| TTS (Pro Voice Clone) | ~1.5 credits per character |
| STT — `/stt/websocket` (ink-2) | 3 credits per second of audio |
| STT — `/stt/turns/websocket` (ink-2) | 3 credits per second of audio |
| STT — `/stt/websocket` (ink-whisper) | 1 credit per second of audio |
| STT — `/stt/turns/websocket` (ink-whisper) | 1 credit per second of audio |
| STT — `/stt` (batch, ink-whisper) | 1 credit per 2 seconds of audio |
| Infill | 300 credits + ~1 credit per character of transcript |
| Voice Changer | 15 credits per second of input audio |
| Pro Voice Clone fine-tune | 1,000,000 credits per fine-tune |

### Line Agent Pricing (USD per minute)

| Feature | Price per Minute |
|---------|-----------------|
| Agent calling | $0.06 |
| Telephony (add-on, Cartesia number) | +$0.014 |

### Billing Notes

- Credits are only consumed by successful requests; errors don't consume credits
- Every subscription plan includes a monthly credit allotment
- Check usage at [play.cartesia.ai/usage](https://play.cartesia.ai/usage)
- Programmatic usage via `/usage/credits` and `/usage/agents` (requires admin API key)

---

## 13. Error Handling

### Structured Error Response (Cartesia-Version 2026-03-01+)

```json
{
  "error_code": "concurrency_limited",
  "title": "Too many concurrent requests",
  "message": "You have exceeded your plan's concurrency limit.",
  "request_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

| Field | Description |
|-------|-------------|
| `error_code` | Machine-readable identifier (can be null) |
| `title` | Short human-readable summary |
| `message` | Detailed human-readable explanation |
| `request_id` | Request identifier for support/debugging |
| `doc_url` | Optional link to docs (when available) |

### WebSocket/SSE Error Events

Include all HTTP error fields plus transport context:

```json
{
  "type": "error",
  "done": true,
  "status_code": 429,
  "error_code": "concurrency_limited",
  "title": "Too many concurrent requests",
  "message": "You have exceeded your plan's concurrency limit.",
  "request_id": "...",
  "context_id": "happy-monkeys-fly"
}
```

### Common Error Codes

| Error Code | Description |
|------------|-------------|
| `quota_exceeded` | Credit quota exceeded |
| `concurrency_limited` | Plan concurrency limit exceeded |
| `voice_model_mismatch` | Voice and model incompatible |
| `voice_not_found` | Invalid voice ID |
| `model_not_found` | Invalid model ID |
| `language_not_supported` | Language not supported by model |
| `file_too_large` | File exceeds size limit |
| `unsupported_audio_format` | Audio format not supported |
| `plan_upgrade_required` | Feature requires plan upgrade |

### Legacy Error Format (pre-2026-03-01)

- HTTP errors: plain text in `Title: Message` format
- WebSocket/SSE errors: legacy envelopes with string-only messages

---

## 14. Audio Output Formats Reference

### TTS Output Formats

| Container | Encoding | Sample Rates | Bit Rates | Notes |
|-----------|----------|--------------|-----------|-------|
| `raw` | `pcm_s16le` | 8000–48000 | — | 16-bit signed LE PCM |
| `raw` | `pcm_f32le` | 8000–48000 | — | 32-bit float LE PCM |
| `raw` | `pcm_mulaw` | 8000–48000 | — | μ-law (telephony) |
| `raw` | `pcm_alaw` | 8000–48000 | — | A-law (telephony) |
| `wav` | (inherits raw encoding) | 8000–48000 | — | WAV container wrapping raw encoding |
| `mp3` | — | 8000–48000 | 32k–192k | MP3 with configurable bit rate |

**SSE limitation:** Only `raw` container supported (no wav/mp3).

### Sample Rates

| Rate | Use Case |
|------|----------|
| 8000 | Telephony (μ-law/A-law) |
| 16000 | Low-latency streaming, speech recognition |
| 22050 | Standard quality |
| 24000 | Voice applications |
| 44100 | CD quality (default) |
| 48000 | Professional audio |

### MP3 Bit Rates

| Bit Rate | Quality |
|----------|---------|
| 32000 | Low (voice-only) |
| 64000 | Medium-low |
| 96000 | Medium |
| 128000 | Standard (default) |
| 192000 | High |

### Format Selection Guide

| Use Case | Recommended Format |
|----------|-------------------|
| General playback | `wav`, `pcm_s16le`, 44100 |
| Telephony / Twilio | `raw`, `pcm_mulaw`, 8000 |
| Low-latency streaming | `raw`, `pcm_s16le`, 16000 |
| Browser playback | `raw`, `pcm_f32le`, 24000 (Web Audio API) |
| File storage | `mp3`, 44100, 128000 |
| Highest quality | `wav`, `pcm_s16le`, 48000 |

---

## 15. Capability Summary & Cross-Reference

### Complete API Endpoint Map

| Capability | Endpoint Base | Methods | Protocol | Auth |
|-----------|--------------|---------|----------|------|
| **TTS (Bytes)** | `/tts/bytes` | POST | HTTP | API Key / Access Token |
| **TTS (SSE)** | `/tts/sse` | POST | HTTP (SSE) | API Key / Access Token |
| **TTS (WebSocket)** | `/tts/websocket` | WS | WebSocket | API Key / Access Token |
| **STT (Batch)** | `/stt` | POST | HTTP | API Key / Access Token |
| **STT (Auto)** | `/stt/turns/websocket` | WS | WebSocket | API Key / Access Token |
| **STT (Manual)** | `/stt/websocket` | WS | WebSocket | API Key / Access Token |
| **Voice Changer (Bytes)** | `/voice-changer/bytes` | POST | HTTP | API Key only |
| **Voice Changer (SSE)** | `/voice-changer/sse` | POST | HTTP (SSE) | API Key only |
| **Infill** | `/infill/bytes` | POST | HTTP | API Key only |
| **Voices — List** | `/voices` | GET | HTTP | API Key |
| **Voices — Clone** | `/voices/clone` | POST | HTTP (multipart) | API Key only |
| **Voices — Localize** | `/voices/localize` | POST | HTTP | API Key only |
| **Voices — Get/Update/Delete** | `/voices/{id}` | GET/POST/DELETE | HTTP | API Key |
| **Pronunciation Dicts** | `/pronunciation-dicts/` | POST/GET | HTTP | API Key only |
| **Fine-Tunes** | `/fine-tunes/...` | POST/GET/DELETE | HTTP | API Key |
| **Datasets** | `/datasets/...` | POST/GET/DELETE | HTTP | API Key |
| **Access Token** | `/access-token` | POST | HTTP | API Key |
| **API Keys** | `/api-keys/...` | GET | HTTP | Admin API Key |
| **Usage — Credits** | `/usage/credits` | GET | HTTP | Admin API Key |
| **Usage — Agents** | `/usage/agents` | GET | HTTP | Admin API Key |
| **API Status** | `/` | GET | HTTP | — |
| **Line Agents** | `/agents/...` | GET/POST/DELETE | HTTP | API Key |
| **Line Calls** | `/agents/calls/...` | GET/POST/DELETE | HTTP | API Key |
| **Line Call Batches** | `/agents/call-batches/...` | GET/POST | HTTP | API Key |
| **Line Deployments** | `/agents/deployments/...` | GET | HTTP | API Key |
| **Line Documents** | `/agents/documents/...` | POST/GET/DELETE | HTTP | API Key |
| **Line Folders** | `/agents/folders/...` | POST/GET/DELETE | HTTP | API Key |
| **Line Metrics** | `/agents/metrics/...` | POST/GET/DELETE | HTTP | API Key |
| **Line Phone Numbers** | `/agents/phone-numbers/...` | GET/POST/DELETE | HTTP | API Key |
| **Line Providers** | `/agents/providers/...` | GET/POST/DELETE | HTTP | API Key |
| **Line Webhooks** | `/agents/webhooks/...` | GET/POST/DELETE | HTTP | API Key |

### Capability Decision Matrix

| If you need... | Use... | Key Parameters |
|----------------|--------|----------------|
| Text → spoken audio (simplest) | TTS Bytes (`/tts/bytes`) | `model_id`, `transcript`, `voice`, `output_format` |
| Text → spoken audio with timestamps (HTTP) | TTS SSE (`/tts/sse`) | + `add_timestamps`, `add_phoneme_timestamps` |
| Stream LLM tokens → speech (real-time) | TTS WebSocket (`/tts/websocket`) | `context_id`, `continue`, `max_buffer_delay_ms` |
| Control speed/volume/emotion | `generation_config` | `volume` (0.5–2.0), `speed` (0.6–1.5), `emotion` |
| Inline mid-transcript controls | SSML tags | `<speed>`, `<volume>`, `<emotion>`, `<break>`, `<spell>` |
| Transcribe pre-recorded audio | Batch STT (`/stt`) | `file`, `model` (ink-whisper), `language` |
| Live transcription with auto turn detection | STT Auto (`/stt/turns/websocket`) | `model` (ink-2), `encoding`, `sample_rate`, turn thresholds |
| Live transcription with manual control | STT Manual (`/stt/websocket`) | `model`, `encoding`, `sample_rate`, send `"finalize"` |
| Change voice in audio | Voice Changer (`/voice-changer/bytes`) | `clip`, `voice[id]`, `output_format` |
| Bridge two audio clips | Infill (`/infill/bytes`) | `left_audio`, `right_audio`, `transcript`, `voice_id` |
| Clone a voice from audio | Instant Clone (`/voices/clone`) | `clip` (≤10s), `name`, `language` |
| Clone a voice professionally | Pro Clone (`/fine-tunes/create`) | Extended data, 1M credits |
| Localize a voice to new language | Localize (`/voices/localize`) | `voice_id`, `language`, `dialect`, `original_speaker_gender` |
| Control pronunciation of specific words | Pronunciation Dictionaries | Create dict → reference via `pronunciation_dict_id` |
| Build a managed voice agent | Line SDK (`cartesia-line`) | `LlmAgent`, `tools`, `LlmConfig` |
| Deploy a voice agent | Line CLI | `cartesia deploy` (<30s) |
| Connect agent to phone | Line Telephony | Provision number or import Twilio |
| Handle interruptions in agent | Line event filters | `cancel_filter: [UserTurnStarted]` |
| Multi-agent handoff | Line `agent_as_handoff` | Transfer to specialized agent with config update |
| Call external APIs from agent | Line `http_server_tool` | JSON schema-based, no function code |
| Browser-side TTS/STT | Access Tokens | Server generates JWT → client uses `?access_token=` |
| Pin model version for production | Dated model ID | `sonic-3.5-2026-05-04` |
| Lowest latency TTS | WebSocket with `max_buffer_delay_ms: 0` | Custom buffering, full sentences |
| Multiplex multiple utterances | WebSocket TTS | Multiple `context_id` on one connection |
| Cancel in-flight generation | WebSocket TTS cancel | `{"context_id": "...", "cancel": true}` |
| Low-latency agent response | STT Auto `turn.eager_end` | Start reply on eager_end, cancel on resume |

### Comparison with ElevenLabs (Summary)

| Aspect | Cartesia | ElevenLabs |
|--------|----------|------------|
| TTS models | Sonic 3.5 (42 langs, ~90ms) | eleven_v3, flash_v2.5, multilingual_v2 |
| STT models | Ink 2 (English, native turn detection), ink-whisper (90+) | scribe_v2 (batch), scribe_v2_realtime |
| Native turn detection | Yes (Ink 2, semantic) | No (separate VAD needed) |
| TTS endpoints | Bytes, SSE, WebSocket | REST, Stream, WebSocket, multi-stream |
| Streaming input (LLM tokens) | WebSocket continuations | WebSocket stream-input |
| Voice cloning | Instant (≤10s), Pro (fine-tune) | IVC, PVC (with captcha verification) |
| Voice design from text | No (Voice Library + clones) | Yes (text-to-voice design) |
| Voice localization | Yes (20 languages, dialects) | No (separate dubbing API) |
| Music generation | No | Yes (music_v2) |
| Sound effects | No | Yes (eleven_text_to_sound_v2) |
| Dubbing | No (separate capability) | Yes (90+ languages) |
| Conversational agent platform | Line (managed, SDK, CLI, telephony) | Speech Engine (BYO LLM) |
| Pronunciation control | Pronunciation dictionaries | Pronunciation dictionaries |
| SSML support | `<speed>`, `<volume>`, `<emotion>`, `<break>`, `<spell>` | Audio tags (`[laughs]`, `[whispering]`) |
| Emotion control | `generation_config.emotion` (60+ values, beta) | Stability/similarity/style settings + audio tags |
| Self-hosted | Yes (Docker, Kubernetes, SageMaker) | No |
| Audio formats | raw (PCM), wav, mp3 | mp3, pcm, wav, opus, μ-law, a-law |
| Versioning | `Cartesia-Version` header (dated) | No explicit API versioning |
| Pricing unit | Credits (per char/second) + agent USD/min | Characters + credits |