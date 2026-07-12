# Gradium API Analysis — Text-to-Speech, Speech-to-Text & Speech-to-Speech

> **Base URL:** `https://api.gradium.ai/api` (REST) | **WebSocket:** `wss://api.gradium.ai/api` | **Servers:** Europe, US
> **Docs:** `https://docs.gradium.ai` | **Product:** `https://gradium.ai` | **Auth:** API key (`x-api-key` header)
> **SDK:** `gradium` (Python) | **OpenAPI:** `https://docs.gradium.ai/api-reference/openapi.json`
> **Description:** Gradium is a low-latency, high-quality voice AI platform providing APIs and a Python SDK for text-to-speech (TTS), speech-to-text (STT), and speech-to-speech (S2S) live translation. It supports five languages (English, French, German, Spanish, Portuguese), offers streaming over WebSocket and one-shot REST, semantic VAD for turn-taking, voice cloning from a 10-second sample, pronunciation dictionaries, and multiplexing over a single WebSocket connection. Expected time-to-first-token is below 300ms when streaming.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Authentication & Access Control](#2-authentication--access-control)
3. [WebSocket Lifecycle & Connection Management](#3-websocket-lifecycle--connection-management)
4. [Text-to-Speech (TTS)](#4-text-to-speech-tts)
5. [Speech-to-Text (STT)](#5-speech-to-text-stt)
6. [Speech-to-Speech (S2S) — Live Translation](#6-speech-to-speech-s2s--live-translation)
7. [Voices — Library, Cloning & Management](#7-voices--library-cloning--management)
8. [Pronunciation Dictionaries](#8-pronunciation-dictionaries)
9. [Voice Settings (TTS)](#9-voice-settings-tts)
10. [Transcription Settings (STT)](#10-transcription-settings-stt)
11. [Speech-to-Speech Settings](#11-speech-to-speech-settings)
12. [Semantic VAD & Turn-Taking](#12-semantic-vad--turn-taking)
13. [Multiplexing](#13-multiplexing)
14. [Browser WebSockets](#14-browser-websockets)
15. [In-Text Control Tags](#15-in-text-control-tags)
16. [Credits & Metering](#16-credits--metering)
17. [Audio Formats Reference](#17-audio-formats-reference)
18. [Error Handling](#18-error-handling)
19. [SDK API Shapes](#19-sdk-api-shapes)
20. [Integrations & Migration](#20-integrations--migration)
21. [Capability Summary & Cross-Reference](#21-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Gradium's platform is organized around these core abstractions:

- **Model** — The underlying AI model, identified by a `model_name` alias. TTS uses `"default"`; STT uses `"default"` (standard transcription) or `"stt-translate"` (translating transcription); S2S uses `"s2s-translate"` (live translation). Models are selected in the `setup` message of each request.
- **Voice** — A distinct speech identity identified by a `voice_id`. Voices come from the flagship Voice Library (60+ curated voices across 5 languages) or custom voice clones created from a short audio sample (≥10 seconds). Each voice has metadata: language, country, perceived age, perceived gender, and description.
- **Stream** — A real-time duplex session over WebSocket. The client sends `setup`, then input messages (`text` for TTS, `audio` for STT/S2S), then `end_of_stream`. The server responds with `ready`, output messages, and a terminal `end_of_stream` or `error`.
- **Setup Message** — The first logical message in every request, configuring model, voice, audio format, and advanced options via `json_config`.
- **json_config** — A nested object (dict in Python SDK, JSON object or string on the wire) carrying advanced per-request settings: temperature, speed (`padding_bonus`), voice similarity (`cfg_coef`), rewrite rules, language, target language, delay control.
- **Pronunciation Dictionary** — A named set of language-specific rewrite rules that override how words or phrases are pronounced in TTS output. Identified by `pronunciation_id`, attachable per request.
- **client_req_id** — A correlation identifier for multiplexing multiple independent requests over a single WebSocket. Every message in a multiplexed request carries the same `client_req_id`.
- **Credit** — The platform billing unit. TTS consumes 1 credit per character synthesized; STT consumes 3 credits per second of audio transcribed.
- **Semantic VAD** — Voice Activity Detection delivered as `step` messages every 80ms during STT WebSocket sessions, containing inactivity probability predictions across future horizons for turn-taking decisions.
- **Flush** — A mechanism to force processing of buffered input: in TTS via the `<flush>` in-text tag; in STT via a `flush` message that triggers a `flushed` response. Used at turn boundaries to emit pending output without waiting for natural silence.
- **delay_in_frames** — Adaptive delay control for STT (each frame = 80ms of context before text is emitted). Higher values improve quality at the cost of latency; lower values are more reactive. Range 0–80.

### Platform Architecture

```
Text Input ──▶ TTS Model ──▶ Audio Output (streamed or buffered)
                  │
    Voice Selection ──▶ Voice Library / Custom Clone
    Voice Settings ──▶ temp, cfg_coef, padding_bonus, rewrite_rules
    Pronunciation Dict ──▶ Override pronunciation
    In-text tags ──▶ <flush>, <break time="..." />
    Timestamps ──▶ Word-level text segments with start_s / stop_s

Audio Input ──▶ STT Model ──▶ Transcript (text segments with timestamps)
                  │
    Language ──▶ en, fr, de, es, pt (or translate via stt-translate)
    Semantic VAD ──▶ step messages every 80ms with inactivity_prob
    Adaptive Delay ──▶ delay_in_frames (latency/quality tradeoff)
    Flush ──▶ Force processing of buffered audio

Audio Input ──▶ S2S Model ──▶ Translated Audio + Transcript (duplex WebSocket)
                  │
    STT inner model ──▶ stt-translate (transcribe + translate)
    TTS inner model ──▶ default (synthesize translated text)
    target_language ──▶ Output language for translation
    voice_id ──▶ Output voice (must match target_language)
```

### End-to-End Flows

**TTS WebSocket pipeline:**
```
Connect (wss + x-api-key) ──▶ setup {voice_id, model_name, output_format, json_config?}
                                  │
                    Server sends ready {request_id, sample_rate, frame_size}
                                  │
                    Client sends text messages ──▶ Server sends audio chunks (base64) + text (timestamps)
                                  │
                    Client sends end_of_stream ──▶ Server sends end_of_stream (or closes socket)
```

**TTS REST pipeline (one-shot):**
```
POST /post/speech/tts?voice_id=...&output_format=...&json_config=...
  body: text block
  ──▶ Response: streamed audio bytes
```

**STT WebSocket pipeline (real-time, with VAD):**
```
Connect (wss + x-api-key) ──▶ setup {model_name, input_format, json_config{language, delay_in_frames}}
                                  │
                    Server sends ready {request_id, sample_rate, frame_size, delay_in_frames}
                                  │
                    Client sends audio (base64) ──▶ Server sends text + end_text + step (VAD) messages
                                  │
                    Optional: client sends flush ──▶ Server sends flushed
                                  │
                    Client sends end_of_stream ──▶ Server sends end_of_stream
```

**S2S WebSocket pipeline (live translation):**
```
Connect (wss + x-api-key) ──▶ setup {model_name: "s2s-translate", stt_model_name: "stt-translate",
                                     tts_model_name: "default", input_format, output_format,
                                     voice_id, json_config{target_language}}
                                  │
                    Server sends ready {request_id, sample_rate, frame_size}
                                  │
                    Client sends audio (base64) ──▶ Server sends text (translated transcript) + audio (synthesized output)
                                  │
                    Client sends end_of_stream ──▶ Server sends end_of_stream
```

---

## 2. Authentication & Access Control

### API Keys

All API endpoints are authenticated using an API key passed via the `x-api-key` HTTP header:

```
x-api-key: your_api_key
```

For WebSocket connections, the header is set when establishing the connection:

```bash
wscat -c "wss://api.gradium.ai/api/speech/tts" \
  -H "x-api-key: YOUR_API_KEY"
```

### SDK Authentication

The Python SDK (`gradium`) accepts the API key in three ways:

```python
import gradium

# 1. Explicit parameter
client = gradium.client.GradiumClient(api_key="gd_your_api_key_here")

# 2. Environment variable (GRADIUM_API_KEY)
#    export GRADIUM_API_KEY=gd_your_api_key_here
client = gradium.client.GradiumClient()

# 3. Both — explicit takes precedence
```

### Browser Tokens

Browser and mobile clients should not expose API keys. Gradium supports short-lived, single-use tokens for WebSocket connections via a `?token=...` query parameter. Tokens are generated server-side and allow browser clients to connect without the API key. See [Browser WebSockets](#14-browser-websockets).

### Server URLs

| Transport | Base URL |
|-----------|----------|
| REST      | `https://api.gradium.ai/api` |
| WebSocket | `wss://api.gradium.ai/api` |

---

## 3. WebSocket Lifecycle & Connection Management

All real-time APIs (TTS, STT, S2S) share the same WebSocket lifecycle:

1. **Connect** with authentication (`x-api-key` header or `?token=...`).
2. **Send `setup`** message with model, voice/format, and configuration.
3. **Receive `ready`** from server (with `request_id`, `sample_rate`, `frame_size`).
4. **Send input** messages (`text` for TTS, `audio` for STT/S2S).
5. **Optionally flush** buffered input (`<flush>` tag for TTS, `flush` message for STT).
6. **Send `end_of_stream`** when input is complete.
7. **Read output** until the server sends `end_of_stream` or `error`.

### WebSocket Endpoints

| Product | Endpoint | Client Input | Server Output |
|---------|----------|-------------|---------------|
| TTS | `wss://api.gradium.ai/api/speech/tts` | `text`, `end_of_stream` | `ready`, `audio`, `text`, `end_of_stream`, `error` |
| STT | `wss://api.gradium.ai/api/speech/asr` | `audio`, `flush`, `end_of_stream` | `ready`, `text`, `end_text`, `step`, `flushed`, `end_of_stream`, `error` |
| S2S | `wss://api.gradium.ai/api/speech/s2s` | `audio`, `end_of_stream` | `ready`, `text`, `audio`, `end_of_stream`, `error` |

### Shared Setup Fields

| Field | Applies to | Required | Description |
|-------|-----------|----------|-------------|
| `type` | All | Yes | Always `"setup"`. |
| `model_name` | TTS, STT, S2S | S2S: Yes | Model alias. TTS: `"default"`. STT: `"default"` or `"stt-translate"`. S2S: `"s2s-translate"`. |
| `json_config` | TTS, STT, S2S | No | Advanced model settings (dict or JSON string). |
| `client_req_id` | All | No | Correlates multiplexed requests. |
| `close_ws_on_eos` | All | No | Defaults to `true`. Set `false` to keep socket open for multiplexing. |
| `retry_for_s` | All | No | Optional setup retry window in seconds. |

### Ready Message

After `setup`, the server sends a `ready` message:

**TTS `ready`:**
```json
{
  "type": "ready",
  "request_id": "req_...",
  "model_name": "default",
  "model_ext": "resolved-model",
  "sample_rate": 48000,
  "frame_size": 3840,
  "audio_stream_names": [],
  "text_stream_names": []
}
```

**STT `ready`:**
```json
{
  "type": "ready",
  "request_id": "req_...",
  "model_name": "default",
  "sample_rate": 24000,
  "frame_size": 1920,
  "delay_in_frames": 16,
  "text_stream_names": []
}
```

**S2S `ready`:**
```json
{
  "type": "ready",
  "request_id": "req_...",
  "model_name": "s2s-translate",
  "sample_rate": 48000,
  "frame_size": 3840
}
```

### Input Chunking Guidelines

For STT raw PCM input, use 80ms chunks when possible:

| Format | Sample rate | Samples per 80ms | Bytes per chunk |
|--------|------------|-------------------|-----------------|
| `pcm` | 24 kHz | 1920 | 3840 |
| `pcm_8000` | 8 kHz | 640 | 1280 |
| `pcm_16000` | 16 kHz | 1280 | 2560 |
| `pcm_48000` | 48 kHz | 3840 | 7680 |

For TTS text input, split on whitespace or sentence boundaries. **Never split inside a word or separate punctuation into a standalone message** — the server inserts a single whitespace between consecutive text messages. Sending `"foo"` followed by `"bar"` produces `"foo bar"`, not `"foobar"`.

---

## 4. Text-to-Speech (TTS)

### Main Concepts

- **Streaming synthesis** — Text is sent incrementally; audio chunks are returned as soon as the model produces them (time-to-first-token < 300ms).
- **Buffered synthesis** — The full text is sent at once; the SDK buffers all audio chunks and returns a single result.
- **Word-level timestamps** — The model returns `text` messages with `start_s` and `stop_s` for each emitted text segment (typically aligned to word boundaries).
- **In-text control** — `<flush>` forces audio emission for all buffered text; `<break time="Ns" />` inserts a pause (0.1–2.0s).
- **Voice selection** — Use `voice_id` (preferred) or `voice` (name fallback). Custom cloned voices use the same `voice_id` mechanism.

### TTS WebSocket Stream (`wss://api.gradium.ai/api/speech/tts`)

**Lifecycle:**
```json
{"type":"setup","voice_id":"YTpq7expH9539ERJ","model_name":"default","output_format":"pcm"}
{"type":"text","text":"Hello, world."}
{"type":"end_of_stream"}
```

#### Client Messages

**`setup`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Always `"setup"`. |
| `model_name` | string | No | Model alias, defaults to `"default"`. |
| `voice_id` | string | Recommended | Voice library or custom voice ID. |
| `voice` | string | No | Voice name fallback when `voice_id` is not provided. |
| `output_format` | string | No | `wav`, `pcm`, `opus`, `ulaw_8000`, `alaw_8000`, `pcm_16000`, etc. Defaults to `wav`. |
| `json_config` | object/string | No | Advanced TTS settings. See [Voice Settings](#9-voice-settings-tts). |
| `pronunciation_id` | string | No | Pronunciation dictionary ID. |
| `client_req_id` | string | No | Correlates multiplexed requests. |
| `close_ws_on_eos` | boolean | No | Defaults to `true`; set `false` to keep socket open. |
| `retry_for_s` | number | No | Optional setup retry window in seconds. |

**`text`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Always `"text"`. |
| `text` | string | Yes | Text chunk to synthesize. |
| `client_req_id` | string | No | Required when routing a multiplexed request. |

**`end_of_stream`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Always `"end_of_stream"`. |
| `client_req_id` | string | No | End the matching multiplexed request. |

#### Server Messages

**`ready`**

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"ready"`. |
| `request_id` | string | Gradium request ID for logging and support. |
| `model_name` | string | Requested model alias. |
| `model_ext` | string | Resolved model identifier, when present. |
| `sample_rate` | integer | Output sample rate. |
| `frame_size` | integer | Output frame size in samples. |
| `audio_stream_names` | string[] | Named audio streams, when present. |
| `text_stream_names` | string[] | Named text streams, when present. |
| `client_req_id` | string | Present for multiplexed requests. |

**`audio`**

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"audio"`. |
| `audio` | string | Base64-encoded audio chunk. |
| `start_s` | number | Chunk start time in seconds. |
| `stop_s` | number | Chunk stop time in seconds. |
| `stream_id` | integer | Stream identifier, when present. |
| `client_req_id` | string | Present for multiplexed requests. |

**`text`** (timestamps)

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"text"`. |
| `text` | string | Text segment associated with generated audio. |
| `start_s` | number | Segment start time in seconds. |
| `stop_s` | number | Segment stop time in seconds. |
| `stream_id` | integer | Stream identifier, when present. |
| `client_req_id` | string | Present for multiplexed requests. |

**Terminal messages:**

| Type | Description |
|------|-------------|
| `end_of_stream` | The request is complete. |
| `flushed` | Reserved for flush acknowledgements. |
| `error` | Terminal error; socket closes after. |

### TTS REST Endpoint (`POST /post/speech/tts`)

One-shot synthesis of a complete text block. The full text is sent in the request body; the server streams audio bytes back as the response body.

Query parameters mirror the WebSocket `setup` fields: `voice_id`, `model_name`, `output_format`, `json_config` (URL-encoded JSON string), `pronunciation_id`.

### SDK TTS API Shapes

| Use case | Method | Description |
|----------|--------|-------------|
| Concurrent send/receive (voice agents, LLM token streaming, browser/telephony bridges) | `client.tts_realtime(...)` | Opens a real-time WebSocket stream with `send_text()`, `send_eos()`, and async iteration over messages. |
| Iterable input (string, list, async generator) → streamed audio chunks | `client.tts_stream(...)` | Pull-based: returns an async iterator of audio bytes via `iter_bytes()`. |
| Complete audio bytes after generation finishes | `client.tts(...)` | Buffered: returns a `TTSResult` with `raw_data`, `sample_rate`, `request_id`, `text_with_timestamps`, `pcm()`, `pcm16()`. |

**`tts_realtime` example:**
```python
async with client.tts_realtime(
    voice_id="YTpq7expH9539ERJ",
    output_format="pcm",
) as tts:
    async def sender():
        await tts.send_text("Hello, world.")
        await tts.send_eos()

    async def receiver():
        async for msg in tts:
            if msg["type"] == "audio":
                audio_chunks.append(msg["audio"])
            elif msg["type"] == "end_of_stream":
                return

    await asyncio.gather(sender(), receiver())
```

**`tts` (buffered) example:**
```python
result = await client.tts(
    setup={
        "model_name": "default",
        "voice_id": "YTpq7expH9539ERJ",
        "output_format": "wav"
    },
    text="Hello, world!"
)

with open("output.wav", "wb") as f:
    f.write(result.raw_data)

# Word-level timestamps
for item in result.text_with_timestamps:
    print(f"{item.text}: {item.start_s:.2f}s - {item.stop_s:.2f}s")
```

**`tts_stream` (async generator input) example:**
```python
async def text_generator():
    yield "Hello, "
    yield "this is "
    yield "a streaming "
    yield "example."

stream = await client.tts_stream(
    setup={"voice_id": "YTpq7expH9539ERJ", "output_format": "pcm"},
    text=text_generator()
)

async for audio_chunk in stream.iter_bytes():
    print(f"Received {len(audio_chunk)} bytes")
```

### Direct WebSocket (without SDK)

For non-Python runtimes or full wire control:

```python
import asyncio, base64, json, websockets

async def synthesise(api_key, voice_id, text):
    setup = {
        "type": "setup",
        "voice_id": voice_id,
        "model_name": "default",
        "output_format": "wav",
    }
    audio_chunks = []

    async with websockets.connect(
        "wss://api.gradium.ai/api/speech/tts",
        additional_headers={"x-api-key": api_key},
    ) as ws:
        await ws.send(json.dumps(setup))
        ready = json.loads(await ws.recv())
        assert ready["type"] == "ready"

        await ws.send(json.dumps({"type": "text", "text": text}))
        await ws.send(json.dumps({"type": "end_of_stream"}))

        while True:
            msg = json.loads(await ws.recv())
            if msg["type"] == "audio":
                audio_chunks.append(base64.b64decode(msg["audio"]))
            elif msg["type"] == "end_of_stream":
                break
            elif msg["type"] == "error":
                raise RuntimeError(msg["message"])

    return b"".join(audio_chunks)
```

---

## 5. Speech-to-Text (STT)

### Main Concepts

- **Two model types** — `"default"` (standard transcription in the audio's language) and `"stt-translate"` (translating transcription — output in a different language chosen via `language`/`target_language`).
- **Two transports** — WebSocket (live audio, lowest latency, VAD signals, flush control) and REST POST (complete audio file, NDJSON streaming response).
- **Semantic VAD** — `step` messages every 80ms with inactivity probability predictions across future horizons, enabling real-time turn detection.
- **Adaptive delay** — `delay_in_frames` (each frame = 80ms) controls the latency/quality tradeoff: lower values are more reactive, higher values improve transcription quality.
- **Flush** — A `flush` message forces the server to process all buffered audio and emit pending transcript, useful at detected turn boundaries.

### Choosing Transport

| Scenario | Use | Why |
|----------|-----|-----|
| Live audio (mic, telephony, agent loop) with sub-second turn-taking | WebSocket | Lowest latency; push audio as captured; get text + VAD in real time. |
| Complete file on disk or in memory | REST POST | One HTTP POST; server streams NDJSON results back. No connection management. |
| In-hand audio but want VAD signals or flush | WebSocket (`stt_stream`) | Pull-based over WebSocket; same low latency, no manual connection management. |
| Browser/mobile microphone | Browser WebSockets | Short-lived tokens instead of exposing API key. |
| Telephony media streams | Telephony audio recipes | Use `ulaw_8000`, `alaw_8000`, or low-sample-rate PCM directly. |

### STT WebSocket Stream (`wss://api.gradium.ai/api/speech/asr`)

**Lifecycle:**
```json
{"type":"setup","model_name":"default","input_format":"pcm","json_config":{"language":"en","delay_in_frames":16}}
{"type":"audio","audio":"base64_encoded_audio"}
{"type":"flush","flush_id":1}
{"type":"end_of_stream"}
```

#### Client Messages

**`setup`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Always `"setup"`. |
| `model_name` | string | No | `"default"` or `"stt-translate"`. Defaults to `"default"`. |
| `input_format` | string | No | `pcm`, `wav`, `opus`, `ulaw_8000`, `alaw_8000`, or explicit PCM rates. Defaults to `wav`. |
| `json_config` | object/string | No | Advanced STT settings. See [Transcription Settings](#10-transcription-settings-stt). |
| `client_req_id` | string | No | Correlates multiplexed requests. |
| `close_ws_on_eos` | boolean | No | Defaults to `true`; set `false` to keep socket open. |
| `retry_for_s` | number | No | Optional setup retry window in seconds. |

**`audio`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Always `"audio"`. |
| `audio` | string | Yes | Base64-encoded audio chunk. |
| `client_req_id` | string | No | Required when routing a multiplexed request. |

**`flush`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Always `"flush"`. |
| `flush_id` | integer | Yes | Echoed in the matching `flushed` response. |
| `client_req_id` | string | No | Required when routing a multiplexed request. |

**`end_of_stream`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Always `"end_of_stream"`. |
| `client_req_id` | string | No | End the matching multiplexed request. |

#### Server Messages

**`ready`**

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"ready"`. |
| `request_id` | string | Gradium request ID. |
| `model_name` | string | Requested model alias. |
| `sample_rate` | integer | Input sample rate after setup. |
| `frame_size` | integer | Frame size in samples. |
| `delay_in_frames` | integer | Model delay, in 80ms frames. |
| `text_stream_names` | string[] | Named text streams, when present. |
| `client_req_id` | string | Present for multiplexed requests. |

**`text`** (transcribed segment)

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"text"`. |
| `text` | string | Transcribed text segment. |
| `start_s` | number | Segment start time in seconds. |
| `stream_id` | integer | Stream identifier, when present. |
| `client_req_id` | string | Present for multiplexed requests. |

**`end_text`** (segment boundary)

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"end_text"`. |
| `stop_s` | number | Stop time for the previous text segment. |
| `stream_id` | integer | Stream identifier, when present. |
| `client_req_id` | string | Present for multiplexed requests. |

**`step`** (semantic VAD)

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `"step"` or legacy `"vad"`. |
| `vad` | object[] | Horizon predictions with `horizon_s` and `inactivity_prob`. |
| `step_idx` | integer | Step index. |
| `step_duration_s` | number | Step duration in seconds, usually `0.08`. |
| `total_duration_s` | number | Audio duration processed so far. |
| `client_req_id` | string | Present for multiplexed requests. |

**`flushed`**

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"flushed"`. |
| `flush_id` | integer | The `flush_id` from the matching request. |
| `client_req_id` | string | Present for multiplexed requests. |

**Terminal messages:**

| Type | Description |
|------|-------------|
| `end_of_stream` | The request is complete. |
| `error` | Terminal error; socket closes after. |

### STT REST Endpoint (`POST /post/speech/asr`)

Transcribe a complete audio file with HTTP POST. The audio is sent as the request body; the server streams NDJSON results back as the response body. Pass `json_config` as a URL-encoded JSON string in the query parameters.

### SDK STT API Shapes

| Use case | Method |
|----------|--------|
| Concurrent send/receive (live mic, agent loop) | `client.stt_realtime(...)` |
| Iterable/async generator audio input → pull-based transcript | `client.stt_stream(...)` |
| Complete audio file → buffered result | `client.stt(...)` |

**`stt_realtime` example:**
```python
config = {"language": "en", "temp": 0.3, "delay_in_frames": 16}

async with client.stt_realtime(
    model_name="default",
    input_format="pcm",
    json_config=config,
) as stt:
    async for msg in stt:
        if msg["type"] == "text":
            print(msg["text"])
        elif msg["type"] == "step":
            turn_done = msg["vad"][-1]["inactivity_prob"] > 0.5
            if turn_done:
                await stt.send_flush(flush_id=1)
```

**`stt_stream` (pull-based) example:**
```python
config = {"language": "en", "temp": 0.3, "delay_in_frames": 16}

stream = await client.stt_stream(
    {"model_name": "default", "input_format": "pcm", "json_config": config},
    audio_generator(audio_data),
)
```

---

## 6. Speech-to-Speech (S2S) — Live Translation

### Main Concepts

Gradium Speech-to-Speech (S2S) translates spoken audio from one language into spoken audio in another, over a single duplex WebSocket. The input audio is transcribed (via the STT inner model), translated into the target language, and re-synthesized as output audio (via the TTS inner model). You stream audio in and receive both the synthesized output audio and the translated transcript back as they're produced.

- **Pipeline**: Input audio → STT (`stt-translate`) → translated text → TTS (`default`) → output audio.
- **Single model**: The only currently supported S2S model is `"s2s-translate"` (live translation).
- **Target language**: Set via `target_language` in `json_config`. If omitted, the speech stays in the original language (transcription + re-synthesis without translation).
- **Output voice**: Must be a voice in the same language as `target_language`.

### S2S WebSocket Stream (`wss://api.gradium.ai/api/speech/s2s`)

**Lifecycle:**
```json
{"type":"setup","model_name":"s2s-translate","stt_model_name":"stt-translate","tts_model_name":"default","input_format":"pcm","output_format":"pcm","voice_id":"YTpq7expH9539ERJ","json_config":{"target_language":"en"}}
{"type":"audio","audio":"base64_encoded_audio"}
{"type":"end_of_stream"}
```

#### Client Messages

**`setup`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Always `"setup"`. |
| `model_name` | string | Yes | Use `"s2s-translate"` (only supported model). |
| `stt_model_name` | string | No | STT model for input transcription. Set to `"stt-translate"` for live translation. |
| `tts_model_name` | string | No | TTS model for output synthesis. Set to `"default"`. |
| `input_format` | string | Yes | `pcm`, `wav`, `opus`, `ulaw_8000`, `alaw_8000`, or explicit PCM rates. For `pcm`, input is 24 kHz, 16-bit signed mono. |
| `output_format` | string | Yes | Same format set as `input_format`. For `pcm`, output is 48 kHz, 16-bit signed mono. |
| `voice_id` | string | No | Voice for synthesized output. Must match `target_language`. |
| `json_config` | object/string | No | Advanced settings. Set `target_language` to translate; omit to keep original language. |
| `client_req_id` | string | No | Correlates multiplexed requests. |
| `close_ws_on_eos` | boolean | No | Defaults to `true`; set `false` to keep socket open. |

**`audio`** — Base64-encoded input audio chunk (same schema as STT).

**`end_of_stream`** — Same schema as STT/TTS.

#### Server Messages

**`ready`**

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"ready"`. |
| `request_id` | string | Gradium request ID. |
| `model_name` | string | Requested model alias. |
| `sample_rate` | integer | Output sample rate in Hz. |
| `frame_size` | integer | Output frame size in samples. |
| `client_req_id` | string | Present for multiplexed requests. |

**`text`** (translated transcript)

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"text"`. |
| `text` | string | Transcribed (and translated, if `target_language` is set) text segment. |
| `start_s` | number | Segment start time in seconds. |
| `stop_s` | number | Segment stop time in seconds. |
| `stream_id` | integer | Stream identifier, when present. |
| `client_req_id` | string | Present for multiplexed requests. |

**`audio`** (synthesized output)

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Always `"audio"`. |
| `audio` | string | Base64-encoded output audio chunk. |
| `start_s` | number | Chunk start time in seconds. |
| `stop_s` | number | Chunk stop time in seconds. |
| `stream_id` | integer | Stream identifier, when present. |
| `client_req_id` | string | Present for multiplexed requests. |

**Terminal messages:** `end_of_stream`, `error` (same as TTS/STT).

### SDK S2S API Shapes

| Use case | Method |
|----------|--------|
| Concurrent send/receive (real-time translation) | `client.s2s_realtime(...)` |
| Iterable/async generator audio input → pull-based | `client.s2s_stream(...)` |
| Complete audio → buffered result | `client.s2s(...)` |

**`s2s_realtime` example:**
```python
config = {"target_language": "en"}

async with client.s2s_realtime(
    model_name="s2s-translate",
    stt_model_name="stt-translate",
    tts_model_name="default",
    input_format="pcm",
    output_format="pcm",
    voice_id="YTpq7expH9539ERJ",
    json_config=config,
) as s2s:
    ...
```

---

## 7. Voices — Library, Cloning & Management

### Main Concepts

- **Flagship Voices** — 60+ curated voices across five languages (en, fr, de, es, pt), hand-picked for quality and range. Each has metadata: name, voice_id, language, country, age group, gender, and description.
- **Custom Voices** — Voice clones created from a short audio sample (≥10 seconds recommended). Cloned voices are used identically to flagship voices via `voice_id`.
- **Voice Management** — REST endpoints to list, create, get, update, and delete voices.

### Flagship Voices (Selection)

| Name | Voice ID | Language | Country | Age Group | Gender |
|------|----------|----------|---------|-----------|--------|
| Emma | `YTpq7expH9539ERJ` | en | us 🇺🇸 | Adult | Feminine |
| Kent | `LFZvm12tW_z0xfGo` | en | us 🇺🇸 | Adult | Masculine |
| Sydney | `jtEKaLYNn6iif5PR` | en | us 🇺🇸 | Adult | Feminine |
| John | `KWJiFWu2O9nMPYcR` | en | us 🇺🇸 | Adult | Masculine |
| Eva | `ubuXFxVQwVYnZQhy` | en | gb 🇬🇧 | Adult | Feminine |
| Jack | `m86j6D7UZpGzHsNu` | en | gb 🇬🇧 | Adult | Masculine |
| Elise | `b35yykvVppLXyw_l` | fr | fr 🇫🇷 | Adult | Feminine |
| Leo | `axlOaUiFyOZhy4nv` | fr | fr 🇫🇷 | Adult | Masculine |
| Mia | `-uP9MuGtBqAvEyxI` | de | de 🇩🇪 | Adult | Feminine |
| Maximilian | `0y1VZjPabOBU3rWy` | de | de 🇩🇪 | Adult | Masculine |
| Valentina | `B36pbz5_UoWn4BDl` | es | mx 🇲🇽 | Adult | Feminine |
| Sergio | `xu7iJ_fn2ElcWp2s` | es | es 🇪🇸 | Adult | Masculine |
| Alice | `pYcGZz9VOo4n2ynh` | pt | br 🇧🇷 | Adult | Feminine |
| Davi | `M-FvVo9c-jGR4PgP` | pt | br 🇧🇷 | Adult | Masculine |

The full library includes 200+ voices spanning multiple countries (us, gb, ca, in, au, fr, ca, de, at, es, mx, ar, co, pt, br, pt) and age groups (Young Adult, Adult, Mature).

### Voice Management REST Endpoints

#### Create Voice — `POST /voices/`

Create a custom voice clone from an uploaded audio sample.

**SDK:**
```python
voice = await gradium.voices.create(
    client,
    audio_file="my_voice_sample.wav",
    name="My Custom Voice",
    description="A voice created from my recording",
    start_s=0.0,
)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `audio_file` | string | Yes | Path to the audio file (WAV, MP3, etc.). |
| `name` | string | Yes | A name for the voice. |
| `description` | string | No | A description of the voice. |
| `language` | string | No | Language hint (`"en"`, `"fr"`, `"de"`, `"es"`, `"pt"`). |
| `start_s` | float | No | Start time in seconds to begin sampling (default `0.0`). |
| `timeout_s` | float | No | Maximum seconds to wait for embedding extraction (default `10.0`). |
| `input_format` | string | No | Audio format. If omitted, inferred from file extension. |

#### Get Voices — `GET /voices/`

List Gradium voices available to the authenticated organization. Returns both flagship and custom voices.

#### Get Voice — `GET /voices/{voice_id}`

Retrieve metadata for a Gradium voice by voice UID.

#### Update Voice — `PATCH /voices/{voice_id}` (or equivalent)

Update metadata for an existing custom Gradium voice.

#### Delete Voice — `DELETE /voices/{voice_id}`

Delete a custom Gradium voice by voice UID.

### Using Custom Voices in TTS

Custom voices are used identically to flagship voices — pass the custom voice's `voice_id` in the setup:

```python
result = await client.tts(
    setup={
        "model_name": "default",
        "voice_id": "YTpq7expH9539ERJ",  # or your custom voice_id
        "output_format": "wav"
    },
    text="Hello with my custom voice!"
)
```

---

## 8. Pronunciation Dictionaries

### Main Concepts

Pronunciation dictionaries customize how specific words or phrases are pronounced in TTS output. They are particularly useful for:

- Brand names, technical terms, or proper nouns
- Acronyms that should be pronounced in a specific way
- Words with non-standard pronunciations

Dictionaries contain language-specific rewrite rules. They are identified by a `pronunciation_id` and attached to TTS requests via the `pronunciation_id` setup field.

### Pronunciation Management REST Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/pronunciations/` | POST | Create a pronunciation dictionary with language-specific rewrite rules. |
| `/pronunciations/` | GET | List pronunciation dictionaries for the authenticated organization. |
| `/pronunciations/{uid}` | GET | Retrieve a pronunciation dictionary and its rewrite rules by UID. |
| `/pronunciations/{uid}` | PATCH/PUT | Update a pronunciation dictionary and its rewrite rules. |
| `/pronunciations/{uid}` | DELETE | Delete a pronunciation dictionary by UID. |

### Using Pronunciation Dictionaries in TTS

Pass the `pronunciation_id` in the setup message:

```python
result = await client.tts(
    setup={
        "voice_id": "YTpq7expH9539ERJ",
        "output_format": "wav",
        "pronunciation_id": "bb1ckYhNHCcIJjdK",
    },
    text="The text you want to generate."
)
```

Dictionaries can also be managed through the Gradium Studio web interface.

---

## 9. Voice Settings (TTS)

TTS models accept advanced options via the `json_config` parameter. In the Python SDK, this is a dict; for REST, it's a URL-encoded JSON string in query parameters.

### Quick Reference

| Parameter | Range | Default | Effect |
|-----------|-------|---------|--------|
| `temp` | `0.0`–`1.4` | `0.7` | Sampling temperature. `0.0` is deterministic; higher values produce more diverse output. |
| `cfg_coef` | `1.0`–`4.0` | `2.0` | Voice similarity. Higher values stay closer to the target voice; very high values can introduce artifacts. |
| `padding_bonus` | `-4.0`–`4.0` | `0.0` | Speech speed. Negative values are faster; positive values are slower. |
| `rewrite_rules` | string | *none* | Text-rewriting rules applied before synthesis. Language codes (`"en"`, `"fr"`, `"de"`, `"es"`, `"pt"`) enable all rules for that language. |
| `pronunciation_id` | string | *none* | Pronunciation dictionary ID, applied per request. |

### Speed Control (`padding_bonus`)

- Default: `0.0`
- Negative values (`-4.0` to `-0.1`): speaker speaks faster
- Positive values (`0.1` to `4.0`): speaker speaks slower

```python
slower_audio = await client.tts(
    setup={'voice_id': 'YTpq7expH9539ERJ', 'output_format': 'wav',
           'json_config': {'padding_bonus': 2.0}},
    text=sample_text,
)

faster_audio = await client.tts(
    setup={'voice_id': 'YTpq7expH9539ERJ', 'output_format': 'wav',
           'json_config': {'padding_bonus': -2.0}},
    text=sample_text,
)
```

### Temperature Control (`temp`)

- Range: `0.0`–`1.4`
- `0.0`: deterministic generation
- Higher values: more diverse outputs
- Default: `0.7`

### Voice Similarity Control (`cfg_coef`)

- Range: `1.0`–`4.0`
- Default: `2.0`
- Higher values: more closely replicates the cloned/target voice, but very high values can lead to audio artifacts

### Rewrite Rules (`rewrite_rules`)

Text-rewriting rules applied before synthesis. Pass a language code string (`"en"`, `"fr"`, `"de"`, `"es"`, `"pt"`) to enable all built-in rewriting rules for that language. See [Text Rewriting Rules](https://docs.gradium.ai/guides/text-rewriting) for custom rule syntax.

### Passing `json_config`

```python
config = {"temp": 0.3, "cfg_coef": 2.5, "padding_bonus": -1.0}

# WebSocket realtime: setup is keyword args; json_config stays nested.
async with client.tts_realtime(
    voice_id="YTpq7expH9539ERJ",
    output_format="pcm",
    json_config=config,
) as stream:
    ...

# One-shot via SDK: setup is a dict containing json_config.
audio = await client.tts(
    setup={"voice_id": "YTpq7expH9539ERJ", "output_format": "wav", "json_config": config},
    text="Hello",
)
```

---

## 10. Transcription Settings (STT)

STT models accept configuration via the `json_config` parameter (dict in SDK, URL-encoded JSON string in REST query).

### Quick Reference

| Option | Type | Allowed values | Effect |
|--------|------|---------------|--------|
| `temp` | float | `0.0`–`1.5` | Sampling temperature for text generation. `0.0` is greedy; higher values produce more diverse output and can help when no text is being recognised. |
| `language` | string | `"en"`, `"fr"`, `"de"`, `"es"`, `"pt"` | Audio language for transcription models, or output language for translating models. |
| `target_language` | string | `"en"`, `"fr"`, `"de"`, `"es"`, `"pt"` | Output language for translating models; ignored otherwise. Interchangeable with `language` for translating models. |
| `padding_bonus` | float | `-4.0`–`4.0` | Biases the model toward emitting text sooner (negative) or later (positive). |
| `delay_in_frames` | int | `0`–`80` | Adaptive delay control. Each frame is 80ms of context before text is emitted. Higher values improve quality at the cost of latency; lower values are more reactive. Legacy alias: `delay_in_tokens`. |

### Language Configuration

- **Non-translating models** (`model_name: "default"`): `language` is the expected language of the audio, grounding the model for better transcription quality. For mixed-language audio, set the dominant language or leave unset. `target_language` has no effect.
- **Translating models** (`model_name: "stt-translate"`): `language` and `target_language` specify the language the transcription should be generated in (not the language of the audio). The two are interchangeable; set either one.

### Semantic VAD and Delay

`delay_in_frames` is most useful with the WebSocket STT stream. The server emits semantic VAD `step` messages every 80ms; each step contains future horizons with `inactivity_prob` values.

```python
if msg["type"] == "step":
    turn_done = msg["vad"][-1]["inactivity_prob"] > 0.5
    if turn_done:
        await stt.send_flush(flush_id=1)
```

- Lower `delay_in_frames` (e.g. `8`): fast back-and-forth assistants, responsiveness matters more than context.
- Higher `delay_in_frames` (e.g. `16+`): transcription quality when a little more latency is acceptable.
- Balanced starting point: `16`.

### Passing `json_config`

```python
config = {"language": "en", "temp": 0.3, "delay_in_frames": 16}

# Pull-based: setup is a dict containing json_config.
stream = await client.stt_stream(
    {"model_name": "default", "input_format": "pcm", "json_config": config},
    audio_generator(audio_data),
)

# Real-time: setup is keyword arguments; json_config stays nested.
async with client.stt_realtime(
    model_name="default",
    input_format="pcm",
    json_config=config,
) as stt:
    ...
```

---

## 11. Speech-to-Speech Settings

S2S is configured through the `setup` message. Translation target is passed through `json_config`.

### Setup Fields

| Field | Type | Required | Effect |
|-------|------|----------|--------|
| `model_name` | string | Yes | Use `"s2s-translate"` (only supported model). |
| `stt_model_name` | string | No | STT model for input transcription. Set to `"stt-translate"` for live translation. |
| `tts_model_name` | string | No | TTS model for output synthesis. Set to `"default"`. |
| `voice_id` | string | Yes | Voice UID for synthesized output. Must be in the same language as `target_language`. |
| `input_format` | string | Yes | `pcm`, `wav`, `opus`, `ulaw_8000`, `alaw_8000`, or explicit PCM rate. For `pcm`: 24 kHz, 16-bit signed mono. |
| `output_format` | string | Yes | Same format set as `input_format`. For `pcm`: 48 kHz, 16-bit signed mono. |
| `json_config` | object/string | No | Advanced pipeline settings. |
| `client_req_id` | string | No | Correlates multiplexed requests. |
| `close_ws_on_eos` | boolean | No | Defaults to `true`; set `false` to keep socket open. |

### `json_config` Options

| Option | Type | Allowed values | Effect |
|--------|------|---------------|--------|
| `target_language` | string | `"en"`, `"fr"`, `"de"`, `"es"`, `"pt"` | Language to translate the speech into before synthesis. Omit to keep original language. |

The transcribed text is translated into `target_language` before the output audio is generated, and the `text` messages received back carry the translated text. The `voice_id` must be a voice in this same language.

---

## 12. Semantic VAD & Turn-Taking

### Main Concepts

Gradium's Semantic VAD (Voice Activity Detection) is a WebSocket-only STT feature that provides real-time turn-taking signals for voice agents. Every 80ms, the server emits a `step` message containing inactivity probability predictions across multiple future horizons.

### `step` Message Structure

```json
{
  "type": "step",
  "vad": [
    {"horizon_s": 0.5, "inactivity_prob": 0.12},
    {"horizon_s": 1.0, "inactivity_prob": 0.34},
    {"horizon_s": 2.0, "inactivity_prob": 0.78}
  ],
  "step_idx": 42,
  "step_duration_s": 0.08,
  "total_duration_s": 3.36
}
```

- `vad`: Array of horizon predictions, each with `horizon_s` (how far into the future) and `inactivity_prob` (probability the speaker has finished).
- `step_idx`: Incrementing step counter.
- `step_duration_s`: Duration of each step (usually 0.08s / 80ms).
- `total_duration_s`: Total audio processed so far.

### Turn-Taking Decision Pattern

A common starting point for voice agents is to watch the longest horizon and flush when the inactivity probability stays above a threshold:

```python
if msg["type"] == "step":
    turn_done = msg["vad"][-1]["inactivity_prob"] > 0.5
    if turn_done:
        await stt.send_flush(flush_id=1)
```

### Adaptive Delay Control

`delay_in_frames` (in `json_config`) tunes the latency/quality tradeoff:

- Each frame = 80ms of context the model processes before emitting text.
- Range: `0`–`80`.
- Lower values (e.g. `8`): more reactive, faster back-and-forth. Use when responsiveness matters more than context.
- Higher values (e.g. `16`): better transcription quality at the cost of latency.
- Default balanced starting point: `16`.

### Flush at Turn Boundaries

Use `flush` to process buffered audio without waiting for natural silence:

```python
# Client sends:
{"type": "flush", "flush_id": 1}

# Server responds:
{"type": "flushed", "flush_id": 1}
```

The `flushed` response confirms all audio sent before the flush has been processed and any pending transcript has been emitted. This is critical for voice agents that need to decide when a speaker has actually finished and hand the turn to the agent.

---

## 13. Multiplexing

### Main Concepts

Multiplexing allows sending multiple independent requests over a single WebSocket connection. Each request is tracked independently using a unique `client_req_id`, enabling concurrent processing of multiple inputs without opening multiple connections.

### Enabling Multiplexing

1. Set `close_ws_on_eos: false` in the `setup` message — this tells the server to keep the WebSocket connection open after completing individual requests.
2. Attach a unique `client_req_id` to every message for a request (setup, text/audio, end_of_stream).
3. Route every server response by its matching `client_req_id`.

### Example: Multiple TTS Requests Over One Socket

```python
setup = {
    "voice_id": "RI2y7oBdsQJmkgFF",
    "output_format": "wav",
    "close_ws_on_eos": False  # Enable multiplexing
}

texts = [
    "First request. Second part, last one.",
    "Second request. Second part, last one again.",
]

client = gradium.client.GradiumClient(base_url=url, api_key=api_key)

async with client.tts_realtime(send_setup_on_start=False) as stream:
    async def send_loop():
        for idx, text in enumerate(texts):
            stamp = {'client_req_id': f'req-{idx:02d}'}
            await stream.send_setup(setup | stamp)
            await stream.send_text(text, **stamp)
            await stream.send_eos(**stamp)

    async def recv_loop():
        audio = collections.defaultdict(list)
        num_eos = 0
        async for msg in stream:
            if msg["type"] == "audio":
                audio[msg.get('client_req_id')].append(msg["audio"])
            elif msg['type'] == 'end_of_stream':
                num_eos += 1
                if num_eos == len(texts):
                    break
        return audio

    _, audio = await asyncio.gather(send_loop(), recv_loop())
    audio = {k: b"".join(v) for k, v in audio.items()}
```

### Multiplexed Setup Fields

| Field | Description |
|-------|-------------|
| `close_ws_on_eos` | Set to `false` to keep socket open after each request's `end_of_stream`. |
| `client_req_id` | Unique identifier per request. Included in every message (setup, text/audio, end_of_stream). Server stamps all responses with the same `client_req_id`. |
| `send_setup_on_start` | SDK parameter. Set to `false` to delay setup and allow manual setup sending per multiplexed request. |

---

## 14. Browser WebSockets

### Main Concepts

Browser and mobile WebSocket clients should not expose API keys. Gradium supports short-lived, single-use tokens for client-side connections.

### Token-Based Connection

1. **Server-side**: Generate a short-lived, single-use token using the API key.
2. **Client-side**: Connect with `?token=...` query parameter instead of the `x-api-key` header.

```javascript
// Browser example (conceptual)
const ws = new WebSocket("wss://api.gradium.ai/api/speech/tts?token=SHORT_LIVED_TOKEN");
```

This pattern allows browser-based voice applications (microphone capture → STT, TTS playback) without exposing the API key in client-side code.

### Use Cases

- **Browser microphone → STT**: Capture microphone audio in a browser and stream it to Gradium STT.
- **Browser TTS playback**: Receive TTS audio chunks in a browser for real-time playback.
- **Mobile clients**: Same token-based pattern for mobile WebSocket connections.

---

## 15. In-Text Control Tags

Gradium TTS recognizes special inline tags within the text to control audio generation behavior.

### `<flush>` Tag

Forces the model to output audio for all text input so far, without waiting for more context. The model normally lags a few words behind the text input (it needs enough context to generate audio). Use `<flush>` when an upstream LLM has finished a thought and you want the model to emit remaining buffered audio.

```python
sample_text = "Hello, this is a test from the Gradium Text to Speech system. <flush> We are testing the flush."

test_audio = await client.tts(
    setup={'voice_id': 'YTpq7expH9539ERJ', 'output_format': 'wav'},
    text=sample_text,
)
```

> **Caution**: Avoid flushing after every token; small text fragments reduce prosody quality.

### `<break time="Ns" />` Tag

Inserts a pause in the generated audio. The break time is specified in seconds and should be between 0.1 and 2.0s. The tag must be preceded and followed by a space.

```python
sample_text = 'Hello, this is a test. <break time="1.5s" /> We are testing the pause.'

test_audio = await client.tts(
    setup={'voice_id': 'YTpq7expH9539ERJ', 'output_format': 'wav'},
    text=sample_text,
)
```

---

## 16. Credits & Metering

### Credit Consumption

Credits are consumed based on audio processed:

| Operation | Rate |
|-----------|------|
| Text-to-Speech | 1 credit per character synthesized (~750 characters/minute → ~45,000 credits/hour) |
| Speech-to-Text | 3 credits per second of audio transcribed |

### Get Credit Balance — `GET /usages/credits`

Retrieve the current Gradium credit balance for the authenticated subscription.

**SDK:**
```python
import json
import gradium

client = gradium.client.GradiumClient()
credits_info = await gradium.usages.get(client)
print(json.dumps(credits_info, indent=2))
```

### Monitoring

Monitor credit consumption to avoid unexpected depletion. The credit balance is available via the REST endpoint and the SDK helper. One hour of continuous TTS generation (at ~750 chars/min) costs approximately 45,000 credits; one hour of STT transcription costs approximately 10,800 credits (3 × 3600s).

---

## 17. Audio Formats Reference

### TTS Output Formats

| Format | Sample Rate | Bit Depth | Channels | Container | Use Case |
|--------|------------|-----------|----------|-----------|----------|
| `wav` | 48 kHz | 16-bit signed | Mono | WAV | General use, file output |
| `pcm` | 48 kHz | 16-bit signed | Mono | Raw PCM | Low-level audio processing, streaming |
| `opus` | — | — | — | Ogg/Opus | Bandwidth-efficient streaming |
| `ulaw_8000` | 8 kHz | μ-law | Mono | Raw | Telephony (US) |
| `alaw_8000` | 8 kHz | A-law | Mono | Raw | Telephony (Europe) |
| `pcm_8000` | 8 kHz | 16-bit | Mono | Raw PCM | Telephony, low-bandwidth |
| `pcm_16000` | 16 kHz | 16-bit | Mono | Raw PCM | Medium quality |
| `pcm_24000` | 24 kHz | 16-bit | Mono | Raw PCM | S2S input default |

**PCM chunk size**: 3840 samples per chunk (80ms at 48kHz).

### STT Input Formats

| Format | Sample Rate | Notes |
|--------|------------|-------|
| `pcm` | 24 kHz | Default for WebSocket STT. 16-bit signed mono. |
| `wav` | Various | WAV container with embedded header. |
| `opus` | — | Ogg/Opus container. |
| `ulaw_8000` | 8 kHz | μ-law, telephony. |
| `alaw_8000` | 8 kHz | A-law, telephony. |
| `pcm_8000` | 8 kHz | Raw PCM, telephony. |
| `pcm_16000` | 16 kHz | Raw PCM. |
| `pcm_24000` | 24 kHz | Raw PCM. |
| `pcm_48000` | 48 kHz | Raw PCM. |

### S2S Audio Formats

- **Input**: `pcm` expects 24 kHz, 16-bit little-endian mono.
- **Output**: `pcm` produces 48 kHz, 16-bit little-endian mono.
- Both `input_format` and `output_format` accept the same format values as TTS/STT.

### PCM Sample Retrieval (SDK)

```python
result = await client.tts(
    setup={"voice_id": "YTpq7expH9539ERJ", "output_format": "pcm"},
    text="Hello"
)

# Get numpy arrays from PCM
pcm_array = result.pcm()
pcm16_array = result.pcm16()
```

---

## 18. Error Handling

### WebSocket Errors

WebSocket errors are sent as JSON messages, then the socket closes:

```json
{"type": "error", "message": "Session not found. Send setup first.", "code": 1002}
```

Treat `error` as terminal for that socket. Open a new connection when retrying.

**Common error codes:**

| Code | Meaning |
|------|---------|
| `1002` | Protocol error (e.g., sending input before setup, reusing an active `client_req_id`). |
| `1008` | Policy violation (invalid auth, missing subscription, invalid request policy). |
| `1011` | Internal server error or unexpected session failure. |

### REST Errors

REST endpoints return standard HTTP status codes with JSON error bodies. The STT REST endpoint streams NDJSON results; errors may appear as error objects within the stream.

### Error Format

```json
{
  "type": "error",
  "message": "Error description",
  "code": 1008
}
```

---

## 19. SDK API Shapes

The Gradium Python SDK (`pip install gradium`) provides three API shapes for each product (TTS, STT, S2S), chosen based on the use case:

### Shape Comparison

| Shape | Input | Output | Use Case |
|-------|-------|--------|----------|
| `_realtime` | Concurrent send/receive loops | Async iterator of messages | Voice agents, LLM token streaming, browser/telephony bridges |
| `_stream` | String, list, or async generator | Async iterator of audio bytes (TTS) or messages (STT/S2S) | Iterable input, pull-based consumption |
| buffered (`tts`/`stt`/`s2s`) | Complete input | Single result object | Scripts, tests, one-shot generation |

### Client Creation

```python
import gradium

# From API key parameter
client = gradium.client.GradiumClient(api_key="gd_your_api_key_here")

# From environment variable GRADIUM_API_KEY
client = gradium.client.GradiumClient()

# With custom base URL (for testing/proxying)
client = gradium.client.GradiumClient(base_url="https://custom.endpoint/api", api_key=api_key)
```

### TTS Methods

| Method | Input | Output |
|--------|-------|--------|
| `client.tts(setup, text)` | `setup` dict + `text` (string or async generator) | `TTSResult` with `raw_data`, `sample_rate`, `request_id`, `text_with_timestamps`, `pcm()`, `pcm16()` |
| `client.tts_stream(setup, text)` | `setup` dict + `text` (string, list, async generator) | Async iterator of audio bytes via `stream.iter_bytes()` |
| `client.tts_realtime(voice_id, output_format, ...)` | Keyword args; `send_text()`, `send_eos()` | Async iterator of message dicts |

### STT Methods

| Method | Input | Output |
|--------|-------|--------|
| `client.stt(setup, audio)` | `setup` dict + audio data | Buffered result |
| `client.stt_stream(setup, audio_gen)` | `setup` dict + async generator of audio | Async iterator of messages |
| `client.stt_realtime(model_name, input_format, json_config, ...)` | Keyword args; `send_audio()`, `send_flush()`, `send_eos()` | Async iterator of message dicts |

### S2S Methods

| Method | Input | Output |
|--------|-------|--------|
| `client.s2s(setup, audio)` | `setup` dict + audio data | Buffered result |
| `client.s2s_stream(setup, audio_gen)` | `setup` dict + async generator of audio | Async iterator of messages |
| `client.s2s_realtime(model_name, stt_model_name, tts_model_name, input_format, output_format, voice_id, json_config, ...)` | Keyword args; `send_audio()`, `send_eos()` | Async iterator of message dicts |

### TTSResult Object

The buffered `client.tts()` method returns a `TTSResult` with:

| Property | Type | Description |
|----------|------|-------------|
| `raw_data` | bytes | Complete audio bytes |
| `sample_rate` | int | Output sample rate (e.g. 48000) |
| `request_id` | string | Gradium request ID for logging |
| `text_with_timestamps` | list | List of `{text, start_s, stop_s}` items |
| `pcm()` | numpy array | PCM audio as numpy array |
| `pcm16()` | numpy array | 16-bit PCM audio as numpy array |

---

## 20. Integrations & Migration

### Agent Framework Integrations

Gradium integrates with several voice agent frameworks:

| Framework | Integration |
|-----------|-------------|
| **LiveKit** | Use Gradium speech models in LiveKit Agents |
| **Pipecat** | Use Gradium streaming text-to-speech in Pipecat voice agents |
| **OpenClaw** | Use Gradium text-to-speech in OpenClaw agents |
| **Gradbot** | Prototype voice agents quickly with Gradium APIs |

### Web Search Integrations

For voice agents that need real-time web search:

| Provider | Use Case |
|----------|----------|
| **Tavily** | Realtime web search with Gradium voice AI agents |
| **Linkup** | Web search with Gradium voice AI agents |
| **Keenable** | Realtime web search with Gradium voice AI agents |

### Migration Guides

Gradium provides migration guides from competing platforms:

| From | Guide |
|------|-------|
| Cartesia | `/guides/migration/cartesia` — Change endpoints, auth, and request fields |
| Deepgram | `/guides/migration/deepgram` — Replace Deepgram speech integrations with Gradium STT and TTS |
| ElevenLabs | `/guides/migration/elevenlabs` — Replace ElevenLabs TTS calls with Gradium REST and WebSocket endpoints |

### Recipes

| Recipe | Description |
|--------|-------------|
| Browser Microphone to STT | Capture microphone audio in a browser and stream it to Gradium STT |
| LLM Tokens to Streaming TTS | Stream generated text into TTS while preserving prosody and low latency |
| Telephony Audio Formats | Use μ-law, A-law, and low-sample-rate PCM with Gradium voice APIs |
| Turn-Taking with Semantic VAD | Use STT semantic VAD and adaptive delay to decide when a user has finished speaking |

---

## 21. Capability Summary & Cross-Reference

### Capability Matrix

| Capability | Transport | Endpoint | Model | Key Parameters |
|------------|-----------|----------|-------|----------------|
| **TTS — Streaming** | WebSocket | `wss://.../speech/tts` | `default` | `voice_id`, `output_format`, `json_config` (temp, cfg_coef, padding_bonus, rewrite_rules), `pronunciation_id` |
| **TTS — One-shot** | REST POST | `POST /post/speech/tts` | `default` | Same as WebSocket, `json_config` as URL-encoded JSON query param |
| **STT — Streaming** | WebSocket | `wss://.../speech/asr` | `default` / `stt-translate` | `input_format`, `json_config` (language, target_language, temp, delay_in_frames, padding_bonus) |
| **STT — One-shot** | REST POST | `POST /post/speech/asr` | `default` / `stt-translate` | Same as WebSocket, audio in request body, NDJSON response |
| **S2S — Live Translation** | WebSocket | `wss://.../speech/s2s` | `s2s-translate` | `stt_model_name`, `tts_model_name`, `voice_id`, `input_format`, `output_format`, `json_config` (target_language) |
| **Voice Cloning** | REST POST | `POST /voices/` | — | `audio_file`, `name`, `description`, `language`, `start_s` |
| **List Voices** | REST GET | `GET /voices/` | — | — |
| **Get Voice** | REST GET | `GET /voices/{voice_id}` | — | — |
| **Update Voice** | REST | `PATCH /voices/{voice_id}` | — | Voice metadata fields |
| **Delete Voice** | REST DELETE | `DELETE /voices/{voice_id}` | — | — |
| **Create Pronunciation** | REST POST | `POST /pronunciations/` | — | Language-specific rewrite rules |
| **List Pronunciations** | REST GET | `GET /pronunciations/` | — | — |
| **Get Pronunciation** | REST GET | `GET /pronunciations/{uid}` | — | — |
| **Update Pronunciation** | REST | `PATCH /pronunciations/{uid}` | — | Updated rewrite rules |
| **Delete Pronunciation** | REST DELETE | `DELETE /pronunciations/{uid}` | — | — |
| **Get Credits** | REST GET | `GET /usages/credits` | — | — |
| **Multiplexing** | WebSocket | Same endpoints | All | `close_ws_on_eos: false`, `client_req_id` on every message |
| **Browser Tokens** | WebSocket | Same endpoints | All | `?token=...` query parameter |

### Feature Comparison Across Products

| Feature | TTS | STT | S2S |
|---------|-----|-----|-----|
| WebSocket streaming | ✅ | ✅ | ✅ |
| REST one-shot | ✅ | ✅ | ❌ |
| Real-time VAD (`step` messages) | ❌ | ✅ | ❌ |
| Flush control | ✅ (`<flush>` tag) | ✅ (`flush` message) | ❌ |
| Adaptive delay | ❌ | ✅ (`delay_in_frames`) | ❌ |
| Word-level timestamps | ✅ (`text` messages) | ✅ (`text` + `end_text`) | ✅ (`text` messages) |
| Multiplexing | ✅ | ✅ | ✅ |
| Browser tokens | ✅ | ✅ | ✅ |
| Pronunciation dictionaries | ✅ | ❌ | ❌ |
| Voice selection | ✅ (`voice_id`) | ❌ | ✅ (`voice_id`) |
| Language selection | ❌ | ✅ (`language` / `target_language`) | ✅ (`target_language`) |
| Speed control | ✅ (`padding_bonus`) | ❌ | ❌ |
| Temperature control | ✅ (`temp`) | ✅ (`temp`) | ❌ |
| Voice similarity | ✅ (`cfg_coef`) | ❌ | ❌ |
| Rewrite rules | ✅ (`rewrite_rules`) | ❌ | ❌ |
| Custom voice cloning | ✅ | ❌ | ✅ (output voice) |

### Supported Languages

| Code | Language | TTS | STT | S2S (target) |
|------|----------|-----|-----|-------------|
| `en` | English | ✅ | ✅ | ✅ |
| `fr` | French | ✅ | ✅ | ✅ |
| `de` | German | ✅ | ✅ | ✅ |
| `es` | Spanish | ✅ | ✅ | ✅ |
| `pt` | Portuguese | ✅ | ✅ | ✅ |

### Key Design Decisions

1. **Unified WebSocket lifecycle** — TTS, STT, and S2S all share the same `setup → ready → input → end_of_stream` protocol, differing only in input/output message types.
2. **Dual transport** — WebSocket for streaming/real-time, REST POST for one-shot/batch. Same models, same `json_config` options.
3. **Semantic VAD as a first-class signal** — STT WebSocket streams include `step` messages every 80ms with multi-horizon inactivity predictions, enabling sophisticated turn-taking logic in voice agents.
4. **Adaptive delay** — `delay_in_frames` gives developers explicit control over the latency/quality tradeoff, from fast back-and-forth (low values) to high-quality transcription (high values).
5. **Multiplexing via `client_req_id`** — A single WebSocket connection can handle multiple independent requests, reducing connection overhead for high-throughput applications.
6. **In-text control** — TTS supports `<flush>` and `<break>` tags for fine-grained control over audio generation timing and pauses.
7. **Pronunciation dictionaries** — Language-specific rewrite rules, manageable via REST API or Gradium Studio, attachable per TTS request.
8. **Instant voice cloning** — Custom voices from a 10-second audio sample, usable identically to flagship voices.
9. **Low latency** — Time-to-first-token below 300ms when streaming, with servers in Europe and the US.
10. **Credit-based metering** — Transparent per-character (TTS) and per-second (STT) credit consumption with a REST endpoint for balance monitoring.