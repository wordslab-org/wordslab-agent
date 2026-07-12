# Google Gemini API Analysis — Text-to-Speech, Audio Understanding, Live Conversational Voice & Live Translation

> **Base URL:** `https://generativelanguage.googleapis.com/v1beta` (REST) | **WebSocket:** `wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent`
> **Docs:** `https://ai.google.dev/gemini-api/docs` | **API Reference:** `https://ai.google.dev/api/live`
> **Auth:** API key via `x-goog-api-key` header (REST) or `?key=` query param (WebSocket) | Ephemeral tokens via `access_token` query param (`v1alpha`)
> **SDKs:** `google-genai` (Python), `@google/genai` (TypeScript/JavaScript), Go, Java, C# | Partner integrations: LiveKit, Pipecat, Fishjam, Stream Vision Agents, Voximplant, Agora, Firebase AI SDK
> **Description:** Google Gemini API is a multimodal AI platform providing native text-to-speech generation (single-speaker and multi-speaker with director-style prompting), audio understanding (transcription, diarization, emotion detection, summarization via structured outputs), real-time bidirectional voice and vision conversations over WebSockets (Live API), and real-time speech-to-speech translation across 70+ languages. Voice capabilities are distributed across the Interactions API (TTS, audio analysis) and the Live API (real-time conversational streaming, translation), with 30 prebuilt voices and 97+ supported languages.

---

## Table of Contents

1. [Platform Overview & Core Concepts](#1-platform-overview--core-concepts)
2. [Authentication & Access Control](#2-authentication--access-control)
3. [Models — Catalog & Selection Guide](#3-models--catalog--selection-guide)
4. [Text-to-Speech (TTS) — Interactions API](#4-text-to-speech-tts--interactions-api)
5. [Audio Understanding — Analysis & Transcription](#5-audio-understanding--analysis--transcription)
6. [Live API — Real-Time Conversational Voice](#6-live-api--real-time-conversational-voice)
7. [Live API — Session Management](#7-live-api--session-management)
8. [Live API — Voice Activity Detection (VAD)](#8-live-api--voice-activity-detection-vad)
9. [Live API — Tool Use & Function Calling](#9-live-api--tool-use--function-calling)
10. [Live API — Native Audio Capabilities](#10-live-api--native-audio-capabilities)
11. [Live Translation — Real-Time Speech-to-Speech Translation](#11-live-translation--real-time-speech-to-speech-translation)
12. [Voice Catalog & Supported Languages](#12-voice-catalog--supported-languages)
13. [Audio Formats & Technical Specifications](#13-audio-formats--technical-specifications)
14. [Partner Integrations](#14-partner-integrations)
15. [Capability Summary & Cross-Reference](#15-capability-summary--cross-reference)

---

## 1. Platform Overview & Core Concepts

### Main Concepts

Google's voice capabilities are organized across two distinct APIs with different mental models:

- **Interaction** — A single request-response cycle via the Interactions API. Used for TTS generation and audio understanding (analysis, transcription). Supports streaming responses. Identified by the `client.interactions.create()` SDK method or `POST /v1beta/interactions` REST endpoint.
- **Session** — A persistent, stateful WebSocket connection via the Live API. Used for real-time bidirectional voice/video conversations and live translation. Identified by a `BidiGenerateContentSetup` message that configures the session for its duration. Sessions support continuous streaming of audio, video, and text.
- **Response Modality** — The output format requested from the model. For TTS, set to `"audio"`. For Live API, set to `["AUDIO"]` (native audio models only support AUDIO output modality). The `response_format` (Interactions API) / `responseModalities` (Live API) parameter controls this.
- **Voice** — A prebuilt speech identity identified by a voice name (e.g. `Kore`, `Puck`, `Charon`). 30 voices available, each with a descriptive personality trait (e.g. "Firm", "Upbeat", "Breathy"). Voices are shared across TTS and Live API. No voice cloning or custom voice creation is available.
- **Speech Config** — Configuration object that maps voices to speakers. For single-speaker TTS: `speech_config: [{voice: "Kore"}]`. For multi-speaker: `speech_config: [{speaker: "Joe", voice: "Kore"}, {speaker: "Jane", voice: "Puck"}]`. For Live API: `speechConfig.voiceConfig.prebuiltVoiceConfig.voiceName`.
- **Audio Tags** — Inline non-speech directives in text, wrapped in square brackets (e.g. `[whispers]`, `[laughs]`, `[cough]`, `[sighs]`, `[gasp]`). Supported by TTS models to influence emotional delivery, pace, and non-verbal sounds. No exhaustive list — experimentation is recommended.
- **Prompting Structure** — TTS models accept director-style prompts with: **Audio Profile** (persona), **Scene** (environment/mood), **Director's Notes** (style, accent, pacing), and **Transcript**. The model uses LLM context awareness to generate expressive performances.
- **Blob** — A data container for media input in the Live API. Contains `data` (base64-encoded) and `mime_type` (e.g. `audio/pcm;rate=16000`, `image/jpeg`).
- **Realtime Input** — Continuous streaming input (audio, video, text) sent via `send_realtime_input` / `sendRealtimeInput`. Optimized for responsiveness over deterministic ordering. End-of-turn is derived from user activity (e.g. end of speech via VAD).
- **Client Content** — Incremental conversation updates sent via `send_client_content` / `sendClientContent`. Unconditionally appended to conversation history. Interrupts current model generation. Used for seeding context and turn-by-turn interactions.
- **VAD (Voice Activity Detection)** — Automatic detection of speech start/end to manage turn boundaries. Configurable sensitivity, padding, and silence duration. Can be disabled for manual client-side VAD control via `activityStart`/`activityEnd` messages.
- **Interruption (Barge-in)** — When VAD detects user speech during model generation, the current response is cancelled and discarded. The server sends an `interrupted` flag in `BidiGenerateContentServerContent`.
- **Session Resumption** — Mechanism to resume a session across WebSocket reconnections using a resumption handle/token. Tokens valid for 2 hours after session termination.
- **Context Window Compression** — Sliding-window mechanism that discards older context to keep sessions running indefinitely. Triggered at a configurable token threshold.
- **Ephemeral Token** — Short-lived authentication token for client-to-server Live API connections. Avoids exposing API keys in browser/mobile apps. Configurable expiration, usage limits, and configuration constraints. Only compatible with Live API (`v1alpha`).
- **Thinking** — Model reasoning before responding. Gemini 3.1 uses `thinkingLevel` (`minimal`/`low`/`medium`/`high`, default `minimal`). Gemini 2.5 uses `thinkingBudget` (token count, `0` to disable).

### Platform Architecture

```
Text Input ──▶ Interactions API (TTS) ──▶ Audio Output (buffered or streamed)
                  │
    Voice Selection ──▶ 30 prebuilt voices
    Speech Config ──▶ single-speaker or multi-speaker (up to 2)
    Audio Tags ──▶ [whispers], [laughs], [excitedly], etc.
    Director Prompts ──▶ Audio Profile, Scene, Director's Notes, Transcript

Audio Input ──▶ Interactions API (Audio Understanding) ──▶ Text Response
                  │
    File Upload ──▶ Files API (>20MB) or inline base64 (<20MB)
    Structured Output ──▶ transcription, diarization, emotion, translation, summary
    Timestamps ──▶ MM:SS format referencing

Audio/Video/Text ──▶ Live API (WebSocket) ──▶ Audio Output + Transcriptions
                       │ (stateful, bidirectional, low-latency)
    Setup ──▶ model, generationConfig, systemInstruction, tools
    Realtime Input ──▶ send_realtime_input (audio/video/text streams)
    Client Content ──▶ send_client_content (turn-by-turn, history seeding)
    VAD ──▶ automatic or manual activity detection
    Tool Calls ──▶ function calling, Google Search grounding
    Session Mgmt ──▶ resumption, context compression, GoAway

Audio Input ──▶ Live Translation (WebSocket) ──▶ Translated Audio Output
                  │
    Translation Config ──▶ target_language_code, echo_target_language
    Audio-only input ──▶ continuous stream processing (no turn-based)
```

### End-to-End Flows

**Text-to-Speech pipeline (single request):**
```
Select voice ──▶ POST /v1beta/interactions
                 │
  model: "gemini-3.1-flash-tts-preview"
  input: "Say cheerfully: Have a wonderful day!"
  response_format: {"type": "audio"}
  generation_config: {"speech_config": [{"voice": "Kore"}]}
  stream: true (optional, for streaming)
                 │
                 ▼
  interaction.output_audio.data (base64 PCM, 24kHz, 16-bit)
  OR streamed: event.delta.data (for stream=True)
```

**Multi-speaker TTS pipeline:**
```
Write dialogue prompt ──▶ POST /v1beta/interactions
                           │
  model: "gemini-3.1-flash-tts-preview"
  input: "TTS the following conversation between Joe and Jane:
          Joe: How's it going? Jane: Not too bad!"
  response_format: {"type": "audio"}
  generation_config: {"speech_config": [
    {"speaker": "Joe", "voice": "Kore"},
    {"speaker": "Jane", "voice": "Puck"}
  ]}
```

**Audio understanding pipeline:**
```
Upload audio file ──▶ POST /v1beta/interactions
                       │
  model: "gemini-3.5-flash"
  input: [
    {"type": "text", "text": "Transcribe with diarization and emotion detection"},
    {"type": "audio", "uri": file_uri, "mime_type": "audio/mp3"}
  ]
  response_format: response_schema (structured output)
                       │
                       ▼
  Structured JSON: {summary, segments: [{speaker, timestamp, content, language, emotion}]}
```

**Real-time conversational voice (Live API):**
```
Client ──WebSocket──▶ Gemini Live API
          │
  1. Connect: wss://...BidiGenerateContent?key=API_KEY
  2. Send setup: {model, generationConfig: {responseModalities: ["AUDIO"], speechConfig}}
  3. Receive: setupComplete
  4. Stream audio: send_realtime_input(audio=Blob(data, "audio/pcm;rate=16000"))
  5. VAD detects end-of-speech ──▶ model generates audio response
  6. Receive: serverContent.modelTurn.parts[].inlineData.data (base64 PCM, 24kHz)
  7. User interrupts ──▶ interrupted=true, generation cancelled
  8. Optional: toolCall ──▶ execute function ──▶ send_tool_response
```

**Live translation pipeline:**
```
Client ──WebSocket──▶ Gemini Live Translate
          │
  1. Connect: wss://...BidiGenerateContent?key=API_KEY
  2. Send setup: {model: "gemini-3.5-live-translate-preview",
     generationConfig: {responseModalities: ["AUDIO"],
       translationConfig: {targetLanguageCode: "pl", echoTargetLanguage: true}}}
  3. Stream source audio: send_realtime_input(audio=Blob(data, "audio/pcm;rate=16000"))
  4. Receive: serverContent.modelTurn.parts[].inlineData.data (translated audio)
  5. Receive: inputTranscription (source text), outputTranscription (translated text)
```

---

## 2. Authentication & Access Control

### API Keys

All Gemini API requests require an API key:

**REST (Interactions API):**
```
x-goog-api-key: YOUR_GEMINI_API_KEY
```

**WebSocket (Live API):**
```
wss://...?key=YOUR_GEMINI_API_KEY
```

API keys are obtained from [Google AI Studio](https://aistudio.google.com/apikey).

### Ephemeral Tokens (Live API Only)

For client-to-server implementations (browser/mobile directly connecting to Gemini), ephemeral tokens avoid exposing API keys:

**Flow:**
1. Client authenticates with your backend
2. Backend requests ephemeral token from Gemini (`v1alpha`)
3. Gemini issues short-lived token
4. Backend sends token to client
5. Client uses token as API key for Live API WebSocket connection (`v1alpha` endpoint)

**Token creation:**

```python
client = genai.Client(http_options={'api_version': 'v1alpha'})
token = client.auth_tokens.create(
    config={
        'uses': 1,                              # Number of sessions (default 1)
        'expire_time': now + timedelta(minutes=30),  # Max 20 hours, default 30 min
        'new_session_expire_time': now + timedelta(minutes=1),  # Default 1 min
        'live_connect_constraints': {            # Optional: lock configuration
            'model': 'gemini-3.1-flash-live-preview',
            'config': {
                'session_resumption': {},
                'temperature': 0.7,
                'response_modalities': ['AUDIO']
            }
        }
    }
)
# Pass token.name to client
```

**Connection with ephemeral token:**
```
wss://...v1alpha.GenerativeService.BidiGenerateContentConstrained?access_token={token}
```

Or via HTTP Authorization header: `Authorization: Token {token}`

### AuthToken Parameters

| Parameter | Type | Default | Constraint | Description |
|-----------|------|---------|------------|-------------|
| `uses` | int32 | 1 | — | Number of sessions token can start. `0` = unlimited. Session resumption doesn't count as a use. |
| `expireTime` | Timestamp | 30 min future | < 20 hours | Time after which messages are rejected |
| `newSessionExpireTime` | Timestamp | 60 sec future | < 20 hours | Time after which new sessions are rejected |
| `fieldMask` | FieldMask | — | — | Controls which setup fields from token override connection setup |
| `bidiGenerateContentSetup` | BidiGenerateContentSetup | — | — | Locked configuration for the token |
| `liveConnectConstraints` | object | — | — | Constrain model and config (used for Live Translate) |

### Best Practices

- Set short `expire_time` for security
- Ephemeral tokens only as secure as your backend auth
- Avoid for backend-to-Gemini connections (typically secure)
- Use `sessionResumption` to reconnect every 10 minutes (same token works even with `uses: 1`)
- For Live Translate, use `live_connect_constraints` to lock `translationConfig`

---

## 3. Models — Catalog & Selection Guide

### Model Catalog

| Model ID | Type | Description | Languages | Key Feature |
|----------|------|-------------|-----------|-------------|
| `gemini-3.1-flash-tts-preview` | TTS | Latest TTS, single + multi-speaker | 95+ (auto-detected) | Director-style prompting, audio tags |
| `gemini-2.5-flash-preview-tts` | TTS | Previous gen TTS | 95+ | Single + multi-speaker |
| `gemini-2.5-pro-preview-tts` | TTS | Pro-quality TTS | 95+ | Higher quality |
| `gemini-3.5-flash` | Audio Understanding | Audio analysis, transcription, structured output | Multilingual | Structured outputs, diarization |
| `gemini-3.1-flash-live-preview` | Live API (Conversational) | Real-time bidirectional voice/video | 97 | `thinkingLevel`, native audio, turn coverage |
| `gemini-2.5-flash-live-preview` | Live API (Conversational) | Previous gen real-time voice/video | 97 | `thinkingBudget`, async function calling, proactive audio, affective dialog |
| `gemini-3.5-live-translate-preview` | Live Translation | Real-time speech-to-speech translation | 70+ | Continuous stream translation |

### Model Comparison: Gemini 3.1 vs 2.5 Flash Live

| Feature | Gemini 3.1 Flash Live Preview | Gemini 2.5 Flash Live Preview |
|---------|-------------------------------|-------------------------------|
| **Thinking** | `thinkingLevel` (`minimal`/`low`/`medium`/`high`), default `minimal` | `thinkingBudget` (token count), `0` to disable, dynamic by default |
| **Server events** | Multiple content parts per event (audio + transcript together) | One content part per event |
| **Client content** | `send_client_content` only for initial history seeding; use `send_realtime_input` during conversation | `send_client_content` supported throughout |
| **Turn coverage** | `TURN_INCLUDES_AUDIO_ACTIVITY_AND_ALL_VIDEO` (default) | `TURN_INCLUDES_ONLY_ACTIVITY` (default) |
| **Custom VAD** | Supported | Supported |
| **Automatic VAD config** | Supported | Supported |
| **Async function calling** | Not supported (sequential only) | Supported (`NON_BLOCKING` + `scheduling`) |
| **Proactive audio** | Not supported | Supported (`proactive_audio: true`, `v1alpha`) |
| **Affective dialog** | Not supported | Supported (`enable_affective_dialog: true`, `v1alpha`) |

### Selection Guide

| Use Case | Recommended Model | Rationale |
|----------|-------------------|-----------|
| Podcast/audiobook generation | `gemini-3.1-flash-tts-preview` | Director-style prompting, audio tags, multi-speaker |
| Real-time voice agent | `gemini-3.1-flash-live-preview` | Lowest latency (minimal thinking), native audio |
| Real-time agent with async tools | `gemini-2.5-flash-live-preview` | Non-blocking function calling |
| Emotion-adaptive conversations | `gemini-2.5-flash-live-preview` | Affective dialog support |
| Real-time translation | `gemini-3.5-live-translate-preview` | Dedicated translation model, 70+ languages |
| Audio transcription/analysis | `gemini-3.5-flash` | Structured outputs, diarization, emotion detection |
| Pre-generate transcript then narrate | `gemini-3.5-flash` + `gemini-3.1-flash-tts-preview` | Two-step: generate text, then TTS |

---

## 4. Text-to-Speech (TTS) — Interactions API

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `POST /v1beta/interactions` | REST | Create TTS (buffered or streamed) |
| `client.interactions.create()` | SDK (Python) | TTS generation |
| `client.interactions.create()` | SDK (JS) | TTS generation |

### Main Concepts

- **Controllable TTS** — Unlike conventional TTS, Gemini TTS uses an LLM that knows not only *what* to say but also *how* to say it. Natural language prompts guide style, accent, pace, and tone.
- **Single-speaker** — One voice configured via `speech_config: [{voice: "Kore"}]`
- **Multi-speaker** — Up to 2 speakers via `speech_config: [{speaker: "Joe", voice: "Kore"}, {speaker: "Jane", voice: "Puck"}]`. Speaker names must match the prompt transcript.
- **Streaming** — Set `stream: true` to receive audio chunks as generated. Events have `event_type: "step.delta"` with `delta.type: "audio"` and `delta.data` (base64).
- **Audio Tags** — Inline modifiers in square brackets. Change tone, pace, emotion, add non-verbal sounds. No exhaustive list; experimentation encouraged. Use English tags even for non-English transcripts.
- **Director's Prompt Structure** — Audio Profile (persona) + Scene (environment) + Director's Notes (style/accent/pacing) + Transcript.

### Request Parameters (POST /v1beta/interactions)

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `model` | string | **Yes** | — | Model ID (e.g. `gemini-3.1-flash-tts-preview`) |
| `input` | string | **Yes** | — | Text to convert to speech (may include audio tags and director prompts) |
| `response_format` | object | No | — | `{"type": "audio"}` to request audio output |
| `generation_config` | object | No | — | Contains `speech_config` array |
| `stream` | boolean | No | false | Stream audio chunks as generated |

### Speech Config Structure

**Single-speaker:**
```json
{
  "speech_config": [
    {"voice": "Kore"}
  ]
}
```

**Multi-speaker (up to 2):**
```json
{
  "speech_config": [
    {"speaker": "Joe", "voice": "Kore"},
    {"speaker": "Jane", "voice": "Puck"}
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `voice` | string | Yes | Voice name from the 30 prebuilt voices |
| `speaker` | string | Multi-speaker only | Speaker name matching the prompt transcript |

### Output Audio

- **Format:** Raw 16-bit PCM, 24kHz, little-endian, mono
- **Retrieval:** `interaction.output_audio.data` (base64-encoded) for buffered; `event.delta.data` for streamed
- **WAV file creation:** channels=1, rate=24000, sample_width=2

### Streaming TTS

```python
stream = client.interactions.create(
    model="gemini-3.1-flash-tts-preview",
    input="Say cheerfully: Have a wonderful day!",
    response_format={"type": "audio"},
    generation_config={"speech_config": [{"voice": "Kore"}]},
    stream=True
)
for event in stream:
    if event.event_type == "step.delta":
        if event.delta.type == "audio":
            audio_data = base64.b64decode(event.delta.data)
            # Process audio chunk
```

### Prompting Guide

**Recommended prompt structure:**

```
# AUDIO PROFILE: Jaz R.
## "The Morning Hype"

## THE SCENE: The London Studio
It is 10:00 PM in a glass-walled studio overlooking the moonlit London skyline...

### DIRECTOR'S NOTES
Style: The "Vocal Smile": You must hear the grin in the audio...
Pace: Speaks at an energetic pace, keeping up with the fast music.
Accent: Jaz is from Brixton, London

#### TRANSCRIPT
Yes, massive vibes in the studio! You are locked in...
```

**Director's Notes elements:**

| Element | Description | Examples |
|---------|-------------|---------|
| **Style** | Tone and emotional quality | "Frustrated and angry developer", "Sassy GenZ beauty YouTuber", "Vocal Smile with raised soft palate" |
| **Accent** | Regional accent specificity | "Southern California valley girl from Laguna Beach", "British English as heard in Croydon, England" |
| **Pacing** | Speed and rhythm | "Speak as fast as possible", "The Drift: incredibly slow and liquid" |

### Audio Tags

Inline modifiers wrapped in square brackets:

| Category | Examples |
|----------|---------|
| Emotions | `[excitedly]`, `[bored]`, `[amazed]`, `[curious]`, `[crying]`, `[panicked]`, `[sarcastic]`, `[serious]` |
| Delivery | `[whispers]`, `[shouting]`, `[tired]`, `[trembling]`, `[reluctantly]`, `[mischievously]` |
| Non-verbal | `[laughs]`, `[giggles]`, `[sighs]`, `[gasp]`, `[cough]` |
| Pace | `[very fast]`, `[very slow]`, `[sarcastically, one painfully slow word at a time]` |
| Creative | `[like a cartoon dog]`, `[like dracula]` |

### Two-Step: Generate Transcript Then TTS

```python
# Step 1: Generate transcript with a text model
transcript_interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input="Generate a short transcript around 100 words..."
)
transcript = transcript_interaction.output_text

# Step 2: Convert to speech with TTS model
tts_interaction = client.interactions.create(
    model="gemini-3.1-flash-tts-preview",
    input=transcript,
    response_format={"type": "audio"},
    generation_config={
        "speech_config": [
            {"speaker": "Dr. Anya", "voice": "Kore"},
            {"speaker": "Liam", "voice": "Puck"}
        ]
    }
)
```

### Limitations (gemini-3.1-flash-tts-preview)

| Constraint | Value |
|------------|-------|
| HTTP status on failure | `500` |
| Error type | `PROHIBITED_CONTENT` |

---

## 5. Audio Understanding — Analysis & Transcription

### Endpoint

| Endpoint | Method | Description |
|----------|--------|-------------|
| `POST /v1beta/interactions` | REST | Audio analysis and transcription |
| `client.interactions.create()` | SDK | Audio understanding |

### Main Concepts

- **Audio Input** — Provided via uploaded file URI (Files API for >20MB) or inline base64 data (<20MB total request). Supports YouTube URLs as video input.
- **Structured Output** — Use `response_format` (response schema) to get structured JSON with transcription, speaker diarization, timestamps, language detection, translation, and emotion detection.
- **Timestamps** — Reference specific audio sections using `MM:SS` format in prompts (e.g. "Provide a transcript from 02:30 to 03:29").
- **Token Counting** — `client.models.count_tokens()` to count tokens in an audio file before processing.

### Input Methods

**Upload audio file (>20MB):**
```python
uploaded_file = client.files.upload(file="path/to/sample.mp3")
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input=[
        {"type": "text", "text": "Describe this audio clip"},
        {"type": "audio", "uri": uploaded_file.uri, "mime_type": uploaded_file.mime_type}
    ]
)
```

**Inline audio data (<20MB):**
```python
with open('path/to/small-sample.mp3', 'rb') as f:
    audio_bytes = f.read()
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input=[
        {"type": "text", "text": "Describe this audio clip"},
        {"type": "audio", "data": base64.b64encode(audio_bytes).decode('utf-8'), "mime_type": "audio/mp3"}
    ]
)
```

**YouTube URL as input:**
```python
interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input=[
        {"type": "video", "uri": "https://www.youtube.com/watch?v=...", "mime_type": "video/mp4"},
        {"type": "text", "text": prompt}
    ]
)
```

### Structured Transcription with Diarization

```python
response_schema = {
    "type": "object",
    "properties": {
        "summary": {"type": "string"},
        "segments": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "speaker": {"type": "string"},
                    "timestamp": {"type": "string"},
                    "content": {"type": "string"},
                    "language": {"type": "string"},
                    "emotion": {"type": "string", "enum": ["happy", "sad", "angry", "neutral"]}
                },
                "required": ["speaker", "timestamp", "content", "emotion"]
            }
        }
    },
    "required": ["summary", "segments"]
}

interaction = client.interactions.create(
    model="gemini-3.5-flash",
    input=[
        {"type": "video", "uri": YOUTUBE_URL, "mime_type": "video/mp4"},
        {"type": "text", "text": prompt}
    ],
    response_format=response_schema
)
```

### Structured Output Schema Fields

| Field | Type | Description |
|-------|------|-------------|
| `summary` | string | Brief summary at the beginning |
| `segments[]` | array | Array of speech segments |
| `segments[].speaker` | string | Speaker identification (e.g. "Speaker 1", "Speaker 2") |
| `segments[].timestamp` | string | Timestamp in MM:SS format |
| `segments[].content` | string | Transcribed speech content |
| `segments[].language` | string | Detected language of segment |
| `segments[].emotion` | enum | `happy` / `sad` / `angry` / `neutral` |

### Supported Audio Formats

| Format | MIME Type |
|--------|-----------|
| WAV | `audio/wav` |
| MP3 | `audio/mp3` |
| AIFF | `audio/aiff` |
| AAC | `audio/aac` |
| OGG | `audio/ogg` |
| FLAC | `audio/flac` |

### Constraints

| Constraint | Value |
|------------|-------|
| Max inline request size | 20 MB (including prompts and all files) |
| Large files | Use Files API to upload first |
| Timestamps format | `MM:SS` |
| Dedicated STT | Use [Google Cloud Speech-to-Text API](https://cloud.google.com/speech-to-text) for real-time transcription |

---

## 6. Live API — Real-Time Conversational Voice

### Endpoints

| Endpoint | Protocol | Description |
|----------|----------|-------------|
| `wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent?key=API_KEY` | WebSocket (v1beta) | Standard Live API |
| `wss://...v1alpha.GenerativeService.BidiGenerateContentConstrained?access_token=TOKEN` | WebSocket (v1alpha) | Ephemeral token / constrained |
| `client.aio.live.connect()` | SDK (Python) | High-level async interface |
| `ai.live.connect()` | SDK (JS) | High-level callback interface |

### Main Concepts

- **Stateful WebSocket** — A persistent, bidirectional connection. Configuration set in the first message (`BidiGenerateContentSetup`) applies for the session duration. Cannot be updated while connection is open (except via session resumption).
- **BidiGenerateContent** — The core RPC. Client sends exactly one of: `setup`, `clientContent`, `realtimeInput`, or `toolResponse`. Server responds with exactly one of: `setupComplete`, `serverContent`, `toolCall`, `toolCallCancellation`, `goAway`, or `sessionResumptionUpdate` (plus optional `usageMetadata`).
- **Setup Message** — First message containing: model, generationConfig, systemInstruction, tools, realtimeInputConfig, sessionResumption, contextWindowCompression, inputAudioTranscription, outputAudioTranscription, proactivity, historyConfig.
- **Realtime Input vs Client Content** — `realtimeInput` is for continuous streaming (audio/video/text), optimized for responsiveness, no guaranteed ordering, end-of-turn derived from VAD. `clientContent` is for incremental conversation updates, appended to history in order, interrupts current generation.

### Session Configuration (BidiGenerateContentSetup)

```json
{
  "model": "models/gemini-3.1-flash-live-preview",
  "generationConfig": {
    "candidateCount": 1,
    "maxOutputTokens": 1024,
    "temperature": 0.7,
    "topP": 0.9,
    "topK": 40,
    "presencePenalty": 0.0,
    "frequencyPenalty": 0.0,
    "responseModalities": ["AUDIO"],
    "speechConfig": {
      "voiceConfig": {
        "prebuiltVoiceConfig": {"voiceName": "Kore"}
      }
    },
    "mediaResolution": "MEDIA_RESOLUTION_LOW"
  },
  "systemInstruction": {"parts": [{"text": "You are a helpful assistant."}]},
  "tools": [],
  "realtimeInputConfig": {
    "automaticActivityDetection": {
      "disabled": false,
      "startOfSpeechSensitivity": "START_SENSITIVITY_HIGH",
      "endOfSpeechSensitivity": "END_SENSITIVITY_HIGH",
      "prefixPaddingMs": 20,
      "silenceDurationMs": 100
    },
    "activityHandling": "START_OF_ACTIVITY_INTERRUPTS",
    "turnCoverage": "TURN_INCLUDES_AUDIO_ACTIVITY_AND_ALL_VIDEO"
  },
  "sessionResumption": {"handle": "previous_handle"},
  "contextWindowCompression": {
    "slidingWindow": {"targetTokens": 5000},
    "triggerTokens": 8000
  },
  "inputAudioTranscription": {},
  "outputAudioTranscription": {},
  "proactivity": {"proactiveAudio": true},
  "historyConfig": {"initialHistoryInClientContent": true}
}
```

### BidiGenerateContentSetup Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | **Yes** | Model resource name: `models/{model}` |
| `generationConfig` | GenerationConfig | No | Generation parameters (see below) |
| `systemInstruction` | Content | No | System instructions (text only in parts) |
| `tools[]` | Tool | No | Function declarations, Google Search |
| `realtimeInputConfig` | RealtimeInputConfig | No | VAD and activity handling config |
| `sessionResumption` | SessionResumptionConfig | No | Session resumption via handle |
| `contextWindowCompression` | ContextWindowCompressionConfig | No | Sliding window compression |
| `inputAudioTranscription` | AudioTranscriptionConfig | No | Enable input audio transcription |
| `outputAudioTranscription` | AudioTranscriptionConfig | No | Enable output audio transcription |
| `proactivity` | ProactivityConfig | No | Proactive audio (v1alpha, 2.5 only) |
| `historyConfig` | HistoryConfig | No | Initial history exchange config |

### GenerationConfig Fields (Live API)

| Field | Type | Supported | Description |
|-------|------|-----------|-------------|
| `responseModalities` | string[] | Yes | `["AUDIO"]` for voice (required for native audio) |
| `speechConfig` | object | Yes | Voice configuration |
| `temperature` | number | Yes | Sampling temperature |
| `topP` | number | Yes | Nucleus sampling |
| `topK` | integer | Yes | Top-k sampling |
| `maxOutputTokens` | integer | Yes | Max output tokens |
| `candidateCount` | integer | Yes | Number of candidates |
| `presencePenalty` | number | Yes | Presence penalty |
| `frequencyPenalty` | number | Yes | Frequency penalty |
| `mediaResolution` | enum | Yes | Input media resolution |
| `responseLogprobs` | — | **Not supported** | |
| `responseMimeType` | — | **Not supported** | |
| `logprobs` | — | **Not supported** | |
| `responseSchema` | — | **Not supported** | |
| `stopSequence` | — | **Not supported** | |
| `routingConfig` | — | **Not supported** | |
| `audioTimestamp` | — | **Not supported** | |

### Speech Config (Live API)

```json
{
  "speechConfig": {
    "voiceConfig": {
      "prebuiltVoiceConfig": {"voiceName": "Kore"}
    }
  }
}
```

### Connecting (SDK)

**Python:**
```python
client = genai.Client()
model = "gemini-3.1-flash-live-preview"
config = {"response_modalities": ["AUDIO"]}
async with client.aio.live.connect(model=model, config=config) as session:
    print("Session started")
```

**JavaScript:**
```javascript
const ai = new GoogleGenAI({});
const session = await ai.live.connect({
    model: 'gemini-3.1-flash-live-preview',
    config: { responseModalities: [Modality.AUDIO] },
    callbacks: {
        onopen: () => console.debug('Opened'),
        onmessage: (message) => console.debug(message),
        onerror: (e) => console.debug('Error:', e.message),
        onclose: (e) => console.debug('Close:', e.reason),
    }
});
```

### Sending Input

**Send audio:**
```python
await session.send_realtime_input(
    audio=types.Blob(data=chunk, mime_type="audio/pcm;rate=16000")
)
```

**Send text:**
```python
await session.send_realtime_input(text="Hello, how are you?")
```

**Send video (max 1 FPS):**
```python
await session.send_realtime_input(
    video=types.Blob(data=frame, mime_type="image/jpeg")
)
```

**Send client content (turn-by-turn):**
```python
await session.send_client_content(
    turns=[{"role": "user", "parts": [{"text": "What is the capital of France?"}]}],
    turn_complete=True
)
```

### Receiving Responses

```python
async for response in session.receive():
    if response.server_content:
        content = response.server_content
        # Audio output
        if content.model_turn:
            for part in content.model_turn.parts:
                if part.inline_data:
                    audio_data = part.inline_data.data
        # Transcriptions
        if content.input_transcription:
            print(f"User: {content.input_transcription.text}")
        if content.output_transcription:
            print(f"Gemini: {content.output_transcription.text}")
        # Turn status
        if content.turn_complete:
            print("Turn complete")
        if content.interrupted:
            print("Generation interrupted by user")
        if content.generation_complete:
            print("Generation complete")
    # Tool calls
    if response.tool_call:
        for fc in response.tool_call.function_calls:
            result = my_function(**fc.args)
    # Session resumption
    if response.session_resumption_update:
        update = response.session_resumption_update
        if update.resumable and update.new_handle:
            handle = update.new_handle
    # GoAway
    if response.go_away:
        print(f"Time left: {response.go_away.time_left}")
```

### BidiGenerateContentServerContent Fields

| Field | Type | Description |
|-------|------|-------------|
| `modelTurn` | Content | Model-generated content (audio parts with `inlineData`) |
| `inputTranscription` | Transcription | Transcription of user's audio input |
| `outputTranscription` | Transcription | Transcription of model's audio output |
| `turnComplete` | bool | Model has completed its turn |
| `generationComplete` | bool | Model finished generating (may precede turnComplete) |
| `interrupted` | bool | User interrupted model generation |
| `groundingMetadata` | GroundingMetadata | Grounding info (Google Search) |
| `urlContextMetadata` | UrlContextMetadata | URL context metadata |

### Incremental Content Updates

For Gemini 3.1, `send_client_content` is only for seeding initial context history (requires `historyConfig.initialHistoryInClientContent = true`). During conversation, use `send_realtime_input` for text updates.

For Gemini 2.5, `send_client_content` is supported throughout for incremental updates and context establishment.

---

## 7. Live API — Session Management

### Session Lifetime

| Session Type | Limit (without compression) | With Compression |
|-------------|---------------------------|------------------|
| Audio-only | 15 minutes | Unlimited |
| Audio + Video | 2 minutes | Unlimited |
| Connection lifetime | ~10 minutes | Use session resumption |

### Context Window Compression

Extends sessions by discarding older context via a sliding window:

```python
config = types.LiveConnectConfig(
    response_modalities=["AUDIO"],
    context_window_compression=types.ContextWindowCompressionConfig(
        sliding_window=types.SlidingWindow(),
    )
)
```

**ContextWindowCompressionConfig fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `slidingWindow` | SlidingWindow | — | Sliding window mechanism |
| `triggerTokens` | int64 | 80% of model's context limit | Token threshold to trigger compression |

**SlidingWindow fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `targetTokens` | int64 | `triggerTokens / 2` | Target tokens to keep after compression. Context always begins at start of a USER turn. System instructions and prefix turns are preserved. |

### Session Resumption

Resumes a session across WebSocket reconnections:

```python
async with client.aio.live.connect(
    model=model,
    config=types.LiveConnectConfig(
        response_modalities=["AUDIO"],
        session_resumption=types.SessionResumptionConfig(
            handle=previous_session_handle  # or None for new session
        )
    )
) as session:
    async for message in session.receive():
        if message.session_resumption_update:
            update = message.session_resumption_update
            if update.resumable and update.new_handle:
                new_handle = update.new_handle  # Save for next reconnection
```

**SessionResumptionConfig fields:**

| Field | Type | Description |
|-------|------|-------------|
| `handle` | string | Handle from previous session's `SessionResumptionUpdate.newHandle`. Empty for new session. |

**SessionResumptionUpdate fields:**

| Field | Type | Description |
|-------|------|-------------|
| `newHandle` | string | New resumable handle. Empty if `resumable` is false. |
| `resumable` | bool | Whether session can be resumed at this point. False during function execution or generation. |

**Resumption token validity:** 2 hours after last session termination.

### GoAway Message

Server sends `GoAway` before connection termination:

| Field | Type | Description |
|-------|------|-------------|
| `timeLeft` | Duration | Remaining time before ABORTED termination. Never less than model-specific minimum. |

### Generation Complete

`generationComplete` signals the model finished generating. With realtime playback, there's a delay between `generationComplete` and `turnComplete` while the model waits for playback to finish.

---

## 8. Live API — Voice Activity Detection (VAD)

### Automatic VAD (Default)

The model automatically detects speech start/end on a continuous audio stream. When VAD detects an interruption, ongoing generation is cancelled and discarded. Only already-sent content is retained in session history.

**Configuration:**

```python
config = {
    "response_modalities": ["AUDIO"],
    "realtime_input_config": {
        "automatic_activity_detection": {
            "disabled": False,  # default
            "start_of_speech_sensitivity": "START_SENSITIVITY_LOW",
            "end_of_speech_sensitivity": "END_SENSITIVITY_LOW",
            "prefix_padding_ms": 20,
            "silence_duration_ms": 100
        }
    }
}
```

### AutomaticActivityDetection Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `disabled` | bool | false | If false (default), voice/text input counts as activity. If true, client must send activity signals manually. |
| `startOfSpeechSensitivity` | StartSensitivity | `START_SENSITIVITY_HIGH` | How likely speech start is detected |
| `prefixPaddingMs` | int32 | — | Duration of detected speech before start-of-speech is committed. Lower = more sensitive, more false positives. |
| `endOfSpeechSensitivity` | EndSensitivity | `END_SENSITIVITY_HIGH` | How likely speech end is detected |
| `silenceDurationMs` | int32 | — | Duration of non-speech before end-of-speech is committed. Larger = longer gaps allowed, more latency. |

### Sensitivity Enums

**StartSensitivity:**

| Value | Description |
|-------|-------------|
| `START_SENSITIVITY_UNSPECIFIED` | Default: `START_SENSITIVITY_HIGH` |
| `START_SENSITIVITY_HIGH` | Detects speech start more often |
| `START_SENSITIVITY_LOW` | Detects speech start less often |

**EndSensitivity:**

| Value | Description |
|-------|-------------|
| `END_SENSITIVITY_UNSPECIFIED` | Default: `END_SENSITIVITY_HIGH` |
| `END_SENSITIVITY_HIGH` | Ends speech more often |
| `END_SENSITIVITY_LOW` | Ends speech less often |

### Activity Handling

Controls what happens when user activity is detected during model generation:

| Value | Description |
|-------|-------------|
| `ACTIVITY_HANDLING_UNSPECIFIED` | Default: `START_OF_ACTIVITY_INTERRUPTS` |
| `START_OF_ACTIVITY_INTERRUPTS` | User activity interrupts model response (barge-in). Model's current response is cut off. |
| `NO_INTERRUPTION` | Model's response continues without interruption. |

### Turn Coverage

Defines which input is included in the user's turn:

| Value | Description |
|-------|-------------|
| `TURN_COVERAGE_UNSPECIFIED` | Model-dependent default |
| `TURN_INCLUDES_ONLY_ACTIVITY` | Activity since last turn, excluding silence |
| `TURN_INCLUDES_ALL_INPUT` | All realtime input since last turn, including silence |
| `TURN_INCLUDES_AUDIO_ACTIVITY_AND_ALL_VIDEO` | Audio activity + all video frames (Gemini 3.1 default) |

### Manual VAD (Custom)

Disable automatic VAD and send activity signals manually:

```python
config = {
    "response_modalities": ["AUDIO"],
    "realtime_input_config": {
        "automatic_activity_detection": {"disabled": True}
    }
}

# Send activity signals manually
await session.send_realtime_input(activity_start=types.ActivityStart())
await session.send_realtime_input(
    audio=types.Blob(data=audio_bytes, mime_type="audio/pcm;rate=16000")
)
await session.send_realtime_input(activity_end=types.ActivityEnd())
```

**Manual VAD best practices:**
- Server's audio buffering is bypassed
- Use end-of-speech silence threshold of **at least 500ms** in client-side VAD
- Thresholds below 500ms cause fragmented audio that degrades quality
- `audioStreamEnd` is not sent in manual mode; stream interruption marked by `activityEnd`

### Audio Stream End

When the audio stream is paused (e.g. microphone off) for more than a second with automatic VAD enabled, send `audioStreamEnd` to flush cached audio:

```python
await session.send_realtime_input(audio_stream_end=True)
```

Client can resume sending audio at any time.

---

## 9. Live API — Tool Use & Function Calling

### Supported Tools

| Tool | Gemini 3.1 Flash Live | Gemini 2.5 Flash Live |
|------|-----------------------|-----------------------|
| Google Search | Supported | Supported |
| Function calling | Synchronous only | Synchronous + asynchronous |
| Google Maps | Not supported | Not supported |
| Code execution | Not supported | Not supported |
| URL context | Not supported | Not supported |

### Function Calling

Define function declarations in session config:

```python
turn_on_the_lights = {"name": "turn_on_the_lights"}
tools = [{"function_declarations": [turn_on_the_lights]}]
config = {"response_modalities": ["AUDIO"], "tools": tools}
```

**Handle tool calls:**

```python
async for response in session.receive():
    if response.tool_call:
        function_responses = []
        for fc in response.tool_call.function_calls:
            result = my_tool_function(**fc.args)
            function_responses.append(types.FunctionResponse(
                name=fc.name, id=fc.id, response={"result": result}
            ))
        await session.send_tool_response(function_responses=function_responses)
```

**Tool call cancellation:** When user interrupts, server sends `toolCallCancellation` with IDs of cancelled calls. Undo side effects if necessary.

### Asynchronous Function Calling (Gemini 2.5 Only)

Non-blocking function calling lets the model continue interacting while functions execute:

```python
# Define non-blocking function
turn_on_the_lights = {"name": "turn_on_the_lights", "behavior": "NON_BLOCKING"}
```

**Scheduling** (in FunctionResponse) controls how the model handles results:

| Scheduling | Description |
|------------|-------------|
| `INTERRUPT` | Immediately interrupt model to deliver the response |
| `WHEN_IDLE` | Deliver response when model is idle |
| `SILENT` | Use knowledge later, no immediate delivery |

```python
function_response = types.FunctionResponse(
    id=fc.id, name=fc.name,
    response={"result": "ok", "scheduling": "INTERRUPT"}
)
```

### Google Search Grounding

```python
tools = [{'google_search': {}}]
config = {"response_modalities": ["AUDIO"], "tools": tools}
```

The model may generate and execute Python code to use Search. Results include `groundingMetadata` in server content.

### Combining Multiple Tools

```python
tools = [
    {"google_search": {}},
    {"function_declarations": [turn_on_the_lights, turn_off_the_lights]}
]
config = {"response_modalities": ["AUDIO"], "tools": tools}
```

---

## 10. Live API — Native Audio Capabilities

### Native Audio Output

Gemini 3.1 and 2.5 Flash Live models feature native audio output — natural, realistic speech with improved multilingual performance. Only `AUDIO` response modality is supported. For text output, use `output_audio_transcription`.

### Thinking Control

**Gemini 3.1 (thinkingLevel):**

```python
config = types.LiveConnectConfig(
    response_modalities=["AUDIO"],
    thinking_config=types.ThinkingConfig(
        thinking_level="low",  # minimal | low | medium | high
        include_thoughts=True  # Optional: receive thought summaries
    )
)
```

| Level | Description |
|-------|-------------|
| `minimal` | Default — lowest latency, minimal reasoning |
| `low` | Light reasoning |
| `medium` | Moderate reasoning |
| `high` | Deep reasoning |

**Gemini 2.5 (thinkingBudget):**

```python
config = {
    "responseModalities": ["AUDIO"],
    "thinkingConfig": {"thinkingBudget": 0}  # 0 to disable, N tokens otherwise
}
```

### Affective Dialog (Gemini 2.5 Only, v1alpha)

Model adapts response style to match input expression and tone:

```python
client = genai.Client(http_options={"api_version": "v1alpha"})
config = types.LiveConnectConfig(
    response_modalities=["AUDIO"],
    enable_affective_dialog=True
)
```

### Proactive Audio (Gemini 2.5 Only, v1alpha)

Model can decide not to respond if input is not relevant:

```python
client = genai.Client(http_options={"api_version": "v1alpha"})
config = types.LiveConnectConfig(
    response_modalities=["AUDIO"],
    proactivity={'proactive_audio': True}
)
```

### Audio Transcriptions

Enable transcription of input and/or output audio:

```python
config = {
    "response_modalities": ["AUDIO"],
    "input_audio_transcription": {},   # Transcribe user's audio input
    "output_audio_transcription": {}   # Transcribe model's audio output
}
```

Transcriptions arrive in `serverContent.inputTranscription.text` and `serverContent.outputTranscription.text`, independently of other server messages (no guaranteed ordering).

### Media Resolution

```python
config = {
    "response_modalities": ["AUDIO"],
    "media_resolution": types.MediaResolution.MEDIA_RESOLUTION_LOW
}
```

### Token Counting

Usage metadata in server messages:

```python
if message.usage_metadata:
    usage = message.usage_metadata
    print(f"Total tokens: {usage.total_token_count}")
    for detail in usage.response_tokens_details:
        print(f"{detail.modality}: {detail.token_count}")
```

**UsageMetadata fields:**

| Field | Type | Description |
|-------|------|-------------|
| `promptTokenCount` | int32 | Tokens in prompt (including cached content) |
| `cachedContentTokenCount` | int32 | Tokens in cached content |
| `responseTokenCount` | int32 | Total tokens across all response candidates |
| `toolUsePromptTokenCount` | int32 | Tokens in tool-use prompts |
| `thoughtsTokenCount` | int32 | Thinking tokens (for thinking models) |
| `totalTokenCount` | int32 | Total (prompt + response) |
| `promptTokensDetails[]` | ModalityTokenCount[] | Input modality token breakdown |
| `responseTokensDetails[]` | ModalityTokenCount[] | Output modality token breakdown |

### Limitations

| Constraint | Value |
|------------|-------|
| Response modalities | `AUDIO` only for native audio models |
| Client authentication | Server-to-server by default; ephemeral tokens for client-to-server |
| Audio-only session | 15 min (without compression) |
| Audio + video session | 2 min (without compression) |
| Connection lifetime | ~10 min (use session resumption) |

---

## 11. Live Translation — Real-Time Speech-to-Speech Translation

### Overview

The Gemini Live API supports low-latency, real-time speech-to-speech translation between 70+ languages using the `gemini-3.5-live-translate-preview` model. Stream audio in one language and receive translated audio output in another language.

### Live Agent vs. Live Translation

| Aspect | Live Agent | Live Translation |
|--------|-----------|-----------------|
| **Role** | Acts as an assistant — listens, reasons, takes actions | Acts as an interpreter — real-time translator pipeline |
| **Interaction model** | Turn-based: pauses, intent detection, interruptions | Continuous stream processing: translates as speaker talks |
| **Tools** | Function calling, Google Search, instructions | Translation only — no tools or instructions |
| **Input** | Fully multimodal: text, audio, video, image | Audio only — restricted for strict real-time latency |
| **Configuration** | Granular: generation, speech, tools, system instructions | Simplified: `target_language_code` + `echo_target_language` |

### Model

| Model | Description |
|-------|-------------|
| `gemini-3.5-live-translate-preview` | Dedicated real-time speech-to-speech translation, 70+ languages |

### Endpoint

Same WebSocket endpoint as Live API:
```
wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent?key=API_KEY
```

### Configuration

```python
config = types.LiveConnectConfig(
    response_modalities=["AUDIO"],
    input_audio_transcription=types.AudioTranscriptionConfig(),
    output_audio_transcription=types.AudioTranscriptionConfig(),
    translation_config=types.TranslationConfig(
        target_language_code="pl",
        echo_target_language=True
    )
)
```

### TranslationConfig Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `targetLanguageCode` | string | **Yes** | — | BCP-47 language code (e.g. `"pl"`, `"es"`, `"en"`) |
| `echoTargetLanguage` | bool | No | false | If true, echoes target language audio back. If false (default), does not echo. |

### Setup Message (WebSocket)

```json
{
  "setup": {
    "model": "models/gemini-3.5-live-translate-preview",
    "generationConfig": {
      "responseModalities": ["AUDIO"],
      "inputAudioTranscription": {},
      "outputAudioTranscription": {},
      "translationConfig": {
        "targetLanguageCode": "pl",
        "echoTargetLanguage": true
      }
    }
  }
}
```

### Sending Audio

Same as Live API — raw 16-bit PCM, 16kHz, little-endian:

```python
await session.send_realtime_input(
    audio=types.Blob(data=chunk, mime_type="audio/pcm;rate=16000")
)
```

### Receiving Translated Audio and Transcriptions

```python
async for response in session.receive():
    if response.server_content:
        if response.server_content.input_transcription:
            print(f"Input transcript: {response.server_content.input_transcription.text}")
        if response.server_content.output_transcription:
            print(f"Output transcript: {response.server_content.output_transcription.text}")
        if response.server_content.model_turn:
            for part in response.server_content.model_turn.parts:
                if part.inline_data:
                    audio_data = part.inline_data.data
                    # Translated audio chunk (base64 PCM, 24kHz)
```

### Ephemeral Tokens for Live Translation

Constrain ephemeral tokens to lock translation configuration (v1alpha):

```python
token = client.auth_tokens.create(
    config={
        'uses': 1,
        'expire_time': now + datetime.timedelta(minutes=30),
        'live_connect_constraints': {
            'model': 'gemini-3.5-live-translate-preview',
            'config': {
                'translation_config': {
                    'target_language_code': 'pl',
                    'echo_target_language': True
                }
            }
        },
        'http_options': {'api_version': 'v1alpha'},
    }
)
```

### Supported Languages (Live Translation)

| Language | BCP-47 | Language | BCP-47 |
|----------|--------|----------|--------|
| Afrikaans | `af` | Kazakh | `kk` |
| Akan | `ak` | Khmer | `km` |
| Albanian | `sq` | Kinyarwanda | `rw` |
| Amharic | `am` | Korean | `ko` |
| Arabic | `ar` | Lao | `lo` |
| Armenian | `hy` | Latvian | `lv` |
| Azerbaijani | `az` | Lithuanian | `lt` |
| Basque | `eu` | Macedonian | `mk` |
| Belarusian | `be` | Malay | `ms` |
| Bengali | `bn` | Malayalam | `ml` |
| Bulgarian | `bg` | Marathi | `mr` |
| Burmese (Myanmar) | `my` | Mongolian | `mn` |
| Catalan | `ca` | Nepali | `ne` |
| Chinese (Simplified) | `zh-Hans` | Norwegian | `no` / `nb` |
| Chinese (Traditional) | `zh-Hant` | Persian | `fa` |
| Croatian | `hr` | Polish | `pl` |
| Czech | `cs` | Portuguese (Brazil) | `pt-BR` |
| Danish | `da` | Portuguese (Portugal) | `pt-PT` |
| Dutch | `nl` | Punjabi | `pa` |
| English | `en` | Romanian | `ro` |
| Estonian | `et` | Russian | `ru` |
| Filipino | `fil` | Serbian | `sr` |
| Finnish | `fi` | Sindhi | `sd` |
| French | `fr` | Sinhala | `si` |
| Galician | `gl` | Slovak | `sk` |
| Georgian | `ka` | Slovenian | `sl` |
| German | `de` | Spanish | `es` |
| Greek | `el` | Sundanese | `su` |
| Gujarati | `gu` | Swahili | `sw` |
| Hausa | `ha` | Swedish | `sv` |
| Hebrew | `he` | Tamil | `ta` |
| Hindi | `hi` | Telugu | `te` |
| Hungarian | `hu` | Thai | `th` |
| Icelandic | `is` | Turkish | `tr` |
| Indonesian | `id` | Ukrainian | `uk` |
| Italian | `it` | Urdu | `ur` |
| Japanese | `ja` | Uzbek | `uz` |
| Javanese | `jv` | Vietnamese | `vi` |
| Kannada | `kn` | Zulu | `zu` |

---

## 12. Voice Catalog & Supported Languages

### Prebuilt Voices (30 voices, shared across TTS and Live API)

| Voice | Personality | Voice | Personality | Voice | Personality |
|-------|------------|-------|------------|-------|------------|
| **Zephyr** | Bright | **Puck** | Upbeat | **Charon** | Informative |
| **Kore** | Firm | **Fenrir** | Excitable | **Leda** | Youthful |
| **Orus** | Firm | **Aoede** | Breezy | **Callirrhoe** | Easy-going |
| **Autonoe** | Bright | **Enceladus** | Breathy | **Iapetus** | Clear |
| **Umbriel** | Easy-going | **Algieba** | Smooth | **Despina** | Smooth |
| **Erinome** | Clear | **Algenib** | Gravelly | **Rasalgethi** | Informative |
| **Laomedeia** | Upbeat | **Achernar** | Soft | **Alnilam** | Firm |
| **Schedar** | Even | **Gacrux** | Mature | **Pulcherrima** | Forward |
| **Achird** | Friendly | **Zubenelgenubi** | Casual | **Vindemiatrix** | Gentle |
| **Sadachbia** | Lively | **Sadaltager** | Knowledgeable | **Sulafat** | Warm |

All voices can be previewed in [AI Studio](https://aistudio.google.com/app/live).

### TTS Supported Languages (95+ languages, auto-detected)

| Language | Code | Language | Code | Language | Code |
|----------|------|----------|------|----------|------|
| Arabic | `ar` | Filipino | `fil` | Finnish | `fi` |
| Dutch | `nl` | Galician | `gl` | Georgian | `ka` |
| English | `en` | Greek | `el` | Gujarati | `gu` |
| French | `fr` | Haitian Creole | `ht` | Hebrew | `he` |
| German | `de` | Hungarian | `hu` | Hindi | `hi` |
| Indonesian | `id` | Icelandic | `is` | Japanese | `ja` |
| Korean | `ko` | Javanese | `jv` | Marathi | `mr` |
| Polish | `pl` | Kannada | `kn` | Konkani | `kok` |
| Portuguese | `pt` | Lao | `lo` | Latin | `la` |
| Romanian | `ro` | Latvian | `lv` | Russian | `ru` |
| Spanish | `es` | Lithuanian | `lt` | Luxembourgish | `lb` |
| Tamil | `ta` | Macedonian | `mk` | Maithili | `mai` |
| Telugu | `te` | Malagasy | `mg` | Thai | `th` |
| Turkish | `tr` | Malay | `ms` | Malayalam | `ml` |
| Ukrainian | `uk` | Mongolian | `mn` | Nepali | `ne` |
| Vietnamese | `vi` | Norwegian (Bokmål) | `nb` | Norwegian (Nynorsk) | `nn` |
| Afrikaans | `af` | Odia | `or` | Albanian | `sq` |
| Amharic | `am` | Pashto | `ps` | Armenian | `hy` |
| Azerbaijani | `az` | Persian | `fa` | Basque | `eu` |
| Belarusian | `be` | Punjabi | `pa` | Bulgarian | `bg` |
| Burmese | `my` | Serbian | `sr` | Catalan | `ca` |
| Sindhi | `sd` | Sinhala | `si` | Chinese (Mandarin) | `cmn` |
| Slovak | `sk` | Cebuano | `ceb` | Croatian | `hr` |
| Slovenian | `sl` | Swahili | `sw` | Czech | `cs` |
| Swedish | `sv` | Danish | `da` | Estonian | `et` |
| Urdu | `ur` | | | | |

### Live API Supported Languages (97 languages)

Same as TTS plus additional languages: Assamese (`as`), Bosnian (`bs`), Faroese (`fo`), Irish (`ga`), Kurdish (`ku`), Kyrgyz (`ky`), Maltese (`mt`), Maori (`mi`), Oromo (`om`), Quechua (`qu`), Romansh (`rm`), Somali (`so`), Southern Sotho (`st`), Tajik (`tg``, Tswana (`tn`), Turkmen (`tk`), Welsh (`cy`), Western Frisian (`fy`), Wolof (`wo`), Yoruba (`yo`).

---

## 13. Audio Formats & Technical Specifications

### Live API Technical Specifications

| Category | Details |
|----------|---------|
| **Input modalities** | Audio (raw 16-bit PCM, 16kHz, little-endian), images (JPEG/PNG, max 1 FPS), text |
| **Output modalities** | Audio (raw 16-bit PCM, 24kHz, little-endian) |
| **Protocol** | Stateful WebSocket connection (WSS) |
| **Input audio** | Natively 16kHz; API resamples any sample rate |
| **Output audio** | Always 24kHz |
| **Input audio MIME** | `audio/pcm;rate=16000` |
| **Video MIME** | `image/jpeg` or `image/png` |
| **Audio encoding** | Base64 for WebSocket transport |

### TTS Audio Output

| Property | Value |
|----------|-------|
| Format | Raw 16-bit PCM |
| Sample rate | 24,000 Hz |
| Channels | 1 (mono) |
| Sample width | 2 bytes (16-bit) |
| Encoding | Little-endian |
| Transport | Base64-encoded in JSON response |

**WAV file creation:**
```python
def wave_file(filename, pcm, channels=1, rate=24000, sample_width=2):
    with wave.open(filename, "wb") as wf:
        wf.setnchannels(channels)
        wf.setsampwidth(sample_width)
        wf.setframerate(rate)
        wf.writeframes(pcm)
```

### Audio Understanding Input Formats

| Format | MIME Type | Max Inline Size |
|--------|-----------|----------------|
| WAV | `audio/wav` | 20 MB total request |
| MP3 | `audio/mp3` | 20 MB total request |
| AIFF | `audio/aiff` | 20 MB total request |
| AAC | `audio/aac` | 20 MB total request |
| OGG | `audio/ogg` | 20 MB total request |
| FLAC | `audio/flac` | 20 MB total request |

Files larger than 20 MB must be uploaded via the Files API first.

### Live API Session Limits

| Session Type | Duration (no compression) | With Compression |
|-------------|---------------------------|------------------|
| Audio-only | 15 minutes | Unlimited |
| Audio + video | 2 minutes | Unlimited |
| Connection | ~10 minutes | Session resumption |
| Context window | Model-specific | Sliding window compression |

---

## 14. Partner Integrations

Third-party integrations that support the Gemini Live API over WebRTC or WebSockets:

| Partner | Description |
|---------|-------------|
| **LiveKit** | Use Gemini Live API with LiveKit Agents |
| **Pipecat by Daily** | Real-time AI chatbot using Gemini Live and Pipecat |
| **Fishjam by Software Mansion** | Live video and audio streaming applications |
| **Vision Agents by Stream** | Real-time voice and video AI applications |
| **Voximplant** | Connect inbound/outbound calls to Live API |
| **Agora** | Real-time conversational AI applications |
| **Firebase AI SDK** | Gemini Live API using Firebase AI Logic |

---

## 15. Capability Summary & Cross-Reference

### Complete API Endpoint Map

| Capability | API | Endpoint / Method | Protocol |
|-----------|-----|-------------------|----------|
| **Text-to-Speech (single)** | Interactions | `POST /v1beta/interactions` | REST / SDK |
| **Text-to-Speech (multi-speaker)** | Interactions | `POST /v1beta/interactions` | REST / SDK |
| **TTS Streaming** | Interactions | `POST /v1beta/interactions` with `stream: true` | REST / SDK |
| **Audio Understanding** | Interactions | `POST /v1beta/interactions` | REST / SDK |
| **Structured Transcription** | Interactions | `POST /v1beta/interactions` with `response_format` schema | REST / SDK |
| **File Upload** | Files API | `client.files.upload()` | REST / SDK |
| **Token Counting** | Models | `client.models.count_tokens()` | REST / SDK |
| **Real-time Voice Agent** | Live API | `wss://...BidiGenerateContent` | WebSocket / SDK |
| **Real-time Voice + Video** | Live API | `wss://...BidiGenerateContent` | WebSocket / SDK |
| **Live Translation** | Live API | `wss://...BidiGenerateContent` | WebSocket / SDK |
| **Ephemeral Token Creation** | Auth | `client.auth_tokens.create()` (`v1alpha`) | REST / SDK |

### Capability Decision Matrix

| If you need... | Use... | Key Parameters |
|----------------|--------|---------------|
| Text → spoken audio (single voice) | Interactions API (TTS) | `model`, `input`, `response_format: {type: "audio"}`, `speech_config: [{voice}]` |
| Text → multi-speaker dialogue audio | Interactions API (TTS) | `speech_config: [{speaker, voice}, ...]` (max 2 speakers) |
| Control speech style/emotion/pace | Interactions API (TTS) | Director prompts + audio tags (`[whispers]`, `[excitedly]`) |
| Stream TTS audio as generated | Interactions API (TTS) | `stream: true`, receive `event.delta.data` |
| Generate transcript then narrate | Interactions + TTS | Step 1: `gemini-3.5-flash` for text; Step 2: TTS model for audio |
| Analyze/describe audio content | Interactions API (Audio) | `input: [{type: "audio", uri/data, mime_type}]` |
| Transcribe with diarization & emotion | Interactions API (Audio) | `response_format` schema with segments, speakers, emotions |
| Reference specific audio timestamps | Interactions API (Audio) | Prompt with `MM:SS` format |
| Real-time voice conversation | Live API | `responseModalities: ["AUDIO"]`, `send_realtime_input(audio)` |
| Real-time voice + video conversation | Live API | `send_realtime_input(audio, video)`, max 1 FPS video |
| Control voice in Live API | Live API | `speechConfig.voiceConfig.prebuiltVoiceConfig.voiceName` |
| Receive input/output transcriptions | Live API | `inputAudioTranscription: {}`, `outputAudioTranscription: {}` |
| Handle user interruptions (barge-in) | Live API | Automatic VAD (default) → `serverContent.interrupted` |
| Configure VAD sensitivity | Live API | `realtimeInputConfig.automaticActivityDetection` fields |
| Manual VAD control | Live API | `automaticActivityDetection.disabled: true`, send `activityStart`/`activityEnd` |
| Real-time translation (speech-to-speech) | Live Translate | `translationConfig: {targetLanguageCode, echoTargetLanguage}` |
| Function calling during conversation | Live API | `tools: [{functionDeclarations}]`, handle `toolCall` |
| Async function calling (non-blocking) | Live API (2.5 only) | `behavior: "NON_BLOCKING"`, `scheduling: "INTERRUPT"/"WHEN_IDLE"/"SILENT"` |
| Google Search grounding | Live API | `tools: [{google_search: {}}]` |
| Control thinking depth | Live API | 3.1: `thinkingLevel`; 2.5: `thinkingBudget` |
| Emotion-adaptive responses | Live API (2.5 only) | `enable_affective_dialog: true` (v1alpha) |
| Model decides not to respond | Live API (2.5 only) | `proactivity: {proactive_audio: true}` (v1alpha) |
| Extend session beyond time limits | Live API | `contextWindowCompression: {slidingWindow: {}}` |
| Resume session after reconnection | Live API | `sessionResumption: {handle}`, save `sessionResumptionUpdate.newHandle` |
| Secure client-side connection | Live API | Ephemeral tokens (`v1alpha`), `liveConnectConstraints` |
| Lock translation config for client | Live Translate | Ephemeral token with `live_connect_constraints.translation_config` |

### Streaming Architecture

| Mode | API | Description | Use Case |
|------|-----|-------------|----------|
| **Buffered** | Interactions (TTS) | Complete audio returned after generation | Pre-generated content |
| **HTTP Streaming** | Interactions (TTS) | `stream: true` — audio chunks as generated | Progressive playback |
| **WebSocket (bidirectional)** | Live API | Continuous bidirectional audio/video/text streaming | Real-time conversations, translation |

### Key Differences from Other Voice Platforms

| Feature | Google Gemini | ElevenLabs | Deepgram |
|---------|--------------|------------|---------|
| Voice cloning | Not available | IVC + PVC | Not available |
| Custom voice design | Not available | Text-to-voice design | Not available |
| Prebuilt voices | 30 | 10,000+ (library) | ~20 (Aura-2) |
| TTS prompting | Director-style (LLM-based) | Voice settings + audio tags (v3) | Rate, pitch, energy |
| Real-time conversation | Live API (WebSocket) | Speech Engine (WebSocket) | Voice Agent API (WebSocket) |
| Live translation | Dedicated model (70+ langs) | Dubbing (batch, 90+) | Not native |
| Audio understanding | Via Interactions API | Scribe (dedicated STT) | Nova-3 (dedicated STT) |
| Function calling in voice | Yes (sync + async on 2.5) | Via custom LLM | Via provider config |
| Multimodal (video) | Yes (Live API) | No | No |
| Session management | Resumption + compression | WebSocket lifecycle | WebSocket lifecycle |