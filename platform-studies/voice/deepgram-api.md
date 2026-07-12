# Deepgram API Analysis — Speech-to-Text, Text-to-Speech, Voice Agents & Audio Intelligence

> **Base URL:** `https://api.deepgram.com` (REST & WebSocket STT/TTS) | **Agent WS:** `wss://agent.deepgram.com` | **STT WS:** `wss://api.deepgram.com`
> **Docs:** `https://developers.deepgram.com` | **API Reference:** `https://developers.deepgram.com/reference/deepgram-api-overview`
> **Auth:** API key via `Authorization: Token <API_KEY>` header (or `Bearer <JWT>` for temporary tokens)
> **SDKs:** Python (`deepgram-sdk`), JavaScript (`@deepgram/sdk`), Go, .NET, Java
> **Playground:** `https://playground.deepgram.com` | **MCP Server:** `https://developers.deepgram.com/_mcp/server`
> **Description:** Deepgram is a voice AI platform providing APIs for speech-to-text (batch, streaming, and conversational turn-based), text-to-speech (single request and continuous streaming), end-to-end voice agents over a single WebSocket, text/audio intelligence features (summarization, sentiment, topics, intents, entities), and account management. Its models span 50+ languages with Nova-3 as the flagship STT model, Flux for conversational turn detection, and Aura-2 for TTS across English, Spanish, German, French, Dutch, Italian, and Japanese.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Authentication & Access Control](#2-authentication--access-control)
3. [Models — Catalog & Selection Guide](#3-models--catalog--selection-guide)
4. [Speech-to-Text — Pre-Recorded (Batch)](#4-speech-to-text--pre-recorded-batch)
5. [Speech-to-Text — Live Streaming (WebSocket)](#5-speech-to-text--live-streaming-websocket)
6. [Speech-to-Text — Flux / Turn-Based (WebSocket v2)](#6-speech-to-text--flux--turn-based-websocket-v2)
7. [Text-to-Speech — Single Request (REST)](#7-text-to-speech--single-request-rest)
8. [Text-to-Speech — Continuous Stream (WebSocket)](#8-text-to-speech--continuous-stream-websocket)
9. [Voice Agent API — End-to-End Conversational Agent](#9-voice-agent-api--end-to-end-conversational-agent)
10. [Text Intelligence / Read API](#10-text-intelligence--read-api)
11. [Audio Intelligence Features (STT Add-Ons)](#11-audio-intelligence-features-stt-add-ons)
12. [Account & Management APIs](#12-account--management-apis)
13. [Concurrency, Limits & Rate Limits](#13-concurrency-limits--rate-limits)
14. [Audio Formats Reference](#14-audio-formats-reference)
15. [TTS Voice Catalog](#15-tts-voice-catalog)
16. [Capability Summary & Cross-Reference](#16-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Deepgram's platform is organized around these core abstractions:

- **Model** — The AI model used to process audio or text. STT models are identified by model name (e.g. `nova-3`, `nova-2-general`, `flux-general-en`, `whisper-large`). TTS models follow the pattern `aura-2-{voicename}-{language}` (e.g. `aura-2-thalia-en`). Domain-specific STT variants exist for medical, meeting, finance, phonecall, voicemail, video, drivethru, automotive, and conversational AI.
- **Language** — BCP-47 language tag (e.g. `en`, `en-US`, `es-419`, `zh-CN`). The `multi` value enables multilingual codeswitching for Nova-3 and Nova-2. Flux supports `flux-general-multi` for 10 languages.
- **Channel** — An audio channel in multi-channel input. The `multichannel` parameter transcribes each channel independently. The `channels` parameter (streaming) specifies the number of audio channels.
- **Alternative** — A transcription hypothesis in the response. Each channel returns one or more `alternatives`, each containing a `transcript`, `confidence`, and `words` array.
- **Utterance** — A semantically meaningful speech segment. Enabled via `utterances=true` with configurable split timing via `utt_split` (default 0.8s).
- **Endpointing** — Streaming feature that detects natural speech pauses to finalize transcripts. Configurable via `endpointing` parameter (default 10 = 10ms of silence). `speech_final` and `is_final` flags distinguish between speech-level and Deepgram-level finality.
- **Smart Format** — Applies formatting to transcripts for improved readability (currency, phone numbers, email addresses, dates, etc.). Enabled via `smart_format=true`.
- **Diarization** — Speaker identification that assigns a speaker number to each word. Configured via `diarize_model` (`latest`, `v1`, `v2` for batch; `latest`, `v1` for streaming).
- **Keyterm / Keywords** — Custom vocabulary boosting to improve recognition of specialized terminology. `keyterm` (Nova-3 only, plain terms) and legacy `keywords` (weighted format `term:boost`).
- **End-of-Turn (EOT)** — Flux-specific concept. Model-integrated turn detection that signals when a user has finished speaking. Configurable via `eot_threshold` (0.5–0.9, default 0.7), `eager_eot_threshold` (0.3–0.9), and `eot_timeout_ms` (default 5000).
- **Provider** — Voice Agent API abstraction for the three pipeline stages: Listen (STT), Think (LLM), Speak (TTS). Each provider has a `type` (e.g. `deepgram`, `open_ai`, `anthropic`, `google`, `groq`, `aws_bedrock`) and model selection.
- **MIP (Model Improvement Program)** — Optional opt-out via `mip_opt_out=true` to exclude requests from Deepgram's model training.

### Platform Architecture

```
Audio Input (file/URL) ──▶ Pre-Recorded STT (POST /v1/listen) ──▶ Transcript JSON
                              │
    Model Selection ──▶ nova-3, nova-2, enhanced, base, whisper
    Features ──▶ smart_format, diarize, punctuate, paragraphs, redact
    Intelligence ──▶ summarize, sentiment, topics, intents, entities

Audio Stream ──▶ Live Streaming STT (WSS /v1/listen) ──▶ Results/Metadata/UtteranceEnd events
                    │
    Interim Results ──▶ is_final=false (preliminary, may evolve)
    Endpointing ──▶ speech_final=true (natural speech pause)
    VAD Events ──▶ SpeechStarted (speech detected)
    KeepAlive ──▶ pause connection without closing

Audio Stream ──▶ Flux STT (WSS /v2/listen) ──▶ TurnInfo events
                    │
    Turn Events ──▶ StartOfTurn, Update, EagerEndOfTurn, TurnResumed, EndOfTurn
    Configurable ──▶ eot_threshold, eager_eot_threshold, eot_timeout_ms
    Multilingual ──▶ flux-general-multi with language_hints

Text Input ──▶ TTS REST (POST /v1/speak) ──▶ Audio file (MP3/WAV/FLAC/Opus/μ-law/a-law/AAC)
Text Stream ──▶ TTS WebSocket (WSS /v1/speak) ──▶ Audio chunks + Metadata/Flushed/Cleared/Warning

Audio Stream ──▶ Voice Agent (WSS /v1/agent/converse) ──▶ Full conversation
                    │
    Listen (STT) ──▶ deepgram (v1=nova-3, v2=flux)
    Think (LLM)  ──▶ open_ai, anthropic, google, groq, aws_bedrock, or custom endpoint
    Speak (TTS)  ──▶ deepgram (aura-2 voices)
    Functions    ──▶ client-side or server-side execution
```

### End-to-End Flows

**Pre-recorded transcription pipeline:**
```
Select model ──▶ POST /v1/listen with query params ──▶ JSON response
                  │                                     │
        audio via URL or binary body        transcript, confidence, words, paragraphs
        features: smart_format, diarize    intelligence: summary, sentiment, topics, intents, entities
        callback URL for async processing  tags for usage reporting
```

**Live streaming transcription pipeline:**
```
Open WSS /v1/listen?model=nova-3 ──▶ Metadata event (connection established)
                  │
    Send audio chunks (binary) ──▶ Results events (interim + final)
    Send KeepAlive ──▶ pause without closing
    Send Finalize ──▶ flush remaining audio
    Send CloseStream ──▶ close connection
```

**Voice Agent pipeline (single WebSocket):**
```
Open WSS /v1/agent/converse ──▶ Welcome event
  │
  ├─ Send Settings ──▶ SettingsApplied event
  │    (audio config, listen/think/speak providers, prompt, functions)
  │
  ├─ Send audio (binary) ──▶ UserStartedSpeaking ──▶ ConversationText (user role)
  │                                          ──▶ AgentThinking ──▶ FunctionCallRequest (if tools)
  │                                          ──▶ AgentStartedSpeaking ──▶ Audio chunks (binary)
  │                                          ──▶ AgentAudioDone
  │
  ├─ UpdateListen/UpdateThink/UpdateSpeak/UpdatePrompt ──▶ *Updated acknowledgements
  ├─ InjectUserMessage/InjectAgentMessage ──▶ InjectionRefused (if invalid timing)
  ├─ SendFunctionCallResponse ──▶ (client-side function results)
  └─ KeepAlive ──▶ keep connection alive
```

---

## 2. Authentication & Access Control

### API Keys

All API requests require an `Authorization` header:

```
Authorization: Token YOUR_DEEPGRAM_API_KEY
```

API keys are project-scoped and can be managed via the Management API:
- **List keys:** `GET /v1/manage/projects/{project_id}/keys`
- **Create key:** `POST /v1/manage/projects/{project_id}/keys` with scopes
- **Delete key:** `DELETE /v1/manage/projects/{project_id}/keys/{key_id}`

### Temporary API Tokens (JWT)

For client-side connections without exposing the API key, temporary JWT tokens can be created:

```
POST /v1/auth/tokens/grant
```

These tokens are used with `Authorization: Bearer <JWT>` and are time-limited.

### SDK Authentication

```python
from deepgram import DeepgramClient
client = DeepgramClient()  # reads DEEPGRAM_API_KEY env var
```

```javascript
const { DeepgramClient } = require("@deepgram/sdk");
const deepgram = new DeepgramClient({ apiKey: process.env.DEEPGRAM_API_KEY });
```

### Model Improvement Program (MIP)

By default, requests may be used to improve Deepgram models. Opt out with `mip_opt_out=true` query parameter (check docs for pricing implications).

---

## 3. Models — Catalog & Selection Guide

### STT Models

| Model | Type | Key Strengths | Languages |
|-------|------|---------------|-----------|
| **Flux** (`flux-general-en`) | Streaming v2 | Model-native turn detection, ultra-low latency, conversational | English |
| **Flux Multi** (`flux-general-multi`) | Streaming v2 | Multilingual turn detection, code-switching | en, es, fr, de, hi, ru, pt, ja, it, nl |
| **Nova-3** (`nova-3` / `nova-3-general`) | Batch + Streaming | Highest accuracy, 50+ languages, multilingual codeswitching, keyterm prompting, self-serve customization | 50+ languages, `multi` for codeswitching |
| **Nova-3 Medical** (`nova-3-medical`) | Batch + Streaming | Medical domain vocabulary | English |
| **Nova-2** (`nova-2-general`) | Batch + Streaming | Languages not yet in Nova-3, filler words, domain variants | 30+ languages, `multi` (es+en) |
| **Nova-2 domain variants** | Batch + Streaming | Optimized: meeting, phonecall, finance, conversationalai, voicemail, video, medical, drivethru, automotive | English (mostly) |
| **Nova** (legacy) | Batch + Streaming | Predecessor to Nova-2 | en, es, hi-Latn |
| **Enhanced** (legacy) | Batch + Streaming | Lower WER than Base, keyword boosting | 13+ languages |
| **Base** (legacy, default) | Batch + Streaming | Large volume, high accuracy timestamps | 20+ languages |
| **Whisper Cloud** | Batch | OpenAI Whisper managed API | Multilingual |

### TTS Models (Aura)

| Model Family | Generation | Languages | Notable Voices |
|-------------|-----------|-----------|----------------|
| **Aura-2** | Latest | en, es, de, fr, nl, it, ja | 40+ English, 10+ Spanish, 9 Dutch, 7 German, 2 French, 10 Italian, 5 Japanese |
| **Aura-1** | Legacy | en | 12 English voices |

TTS model naming convention: `aura-{version}-{voicename}-{language}` (e.g. `aura-2-thalia-en`).

### LLM Models (Voice Agent "Think" Provider)

| Provider | Type | Models |
|----------|------|--------|
| **OpenAI** | `open_ai` | gpt-5, gpt-5-mini, gpt-5-nano, gpt-4.1, gpt-4.1-mini, gpt-4.1-nano, gpt-4o, gpt-4o-mini |
| **Anthropic** | `anthropic` | claude-3-5-haiku-latest, claude-sonnet-4-20250514 |
| **Google** | `google` | gemini-2.0-flash, gemini-2.0-flash-lite, gemini-2.5-flash |
| **Groq** | `groq` | openai/gpt-oss-20b |
| **AWS Bedrock** | `aws_bedrock` | anthropic/claude-3-5-sonnet, anthropic/claude-3-5-haiku |
| **Custom** | any | Custom LLM endpoint via `endpoint.url` + `endpoint.headers` |

---

## 4. Speech-to-Text — Pre-Recorded (Batch)

### API Function: Transcribe Pre-Recorded Audio

| Property | Value |
|----------|-------|
| **Endpoint** | `POST https://api.deepgram.com/v1/listen` |
| **Protocol** | REST (HTTP POST) |
| **Auth** | `Authorization: Token <API_KEY>` |
| **Content-Type** | `application/json` (URL payload) or audio MIME type (binary file) |
| **Response** | JSON (`ListenV1Response`) or async acceptance (`ListenV1AcceptedResponse` with `request_id`) |

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string (URI) | Yes (URL mode) | Audio file URL to transcribe |

For local files, send raw audio bytes as the request body with the appropriate `Content-Type` header (e.g. `audio/wav`, `audio/mp3`).

### Query Parameters

#### Model & Language

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | `base-general` | AI model: `nova-3`, `nova-2-general`, `enhanced`, `base`, `whisper-large`, etc. |
| `language` | string | `en` | BCP-47 language tag. Use `multi` for codeswitching |
| `version` | string | `latest` | Model version |
| `detect_language` | boolean\|array | `false` | Auto-detect dominant language. Can pass array of candidate languages |
| `multichannel` | boolean | `false` | Transcribe each audio channel independently |

#### Formatting & Readability

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `smart_format` | boolean | `false` | Apply formatting (currency, phone numbers, emails, dates) |
| `punctuate` | boolean | `false` | Add punctuation and capitalization |
| `paragraphs` | boolean | `false` | Split transcript into paragraphs |
| `numerals` | boolean | `false` | Convert written numbers to numerical format |
| `dictation` | boolean | `false` | Dictation mode for formatting dictated speech |
| `profanity_filter` | boolean | `false` | Filter recognized profanity |
| `filler_words` | boolean | `false` | Include filler words ("uh", "um") |
| `measurements` | boolean | `false` | Convert spoken measurements to abbreviations |

#### Speaker Identification

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `diarize` | boolean | `false` | **Deprecated** — use `diarize_model` instead |
| `diarize_model` | string | — | Diarization model: `latest` (v2), `v1`, `v2`. Enables diarization |
| `utterances` | boolean | `false` | Segment speech into semantic units |
| `utt_split` | number | `0.8` | Seconds to wait before detecting pause between words |

#### Custom Vocabulary

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `keyterm` | array<string> | — | Keyterm prompting (Nova-3 only). Plain terms, no weights |
| `keywords` | string\|array | — | Legacy keyword boosting. Format: `term:boost` (boost 0–100) |
| `search` | string\|array | — | Search for terms/phrases in audio. Returns hits with confidence + timestamps |
| `replace` | string\|array | — | Search and replace terms in transcript |
| `redact` | string\|array | `false` | Redact sensitive info: `pci`, `pii`, `numbers`, `ssn`, `aggressive_numbers` |

#### Audio Intelligence (Audio Intelligence Features)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `summarize` | boolean\|string | `false` | Summarize content. Accepts `true`, `false`, or `"v2"` |
| `sentiment` | boolean | `false` | Recognize sentiment throughout transcript |
| `topics` | boolean | `false` | Detect topics throughout transcript |
| `custom_topic` | string\|array | — | Custom topics to detect (up to 100) |
| `custom_topic_mode` | string | `extended` | `strict` (only custom topics) or `extended` (custom + detected) |
| `intents` | boolean | `false` | Recognize speaker intent |
| `custom_intent` | string\|array | — | Custom intents to detect |
| `custom_intent_mode` | string | `extended` | `strict` or `extended` |
| `detect_entities` | boolean | `false` | Extract key entities (names, orgs, dates, etc.) |

#### Result Processing

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `callback` | string | — | URL for async callback delivery of results |
| `callback_method` | string | `POST` | HTTP method for callback: `POST` or `PUT` |
| `tag` | string\|array | — | Labels for usage reporting identification |
| `extra` | string\|array | — | Arbitrary key-value pairs attached to response |
| `mip_opt_out` | boolean | `false` | Opt out of Model Improvement Program |

### Response Structure

```json
{
  "metadata": {
    "request_id": "uuid",
    "sha256": "hash",
    "created": "ISO-8601",
    "duration": 25.93,
    "channels": 1,
    "models": ["model-uuid"],
    "model_info": { "uuid": { "name": "...", "version": "...", "arch": "..." } },
    "summary_info": { "model_uuid": "...", "input_tokens": 95, "output_tokens": 63 },
    "tags": ["tag1"]
  },
  "results": {
    "channels": [{
      "alternatives": [{
        "transcript": "Full transcript text...",
        "confidence": 0.999,
        "words": [{ "word": "...", "start": 0.08, "end": 0.32, "confidence": 0.99, "punctuated_word": "..." }],
        "paragraphs": { "transcript": "...", "paragraphs": [{ "sentences": [...], "speaker": 0, "num_words": 63 }] },
        "entities": [{ "label": "Event", "value": "...", "confidence": 0.95, "start_word": 2, "end_word": 3 }],
        "summaries": [{ "summary": "...", "start_word": 0, "end_word": 12 }],
        "topics": [{ "text": "...", "topics": ["Space Exploration"] }]
      }],
      "detected_language": "en"
    }],
    "utterances": [{ "start": 0, "end": 6, "confidence": 0.95, "transcript": "...", "speaker": 0, "id": "uuid" }],
    "summary": { "result": "success", "short": "Short summary..." },
    "topics": { "results": { "topics": { "segments": [{ "text": "...", "topics": [{ "topic": "...", "confidence_score": 0.9 }] }] } } },
    "intents": { "results": { "intents": { "segments": [{ "text": "...", "intents": [{ "intent": "...", "confidence_score": 0.9 }] }] } } },
    "sentiments": { "segments": [{ "text": "...", "sentiment": "positive", "sentiment_score": 0.58 }], "average": { "sentiment": "positive", "sentiment_score": 0.58 } }
  }
}
```

### Limits

- **File size:** Maximum 2 GB
- **Concurrent requests:** Up to 100 per project (Nova/Base/Enhanced)
- **Processing timeout:** 10 minutes (Nova/Base/Enhanced), 20 minutes (Whisper)

---

## 5. Speech-to-Text — Live Streaming (WebSocket)

### API Function: Transcribe Streaming Audio

| Property | Value |
|----------|-------|
| **Endpoint** | `WSS wss://api.deepgram.com/v1/listen` |
| **Protocol** | WebSocket (AsyncAPI) |
| **Auth** | `Authorization: Token <API_KEY>` (header) |
| **Connection** | Query parameters configure the session |

### Query Parameters (WebSocket connection URL)

All pre-recorded formatting/vocabulary parameters apply, plus streaming-specific parameters:

#### Streaming-Specific Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `encoding` | string | — | Required for raw audio: `linear16`, `linear32`, `flac`, `alaw`, `mulaw`, `amr-nb`, `amr-wb`, `opus`, `ogg-opus`, `speex`, `g729` |
| `sample_rate` | integer | — | Required when `encoding` is specified (e.g. 16000, 24000, 48000) |
| `channels` | integer | `1` | Number of audio channels |
| `interim_results` | string | `false` | Send ongoing transcription updates as more audio arrives |
| `endpointing` | string | `10` | Milliseconds of silence to detect end-of-speech (10–1000) |
| `vad_events` | string | `false` | Emit `SpeechStarted` events when speech is detected |
| `utterance_end_ms` | string | — | Emit `UtteranceEnd` events after specified ms of silence |
| `mip_opt_out` | string | `false` | Opt out of Model Improvement Program |

#### Streaming Model Selection

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | — | `nova-3`, `nova-2-general`, `enhanced`, `base`, `custom`, etc. |
| `language` | string | `en` | BCP-47 tag |
| `version` | string | `latest` | Model version |

### Client → Server Messages

| Message | Type | Description |
|---------|------|-------------|
| **Audio data** | binary | Raw audio chunks to transcribe |
| **Finalize** | `{ "type": "Finalize" }` | Flush the WebSocket stream, get final results for all sent audio |
| **CloseStream** | `{ "type": "CloseStream" }` | Close the WebSocket stream |
| **KeepAlive** | `{ "type": "KeepAlive" }` | Keep connection alive without sending audio (pause) |

### Server → Client Messages

| Message | Type Field | Description |
|---------|-----------|-------------|
| **Results** | `Results` | Transcription results with `is_final`, `speech_final`, `channel.alternatives`, `words`, `entities` |
| **Metadata** | `Metadata` | Connection metadata: `request_id`, `sha256`, `created`, `duration`, `channels` |
| **UtteranceEnd** | `UtteranceEnd` | Utterance end event with `channel` and `last_word_end` timestamp |
| **SpeechStarted** | `SpeechStarted` | Speech detected event with `channel` and `timestamp` |

### Results Message Structure

```json
{
  "type": "Results",
  "channel_index": [0, 1],
  "duration": 1.98,
  "start": 5.99,
  "is_final": true,
  "speech_final": true,
  "channel": {
    "alternatives": [{
      "transcript": "Tell me more about this.",
      "confidence": 0.9996,
      "words": [{ "word": "tell", "start": 6.07, "end": 6.35, "confidence": 0.998, "punctuated_word": "Tell" }]
    }]
  },
  "metadata": { "request_id": "uuid", "model_info": { "name": "...", "version": "...", "arch": "..." } },
  "from_finalize": false
}
```

### Key Concepts: `is_final` vs `speech_final`

- **`is_final: false`** — Interim result; Deepgram will continue processing and may refine the transcript
- **`is_final: true`** — Deepgram has finalized this segment; won't change with more audio
- **`speech_final: true`** — Natural speech pause detected; the utterance naturally ended here
- These are independent flags; a result can be `is_final=true` but `speech_final=false` (Deepgram committed to the text but speech continues)

---

## 6. Speech-to-Text — Flux / Turn-Based (WebSocket v2)

### API Function: Conversational Speech Recognition with Turn Detection

| Property | Value |
|----------|-------|
| **Endpoint** | `WSS wss://api.deepgram.com/v2/listen` |
| **Protocol** | WebSocket (AsyncAPI) |
| **Auth** | `Authorization: Token <API_KEY>` (header) |
| **Purpose** | Real-time conversational STT with model-native end-of-turn detection |

### Main Concepts

Flux is purpose-built for voice agents. Instead of passively transcribing, it understands conversational flow and automatically handles turn-taking:

- **Turn** — A single user speaking segment from start to end
- **End-of-Turn (EOT)** — Model-integrated detection that the user has finished speaking
- **Eager EOT** — Moderate-confidence signal to begin preparing agent reply early (reduces latency)
- **Turn Resumed** — Speech continued after an Eager EOT was fired (false alarm correction)

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | — | **Required.** `flux-general-en` or `flux-general-multi` |
| `encoding` | string | — | `linear16`, `linear32`, `mulaw`, `alaw`, `opus`, `ogg-opus` (omit for containerized audio) |
| `sample_rate` | integer | — | Sample rate in Hz |
| `eot_threshold` | number | `0.7` | End-of-turn confidence required (0.5–0.9) |
| `eager_eot_threshold` | number | — | Eager EOT confidence (0.3–0.9). Enables EagerEndOfTurn + TurnResumed events |
| `eot_timeout_ms` | integer | `5000` | Force-finish turn after this many ms post-speech |
| `keyterm` | string\|array | — | Keyterm prompting (plain terms, no weights) |
| `language_hint` | string\|array | — | Language hints for `flux-general-multi` only (BCP-47 codes) |
| `profanity_filter` | string | `false` | Filter profanity |
| `tag` | string | — | Labels for usage reporting |
| `mip_opt_out` | boolean | `false` | Opt out of MIP |

### Client → Server Messages

| Message | Type | Description |
|---------|------|-------------|
| **Audio data** | binary | Raw audio chunks |
| **CloseStream** | `{ "type": "CloseStream" }` | Close the stream |
| **Configure** | `{ "type": "Configure", "thresholds": {...}, "keyterms": [...], "language_hints": [...] }` | Update settings mid-session |

### Server → Client Messages

| Message | Type | Description |
|---------|------|-------------|
| **Connected** | `Connected` | Connection established with `request_id` and `sequence_id` |
| **TurnInfo** | `TurnInfo` | Turn state update (see events below) |
| **ConfigureSuccess** | `ConfigureSuccess` | Configure message applied successfully |
| **ConfigureFailure** | `ConfigureFailure` | Configure message rejected |
| **FatalError** | `Error` | Fatal error with `code` and `description` |

### TurnInfo Event Types

| Event | Description |
|-------|-------------|
| **`StartOfTurn`** | User has begun speaking for the first time in the turn |
| **`Update`** | Additional audio transcribed; turn state unchanged |
| **`EagerEndOfTurn`** | Moderate confidence user finished — opportunity to prepare agent reply |
| **`TurnResumed`** | Speech continued after EagerEndOfTurn (false alarm) |
| **`EndOfTurn`** | User has finished speaking for the turn |

### TurnInfo Message Structure

```json
{
  "type": "TurnInfo",
  "request_id": "uuid",
  "sequence_id": 5,
  "event": "EndOfTurn",
  "turn_index": 0,
  "audio_window_start": 0.0,
  "audio_window_end": 3.5,
  "transcript": "Hello, how can I help you?",
  "words": [{ "word": "Hello", "confidence": 0.99, "start": 0.0, "end": 0.5 }],
  "end_of_turn_confidence": 0.85,
  "languages": ["en"],
  "languages_hinted": ["en"]
}
```

### Supported Languages (Flux)

| Model | Languages |
|-------|----------|
| `flux-general-en` | English (all accents) |
| `flux-general-multi` | English, Spanish, French, German, Hindi, Russian, Portuguese, Japanese, Italian, Dutch |

---

## 7. Text-to-Speech — Single Request (REST)

### API Function: Generate Speech from Text

| Property | Value |
|----------|-------|
| **Endpoint** | `POST https://api.deepgram.com/v1/speak` |
| **Protocol** | REST (HTTP POST) |
| **Auth** | `Authorization: Token <API_KEY>` |
| **Content-Type** | `application/json` |
| **Response** | Audio stream (binary) with metadata headers |

### Request Body

```json
{
  "text": "Hello, how can I help you today?"
}
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | `aura-asteria-en` | TTS voice model (e.g. `aura-2-thalia-en`) |
| `encoding` | string | `mp3` | Output encoding: `linear16`, `flac`, `mulaw`, `alaw`, `mp3`, `opus`, `aac` |
| `container` | string | `wav` | File container: `wav`, `ogg`, `none` |
| `sample_rate` | string | `24000` | Sample rate (Hz). Depends on encoding |
| `bit_rate` | string\|number | `48000` | Bit rate (bps). For mp3: `32000` or `48000` |
| `speed` | number | `1` | Speaking rate multiplier (preserves prosody). Not all languages |
| `callback` | string | — | URL for async callback |
| `callback_method` | string | `POST` | `POST` or `PUT` |
| `tag` | string\|array | — | Labels for usage reporting |
| `mip_opt_out` | boolean | `false` | Opt out of MIP |

### Audio Format Combinations

| Encoding | Container | Sample Rate | Bit Rate |
|----------|-----------|------------|---------|
| `linear16` | `wav` or `none` | 8000, 16000, 24000, 32000, 48000 | — |
| `flac` | — | — | — |
| `mulaw` | `wav` or `none` | 8000, 16000 | — |
| `alaw` | `wav` or `none` | 8000, 16000 | — |
| `mp3` | — | 22050 (fixed) | 32000, 48000 |
| `opus` | `ogg` | 48000 (fixed) | — |
| `aac` | — | — | — |

### Response Headers

| Header | Description |
|--------|-------------|
| `content-type` | Audio MIME type (e.g. `audio/mpeg`) |
| `dg-model-name` | Model used (e.g. `aura-2-thalia-en`) |
| `dg-model-uuid` | Unique model identifier |
| `dg-char-count` | Character count of input text |
| `dg-request-id` | Unique request identifier |
| `transfer-encoding` | `chunked` (streamed response) |

### SDK Methods

| Language | Method |
|----------|--------|
| Python | `deepgram.speak.v1.audio.generate(text="...", model="aura-2-thalia-en")` |
| JavaScript | `deepgram.speak.v1.audio.generate({ text, model, encoding, container })` |
| Go | `speak.New(client).ToSave(ctx, filePath, text, options)` |
| C# | `SpeakRESTClient.ToFile(TextSource, "output.mp3", SpeakSchema)` |
| Java | `client.speak().v1().audio().generate(SpeakV1Request.builder().text("...").build())` |

### Limits

- **Input text:** Maximum 2000 characters (Aura-2 and Aura-1)
- **Error codes:** 413 (text exceeds limit), 422 (unprocessable content), 429 (rate limited)

---

## 8. Text-to-Speech — Continuous Stream (WebSocket)

### API Function: Stream Text to Speech

| Property | Value |
|----------|-------|
| **Endpoint** | `WSS wss://api.deepgram.com/v1/speak` |
| **Protocol** | WebSocket (AsyncAPI) |
| **Auth** | `Authorization: Token <API_KEY>` (header) |
| **Purpose** | Continuous text streaming with real-time audio generation |

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | `aura-asteria-en` | TTS voice model |
| `encoding` | string | `linear16` | Streaming-compatible: `linear16`, `mulaw`, `alaw` |
| `sample_rate` | string | `24000` | 8000, 16000, 24000, 32000, 48000 |
| `speed` | number | `1` | Speaking rate multiplier |
| `mip_opt_out` | boolean | `false` | Opt out of MIP |

### Client → Server Messages

| Message | Type | Description |
|---------|------|-------------|
| **Speak** | `{ "type": "Speak", "text": "Hello world" }` | Text to convert to audio |
| **Flush** | `{ "type": "Flush" }` | Flush buffer and receive final audio for text sent so far |
| **Clear** | `{ "type": "Clear" }` | Clear buffer and start new generation (potentially destructive) |
| **Close** | `{ "type": "Close" }` | Flush buffer and close connection gracefully |

### Server → Client Messages

| Message | Type | Description |
|---------|------|-------------|
| **Audio** | binary | Audio chunk as it is generated |
| **Metadata** | `Metadata` | Generation metadata: `request_id`, `model_name`, `model_version`, `model_uuid` |
| **Flushed** | `Flushed` | Flush operation completed, with `sequence_id` |
| **Cleared** | `Cleared` | Clear operation completed, with `sequence_id` |
| **Warning** | `Warning` | Non-fatal warning with `description` and `code` |

### Metadata Message Structure

```json
{
  "type": "Metadata",
  "request_id": "uuid",
  "model_name": "aura-2-thalia-en",
  "model_version": "...",
  "model_uuid": "uuid",
  "additional_model_uuids": ["uuid"]
}
```

---

## 9. Voice Agent API — End-to-End Conversational Agent

### API Function: Build a Conversational Voice Agent

| Property | Value |
|----------|-------|
| **Endpoint** | `WSS wss://agent.deepgram.com/v1/agent/converse` |
| **Protocol** | WebSocket (AsyncAPI) |
| **Auth** | `Authorization: Token <API_KEY>` (header) |
| **Purpose** | Full speech pipeline (STT → LLM → TTS) over a single WebSocket connection |

### Main Concepts

- **Listen** — The STT component. Two API versions: V1 (Nova-3/Nova-2 standard STT) and V2 (Flux with turn detection)
- **Think** — The LLM/reasoning component. Supports OpenAI, Anthropic, Google, Groq, AWS Bedrock, or custom endpoints
- **Speak** — The TTS component. Uses Deepgram Aura voices
- **Function Calling** — Agent can call external APIs/tools mid-conversation. Functions can be client-side (client executes and returns response) or server-side (Deepgram calls the endpoint)
- **Context** — Conversation history including messages and function call records, passed via settings
- **Injection** — Inject user or agent messages into the conversation (may be refused if timing is invalid)
- **Latency Metrics** — `total_latency`, `tts_latency`, `ttt_latency` (text-to-text/LLM) reported on `AgentStartedSpeaking`

### Client → Server Messages

| Message | Type | Description |
|---------|------|-------------|
| **Settings** | `Settings` | Initial configuration (audio, listen, think, speak, agent prompt, functions, context) |
| **UpdateListen** | `UpdateListen` | Update STT provider settings |
| **UpdateThink** | `UpdateThink` | Update LLM provider settings |
| **UpdateSpeak** | `UpdateSpeak` | Update TTS provider settings |
| **UpdatePrompt** | `UpdatePrompt` | Update the system prompt |
| **InjectUserMessage** | `InjectUserMessage` | Inject a user message |
| **InjectAgentMessage** | `InjectAgentMessage` | Inject an agent message |
| **SendFunctionCallResponse** | `SendFunctionCallResponse` | Return client-side function execution result |
| **KeepAlive** | `KeepAlive` | Keep connection alive |
| **Media** | binary | Raw audio data for processing |

### Server → Client Messages

| Message | Type | Description |
|---------|------|-------------|
| **Welcome** | `Welcome` | Connection established with `request_id` |
| **SettingsApplied** | `SettingsApplied` | Settings configuration accepted |
| **ConversationText** | `ConversationText` | Transcript of what was said (`role`: user/assistant, `content`) |
| **UserStartedSpeaking** | `UserStartedSpeaking` | User has begun speaking |
| **AgentThinking** | `AgentThinking` | Agent's thought process text |
| **AgentStartedSpeaking** | `AgentStartedSpeaking` | Agent begins speaking, includes latency metrics |
| **AgentAudioDone** | `AgentAudioDone` | Agent has finished sending audio |
| **Audio** | binary | Raw audio data generated by the agent |
| **FunctionCallRequest** | `FunctionCallRequest` | Server requests function execution (array of functions with `id`, `name`, `arguments`, `client_side`) |
| **FunctionCallResponse** | `FunctionCallResponse` | Server-side function execution result |
| **History** | `History` | Conversation history (messages or function calls) |
| **ListenUpdated** | `ListenUpdated` | Listen settings update confirmed |
| **ThinkUpdated** | `ThinkUpdated` | Think settings update confirmed |
| **SpeakUpdated** | `SpeakUpdated` | Speak settings update confirmed |
| **PromptUpdated** | `PromptUpdated` | Prompt update confirmed |
| **InjectionRefused** | `InjectionRefused` | Injection was refused with reason |
| **Error** | `Error` | Error with `description` and `code` |
| **Warning** | `Warning` | Non-fatal warning |

### Settings Message Structure

The `Settings` message is the primary configuration sent at connection start:

```json
{
  "type": "Settings",
  "audio": {
    "input": { "encoding": "linear16", "sample_rate": 24000 },
    "output": { "encoding": "linear16", "sample_rate": 24000, "container": "none" }
  },
  "agent": {
    "listen": {
      "provider": {
        "type": "deepgram",
        "version": "v1",
        "model": "nova-3",
        "language": "en-US",
        "keyterms": ["custom term"],
        "smart_format": true
      }
    },
    "think": {
      "provider": {
        "type": "open_ai",
        "model": "gpt-4o",
        "temperature": 0.7
      },
      "prompt": "You are a helpful assistant...",
      "functions": [{
        "name": "get_weather",
        "description": "Get weather for a location",
        "parameters": {},
        "endpoint": { "url": "https://api.weather.com", "method": "GET", "headers": {} }
      }],
      "context_length": "max"
    },
    "speak": {
      "provider": {
        "type": "deepgram",
        "model": "aura-2-thalia-en",
        "speed": 1.0
      }
    },
    "context": {
      "messages": [
        { "type": "History", "role": "user", "content": "Previous user message" },
        { "type": "History", "role": "assistant", "content": "Previous agent response" }
      ]
    }
  },
  "flags": { "history": true }
}
```

### Listen Provider Configuration

**V1 (Standard STT — Nova-3/Nova-2):**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | string | — | `deepgram` (required) |
| `version` | string | — | `v1` |
| `model` | string | — | `nova-3`, `nova-2`, etc. |
| `language` | string | `en-US` | BCP-47 tag or `multi` |
| `keyterms` | array<string> | — | Keyterm prompting |
| `smart_format` | boolean | `false` | Smart formatting |

**V2 (Flux — Conversational with turn detection):**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | string | — | `deepgram` (required) |
| `version` | string | — | `v2` |
| `model` | string | — | `flux-general-en` or `flux-general-multi` (required) |
| `language_hints` | array<string> | — | Language hints (multi model only) |
| `eot_threshold` | number | `0.7` | End-of-turn confidence (0.5–0.9) |
| `eager_eot_threshold` | number | — | Eager EOT confidence (0.3–0.9) |
| `eot_timeout_ms` | integer | `5000` | Force-finish after ms post-speech |
| `keyterms` | array<string> | — | Keyterm prompting |

### Think Provider Configuration

| Provider | Key Fields |
|----------|------------|
| **OpenAI** | `type=open_ai`, `model` (gpt-5/gpt-4o/etc.), `temperature`, `reasoning_mode` (none/minimal/low/medium/high) |
| **Anthropic** | `type=anthropic`, `model`, `temperature` (0–1) |
| **Google** | `type=google`, `model` (gemini-2.x), `temperature` |
| **Groq** | `type=groq`, `model`, `temperature`, `reasoning_mode` |
| **AWS Bedrock** | `type=aws_bedrock`, `model`, `temperature`, `credentials` (STS or IAM) |
| **Custom** | `provider` + `endpoint` (`url`, `headers`) |

**Think-level settings:**

| Field | Type | Description |
|-------|------|-------------|
| `provider` | object | LLM provider config (required) |
| `endpoint` | object | Custom LLM endpoint: `{ url, headers }` |
| `functions` | array | Function definitions: `name`, `description`, `parameters`, `endpoint` (omit for client-side) |
| `prompt` | string | System prompt for the agent |
| `context_length` | string\|number | `"max"` or character count (custom endpoint only) |

### Speak Provider Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | string | — | `deepgram` (required) |
| `model` | string | — | Aura voice model (required) |
| `speed` | number | `1` | Speaking rate multiplier |

### Audio Configuration

| Component | Fields | Defaults |
|-----------|--------|----------|
| **Input** | `encoding`, `sample_rate` | `linear16`, `24000` |
| **Output** | `encoding`, `sample_rate`, `bitrate`, `container` | `linear16`, `none` |

Input encodings: `linear16`, `linear32`, `flac`, `alaw`, `mulaw`, `amr-nb`, `amr-wb`, `opus`, `ogg-opus`, `speex`, `g729`

Output encodings: `linear16`, `mulaw`, `alaw`, `mp3`, `opus`, `flac`, `aac`

### Function Calling

Functions can be defined in the `think` settings:

```json
{
  "name": "get_weather",
  "description": "Get current weather for a location",
  "parameters": { "type": "object", "properties": { "location": { "type": "string" } } },
  "endpoint": { "url": "https://api.example.com/weather", "method": "POST", "headers": {} }
}
```

- **Server-side** (`endpoint` provided): Deepgram calls the endpoint directly, returns `FunctionCallResponse`
- **Client-side** (`endpoint` omitted, `client_side: true`): Server sends `FunctionCallRequest`, client executes and returns `SendFunctionCallResponse`

### Latency Metrics

The `AgentStartedSpeaking` event includes:

| Field | Description |
|-------|-------------|
| `total_latency` | Seconds from user utterance to agent reply |
| `tts_latency` | Portion attributable to text-to-speech |
| `ttt_latency` | Portion attributable to text-to-text (LLM) |

### Additional Features

- **Multi-Agent Architecture** — Orchestrate multiple specialized agents with context-based handoff
- **Telephony** — Connect voice agents to phone networks (inbound/outbound calls)
- **Browser Agent SDK** — Four composable packages for web applications
- **Reusable Configurations** — Save and reuse agent settings across projects
- **Prompting** — System prompts shape live-call behavior

---

## 10. Text Intelligence / Read API

### API Function: Analyze Text Content

| Property | Value |
|----------|-------|
| **Endpoint** | `POST https://api.deepgram.com/v1/read` |
| **Protocol** | REST (HTTP POST) |
| **Auth** | `Authorization: Token <API_KEY>` |
| **Content-Type** | `application/json` |
| **Response** | JSON (`ReadV1Response`) |

### Request Body

```json
// Option 1: Plain text
{ "text": "The text to analyze..." }

// Option 2: URL to text source
{ "url": "https://example.com/document.txt" }
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `language` | string | `en` | BCP-47 language tag |
| `summarize` | boolean\|string | `false` | Summarize content (boolean or `"v2"`) |
| `sentiment` | boolean | `false` | Recognize sentiment throughout text |
| `topics` | boolean | `false` | Detect topics throughout text |
| `custom_topic` | string\|array | — | Custom topics to detect (up to 100) |
| `custom_topic_mode` | string | `extended` | `strict` or `extended` |
| `intents` | boolean | `false` | Recognize intent throughout text |
| `custom_intent` | string\|array | — | Custom intents to detect |
| `custom_intent_mode` | string | `extended` | `strict` or `extended` |
| `callback` | string | — | URL for async callback |
| `callback_method` | string | `POST` | `POST` or `PUT` |
| `tag` | string\|array | — | Labels for usage reporting |

### Response Structure

```json
{
  "metadata": {
    "metadata": {
      "request_id": "uuid",
      "created": "ISO-8601",
      "language": "en",
      "summary_info": { "model_uuid": "...", "input_tokens": 350, "output_tokens": 75 },
      "sentiment_info": { ... },
      "topics_info": { ... },
      "intents_info": { ... }
    }
  },
  "results": {
    "summary": { "results": { "summary": { "text": "Summary text..." } } },
    "topics": { "results": { "topics": { "segments": [{ "text": "...", "topics": [{ "topic": "...", "confidence_score": 0.9 }] }] } } },
    "intents": { "results": { "intents": { "segments": [{ "text": "...", "intents": [{ "intent": "...", "confidence_score": 0.9 }] }] } } },
    "sentiments": { "segments": [{ "text": "...", "sentiment": "positive", "sentiment_score": 0.58 }], "average": { "sentiment": "positive", "sentiment_score": 0.58 } }
  }
}
```

---

## 11. Audio Intelligence Features (STT Add-Ons)

These features are available as query parameters on the Pre-Recorded STT API (`POST /v1/listen`) and the Text Intelligence API (`POST /v1/read`). Some are also available on the Streaming STT API.

### Feature Summary

| Feature | Parameter | Available On | Description |
|---------|-----------|-------------|-------------|
| **Summarization** | `summarize` | Pre-recorded, Read | Generate concise summary of content. Accepts `true`/`false`/`"v2"` |
| **Sentiment Analysis** | `sentiment` | Pre-recorded, Read | Recognize sentiment (positive/negative/neutral) with score per segment + average |
| **Topic Detection** | `topics` | Pre-recorded, Read | Detect topics with confidence scores. Custom topics via `custom_topic` |
| **Intent Recognition** | `intents` | Pre-recorded, Read | Recognize speaker intent with confidence scores. Custom intents via `custom_intent` |
| **Entity Detection** | `detect_entities` | Pre-recorded, Streaming | Extract entities (NAME, PHONE_NUMBER, EMAIL_ADDRESS, ORGANIZATION, CARDINAL, etc.) |

### Intelligence Response Objects

**Sentiment:**
```json
{
  "segments": [{ "text": "...", "start_word": 0, "end_word": 69, "sentiment": "positive", "sentiment_score": 0.58 }],
  "average": { "sentiment": "positive", "sentiment_score": 0.58 }
}
```

**Topics:**
```json
{
  "results": { "topics": { "segments": [{ "text": "...", "start_word": 32, "end_word": 69, "topics": [{ "topic": "Spacewalk", "confidence_score": 0.92 }] }] } }
}
```

**Intents:**
```json
{
  "results": { "intents": { "segments": [{ "text": "...", "start_word": 354, "end_word": 414, "intents": [{ "intent": "Encourage podcasting", "confidence_score": 0.004 }] }] } }
}
```

**Entities:**
```json
[{ "label": "Event", "value": "spacewalk", "raw_value": "spacewalk", "confidence": 0.95, "start_word": 2, "end_word": 3 }]
```

**Summary:**
```json
{ "result": "success", "short": "Short summary text..." }
```

### Custom Topics & Intents

- **`custom_topic`** — Submit up to 100 custom topics. `custom_topic_mode`: `strict` (only return custom topics) or `extended` (custom + model-detected)
- **`custom_intent`** — Submit custom intents. `custom_intent_mode`: `strict` or `extended` (same semantics)

---

## 12. Account & Management APIs

### Management Endpoints

| Capability | Endpoint | Method |
|-----------|----------|--------|
| **List Projects** | `/v1/manage/projects` | GET |
| **List Project Members** | `/v1/manage/projects/{id}/members` | GET |
| **List Invitations** | `/v1/manage/projects/{id}/invites` | GET |
| **List Member Scopes** | `/v1/manage/projects/{id}/members/{member_id}/scopes` | GET |
| **Get Billing Balances** | `/v1/manage/projects/{id}/balances` | GET |
| **Get Usage Breakdown** | `/v1/manage/projects/{id}/usage` | GET |
| **List Project Requests** | `/v1/manage/projects/{id}/requests` | GET |
| **Manage API Keys** | `/v1/manage/projects/{id}/keys` | GET, POST, DELETE |
| **List Models** | `/v1/manage/models` | GET |
| **Get Model Metadata** | `/v1/manage/models/{id}` | GET |
| **Create Temporary Token** | `/v1/auth/tokens/grant` | POST |
| **Self-Hosted Credentials** | `/v1/self-hosted/distribution-credentials` | GET, POST, DELETE |

### Self-Hosted Deepgram

Deepgram offers a self-hosted solution for enterprises needing:
- Better performance and latency control
- Compliance and data residency requirements
- On-premise deployment

Self-hosted add-ons are available for models and features.

---

## 13. Concurrency, Limits & Rate Limits

### STT Pre-Recorded Limits

| Limit | Value |
|-------|-------|
| File size | 2 GB max |
| Concurrent requests | 100 per project (Nova/Base/Enhanced) |
| Processing timeout | 10 min (Nova/Base/Enhanced), 20 min (Whisper) |
| Whisper concurrency | 15 (paid), 5 (pay-as-you-go) |

### STT Streaming Limits

- Rate limits apply per project
- Connections are stateful WebSocket sessions

### TTS Limits

| Limit | Value |
|-------|-------|
| Input text | 2000 characters (Aura-2 and Aura-1) |
| Rate limit | Per project concurrency limits (see API Rate Limits docs) |
| Error 413 | Input text exceeds character limit |
| Error 422 | Unprocessable content |
| Error 429 | Too many requests (concurrent limit reached) |

### Voice Agent Limits

- Per-project concurrency limits apply
- WebSocket connection-based sessions

---

## 14. Audio Formats Reference

### STT Input Encodings (Pre-Recorded)

| Encoding | Description |
|----------|-------------|
| `linear16` | 16-bit PCM (uncompressed) |
| `flac` | Free Lossless Audio Codec |
| `mulaw` | μ-law (telephony) |
| `amr-nb` | AMR narrowband |
| `amr-wb` | AMR wideband |
| `opus` | Opus codec |
| `speex` | Speex codec |
| `g729` | G.729 voice codec |

### STT Input Encodings (Streaming)

All pre-recorded encodings plus: `linear32`, `alaw`, `ogg-opus`

### TTS Output Encodings

| Encoding | Container | Sample Rates | Use Case |
|----------|-----------|--------------|----------|
| `mp3` | — | 22050 (fixed) | General playback |
| `linear16` | `wav` or `none` | 8000–48000 | High-quality, telephony |
| `flac` | — | — | Lossless |
| `mulaw` | `wav` or `none` | 8000, 16000 | Telephony |
| `alaw` | `wav` or `none` | 8000, 16000 | International telephony |
| `opus` | `ogg` | 48000 (fixed) | Real-time communication |
| `aac` | — | — | Better quality at smaller sizes |

### TTS Streaming (WebSocket) Encodings

Only streaming-compatible encodings: `linear16`, `mulaw`, `alaw`

### Voice Agent Audio

**Input encodings:** `linear16`, `linear32`, `flac`, `alaw`, `mulaw`, `amr-nb`, `amr-wb`, `opus`, `ogg-opus`, `speex`, `g729`

**Output encodings:** `linear16`, `mulaw`, `alaw`, `mp3`, `opus`, `flac`, `aac`

**Output containers:** `none`, `wav`, `ogg`

---

## 15. TTS Voice Catalog

### Aura-2 Voice Selection Guide

| Use Case | Recommended English Voices |
|----------|--------------------------|
| **Casual chat** | `aura-2-thalia-en`, `aura-2-apollo-en`, `aura-2-aries-en` |
| **Customer service** | `aura-2-arcas-en`, `aura-2-mars-en`, `aura-2-neptune-en`, `aura-2-helena-en`, `aura-2-harmonia-en` |
| **IVR** | `aura-2-thalia-en`, `aura-2-andromeda-en`, `aura-2-callista-en`, `aura-2-luna-en`, `aura-2-zeus-en` |
| **Storytelling** | `aura-2-athena-en`, `aura-2-cora-en`, `aura-2-cordelia-en`, `aura-2-minerva-en`, `aura-2-janus-en` |
| **Advertising** | `aura-2-asteria-en`, `aura-2-atlas-en`, `aura-2-odysseus-en` |
| **Interview** | `aura-2-aurora-en`, `aura-2-delia-en`, `aura-2-juno-en`, `aura-2-ophelia-en`, `aura-2-vesta-en` |
| **Informative** | `aura-2-hera-en`, `aura-2-hermes-en`, `aura-2-orion-en`, `aura-2-selene-en`, `aura-2-theia-en` |

### Featured Voices by Language

| Language | Featured Voices | Total Available |
|----------|----------------|-----------------|
| **English** | thalia, andromeda, helena, apollo, arcas, aries | 40+ |
| **Spanish** | celeste (Colombian), estrella (Mexican), nestor (Peninsular) | 10+ (codeswitching available) |
| **Dutch** | rhea, sander, beatrix | 9 |
| **German** | julius, viktoria | 7 |
| **French** | agathe, hector | 2 |
| **Italian** | livia, dionisio | 10 |
| **Japanese** | fujin, izanami | 5 |

### Language Support

| Language | Accents |
|----------|---------|
| English | American, British, Australian, Irish, Filipino |
| Spanish | Mexican, Peninsular, Colombian, Latin American, Argentine |
| German | German |
| French | French |
| Dutch | Dutch |
| Italian | Italian |
| Japanese | Japanese |

### Spanish Codeswitching Voices

The following Spanish voices can seamlessly switch between English and Spanish: Aquila, Carina, Diana, Javier, and Selena.

---

## 16. Capability Summary & Cross-Reference

### Complete API Endpoint Map

| Capability | Endpoint | Protocol | Method |
|-----------|----------|----------|--------|
| **STT Pre-Recorded** | `/v1/listen` | REST | POST |
| **STT Live Streaming** | `wss://api.deepgram.com/v1/listen` | WebSocket | WSS |
| **STT Flux (Turn-Based)** | `wss://api.deepgram.com/v2/listen` | WebSocket | WSS |
| **TTS Single Request** | `/v1/speak` | REST | POST |
| **TTS Continuous Stream** | `wss://api.deepgram.com/v1/speak` | WebSocket | WSS |
| **Voice Agent** | `wss://agent.deepgram.com/v1/agent/converse` | WebSocket | WSS |
| **Text Intelligence** | `/v1/read` | REST | POST |
| **Temporary Token** | `/v1/auth/tokens/grant` | REST | POST |
| **Manage Projects** | `/v1/manage/projects` | REST | GET |
| **Manage Members** | `/v1/manage/projects/{id}/members` | REST | GET |
| **Manage Invitations** | `/v1/manage/projects/{id}/invites` | REST | GET |
| **Manage Scopes** | `/v1/manage/projects/{id}/members/{mid}/scopes` | REST | GET |
| **Billing Balances** | `/v1/manage/projects/{id}/balances` | REST | GET |
| **Usage Breakdown** | `/v1/manage/projects/{id}/usage` | REST | GET |
| **Project Requests** | `/v1/manage/projects/{id}/requests` | REST | GET |
| **Manage API Keys** | `/v1/manage/projects/{id}/keys` | REST | GET/POST/DELETE |
| **List Models** | `/v1/manage/models` | REST | GET |
| **Model Metadata** | `/v1/manage/models/{id}` | REST | GET |
| **Self-Hosted Credentials** | `/v1/self-hosted/distribution-credentials` | REST | GET/POST/DELETE |

### Capability Decision Matrix

| If you need... | Use... | Key Parameters |
|----------------|--------|---------------|
| Transcribe a recorded audio file | STT Pre-Recorded (`POST /v1/listen`) | `model`, `smart_format`, `diarize_model`, `utterances` |
| Transcribe audio from a URL | STT Pre-Recorded with URL body | `{ "url": "..." }`, `model=nova-3` |
| Transcribe a local file | STT Pre-Recorded with binary body | `Content-Type: audio/wav`, `--data-binary @file` |
| Transcribe audio in real-time | STT Streaming (`WSS /v1/listen`) | `model`, `encoding`, `sample_rate`, `interim_results`, `endpointing` |
| Detect when user finishes speaking (voice agent) | STT Flux (`WSS /v2/listen`) | `model=flux-general-en`, `eot_threshold`, `eager_eot_threshold` |
| Multilingual conversational STT | STT Flux Multi | `model=flux-general-multi`, `language_hint` |
| Generate speech from text | TTS REST (`POST /v1/speak`) | `model`, `encoding`, `container`, `sample_rate`, `speed` |
| Stream text to speech continuously | TTS WebSocket (`WSS /v1/speak`) | `model`, `encoding`, `sample_rate` |
| Build a full voice agent (STT+LLM+TTS) | Voice Agent (`WSS /v1/agent/converse`) | `listen.provider`, `think.provider`, `speak.provider`, `prompt` |
| Agent calls external APIs | Voice Agent Function Calling | `think.functions[]` with `endpoint` (server-side) or client-side |
| Summarize audio content | STT + `summarize=true` | `summarize`, `summarize=v2` |
| Analyze sentiment | STT + `sentiment=true` or Read API | `sentiment=true` |
| Detect topics | STT + `topics=true` or Read API | `topics=true`, `custom_topic` |
| Recognize intent | STT + `intents=true` or Read API | `intents=true`, `custom_intent` |
| Extract entities from audio | STT + `detect_entities=true` | `detect_entities=true` |
| Redact sensitive information | STT + `redact` | `redact=pii`, `redact=pci`, `redact=numbers`, `redact=ssn` |
| Identify speakers | STT + diarization | `diarize_model=latest`, `utterances=true` |
| Improve recognition of specific terms | STT + keyterm/keywords | `keyterm=term` (Nova-3), `keywords=term:boost` (legacy) |
| Search for terms in audio | STT + search | `search=term` |
| Replace terms in transcript | STT + replace | `replace=old:new` |
| Format transcripts (currency, phones) | STT + smart_format | `smart_format=true` |
| Split transcript into paragraphs | STT + paragraphs | `paragraphs=true`, `punctuate=true` |
| Include filler words | STT + filler_words | `filler_words=true` |
| Async processing with callback | STT/TTS + callback | `callback=https://...`, `callback_method=POST` |
| Label requests for reporting | STT/TTS + tag | `tag=label` |
| Connect agent to phone | Voice Agent Telephony | Inbound/outbound call integration |
| Update agent settings mid-conversation | Voice Agent Update messages | `UpdateListen`, `UpdateThink`, `UpdateSpeak`, `UpdatePrompt` |
| Inject messages into conversation | Voice Agent Inject | `InjectUserMessage`, `InjectAgentMessage` |
| Track agent latency | Voice Agent `AgentStartedSpeaking` | `total_latency`, `tts_latency`, `ttt_latency` |
| Create client-safe tokens | Auth API | `POST /v1/auth/tokens/grant` |
| Manage API keys | Management API | `GET/POST/DELETE /v1/manage/projects/{id}/keys` |
| Self-hosted deployment | Self-Hosted | Distribution credentials + on-premise models |

### SDK Method Reference

| Capability | Python | JavaScript | Go | C# | Java |
|-----------|--------|------------|----|----|------|
| Pre-recorded URL | `deepgram.listen.v1.media.transcribe_url(url=..., model=...)` | `deepgram.listen.v1.media.transcribeUrl({ url, model })` | `dg.FromURL(ctx, url, options)` | `deepgramClient.TranscribeUrl(UrlSource, PreRecordedSchema)` | `client.listen().v1().media().transcribeUrl(req)` |
| Pre-recorded file | `deepgram.listen.v1.media.transcribe_file(request=bytes, model=...)` | `deepgram.listen.v1.media.transcribeFile(stream, options)` | `dg.FromFile(ctx, path, &options)` | `deepgramClient.TranscribeFile(bytes, schema)` | `client.listen().v1().media().transcribeFile(bytes)` |
| Live streaming | `deepgram.listen.v1.connect(model=...)` | `deepgram.listen.v1.connect({ model, language })` | `client.NewForDemo(ctx, &options)` | `ListenWebSocketClient.Connect(LiveSchema)` | `wsClient.connect(V1ConnectOptions)` |
| TTS generate | `deepgram.speak.v1.audio.generate(text=..., model=...)` | `deepgram.speak.v1.audio.generate({ text, model })` | `speak.ToSave(ctx, path, text, options)` | `SpeakRESTClient.ToFile(TextSource, path, SpeakSchema)` | `client.speak().v1().audio().generate(req)` |