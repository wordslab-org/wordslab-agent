# OpenAI API Analysis — Realtime Voice Agents, Live Translation, Transcription & Speech Generation

> **Base URL:** `https://api.openai.com/v1` (REST) | **WebSocket:** `wss://api.openai.com/v1/realtime` | **WebRTC:** `/v1/realtime/calls`
> **Docs:** `https://developers.openai.com/api/docs/guides/realtime` | **Product:** `https://platform.openai.com` | **Auth:** API key (`Authorization: Bearer` header) or ephemeral client secrets (`ek_...`)
> **SDKs:** `openai` (Python), `openai` (TypeScript/Node), `@openai/agents/realtime` (Agents SDK Realtime), OpenAI CLI
> **Description:** OpenAI offers voice and audio capabilities across two broad architectures — request-based audio APIs for files and bounded requests, and stateful realtime sessions for live, low-latency audio. The platform covers speech-to-speech voice agents, live translation, realtime transcription, file-based speech-to-text, and text-to-speech, with custom voice creation. Realtime models like `gpt-realtime-2.1` are natively multimodal, supporting audio input/output, text, and image input within a single session.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Authentication & Access Control](#2-authentication--access-control)
3. [Models — Catalog & Selection Guide](#3-models--catalog--selection-guide)
4. [Realtime Sessions — Architecture & Connection Methods](#4-realtime-sessions--architecture--connection-methods)
5. [Voice Agents (Speech-to-Speech)](#5-voice-agents-speech-to-speech)
6. [Realtime Translation (Live Interpretation)](#6-realtime-translation-live-interpretation)
7. [Realtime Transcription (Streaming STT)](#7-realtime-transcription-streaming-stt)
8. [Speech to Text — File-Based Transcription & Translation](#8-speech-to-text--file-based-transcription--translation)
9. [Text to Speech (Speech Generation)](#9-text-to-speech-speech-generation)
10. [Audio in Chat Completions (Multimodal Audio Chat)](#10-audio-in-chat-completions-multimodal-audio-chat)
11. [Custom Voices — Creation & Management](#11-custom-voices--creation--management)
12. [Realtime Session Configuration Reference](#12-realtime-session-configuration-reference)
13. [Realtime Event Reference](#13-realtime-event-reference)
14. [Voice Activity Detection (VAD)](#14-voice-activity-detection-vad)
15. [Interruption & Barge-in Handling](#15-interruption--barge-in-handling)
16. [Function Calling in Realtime Sessions](#16-function-calling-in-realtime-sessions)
17. [Image Input in Realtime Sessions](#17-image-input-in-realtime-sessions)
18. [Capability Summary & Cross-Reference](#18-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

OpenAI's audio platform is organized around these core abstractions:

- **Audio Modality** — Four modalities combine to form audio applications: audio input (model receives sound), audio output (model/API returns spoken audio), text transcript (speech becomes text), and text prompt (text controls what the model says/does).
- **Realtime Session** — A stateful, persistent connection between the client and model over WebRTC or WebSocket. The session keeps a connection open while the application sends audio, receives events, and updates session state. Maximum duration is **60 minutes**.
- **Conversation** — Represents user input Items and model output Items generated during the current realtime session.
- **Response** — Model-generated audio or text Items that are added to the Conversation.
- **Input Audio Buffer** — A server-side buffer for streaming audio input over WebSocket. Clients append base64-encoded audio chunks; with VAD enabled, the server auto-commits when it detects speech turns. Without VAD, the client manually commits.
- **Voice** — A named speech identity for audio output. Built-in voices (e.g. `alloy`, `coral`, `marin`) or custom voices (identified by `voice_id`). Once a realtime session emits audio, the voice cannot be changed for that session.
- **Turn Detection (VAD)** — Automatic detection of when the user starts/stops speaking. Two modes: `server_vad` (silence-based) and `semantic_vad` (semantic classifier). Can be disabled (`null`) for push-to-talk.
- **Ephemeral Client Secret** — A short-lived credential (`ek_...`) created server-side, used by browser/mobile clients to connect to realtime sessions without exposing the main API key.
- **Safety Identifier** — A stable, privacy-preserving value (e.g. hashed user ID) sent via the `OpenAI-Safety-Identifier` header to help OpenAI monitor/detect abuse at the individual user level rather than the organization level.

### Speech Tasks

| Task | Description | Architecture |
|------|-------------|-------------|
| **Speech to Text (STT)** | Converts speech into text. For captions, notes, transcripts, analytics, search. | Request-based (files) or Realtime (streaming) |
| **Text to Speech (TTS)** | Converts text into spoken audio. For narration, assistants, accessibility. | Request-based (streaming HTTP) |
| **Speech to Speech** | Model listens, reasons, and speaks in one low-latency session. For conversational voice agents. | Realtime session |
| **Speech Translation** | Listens to speech in one language, returns translated speech or transcript in another. | Realtime translation session |

### Platform Architecture

```
Request-based APIs:
  Audio File ──▶ STT Model ──▶ Text transcript (json/text/srt/vtt/verbose_json/diarized_json)
  Text Input ──▶ TTS Model ──▶ Audio output (mp3/opus/aac/flac/wav/pcm)

Realtime sessions:
  Audio Stream ──▶ Realtime Model ──▶ Audio output + text transcript (streaming)
                    │
    Voice Agent:      conversation lifecycle, tool calls, turn-taking, interruptions
    Translation:       continuous stream, no turn lifecycle, translated audio + transcripts
    Transcription:     continuous stream, transcript deltas only, no model responses

Multimodal chat:
  Text + Audio Input ──▶ Audio-capable Chat Model ──▶ Text + Audio Output
```

### End-to-End Flows

**Voice agent (browser, WebRTC):**
```
1. App server creates ephemeral client secret (POST /v1/realtime/client_secrets)
2. Frontend creates RealtimeSession (Agents SDK) or RTCPeerConnection
3. Session connects over WebRTC (browser) or WebSocket (server)
4. Agent handles audio turns, tools, interruptions, handoffs in the session
```

**Chained voice pipeline (Python, server-side):**
```
Audio Input ──▶ Speech-to-Text ──▶ Agent (text reasoning) ──▶ Text-to-Speech ──▶ Audio Output
```

**Realtime translation (WebSocket):**
```
Source audio (base64 PCM16) ──▶ /v1/realtime/translations ──▶ translated audio + transcript deltas
```

---

## 2. Authentication & Access Control

### API Keys

All REST and WebSocket requests require an `Authorization` header:

```
Authorization: Bearer $OPENAI_API_KEY
```

### Ephemeral Client Secrets

For browser/mobile clients, the application server creates a short-lived client secret so the API key is never exposed client-side:

```
POST /v1/realtime/client_secrets
POST /v1/realtime/translations/client_secrets
```

The returned secret (`ek_...`) is used by the frontend to connect via WebRTC or WebSocket. When using ephemeral tokens, set the `OpenAI-Safety-Identifier` header on the server-side request that creates the client secret so the identifier is bound to that session.

### SDK Authentication

```python
from openai import OpenAI
client = OpenAI()  # reads OPENAI_API_KEY env var
```

```typescript
import OpenAI from "openai";
const openai = new OpenAI();  // reads OPENAI_API_KEY env var
```

### Safety Identifiers

- Sent via `OpenAI-Safety-Identifier` header on Realtime API requests
- Recommended but not required
- Use a stable, privacy-preserving value (e.g. hashed internal user ID)
- Does not carry over from Responses API requests or other sessions
- Must be passed separately when creating/connecting each Realtime session

### Beta to GA Migration

Key changes from beta to GA:
- Remove `OpenAI-Beta: realtime=v1` header
- Use `POST /v1/realtime/client_secrets` for ephemeral credentials
- Use `/v1/realtime/calls` for WebRTC sessions
- Set `session.type`, move output audio config under `session.audio.output`
- Use newer response event names: `response.output_text.delta`, `response.output_audio.delta`, `response.output_audio_transcript.delta`

---

## 3. Models — Catalog & Selection Guide

### Model Catalog

| Model ID | Type | Description | Use Case |
|---------|------|-------------|----------|
| `gpt-realtime-2.1` | Realtime (voice agent) | Latest realtime model with reasoning, speech-to-speech | Low-latency voice agents, tool use, conversations |
| `gpt-realtime-2` | Realtime (voice agent) | Previous realtime generation | Voice agents, image input support |
| `gpt-realtime-translate` | Realtime (translation) | Dedicated translation model | Live speech translation, interpretation |
| `gpt-realtime-whisper` | Realtime (transcription) | Streaming transcription with controllable latency | Live transcription, captions |
| `gpt-4o-mini-tts` | TTS | Newest, most reliable text-to-speech model | Speech generation with instruction-based control |
| `tts-1` | TTS | Lower latency, lower quality | Realtime applications where latency matters |
| `tts-1-hd` | TTS | Higher quality | Higher-fidelity speech generation |
| `gpt-4o-transcribe` | STT (file) | GPT-4o based transcription | File transcription with prompts and logprobs |
| `gpt-4o-mini-transcribe` | STT (file) | Smaller GPT-4o transcription | Cost-effective file transcription |
| `gpt-4o-transcribe-diarize` | STT (file) | Speaker-aware transcription | Meeting recordings, multi-speaker audio |
| `whisper-1` | STT (file) | Open-source Whisper model | Transcription + translation, timestamps, multiple formats |
| `gpt-audio-1.5` | Multimodal chat | Natively multimodal (audio + text in/out) | Audio in existing chat completions apps |

### Selection Guide

| Use Case | Recommended Model | Architecture |
|----------|-------------------|-------------|
| Low-latency voice agent | `gpt-realtime-2.1` | Realtime session (`/v1/realtime`) |
| Live speech translation | `gpt-realtime-translate` | Translation session (`/v1/realtime/translations`) |
| Live streaming transcription | `gpt-realtime-whisper` | Transcription session (Realtime) |
| File/bounded audio transcription | `gpt-4o-transcribe` | Request-based (`/v1/audio/transcriptions`) |
| Speaker-diarized transcription | `gpt-4o-transcribe-diarize` | Request-based (`/v1/audio/transcriptions`) |
| Text to speech | `gpt-4o-mini-tts` | Request-based (`/v1/audio/speech`) |
| Audio in chat app | `gpt-audio-1.5` | Chat Completions (`/v1/chat/completions`) |

### Architecture Decision Matrix

| Goal | Model or API | Start Here |
|------|-------------|------------|
| Build a low-latency voice agent | `gpt-realtime-2.1` | Voice agents |
| Translate live speech into another language | `gpt-realtime-translate` | Realtime translation |
| Transcribe live audio into streaming text | `gpt-realtime-whisper` | Realtime transcription |
| Transcribe files or bounded audio requests | Audio transcription models | Speech to text |
| Generate speech from text | Speech generation models | Text to speech |
| Add audio to an existing Chat Completions app | Audio-capable chat models | Audio and speech |

---

## 4. Realtime Sessions — Architecture & Connection Methods

### Session Types

| Session Type | Use When | Endpoint |
|-------------|----------|----------|
| **Voice-agent session** | Model should respond, call tools, manage conversation state | `/v1/realtime` |
| **Translation session** | App should continuously translate speech as it arrives | `/v1/realtime/translations` |
| **Transcription session** | App needs streaming transcript deltas without model-generated spoken responses | Transcription session (emits transcript deltas) |

### Connection Methods

| Transport | Use When | Key Feature |
|-----------|----------|-------------|
| **WebRTC** | Browser and mobile clients that capture or play audio directly | Peer connection handles media tracks; audio I/O is automatic |
| **WebSocket** | Server already receives raw audio from a media pipeline, call system, or worker | Manual base64-encoded PCM audio chunks; lowest-level interface |
| **SIP** | Telephony voice agents | Confirm model support before using for translation or transcription |

### WebRTC Connection Flow

```
1. App server creates ephemeral client secret: POST /v1/realtime/client_secrets
2. Browser creates RTCPeerConnection
3. Adds local audio track (microphone): pc.addTrack(ms.getTracks()[0])
4. Sets up remote audio playback: pc.ontrack -> audio element
5. Creates data channel for events: pc.createDataChannel("oai-events")
6. Creates SDP offer, sends to: POST /v1/realtime/calls (with client secret)
7. Server returns SDP answer; pc.setRemoteDescription(answer)
8. Session is live — send/receive events via data channel
```

### WebSocket Connection Flow

```
1. Connect to: wss://api.openai.com/v1/realtime?model=gpt-realtime-2.1
   Headers: Authorization: Bearer $OPENAI_API_KEY
2. Receive: session.created (session ready)
3. Send: session.update (configure session)
4. Receive: session.updated (new state)
5. Send audio: input_audio_buffer.append (base64 PCM chunks)
6. Receive: input_audio_buffer.speech_started / speech_stopped (VAD events)
7. Receive: response.output_audio.delta (audio output)
8. Receive: response.done (response complete)
```

---

## 5. Voice Agents (Speech-to-Speech)

### Architecture Choices

| Architecture | Best For | Why |
|-------------|----------|-----|
| **Speech-to-speech with live audio sessions** | Natural, low-latency conversations | Model handles live audio input and output directly |
| **Chained voice pipeline** | Predictable workflows, extending existing text agents | App explicitly chains STT → text reasoning → TTS |

### Speech-to-Speech (RealtimeAgent)

The fastest path to a browser-based voice assistant uses the Agents SDK:

```typescript
import { RealtimeAgent, RealtimeSession } from "@openai/agents/realtime";

const agent = new RealtimeAgent({
  name: "Assistant",
  instructions: "You are a helpful voice assistant.",
});

const session = new RealtimeSession(agent, {
  model: "gpt-realtime-2.1",
});

await session.connect({
  apiKey: "ek_...(ephemeral key from your server)",
});
```

From there, attach tools, handoffs, and guardrails to the `RealtimeAgent` the same way as a text agent. Keep audio transport in the session layer, and business logic in the agent definition.

### Chained Voice Pipeline (Python)

```python
from agents import Agent, function_tool
from agents.voice import AudioInput, SingleAgentVoiceWorkflow, VoicePipeline

@function_tool
def get_weather(city: str) -> str:
    return f"The weather in {city} is sunny."

agent = Agent(
    name="Assistant",
    instructions="You are a helpful voice assistant.",
    model="gpt-5.6",
    tools=[get_weather],
)

pipeline = VoicePipeline(workflow=SingleAgentVoiceWorkflow(agent))
audio_input = AudioInput(buffer=np.zeros(24000 * 3, dtype=np.int16))
result = await pipeline.run(audio_input)
async for event in result.stream():
    if event.type == "voice_stream_event_audio":
        print("Received audio bytes", len(event.data))
```

### Key Features

- **Realtime 2 reasoning** — Adds reasoning to speech-to-speech workflows. Start with `reasoning.effort` set to `low` for production voice agents, adjust based on latency tolerance and task complexity.
- **Barge-in / interruption** — User speech mid-response cancels in-flight response and truncates unplayed audio.
- **Tool use** — Function calling, MCP servers, and connectors available in realtime sessions.
- **Image input** — `gpt-realtime-2` and later support image input via `input_image` content parts.
- **Same agent building blocks** — Tools, orchestration/handoffs, guardrails, integrations/observability all work with voice.

### Endpoint

| Endpoint | Protocol | Description |
|----------|----------|-------------|
| `/v1/realtime` | WebSocket | Standard realtime conversation session |
| `/v1/realtime/calls` | WebRTC (SDP) | WebRTC call establishment |
| `/v1/realtime/client_secrets` | POST (HTTP) | Create ephemeral client secret |

---

## 6. Realtime Translation (Live Interpretation)

### Endpoints

| Endpoint | Protocol | Description |
|----------|----------|-------------|
| `/v1/realtime/translations` | WebSocket | Translation session (WebSocket) |
| `/v1/realtime/translations/calls` | WebRTC (SDP) | Translation session (WebRTC) |
| `/v1/realtime/translations/client_secrets` | POST (HTTP) | Create translation client secret |

### Main Concepts

Realtime translation uses a dedicated translation endpoint instead of the standard voice-agent endpoint. Translation sessions are **continuous**: the client streams audio in, and the service streams translated audio and transcript deltas out.

- **No conversation lifecycle** — Translation sessions don't use the normal assistant turn lifecycle.
- **No `response.create`** — Don't call `response.create`; don't wait for a client to commit a user turn before translation begins.
- **Continuous streaming** — Keep appending audio, including silence between phrases, and handle output events as they arrive.
- **Model acts as interpreter** (not assistant) — Translates what a human says rather than answering questions.
- **One session per target language** — For multi-language output, create separate sessions per language.

### Session Configuration

```json
{
  "type": "session.update",
  "session": {
    "audio": {
      "output": {
        "language": "es"
      }
    }
  }
}
```

### WebSocket Connection

```
wss://api.openai.com/v1/realtime/translations?model=gpt-realtime-translate
Headers: Authorization: Bearer $OPENAI_API_KEY
         OpenAI-Safety-Identifier: hashed-user-id
```

### Translation Events

| Event | Direction | Description |
|-------|-----------|-------------|
| `session.update` | Client → Server | Configure target language |
| `session.input_audio_buffer.append` | Client → Server | Send base64 PCM16 audio |
| `session.close` | Client → Server | Flush pending audio, emit remaining output |
| `session.output_audio.delta` | Server → Client | Translated audio chunks (base64 PCM16) |
| `session.output_transcript.delta` | Server → Client | Translated text transcript deltas |
| `session.input_transcript.delta` | Server → Client | Source language transcript deltas |
| `session.closed` | Server → Client | Session fully closed |

### Architecture Patterns

**Listen-along translation** (one source, many listeners):
```
source audio → translation session → translated audio + subtitles
One session per target language
```

**Conversational translation** (two or more participants):
```
Caller A audio → translate into Caller B language → play to Caller B
Caller B audio → translate into Caller A language → play to Caller A
One session per direction; keep speaker tracks separate
```

### Session Close

Send `session.close` before closing the WebSocket. The service flushes pending input audio, emits remaining translated output, then sends `session.closed`. Stop appending audio after `session.close` and continue reading events until `session.closed` arrives.

---

## 7. Realtime Transcription (Streaming STT)

### Main Concepts

Realtime transcription sessions produce streaming transcript deltas from live audio without model-generated spoken responses. Use this for live captions, call analysis, and media stream transcription.

- **Transcription-only session** — `session.type` set to `"transcription"`.
- **`gpt-realtime-whisper`** — Streaming transcription model with controllable latency via `delay` setting.
- **Manual commit** — For `gpt-realtime-whisper`, omit `turn_detection` or set to `null`, then commit audio manually with `input_audio_buffer.commit`.

### Session Configuration

```json
{
  "type": "session.update",
  "session": {
    "type": "transcription",
    "audio": {
      "input": {
        "format": {
          "type": "audio/pcm",
          "rate": 24000
        },
        "transcription": {
          "model": "gpt-realtime-whisper",
          "language": "en",
          "delay": "low"
        }
      }
    }
  }
}
```

### Session Fields

| Field | Description |
|-------|-------------|
| `type` | Set to `"transcription"` for transcription-only sessions |
| `audio.input.format` | Input encoding — use 24 kHz mono PCM for `audio/pcm` |
| `audio.input.transcription.model` | Use `gpt-realtime-whisper` for streaming transcription |
| `audio.input.transcription.language` | Optional language hint (e.g. `"en"`) |
| `audio.input.transcription.delay` | Latency/accuracy tradeoff: `minimal`, `low`, `medium`, `high`, `xhigh` |
| `audio.input.turn_detection` | Optional VAD for models that support it. For `gpt-realtime-whisper`, omit or set to `null`, then commit audio manually |

### Streaming Audio

Send audio chunks:
```json
{
  "type": "input_audio_buffer.append",
  "audio": "base64Pcm16"
}
```

If turn detection is disabled, commit the buffer when transcription should begin:
```json
{
  "type": "input_audio_buffer.commit"
}
```

For models that support server VAD, the session commits audio automatically when it detects a turn boundary.

### Latency/Accuracy Tradeoff (`delay`)

| Delay Value | Behavior |
|-------------|----------|
| `minimal` | Earliest partial text, lowest accuracy |
| `low` | Early partial text, good for live captions |
| `medium` | Balanced |
| `high` | Higher accuracy, later partials |
| `xhigh` | Best accuracy, latest partials |

Test with real production audio, target languages, accents, and domain vocabulary before choosing a production default.

### Production Checklist

- Pick a target latency and accuracy threshold before tuning
- Test against real production audio, not only clean samples
- Test each target language
- Include numbers, dates, currency, email addresses, product names, and domain terms in eval set
- Track empty, truncated, and delayed transcripts apart from word error rate
- Decide how UI should revise partial text when later deltas correct earlier text
- Use `item_id` to order and reconcile final transcripts
- Keep a fallback path for unsupported timestamps, diarization, or confidence fields

---

## 8. Speech to Text — File-Based Transcription & Translation

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/audio/transcriptions` | POST | Transcribe audio (file upload) |
| `/v1/audio/translations` | POST | Translate audio to English text |

### Supported Input Formats

`mp3`, `mp4`, `mpeg`, `mpga`, `m4a`, `wav`, `webm`

| Limit | Value |
|-------|-------|
| Max file size | 25 MB |
| Input types | `mp3`, `mp4`, `mpeg`, `mpga`, `m4a`, `wav`, `webm` |
| Known speaker references | Same formats, as data URLs (2–10 seconds) |

### Transcription Models & Response Formats

| Model | Supported `response_format` | Supports Prompt | Supports Logprobs | Supports `timestamp_granularities[]` |
|-------|------------------------------|-----------------|--------------------|--------------------------------------|
| `whisper-1` | `json`, `text`, `srt`, `verbose_json`, `vtt` | Yes (224 tokens max) | No | Yes (segment, word) |
| `gpt-4o-transcribe` | `json`, `text` | Yes (GPT-4o style) | Yes | No |
| `gpt-4o-mini-transcribe` | `json`, `text` | Yes (GPT-4o style) | Yes | No |
| `gpt-4o-transcribe-diarize` | `json`, `text`, `diarized_json` | No | No | No |

### Transcription Request Parameters (POST `/v1/audio/transcriptions`)

Content-Type: `multipart/form-data`

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `file` | binary | **Yes** | — | Audio file to transcribe |
| `model` | string | **Yes** | — | Model ID (`whisper-1`, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe`, `gpt-4o-transcribe-diarize`) |
| `language` | string | No | null | ISO 639-1 or 639-3 language hint (GPT-4o models) |
| `prompt` | string | No | null | Context to improve transcription quality. Whisper: first 224 tokens only. Not for `gpt-4o-transcribe-diarize` |
| `response_format` | enum | No | `json` | Output format (see table above) |
| `timestamp_granularities[]` | array | No | — | `segment` and/or `word` — only `whisper-1`, requires `verbose_json` |
| `chunking_strategy` | string/object | No | — | `"auto"` or VAD config — required for `gpt-4o-transcribe-diarize` when audio > 30s |
| `stream` | boolean | No | false | Stream transcript events. Not supported for `whisper-1` |
| `include[]` | array | No | — | Include `logprobs` in response (GPT-4o models) |
| `known_speaker_names[]` | array | No | — | Up to 4 speaker names for diarization mapping |
| `known_speaker_references[]` | array | No | — | Up to 4 short audio references (2–10s, data URLs) for diarization |

### Diarization (`gpt-4o-transcribe-diarize`)

Produces speaker-aware transcripts with `diarized_json` response format:

```json
{
  "segments": [
    {
      "speaker": "speaker_0",
      "text": "Hello, how are you?",
      "start": 0.0,
      "end": 2.5
    }
  ]
}
```

- Requires `chunking_strategy` when audio > 30 seconds (`"auto"` recommended)
- Optional `known_speaker_names[]` + `known_speaker_references[]` (up to 4, 2–10s each, data URLs)
- When `stream=true`, emits `transcript.text.segment` events when segments complete
- Not yet supported in the Realtime API

### Streaming Transcription (file-based)

With `stream=true` (GPT-4o models only, not `whisper-1`):
- Emits `transcript.text.delta` events as each part is transcribed
- Emits `transcript.text.done` with the full transcript when complete
- With `diarized_json`, also emits `transcript.text.segment` events with speaker labels

### Translation Endpoint (POST `/v1/audio/translations`)

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file` | binary | **Yes** | Audio file (any supported language) |
| `model` | string | **Yes** | Must be `whisper-1` (only supported model) |
| `prompt` | string | No | Context hint |

- Output is always **English text** (only translation into English supported)
- Same input formats and 25 MB limit as transcriptions

### Improving Reliability

**Method 1 — Prompt parameter (Whisper):**
- Use `prompt` to pass correct spellings of uncommon words/acronyms
- Whisper only considers first 224 tokens of prompt
- Useful for: correcting misrecognized words, preserving context across split files, forcing punctuation, keeping filler words, controlling writing style

**Method 2 — Post-processing with GPT-4:**
- Transcribe with Whisper, then post-process transcript with GPT-4/GPT-3.5-Turbo
- Larger context window, more scalable than 224-token prompt limit
- GPT-4 can be instructed and guided in ways impossible with Whisper

### Supported Languages (57)

Afrikaans, Arabic, Armenian, Azerbaijani, Belarusian, Bosnian, Bulgarian, Catalan, Chinese, Croatian, Czech, Danish, Dutch, English, Estonian, Finnish, French, Galician, German, Greek, Hebrew, Hindi, Hungarian, Icelandic, Indonesian, Italian, Japanese, Kannada, Kazakh, Korean, Latvian, Lithuanian, Macedonian, Malay, Marathi, Maori, Nepali, Norwegian, Persian, Polish, Portuguese, Romanian, Russian, Serbian, Slovak, Slovenian, Spanish, Swahili, Swedish, Tagalog, Tamil, Thai, Turkish, Ukrainian, Urdu, Vietnamese, Welsh

> Model trained on 98 languages; only languages with <50% WER (word error rate) are listed.

---

## 9. Text to Speech (Speech Generation)

### Endpoint

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/audio/speech` | POST | Generate spoken audio from text |

### Models

| Model | Description | Quality | Latency |
|-------|-------------|---------|---------|
| `gpt-4o-mini-tts` | Newest, most reliable TTS; instruction-controllable | Highest | Standard |
| `tts-1` | Lower latency | Lower | Lower |
| `tts-1-hd` | Higher quality | Higher | Higher |

### Request Parameters (POST `/v1/audio/speech`)

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `model` | string | **Yes** | — | `gpt-4o-mini-tts`, `tts-1`, or `tts-1-hd` |
| `input` | string | **Yes** | — | Text to convert to speech |
| `voice` | string/object | **Yes** | — | Built-in voice name (e.g. `"coral"`) or `{ "id": "voice_123abc" }` for custom voice |
| `instructions` | string | No | — | Control speech aspects: accent, emotional range, intonation, speed, tone, whispering (gpt-4o-mini-tts only) |
| `response_format` | enum | No | `mp3` | `mp3`, `opus`, `aac`, `flac`, `wav`, `pcm` |
| `language` | string | No | — | Language for custom voices (e.g. `"fr"`) |

### Built-in Voices

**`gpt-4o-mini-tts` (13 voices):** `alloy`, `ash`, `ballad`, `coral`, `echo`, `fable`, `nova`, `onyx`, `sage`, `shimmer`, `verse`, `marin`, `cedar`

**`tts-1` / `tts-1-hd` (9 voices):** `alloy`, `ash`, `coral`, `echo`, `fable`, `onyx`, `nova`, `sage`, `shimmer`

> For best quality, use `marin` or `cedar`. Voices optimized for English. Try voices at [OpenAI.fm](https://openai.fm).

### Output Formats

| Format | Description | Use Case |
|--------|-------------|----------|
| `mp3` | Default, general use | General playback |
| `opus` | Low latency, internet streaming | Communication, streaming |
| `aac` | Digital audio compression | YouTube, Android, iOS |
| `flac` | Lossless compression | Audio archiving |
| `wav` | Uncompressed | Low-latency applications (no decode overhead) |
| `pcm` | Raw 24kHz 16-bit signed low-endian, no header | Lowest latency streaming |

### Streaming

The Speech API supports realtime audio streaming via chunk transfer encoding — audio can be played before the full file is generated. For fastest response times, use `wav` or `pcm` as the response format.

### Instruction-Based Speech Control (`gpt-4o-mini-tts`)

Prompt the model to control:
- Accent
- Emotional range
- Intonation
- Impressions
- Speed of speech
- Tone
- Whispering

### Supported Languages

Follows Whisper language support (57 languages listed in §8). Despite voices being optimized for English, the model performs well across listed languages.

### Usage Policy

Must provide clear disclosure to end users that the TTS voice is AI-generated and not a human voice.

---

## 10. Audio in Chat Completions (Multimodal Audio Chat)

### Endpoint

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/chat/completions` | POST | Chat with audio input/output (model `gpt-audio-1.5`) |

### Audio Output from Model

Generate a human-like audio response to a text prompt:

```javascript
const response = await openai.chat.completions.create({
  model: "gpt-audio-1.5",
  modalities: ["text", "audio"],
  audio: { voice: "alloy", format: "wav" },
  messages: [
    { role: "user", content: "Is a golden retriever a good family dog?" }
  ],
  store: true,
});
// response.choices[0].message.audio.data — base64 encoded WAV
```

### Audio Input to Model

Use audio inputs for prompting:

```javascript
const response = await openai.chat.completions.create({
  model: "gpt-audio-1.5",
  modalities: ["text", "audio"],
  audio: { voice: "alloy", format: "wav" },
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "What is in this recording?" },
        { type: "input_audio", input_audio: { data: base64str, format: "wav" } }
      ]
    }
  ],
  store: true,
});
```

### Key Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | string | `gpt-audio-1.5` (natively multimodal) |
| `modalities` | array | `["text", "audio"]` — enable audio output |
| `audio` | object | `{ voice: "alloy", format: "wav" }` — output voice and format |
| `messages[].content[].type` | string | `"text"`, `"input_audio"` |
| `messages[].content[].input_audio` | object | `{ data: base64str, format: "wav" }` |
| `store` | boolean | Whether to store the completion |

> For live browser speech-to-speech, use Realtime sessions instead. Chat Completions audio is for extending existing chat flows.

---

## 11. Custom Voices — Creation & Management

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/audio/voice_consents` | POST | Create a voice consent recording |
| `/v1/audio/voices` | POST | Create a custom voice from audio sample + consent ID |

### Main Concepts

Custom voices enable a unique voice for your agent/application. Created voices can be used with:
- Text to Speech API (`/v1/audio/speech`)
- Realtime API (`session.audio.output.voice`)
- Chat Completions API with audio output

### Creating a Custom Voice

Requires **two separate audio recordings**:

1. **Consent recording** — Voice actor reads a consent phrase (in one of 16 supported languages). Uploaded via `POST /v1/audio/voice_consents` with `name`, `language`, and `recording` fields. Returns a consent ID (`cons_...`).

2. **Sample recording** — The actual audio sample the model will replicate (≤30 seconds). Uploaded via `POST /v1/audio/voices` with `name`, `audio_sample`, and `consent` (consent ID).

### Consent Phrases (16 languages)

| Language | Phrase |
|----------|-------|
| `en` | "I am the owner of this voice and I consent to OpenAI using this voice to create a synthetic voice model." |
| `de` | "Ich bin der Eigentümer dieser Stimme und bin damit einverstanden..." |
| `es` | "Soy el propietario de esta voz y doy mi consentimiento..." |
| `fr` | "Je suis le propriétaire de cette voix et j'autorise OpenAI..." |
| `ja` | "私はこの音声の所有者であり..." |
| `zh` | "我是此声音的拥有者并授权OpenAI..." |
| `hi`, `id`, `it`, `ko`, `nl`, `pl`, `pt`, `ru`, `uk`, `vi` | (see docs) |

### Requirements & Limitations

| Constraint | Value |
|------------|-------|
| Max voices per organization | 20 |
| Sample duration | ≤ 30 seconds |
| Sample formats | `mpeg`, `wav`, `ogg`, `aac`, `flac`, `webm`, `mp4` |
| Eligibility | Limited to eligible customers (contact sales) |

### Using a Custom Voice

**Text to speech:**
```json
{
  "model": "gpt-4o-mini-tts",
  "voice": { "id": "voice_123abc" },
  "input": "Text to speak",
  "language": "fr",
  "format": "wav"
}
```

**Realtime API:**
```json
{
  "session": {
    "type": "realtime",
    "model": "gpt-realtime-2",
    "audio": {
      "output": {
        "voice": { "id": "voice_123abc" }
      }
    }
  }
}
```

### Recording Tips

- Record in a quiet space with minimal echo
- Use a professional XLR microphone
- Stay 7–8 inches from mic with a pop filter
- Model copies exactly what you give it — tone, cadence, energy, pauses, habits
- Try multiple examples to find the best fit

---

## 12. Realtime Session Configuration Reference

### `session.update` Client Event

The `session.update` event configures the realtime session. Most properties can be updated at any time, except the `voice` after the model has emitted audio once.

```json
{
  "type": "session.update",
  "session": {
    "type": "realtime",
    "model": "gpt-realtime-2.1",
    "output_modalities": ["audio"],
    "audio": {
      "input": {
        "format": { "type": "audio/pcm", "rate": 24000 },
        "turn_detection": {
          "type": "semantic_vad",
          "eagerness": "medium",
          "create_response": true,
          "interrupt_response": true
        }
      },
      "output": {
        "format": { "type": "audio/pcm" },
        "voice": "marin"
      }
    },
    "instructions": "Speak clearly and briefly. Confirm understanding before taking actions.",
    "prompt": {
      "id": "pmpt_123",
      "version": "89",
      "variables": { "city": "Paris" }
    },
    "tools": [ ... ],
    "tool_choice": "auto"
  }
}
```

### Session Fields

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `"realtime"` (voice agent) or `"transcription"` (transcription-only) |
| `model` | string | Model ID (e.g. `"gpt-realtime-2.1"`) |
| `output_modalities` | array | `["audio"]`, `["text"]`, or `["audio", "text"]` |
| `instructions` | string | System instructions for the model |
| `prompt` | object | Server-stored prompt: `{ id, version (optional), variables (optional) }`. Direct session fields override prompt fields if they overlap |
| `audio.input.format` | object | `{ type: "audio/pcm", rate: 24000 }` |
| `audio.input.turn_detection` | object/null | VAD config (see §14) or `null` to disable |
| `audio.output.format` | object | `{ type: "audio/pcm" }` or `{ type: "audio/pcmu" }` |
| `audio.output.voice` | string/object | Built-in voice name (e.g. `"marin"`) or `{ id: "voice_123abc" }` for custom voice |
| `audio.output.language` | string | Target language for translation sessions (e.g. `"es"`) |
| `tools` | array | Function tool definitions |
| `tool_choice` | string | e.g. `"auto"` |

### Response-Level Overrides (`response.create`)

| Field | Description |
|-------|-------------|
| `output_modalities` | Override session modalities for this response |
| `instructions` | Per-response instructions override |
| `input` | Custom context array — `item_reference` (existing items by `id`) and new `message` items. Empty `[]` removes all prior context |
| `conversation` | Set to `"none"` for out-of-band response not added to the default conversation |
| `metadata` | Arbitrary object to identify the response |
| `tools` | Per-response tool list override |
| `tool_choice` | Per-response tool choice override |
| `input_audio_format` | Override input format for this response |
| `audio.output.format` | Override output format for this response |

---

## 13. Realtime Event Reference

### Client Events (sent by the application)

| Event | Description | Used In |
|-------|-------------|---------|
| `session.update` | Update session configuration | All sessions |
| `conversation.item.create` | Add a conversation item (message, function_call_output) | Voice agent |
| `conversation.item.truncate` | Truncate unplayed audio after interruption (WebSocket) | Voice agent |
| `input_audio_buffer.append` | Append base64 audio chunk to buffer | Voice agent, transcription, translation |
| `input_audio_buffer.commit` | Commit audio buffer (when VAD disabled) | Voice agent, transcription |
| `input_audio_buffer.clear` | Clear audio buffer | Voice agent |
| `output_audio_buffer.clear` | Clear unplayed output audio (WebRTC/SIP) | Voice agent |
| `response.create` | Request model response | Voice agent |
| `response.cancel` | Cancel in-progress response | Voice agent (push-to-talk) |
| `session.close` | Close translation session gracefully | Translation only |
| `session.input_audio_buffer.append` | Append audio (translation session naming) | Translation |

### Server Events (received from the API)

| Event | Description | Used In |
|-------|-------------|---------|
| `session.created` | Session is ready | All sessions |
| `session.updated` | Session config updated | All sessions |
| `session.closed` | Translation session fully closed | Translation |
| `conversation.item.added` | Conversation item added | Voice agent |
| `conversation.item.done` | Conversation item complete | Voice agent |
| `input_audio_buffer.speech_started` | User started speaking (VAD) | Voice agent, transcription |
| `input_audio_buffer.speech_stopped` | User stopped speaking (VAD) | Voice agent, transcription |
| `input_audio_buffer.committed` | Audio buffer committed | Voice agent |
| `response.created` | Response started | Voice agent |
| `response.output_item.added` | New output item added | Voice agent |
| `response.content_part.added` | Content part started | Voice agent |
| `response.output_text.delta` | Text streaming delta | Voice agent |
| `response.output_text.done` | Text output complete | Voice agent |
| `response.output_audio.delta` | Audio streaming delta (base64) | Voice agent, translation |
| `response.output_audio.done` | Audio output complete | Voice agent |
| `response.output_audio_transcript.delta` | Audio transcript streaming delta | Voice agent |
| `response.output_audio_transcript.done` | Audio transcript complete | Voice agent |
| `response.function_call_arguments.delta` | Function call args streaming | Voice agent |
| `response.content_part.done` | Content part complete | Voice agent |
| `response.output_item.done` | Output item complete | Voice agent |
| `response.done` | Full response complete | Voice agent |
| `response.cancelled` | Response cancelled (barge-in) | Voice agent |
| `rate_limits.updated` | Rate limit info updated | Voice agent |
| `session.output_audio.delta` | Translated audio delta | Translation |
| `session.output_transcript.delta` | Translated text delta | Translation |
| `session.input_transcript.delta` | Source transcript delta | Translation |
| `transcript.text.delta` | File transcription streaming delta | STT (stream=true) |
| `transcript.text.done` | File transcription complete | STT (stream=true) |
| `transcript.text.segment` | Diarized segment complete | STT (stream=true, diarized) |
| `error` | Error event | All sessions |

### Text Generation Event Flow (Voice Agent)

```
Client: conversation.item.create (user text)
Client: response.create
Server: conversation.item.added → conversation.item.done
Server: response.created → response.output_item.added → response.content_part.added
Server: response.output_text.delta (repeated)
Server: response.output_text.done → response.content_part.done → response.output_item.done
Server: response.done → rate_limits.updated
```

### Audio I/O Event Flow (WebSocket, Voice Agent)

| Stage | Client Events | Server Events |
|-------|--------------|---------------|
| Session init | `session.update` | `session.created`, `session.updated` |
| User audio input | `conversation.item.create` (whole audio), `input_audio_buffer.append` (stream chunks), `input_audio_buffer.commit` (VAD off), `response.create` (VAD off) | `input_audio_buffer.speech_started`, `input_audio_buffer.speech_stopped`, `input_audio_buffer.committed` |
| Server audio output | `input_audio_buffer.clear` (VAD off) | `conversation.item.added`, `conversation.item.done`, `response.created`, `response.output_item.added/created`, `response.content_part.added`, `response.output_audio.delta`, `response.output_audio.done`, `response.output_audio_transcript.delta`, `response.output_audio_transcript.done`, `response.output_text.delta`, `response.output_text.done`, `response.content_part.done`, `response.output_item.done`, `response.done`, `rate_limits.updated` |

---

## 14. Voice Activity Detection (VAD)

### Overview

VAD automatically detects when the user has started or stopped speaking. Enabled by default in speech-to-speech sessions. Optional in transcription sessions (model-dependent). Configured via `session.audio.input.turn_detection` in `session.update`.

### Events

- `input_audio_buffer.speech_started` — Start of a speech turn
- `input_audio_buffer.speech_stopped` — End of a speech turn

### Two VAD Modes

| Mode | Detection Method | Default |
|------|------------------|---------|
| `server_vad` | Periods of silence | Yes (for supporting models) |
| `semantic_vad` | Semantic classifier — scores probability user is done speaking based on words uttered | No |

### Server VAD Configuration

```json
{
  "type": "session.update",
  "session": {
    "audio": {
      "input": {
        "turn_detection": {
          "type": "server_vad",
          "threshold": 0.5,
          "prefix_padding_ms": 300,
          "silence_duration_ms": 500,
          "create_response": true,
          "interrupt_response": true
        }
      }
    }
  }
}
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `threshold` | float | 0.5 | Activation threshold (0–1). Higher = requires louder audio (better in noisy environments) |
| `prefix_padding_ms` | int | 300 | Audio (ms) to include before VAD-detected speech |
| `silence_duration_ms` | int | 500 | Silence duration (ms) to detect speech stop. Shorter = quicker turn detection |
| `create_response` | boolean | true | Whether to auto-create a response when turn ends (conversation only) |
| `interrupt_response` | boolean | true | Whether user speech interrupts in-progress response (conversation only) |

### Semantic VAD Configuration

```json
{
  "type": "session.update",
  "session": {
    "audio": {
      "input": {
        "turn_detection": {
          "type": "semantic_vad",
          "eagerness": "medium",
          "create_response": true,
          "interrupt_response": true
        }
      }
    }
  }
}
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `eagerness` | enum | `auto` (= `medium`) | How eager to interrupt: `auto` (=`medium`), `low` (let user take time), `medium`, `high` (chunk ASAP) |
| `create_response` | boolean | true | Auto-create response (conversation only) |
| `interrupt_response` | boolean | true | Allow interruption (conversation only) |

> `create_response` and `interrupt_response` are conversation-only fields. In transcription sessions, VAD only controls audio chunking.

### Disabling VAD

Set `turn_detection` to `null` for push-to-talk interfaces:
```json
{
  "type": "session.update",
  "session": {
    "audio": {
      "input": {
        "turn_detection": null
      }
    }
  }
}
```

When VAD is disabled, the client must manually:
- `input_audio_buffer.commit` (creates a new user input item)
- `response.create` (triggers a response)
- `input_audio_buffer.clear` (before beginning new user input)

### VAD Without Auto-Response

Keep VAD enabled but prevent automatic responses:
```json
{
  "turn_detection": {
    "type": "server_vad",
    "create_response": false,
    "interrupt_response": false
  }
}
```
Client triggers responses manually with `response.create`.

### Transcription Session VAD

- Models that support VAD default to `server_vad`
- `gpt-realtime-whisper` requires `turn_detection` omitted or set to `null` (manual commit)
- In transcription mode, `eagerness` affects how audio is chunked even without model replies

---

## 15. Interruption & Barge-in Handling

### Concept

In voice applications, users can interrupt the model while it's speaking. The Realtime API handles this by detecting user speech, cancelling the ongoing response, and starting a new one. This is called **truncating** the model's last response — removing the unplayed portion from the conversation.

### Platform Differences

| Transport | Truncation Handling |
|-----------|-------------------|
| **WebRTC / SIP** | Server manages output audio buffer, knows how much has been played. Auto-truncates unplayed audio on interruption. |
| **WebSocket** | Client manages audio playback. Must stop playback and handle truncation manually. |

### WebSocket Truncation Procedure

1. Client monitors for `input_audio_buffer.speech_started` (user started speaking)
2. Server auto-cancels in-progress response → emits `response.cancelled`
3. Client immediately stops playback of current model audio
4. Client notes how much audio was played before interruption
5. Client sends `conversation.item.truncate` to remove unplayed audio:

```json
{
  "type": "conversation.item.truncate",
  "item_id": "item_1234",
  "content_index": 0,
  "audio_end_ms": 1500
}
```

| Field | Description |
|-------|-------------|
| `item_id` | ID of the model's last response item |
| `content_index` | Content part index |
| `audio_end_ms` | Truncate point in milliseconds |

### Transcript Truncation Caveat

The realtime model cannot precisely align transcript and audio. `conversation.item.truncate` cuts audio at a given point and removes the text transcript for the unplayed portion, but does not provide a truncated transcript.

---

## 16. Function Calling in Realtime Sessions

### Overview

When updating the session or creating a response, you can specify a list of available functions for the model to call. If the model determines it should make a function call, it adds items to the conversation representing arguments to a function call.

### Registering Tools

**Session level** — `session.tools` in `session.update`:
```json
{
  "type": "session.update",
  "session": {
    "tools": [
      {
        "type": "function",
        "name": "generate_horoscope",
        "description": "Give today's horoscope for an astrological sign.",
        "parameters": {
          "type": "object",
          "properties": {
            "sign": {
              "type": "string",
              "description": "The sign for the horoscope.",
              "enum": ["aries", "taurus", ...]
            }
          },
          "required": ["sign"]
        }
      }
    ],
    "tool_choice": "auto"
  }
}
```

**Response level** — `response.tools` in `response.create`.

### Function Call Flow

```
1. Client: session.update (with tools + tool_choice) OR response.create (with tools)
2. Client: conversation.item.create (user input)
3. Client: response.create
4. Server: response.function_call_arguments.delta (streaming arguments)
5. Server: response.done (contains function_call output item)
6. Client: conversation.item.create (function_call_output, referencing call_id)
7. Client: response.create (model responds using tool output)
```

### Function Call Response Item (`response.done`)

```json
{
  "type": "response.done",
  "response": {
    "id": "resp_...",
    "status": "completed",
    "output": [
      {
        "type": "function_call",
        "status": "completed",
        "name": "generate_horoscope",
        "call_id": "call_sHlR7iaFwQ2YQOqm",
        "arguments": "{\"sign\":\"Aquarius\"}"
      }
    ]
  }
}
```

| Property | Purpose |
|----------|---------|
| `type` | `"function_call"` — indicates this response contains function call arguments |
| `name` | The name of the configured function to call |
| `arguments` | JSON string containing arguments to the function |
| `call_id` | System-generated ID — **required to pass function call result back to the model** |

### Returning Function Results

```json
{
  "type": "conversation.item.create",
  "item": {
    "type": "function_call_output",
    "call_id": "call_sHlR7iaFwQ2YQOqm",
    "output": "{\"horoscope\": \"You will soon meet a new friend.\"}"
  }
}
```

Then send `response.create` to let the model respond using the function output.

---

## 17. Image Input in Realtime Sessions

`gpt-realtime-2` and later support image input. Attach an image as a content part in a user message:

```javascript
const event = {
  type: "conversation.item.create",
  item: {
    type: "message",
    role: "user",
    content: [
      {
        type: "input_image",
        image_url: `data:image/${format};base64,${base64Image}`
      }
    ]
  }
};
```

| Field | Description |
|-------|-------------|
| `type` | `"input_image"` |
| `image_url` | Base64 data URI (`data:image/{format};base64,...`) |

The model can incorporate what's in the image when it responds. No separate `image_input` event type — images ride on the standard `conversation.item.create` message mechanism.

---

## 18. Capability Summary & Cross-Reference

### Complete API Endpoint Map

| Capability | Endpoint Base | Methods | Protocol |
|-----------|--------------|---------|----------|
| **Voice Agents (Realtime)** | `/v1/realtime` | WebSocket session | WebSocket |
| **Realtime WebRTC** | `/v1/realtime/calls` | POST (SDP exchange) | WebRTC |
| **Realtime Client Secrets** | `/v1/realtime/client_secrets` | POST | HTTP |
| **Realtime Translation** | `/v1/realtime/translations` | WebSocket session | WebSocket |
| **Translation WebRTC** | `/v1/realtime/translations/calls` | POST (SDP exchange) | WebRTC |
| **Translation Client Secrets** | `/v1/realtime/translations/client_secrets` | POST | HTTP |
| **Text to Speech** | `/v1/audio/speech` | POST | HTTP |
| **Speech to Text** | `/v1/audio/transcriptions` | POST | HTTP |
| **Audio Translation** | `/v1/audio/translations` | POST | HTTP |
| **Custom Voice Consents** | `/v1/audio/voice_consents` | POST | HTTP |
| **Custom Voices** | `/v1/audio/voices` | POST | HTTP |
| **Multimodal Audio Chat** | `/v1/chat/completions` | POST | HTTP |

### Capability Decision Matrix

| If you need... | Use... | Key Parameters |
|----------------|--------|----------------|
| Low-latency voice agent (listen, reason, speak) | Realtime session (`/v1/realtime`) with `gpt-realtime-2.1` | `model`, `voice`, `instructions`, `turn_detection`, `tools` |
| Live speech translation | Translation session (`/v1/realtime/translations`) with `gpt-realtime-translate` | `audio.output.language`, continuous audio append |
| Live streaming transcription | Transcription session with `gpt-realtime-whisper` | `audio.input.transcription.model`, `delay`, `language` |
| File transcription | `/v1/audio/transcriptions` with `gpt-4o-transcribe` | `file`, `model`, `response_format`, `prompt`, `stream` |
| Speaker-diarized transcription | `/v1/audio/transcriptions` with `gpt-4o-transcribe-diarize` | `response_format=diarized_json`, `chunking_strategy`, `known_speaker_names[]` |
| Audio translation to English | `/v1/audio/translations` with `whisper-1` | `file`, `model` |
| Text to speech | `/v1/audio/speech` with `gpt-4o-mini-tts` | `model`, `input`, `voice`, `instructions`, `response_format` |
| Audio in existing chat app | `/v1/chat/completions` with `gpt-audio-1.5` | `modalities`, `audio`, `messages` with `input_audio` |
| Create a custom voice | `/v1/audio/voices` (consent + sample) | `name`, `audio_sample`, `consent` |
| Use custom voice in TTS | `/v1/audio/speech` | `voice: { id: "voice_123abc" }` |
| Use custom voice in Realtime | Realtime session config | `audio.output.voice: { id: "voice_123abc" }` |
| Push-to-talk voice agent | Realtime session with `turn_detection: null` | Manual `input_audio_buffer.commit` + `response.create` |
| Barge-in / interruption handling | Realtime session with VAD enabled | `input_audio_buffer.speech_started` → `conversation.item.truncate` |
| Function calling in voice | Realtime session with `tools` | `session.tools`, `tool_choice`, `function_call_output` |
| Image input in voice | Realtime session with `gpt-realtime-2+` | `input_image` content part with `image_url` data URI |
| Server-stored prompts in Realtime | Realtime session `prompt` field | `prompt: { id, version, variables }` |
| Out-of-band response (not in conversation) | `response.create` with `conversation: "none"` | `response.input`, `response.metadata` |
| Control TTS speech style | `gpt-4o-mini-tts` `instructions` parameter | Accent, emotion, intonation, speed, tone, whispering |
| Transcription with word timestamps | `/v1/audio/transcriptions` with `whisper-1` | `response_format=verbose_json`, `timestamp_granularities=["word"]` |
| Transcription with logprobs | `/v1/audio/transcriptions` with GPT-4o models | `include[]=logprobs` |