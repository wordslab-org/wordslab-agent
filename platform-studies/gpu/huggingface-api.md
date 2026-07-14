# Hugging Face On-Demand Inference — API Capabilities Analysis

> Source: documentation reachable from https://huggingface.co/docs/inference-endpoints/index
> Scope: on-demand LLM inference services — both the serverless multi-provider API (Inference Providers) and the dedicated GPU rental service (Inference Endpoints). Core AI task APIs, infrastructure management, provider routing, and cross-cutting operational capabilities are covered.
> Generated: 2026-07-14

This document analyzes the capabilities exposed by Hugging Face for on-demand LLM inference. Hugging Face operates **two complementary services**:

- **Inference Providers** (serverless API) — a unified, multi-provider proxy layer that routes requests to third-party inference partners (Groq, Together, Cerebras, Novita, …) or to Hugging Face's own managed infrastructure. Pay-as-you-go per request. Base URL: `https://router.huggingface.co/v1`.
- **Inference Endpoints** (dedicated GPU) — a managed service that deploys a single model from the Hugging Face Hub onto a dedicated, autoscaling GPU/CPU instance. Billed per minute of compute. Base URL per endpoint: `https://<id>.<region>.<cloud>.endpoints.huggingface.cloud/`.

For each capability, this document lists the **main concepts**, the **API functions/endpoints**, and the **parameters** that govern behavior.

---

## Table of Contents

### Inference Providers (Serverless API)
1. [Provider Routing & Selection](#1-provider-routing--selection)
2. [Chat Completion (LLM)](#2-chat-completion-llm)
3. [Chat Completion (VLM — Vision-Language Models)](#3-chat-completion-vlm--vision-language-models)
4. [Text Generation](#4-text-generation)
5. [Feature Extraction (Embeddings)](#5-feature-extraction-embeddings)
6. [Text to Image](#6-text-to-image)
7. [Text to Video](#7-text-to-video)
8. [Image to Image](#8-image-to-image)
9. [Automatic Speech Recognition](#9-automatic-speech-recognition)
10. [Summarization](#10-summarization)
11. [Translation](#11-translation)
12. [Text Classification](#12-text-classification)
13. [Token Classification (NER)](#13-token-classification-ner)
14. [Question Answering](#14-question-answering)
15. [Fill-Mask](#15-fill-mask)
16. [Image Classification](#16-image-classification)
17. [Object Detection](#17-object-detection)
18. [Image Segmentation](#18-image-segmentation)
19. [Zero-Shot Classification](#19-zero-shot-classification)
20. [Table Question Answering](#20-table-question-answering)
21. [Audio Classification](#21-audio-classification)
22. [Function Calling / Tool Use](#22-function-calling--tool-use)
23. [Structured Output (JSON Schema)](#23-structured-output-json-schema)
24. [Responses API (Agentic)](#24-responses-api-agentic)
25. [Model Discovery & Hub API](#25-model-discovery--hub-api)
26. [Billing & Organization Cost Attribution](#26-billing--organization-cost-attribution)

### Inference Endpoints (Dedicated GPU)
27. [Endpoint Lifecycle Management](#27-endpoint-lifecycle-management)
28. [Hardware & Cloud Configuration](#28-hardware--cloud-configuration)
29. [Inference Engine Configuration](#29-inference-engine-configuration)
30. [Autoscaling](#30-autoscaling)
31. [Authentication & Access Control](#31-authentication--access-control)
32. [Network & PrivateLink](#32-network--privatelink)
33. [Observability (Analytics & Metrics)](#33-observability-analytics--metrics)
34. [Runtime Logs](#34-runtime-logs)
35. [Security & Compliance](#35-security--compliance)
36. [Pricing & Cost Calculation](#36-pricing--cost-calculation)
37. [Custom Containers](#37-custom-containers)
38. [Custom Inference Handlers (Toolkit)](#38-custom-inference-handlers-toolkit)

---

## Inference Providers (Serverless API)

### 1. Provider Routing & Selection

**Source pages:** `inference-providers/index`, `inference-providers/hub-api`

#### Main concepts

- **Inference Providers**: a unified proxy layer (the "router") sitting between the application and multiple third-party inference partners. Requests are authenticated with a single Hugging Face token; billing is centralized.
- **Provider**: a backend partner serving models — Cerebras, Cohere, DeepInfra, Fal AI, Featherless AI, Fireworks, Groq, HF Inference, Novita, Nscale, OVHcloud, Public AI, Replicate, Scaleway, Together, WaveSpeedAI, Z.ai. Each provider supports a subset of task types.
- **Provider selection policy**: a suffix appended to the model id controlling which provider handles the request:
  - `:fastest` (default) — highest throughput in tokens/second
  - `:cheapest` — lowest price per output token
  - `:preferred` — follows user-defined preference order in Inference Provider settings
  - `:<provider-name>` — forces a specific provider (e.g., `:groq`)
- **Routed requests vs. Custom provider key**: by default, requests route through HF and are billed by HF. Alternatively, a user can set a custom provider key so the provider bills directly (HF adds no markup in either case).
- **Automatic failover**: when using `provider="auto"`, if the primary provider is unavailable, the router falls back to alternative providers.

#### API functions

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Chat completion (OpenAI-compatible) | `POST https://router.huggingface.co/v1/chat/completions` | Server-side provider selection for chat tasks; model id carries the policy suffix |
| List models (OpenAI-compatible) | `GET https://router.huggingface.co/v1/models` | Returns all chat-completion models across providers with per-provider metadata (pricing, context length, latency, throughput, tool/structured-output support) |
| Get single model | `GET https://router.huggingface.co/v1/models/{model_id}` | Returns one model with all provider entries |
| Responses API | `POST https://router.huggingface.co/v1/responses` | OpenAI Responses-compatible endpoint for agentic workflows |

#### Key parameters (model id suffix)

- `model` (string, required) — Hub model id with optional `:policy` or `:provider` suffix
- `provider` (string, client-side) — set on `InferenceClient` to force a specific provider (`"auto"` by default)

#### Provider metadata fields (from `GET /v1/models`)

| Field | Type | Description |
| --- | --- | --- |
| `provider` | string | Provider identifier |
| `status` | string | `live` or `error` |
| `context_length` | number | Max context length on this provider |
| `pricing.input` / `pricing.output` | number | USD per million tokens |
| `is_free` | boolean | Temporary free promo |
| `supports_tools` | boolean | Tool-calling support |
| `supports_structured_output` | boolean | JSON schema output support |
| `first_token_latency_ms` | number | TTFT from latest probe |
| `throughput` | number | Output tokens/second from latest probe |
| `is_model_author` | boolean | Whether provider published the model |

---

### 2. Chat Completion (LLM)

**Source pages:** `inference-providers/tasks/chat-completion`, `inference-providers/index`

#### Main concepts

- **Chat completion**: generates a response given a list of messages in a conversational context. Supports both LLMs and VLMs. This is the primary interface for conversational AI.
- **OpenAI SDK compatibility**: the `/v1/chat/completions` endpoint is a drop-in replacement for OpenAI's API. Any OpenAI client (Python, JS, Go, Java) works by changing the `base_url`.
- **Streaming**: Server-Sent Events (SSE) when `stream=true`; tokens arrive incrementally as `data: {...}` lines terminated by `data: [DONE]`.
- **Tool calling**: the model can decide to call external functions defined in the `tools` array.
- **Structured output**: `response_format` with `json_schema` forces schema-compliant JSON.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create chat completion | `POST /v1/chat/completions` | Generate a conversational response |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `messages` | object[] (required) | Conversation history; each message has `role` (system/user/assistant/tool) and `content` |
| `content` | string \| object[] | Plain text, or array of content parts (text parts: `{type:"text", text}`, image parts: `{type:"image_url", image_url:{url}}`) |
| `model` | string (required) | Hub model id with optional `:policy`/`:provider` suffix |
| `max_tokens` | integer | Maximum tokens to generate |
| `temperature` | number (0–2) | Sampling temperature |
| `top_p` | number | Nucleus sampling probability mass |
| `frequency_penalty` | number (−2.0 to 2.0) | Penalizes repeated tokens by frequency |
| `presence_penalty` | number (−2.0 to 2.0) | Penalizes tokens already present |
| `seed` | integer | Reproducibility seed |
| `stop` | string[] | Up to 4 stop sequences |
| `stream` | boolean | Enable SSE streaming |
| `stream_options.include_usage` | boolean | Stream token-usage stats before `[DONE]` |
| `tools` | object[] | Function definitions the model may call; each has `type`, `function{name, description, parameters}` |
| `tool_choice` | enum \| object | `auto`, `none`, `required`, or `{type:"function", function:{name}}` to force a specific function |
| `tool_prompt` | string | Prompt appended before tools |
| `response_format` | object | `{type:"text"}`, `{type:"json_object"}`, or `{type:"json_schema", json_schema:{name, description, schema, strict}}` |
| `reasoning_effort` | string | Constrains reasoning effort for reasoning models: `none`, `minimal`, `low`, `medium`, `high`, `xhigh` |
| `logprobs` | boolean | Return log probabilities of output tokens |
| `top_logprobs` | integer (0–5) | Number of most-likely tokens per position (requires `logprobs=true`) |

#### Response (non-streaming)

| Field | Type | Description |
| --- | --- | --- |
| `choices[]` | object[] | Array of completion choices |
| `choices[].message` | object | `{role, content}` or `{role, tool_calls[]}` |
| `choices[].finish_reason` | string | Why generation stopped |
| `choices[].index` | integer | Choice index |
| `choices[].logprobs` | object | Token logprobs if requested |
| `id` | string | Completion id |
| `model` | string | Model used |
| `created` | integer | Timestamp |
| `system_fingerprint` | string | System fingerprint |
| `usage` | object | `{prompt_tokens, completion_tokens, total_tokens}` |

#### Response (streaming)

Each SSE chunk contains `choices[]` with a `delta` object (incremental content/role/tool_calls) instead of a full `message`.

---

### 3. Chat Completion (VLM — Vision-Language Models)

**Source pages:** `inference-providers/tasks/image-text-to-text`, `inference-providers/tasks/chat-completion`

#### Main concepts

- **Vision-Language Models (VLMs)**: models that accept images alongside text (e.g., GLM-4.5V, Qwen2.5-VL). They use the same `/v1/chat/completions` endpoint as LLMs.
- **Multimodal content**: the `content` field of a user message is an array of typed parts — `text` parts and `image_url` parts.

#### API function

Same as Chat Completion (`POST /v1/chat/completions`). The `content` array differentiates VLM from LLM requests.

#### Image content part structure

```json
{
  "type": "image_url",
  "image_url": { "url": "https://..." }
}
```

#### Providers supporting VLM chat

Cohere, DeepInfra, Featherless AI, Fireworks, Groq, HF Inference, Novita, Nscale, OVHcloud, Together, Z.ai.

---

### 4. Text Generation

**Source pages:** `inference-providers/tasks/text-generation`

#### Main concepts

- **Text generation**: generate text from a raw prompt string (as opposed to chat completion's message list). This is the TGI/Featherless-style interface. Chat completion is recommended for conversational use; text generation is for raw prompt-based generation.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Text generation | `POST /models/{model_id}` (provider-routed) | Generate text from a prompt |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | The prompt text |
| `parameters` | object | Generation parameters |
| `parameters.do_sample` | boolean | Activate logits sampling |
| `parameters.max_new_tokens` | integer | Max tokens to generate |
| `parameters.temperature` | number | Logits distribution modulation |
| `parameters.top_k` | integer | Top-k filtering |
| `parameters.top_p` | number | Nucleus sampling |
| `parameters.top_n_tokens` | integer | Top-n-token filtering |
| `parameters.repetition_penalty` | number | Repetition penalty (1.0 = none) |
| `parameters.frequency_penalty` | number | Frequency penalty (1.0 = none) |
| `parameters.typical_p` | number | Typical decoding mass |
| `parameters.seed` | integer | Random seed |
| `parameters.truncate` | integer | Truncate input to N tokens |
| `parameters.return_full_text` | boolean | Prepend prompt to output |
| `parameters.watermark` | boolean | Apply watermark |
| `parameters.best_of` | integer | Generate N sequences, return best |
| `parameters.adapter_id` | string | LoRA adapter id |
| `parameters.stop` | string[] | Stop sequences |
| `parameters.details` | boolean | Return generation details |
| `parameters.decoder_input_details` | boolean | Return decoder input logprobs/ids |
| `parameters.grammar` | object | Grammar constraint: `{type:"json", value}`, `{type:"regex", value}`, or `{type:"json_schema", value:{name, schema}}` |
| `stream` | boolean | Enable SSE streaming |

#### Response (non-streaming)

| Field | Type | Description |
| --- | --- | --- |
| `generated_text` | string | The generated text |
| `details` | object | Finish reason, generated token count, seed, prefill tokens, output tokens with logprobs |
| `details.finish_reason` | enum | `length`, `eos_token`, `stop_sequence` |
| `details.best_of_sequences` | object[] | Additional sequences if `best_of` > 1 |

---

### 5. Feature Extraction (Embeddings)

**Source pages:** `inference-providers/tasks/feature-extraction`

#### Main concepts

- **Feature extraction / embeddings**: convert text into a vector. Used for RAG retrieval, semantic search, clustering, and similarity computation.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Feature extraction | `POST /models/{model_id}` (provider-routed) | Generate embeddings |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string \| string[] (required) | Text or batch of texts to embed |
| `normalize` | boolean | Normalize output vectors |
| `prompt_name` | string | Named prompt from sentence-transformers config (e.g., `"query"`) |
| `truncate` | boolean | Truncate input to model max length |
| `truncation_direction` | enum | `left` or `right` |

#### Response

An array of arrays (float vectors) — one vector per input string.

#### Providers supporting feature extraction

HF Inference, Scaleway, Together.

---

### 6. Text to Image

**Source pages:** `inference-providers/tasks/text-to-image`

#### Main concepts

- **Text to image**: generate an image from a text prompt using diffusion models (e.g., FLUX.1-dev, FLUX.1-schnell).

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Text to image | `POST /models/{model_id}` (provider-routed) | Generate an image |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | The text prompt |
| `parameters.guidance_scale` | number | Higher = closer to prompt |
| `parameters.negative_prompt` | string | What NOT to include |
| `parameters.num_inference_steps` | integer | Denoising steps |
| `parameters.width` | integer | Output image width (px) |
| `parameters.height` | integer | Output image height (px) |
| `parameters.scheduler` | string | Override scheduler |
| `parameters.seed` | integer | Random seed |

#### Response

Raw image bytes in the payload (content type: `image/png` or similar).

#### Providers supporting text-to-image

Fal AI, HF Inference, Nscale, Replicate, Together, WaveSpeedAI.

---

### 7. Text to Video

**Source pages:** `inference-providers/tasks/text-to-video`

#### Main concepts

- **Text to video**: generate a video from a text prompt (e.g., HunyuanVideo, LTX-Video, Wan2.2).

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Text to video | `POST /models/{model_id}` (provider-routed) | Generate a video |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | The text prompt |
| `parameters.num_frames` | number | Number of video frames |
| `parameters.guidance_scale` | number | Guidance strength |
| `parameters.negative_prompt` | string[] | What NOT to include |
| `parameters.num_inference_steps` | integer | Denoising steps |
| `parameters.seed` | integer | Random seed |

#### Response

Raw video bytes in the payload.

#### Providers supporting text-to-video

Fal AI, Novita, Replicate, Together, WaveSpeedAI.

---

### 8. Image to Image

**Source pages:** `inference-providers/tasks/image-to-image`

#### Main concepts

- **Image to image**: transform a source image (style transfer, colorization, upscaling, editing). Input image is base64-encoded.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Image to image | `POST /models/{model_id}` (provider-routed) | Transform an image |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Base64-encoded input image |
| `parameters.prompt` | string | Text prompt to guide transformation |
| `parameters.guidance_scale` | number | Guidance strength |
| `parameters.negative_prompt` | string | What NOT to include |
| `parameters.num_inference_steps` | integer | Denoising steps |
| `parameters.target_size` | object | `{width, height}` — output dimensions (provider/model-dependent) |

#### Response

Raw image bytes.

#### Providers supporting image-to-image

Fal AI, Replicate, Together, WaveSpeedAI.

---

### 9. Automatic Speech Recognition

**Source pages:** `inference-providers/tasks/automatic-speech-recognition`

#### Main concepts

- **ASR / Speech-to-Text**: transcribe audio to text. Input audio is base64-encoded (or raw bytes if no parameters).

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| ASR | `POST /models/{model_id}` (provider-routed) | Transcribe audio |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Base64-encoded audio |
| `parameters.return_timestamps` | boolean | Output timestamps with text |
| `parameters.generation_parameters` | object | Generation controls: `temperature`, `top_k`, `top_p`, `typical_p`, `epsilon_cutoff`, `eta_cutoff`, `max_length`, `max_new_tokens`, `min_length`, `min_new_tokens`, `do_sample`, `early_stopping`, `num_beams`, `num_beam_groups`, `penalty_alpha`, `use_cache` |

#### Response

| Field | Type | Description |
| --- | --- | --- |
| `text` | string | Recognized text |
| `chunks[]` | object[] | When `return_timestamps=true`: `{text, timestamp:[start, end]}` |

#### Providers supporting ASR

Fal AI, HF Inference, Replicate, Together.

---

### 10. Summarization

**Source pages:** `inference-providers/tasks/summarization`

#### Main concepts

- **Summarization**: produce a shorter version of a document while preserving important information.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Summarize | `POST /models/{model_id}` (provider-routed) | Summarize text |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Text to summarize |
| `parameters.clean_up_tokenization_spaces` | boolean | Clean extra spaces in output |
| `parameters.truncation` | enum | `do_not_truncate`, `longest_first`, `only_first`, `only_second` |
| `parameters.generate_parameters` | object | Additional generation params |

#### Response

| Field | Type | Description |
| --- | --- | --- |
| `summary_text` | string | The summary |

#### Provider

HF Inference.

---

### 11. Translation

**Source pages:** `inference-providers/tasks/translation`

#### Main concepts

- **Translation**: convert text from one language to another.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Translate | `POST /models/{model_id}` (provider-routed) | Translate text |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Text to translate |
| `parameters.src_lang` | string | Source language (required for multilingual models) |
| `parameters.tgt_lang` | string | Target language (required for multilingual models) |
| `parameters.clean_up_tokenization_spaces` | boolean | Clean extra spaces |
| `parameters.truncation` | enum | Truncation strategy |
| `parameters.generate_parameters` | object | Additional generation params |

#### Response

| Field | Type | Description |
| --- | --- | --- |
| `translation_text` | string | The translated text |

#### Provider

HF Inference.

---

### 12. Text Classification

**Source pages:** `inference-providers/tasks/text-classification`

#### Main concepts

- **Text classification**: assign a label/class to text (sentiment analysis, language detection, prompt-injection detection).

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Classify text | `POST /models/{model_id}` (provider-routed) | Classify text |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Text to classify |
| `parameters.function_to_apply` | enum | `sigmoid`, `softmax`, `none` |
| `parameters.top_k` | integer | Limit to top K classes |

#### Response

Array of `{label, score}` objects.

#### Provider

HF Inference.

---

### 13. Token Classification (NER)

**Source pages:** `inference-providers/tasks/token-classification`

#### Main concepts

- **Token classification**: assign labels to individual tokens (NER, POS tagging).

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Token classification | `POST /models/{model_id}` (provider-routed) | Classify tokens |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Input text |
| `parameters.ignore_labels` | string[] | Labels to ignore |
| `parameters.stride` | integer | Overlap tokens between chunks |
| `parameters.aggregation_strategy` | string | `none`, `simple`, `first`, `average`, `max` |

#### Response

Array of `{entity_group, entity, score, word, start, end}` objects.

#### Provider

HF Inference.

---

### 14. Question Answering

**Source pages:** `inference-providers/tasks/question-answering`

#### Main concepts

- **Extractive QA**: retrieve an answer span from a given context document.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Question answering | `POST /models/{model_id}` (provider-routed) | Extract answer |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | object (required) | `{context, question}` |
| `parameters.top_k` | integer | Number of answers |
| `parameters.doc_stride` | integer | Chunk overlap size |
| `parameters.max_answer_len` | integer | Max answer length |
| `parameters.max_seq_len` | integer | Max total sentence length |
| `parameters.max_question_len` | integer | Max question length |
| `parameters.handle_impossible_answer` | boolean | Accept impossible answer |
| `parameters.align_to_words` | boolean | Align to real words |

#### Response

Array of `{answer, score, start, end}` objects.

#### Provider

HF Inference.

---

### 15. Fill-Mask

**Source pages:** `inference-providers/tasks/fill-mask`

#### Main concepts

- **Fill-mask**: predict the masked token(s) in a sequence.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Fill mask | `POST /models/{model_id}` (provider-routed) | Predict masked token |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Text with mask token |
| `parameters.top_k` | integer | Number of predictions |
| `parameters.targets` | string[] | Limit scores to these targets |

#### Response

Array of `{sequence, score, token, token_str}` objects.

#### Provider

HF Inference.

---

### 16. Image Classification

**Source pages:** `inference-providers/tasks/image-classification`

#### Main concepts

- **Image classification**: assign a single label to an entire image. Input is base64-encoded image.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Classify image | `POST /models/{model_id}` (provider-routed) | Classify image |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Base64-encoded image |
| `parameters.function_to_apply` | enum | `sigmoid`, `softmax`, `none` |
| `parameters.top_k` | integer | Top K classes |

#### Response

Array of `{label, score}` objects.

#### Providers

Fal AI, HF Inference.

---

### 17. Object Detection

**Source pages:** `inference-providers/tasks/object-detection`

#### Main concepts

- **Object detection**: identify objects with bounding boxes and labels.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Detect objects | `POST /models/{model_id}` (provider-routed) | Detect objects in image |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Base64-encoded image |
| `parameters.threshold` | number | Probability threshold |

#### Response

Array of `{label, score, box:{xmin, ymin, xmax, ymax}}` objects.

#### Provider

HF Inference.

---

### 18. Image Segmentation

**Source pages:** `inference-providers/tasks/image-segmentation`

#### Main concepts

- **Image segmentation**: divide an image into segments (instance, panoptic, semantic).

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Segment image | `POST /models/{model_id}` (provider-routed) | Segment image |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Base64-encoded image |
| `parameters.mask_threshold` | number | Binary mask threshold |
| `parameters.overlap_mask_area_threshold` | number | Eliminate small segments |
| `parameters.subtask` | enum | `instance`, `panoptic`, `semantic` |
| `parameters.threshold` | number | Probability threshold for masks |

#### Response

Array of `{label, mask, score}` objects (mask is base64-encoded black-and-white image).

#### Providers

Fal AI, HF Inference.

---

### 19. Zero-Shot Classification

**Source pages:** `inference-providers/tasks/zero-shot-classification`

#### Main concepts

- **Zero-shot classification**: classify text into candidate labels without task-specific training.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Zero-shot classify | `POST /models/{model_id}` (provider-routed) | Zero-shot classify |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Text to classify |
| `parameters.candidate_labels` | string[] (required) | Possible class labels |
| `parameters.hypothesis_template` | string | Template with placeholder for candidate labels |
| `parameters.multi_label` | boolean | Multiple labels can be true |

#### Response

Array of `{label, score}` objects.

#### Provider

HF Inference.

---

### 20. Table Question Answering

**Source pages:** `inference-providers/tasks/table-question-answering`

#### Main concepts

- **Table QA**: answer a question about information in a table.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Table QA | `POST /models/{model_id}` (provider-routed) | Answer question about table |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | object (required) | `{table, question}` |
| `parameters.padding` | enum | `do_not_pad`, `longest`, `max_length` |
| `parameters.sequential` | boolean | Sequential vs batch inference |
| `parameters.truncation` | boolean | Truncation control |

#### Response

Array of `{answer, coordinates, cells, aggregator}` objects.

#### Provider

HF Inference.

---

### 21. Audio Classification

**Source pages:** `inference-providers/tasks/audio-classification`

#### Main concepts

- **Audio classification**: assign a label to audio (command recognition, speaker ID, genre detection).

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Classify audio | `POST /models/{model_id}` (provider-routed) | Classify audio |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `inputs` | string (required) | Base64-encoded audio |
| `parameters.function_to_apply` | enum | `sigmoid`, `softmax`, `none` |
| `parameters.top_k` | integer | Top K classes |

#### Response

Array of `{label, score}` objects.

> Note: no providers currently support audio classification via Inference Providers.

---

### 22. Function Calling / Tool Use

**Source pages:** `inference-providers/guides/function-calling`

#### Main concepts

- **Function calling**: enables LLMs to call external functions/tools by generating structured JSON function calls. The model decides when to call based on user input, you execute the function, then feed the result back for a final response.
- **Tool schema**: JSON Schema describing each function's name, description, and parameters.
- **Tool choice**: controls when functions are called — `auto` (model decides), `none`, `required` (must call at least one), or a specific function object.
- **Strict mode**: `additionalProperties: false` + `strict: true` enforces exact schema compliance. Provider-dependent support.

#### Workflow

1. Define functions (Python implementations)
2. Define tool schemas (JSON Schema)
3. Send initial request with `tools` and `tool_choice`
4. Check `response.choices[0].message.tool_calls`
5. Execute functions, add results as `role: "tool"` messages
6. Send follow-up request with full conversation to get final response

#### Key parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `tools` | object[] | Function definitions: `{type:"function", function:{name, description, parameters}}` |
| `tool_choice` | enum \| object | `auto`, `none`, `required`, or `{type:"function", function:{name}}` |
| `strict` | boolean | Enforce strict schema (in function.parameters) |
| `additionalProperties` | boolean | Must be `false` for strict mode |

#### Streaming with tools

When `stream=true`, tool calls arrive incrementally in `chunk.choices[0].delta.tool_calls`.

> Note: `huggingface_hub.InferenceClient` does not currently support `tool_choice` with a specific function. Use the OpenAI client for full `tool_choice` support.

---

### 23. Structured Output (JSON Schema)

**Source pages:** `inference-providers/guides/structured-output`, `inference-providers/tasks/chat-completion`

#### Main concepts

- **Structured outputs**: guarantee the model returns JSON matching a specific schema every time. Eliminates parsing/retry logic.
- **Response format types**:
  - `{type: "text"}` — plain text
  - `{type: "json_object"}` — JSON object (no schema enforcement)
  - `{type: "json_schema", json_schema: {name, description, schema, strict}}` — schema-validated JSON

#### Using response_format

```python
response_format = {
    "type": "json_schema",
    "json_schema": {
        "name": "PaperAnalysis",
        "schema": PaperAnalysis.model_json_schema(),  # from Pydantic
        "strict": True,
    },
}
```

#### OpenAI client `.parse` helper

The OpenAI Python SDK's `client.beta.chat.completions.parse()` accepts a Pydantic model directly as `response_format` and returns a parsed Python object via `completion.choices[0].message.parsed`.

#### Key parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `response_format.type` | enum | `text`, `json_object`, `json_schema` |
| `response_format.json_schema.name` | string | Schema name |
| `response_format.json_schema.description` | string | Description for the model |
| `response_format.json_schema.schema` | object | JSON Schema definition |
| `response_format.json_schema.strict` | boolean | Strict schema adherence |

---

### 24. Responses API (Agentic)

**Source pages:** `inference-providers/guides/responses-api`

#### Main concepts

- **Responses API** (beta): OpenAI's unified agentic interface, accessible via `POST /v1/responses` on the HF router. Supports multi-provider routing, event streaming, structured outputs, tool orchestration, and Remote MCP tools.
- **Event-driven streaming**: semantic events like `response.created`, `output_text.delta`, `response.completed`.
- **Remote MCP**: call server-hosted Model Context Protocol tools without client-side implementation.
- **Reasoning effort controls**: `reasoning.effort` parameter (`low`, `medium`, `high`) trades latency for depth.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Create response | `POST https://router.huggingface.co/v1/responses` | Agentic model interaction |

#### Request parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `model` | string (required) | Model id with optional `:provider` suffix |
| `instructions` | string | System-level instructions |
| `input` | string \| object[] | Plain text, or message array (roles: `developer`, `system`, `user`) |
| `input[].content` | string \| object[] | Multimodal: `input_text` and `input_image` parts |
| `stream` | boolean | Event-based streaming |
| `tools` | object[] | Function tools or MCP tools |
| `tools[].type` | string | `function` or `mcp` |
| `tools[].server_label` | string | MCP server label |
| `tools[].server_url` | string | MCP server URL |
| `tools[].allowed_tools` | string[] | Allowed MCP tool names |
| `tools[].require_approval` | string | `"never"` to skip approval |
| `tool_choice` | enum | `auto`, etc. |
| `response_format` | object | JSON schema structured output |
| `reasoning.effort` | string | `low`, `medium`, `high` |
| `text_format` | Pydantic model | (Python SDK `.parse` helper) — target type for structured output |

#### Response

| Field | Type | Description |
| --- | --- | --- |
| `output_text` | string | Convenience text output |
| `output[]` | object[] | Full output items (text, tool calls, reasoning) |
| `output_parsed` | object | Parsed structured object (with `.parse` helper) |

---

### 25. Model Discovery & Hub API

**Source pages:** `inference-providers/hub-api`, `inference-providers/index`

#### Main concepts

- **Model inference status**: `"warm"` means at least one provider serves the model; absent means no inference provider.
- **inferenceProviderMapping**: per-model metadata showing which providers serve it, their status (`staging`/`live`), the provider-specific model id, and the task type.

#### API functions

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| List models by provider | `GET https://huggingface.co/api/models?inference_provider={provider}` | Filter models served by a specific provider (or `all`, or comma-separated list) |
| List models by task + provider | `GET https://huggingface.co/api/models?inference_provider={provider}&pipeline_tag={task}` | Combined filter |
| Get model inference status | `GET https://huggingface.co/api/models/{model_id}?expand[]=inference` | Returns `inference: "warm"` or undefined |
| Get model provider mapping | `GET https://huggingface.co/api/models/{model_id}?expand[]=inferenceProviderMapping` | Returns provider→{status, providerId, task, isModelAuthor} |
| List OpenAI-compatible models | `GET https://router.huggingface.co/v1/models` | All chat models with per-provider pricing/latency/throughput metadata |
| Get single model (router) | `GET https://router.huggingface.co/v1/models/{model_id}` | One model with all provider entries |

#### CLI equivalent

```bash
hf models ls --warm                          # all models with at least one provider
hf models ls --warm --search GLM-5.2         # search
hf models ls --inference-provider fal-ai --pipeline-tag text-to-image
hf models ls --warm --json                   # machine-readable
```

---

### 26. Billing & Organization Cost Attribution

**Source pages:** `inference-providers/pricing`

#### Main concepts

- **Pay-as-you-go**: no markup on provider rates. Monthly credits apply to routed requests.
- **Free tier credits**: Free users $0.10/mo, PRO users $2.00/mo, Team/Enterprise $2.00/seat/mo.
- **Routed vs. Custom provider key**: routed requests bill through HF (credits apply); custom key bills directly through the provider (no credits).
- **Organization billing**: Team/Enterprise can centralize billing via `X-HF-Bill-To` header or `bill_to` client parameter.
- **HF-Inference cost**: billed per compute time × hardware price per second (e.g., 10s on a $0.00012/s GPU = $0.0012).

#### Billing parameters

| Parameter | Location | Description |
| --- | --- | --- |
| `X-HF-Bill-To` | HTTP header | Organization name or resource group ID to bill |
| `bill_to` | `InferenceClient` constructor (Python) | Organization name |
| `billTo` | `InferenceClient` constructor (JS) | Organization name |
| `extra_headers: {"X-HF-Bill-To": "org-name"}` | OpenAI client method call | Header injection for OpenAI SDK |

---

## Inference Endpoints (Dedicated GPU)

### 27. Endpoint Lifecycle Management

**Source pages:** `inference-endpoints/quick_start`, `inference-endpoints/guides/foundations`, `inference-endpoints/about`

#### Main concepts

- **Inference Endpoint**: a managed deployment of a single model from the Hugging Face Hub onto dedicated infrastructure. Packaged as a Docker container running an inference engine (vLLM, TGI, SGLang, etc.).
- **Model repository**: the Hugging Face Hub repo containing model weights/artifacts. Mounted at `/repository` inside the container.
- **Endpoint states**: `Running`, `Initializing`, `Paused`, `Scaled to Zero`, `Failed`.
- **Model Catalog**: 100+ pre-configured models for one-click deployment, filterable by name, task, hardware, price.
- **Dashboard** (`endpoints.huggingface.co`): central interface to manage, monitor, and deploy endpoints across organizations.

#### API functions (management)

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| List endpoints | `GET https://api.endpoints.huggingface.cloud/v2/endpoint/{namespace}` | Enumerate endpoints for a namespace |
| Create endpoint | `POST https://api.endpoints.huggingface.cloud/v2/endpoint/{namespace}` | Create a new endpoint |
| Get endpoint | `GET https://api.endpoints.huggingface.cloud/v2/endpoint/{namespace}/{name}` | Get endpoint details |
| Update endpoint | `PUT https://api.endpoints.huggingface.cloud/v2/endpoint/{namespace}/{name}` | Modify configuration |
| Delete endpoint | `DELETE https://api.endpoints.huggingface.cloud/v2/endpoint/{namespace}/{name}` | Remove endpoint |
| Pause endpoint | `POST .../{name}/pause` | Stop endpoint (still counts toward quota) |
| Resume endpoint | `POST .../{name}/resume` | Wake up a scaled-to-zero or paused endpoint |
| Get OpenMetrics | `GET https://api.endpoints.huggingface.cloud/v2/endpoint/{namespace}/{name}/open-metrics` | Real-time metrics in OpenMetrics format (Team/Enterprise) |

> The full OpenAPI specification is available at `https://api.endpoints.huggingface.cloud/`. Management is also possible through the `huggingface_hub` Python client and the web UI.

#### Inference API (per endpoint)

Once running, each endpoint exposes an OpenAI-compatible API:

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Chat completions | `POST https://{id}.{region}.{cloud}.endpoints.huggingface.cloud/v1/chat/completions` | OpenAI-compatible chat |
| Completions | `POST .../v1/completions` | OpenAI-compatible completions |
| Health check | `GET .../health` | Readiness probe (returns 503 if model not loaded) |

---

### 28. Hardware & Cloud Configuration

**Source pages:** `inference-endpoints/guides/configuration`, `inference-endpoints/support/pricing`

#### Main concepts

- **Cloud providers**: AWS, Microsoft Azure, Google Cloud Platform.
- **Accelerator types**: CPU, GPU, INF2 (AWS Inferentia2).
- **Instance types**: defined by GPU type, GPU count, memory, vCPUs, and hourly rate.
- **Region**: deployment region per cloud provider (e.g., East US).

#### Available GPU instances (AWS, selected)

| Instance Type | Size | GPUs | Memory | Hourly Rate | Architecture |
| --- | --- | --- | --- | --- | --- |
| nvidia-t4 | x1 | 1 | 14 GB | $0.5 | NVIDIA T4 |
| nvidia-l4 | x1 | 1 | 24 GB | $0.8 | NVIDIA L4 |
| nvidia-a10g | x1 | 1 | 24 GB | $1 | NVIDIA A10G |
| nvidia-l40s | x1 | 1 | 48 GB | $1.8 | NVIDIA L40S |
| nvidia-a100 | x1 | 1 | 80 GB | $2.5 | NVIDIA A100 |
| nvidia-a100 | x4 | 4 | 320 GB | $10 | NVIDIA A100 |
| nvidia-h200 | x1 | 1 | 141 GB | $5 | NVIDIA H200 |
| nvidia-h200 | x8 | 8 | 1128 GB | $40 | NVIDIA H200 |
| inf2 | x1 | 1 | 32 GB | $0.75 | AWS Inferentia2 |
| inf2 | x12 | 12 | 384 GB | $12 | AWS Inferentia2 |

#### CPU instances (AWS, selected)

| Instance Type | Size | vCPUs | Memory | Hourly Rate |
| --- | --- | --- | --- | --- |
| intel-spr | x1 | 1 | 2 GB | $0.033 |
| intel-spr | x2 | 2 | 4 GB | $0.067 |
| intel-spr | x4 | 4 | 8 GB | $0.134 |

#### Configuration parameters

| Parameter | Description |
| --- | --- |
| Cloud provider | AWS, Azure, GCP |
| Accelerator type | CPU, GPU, INF2 |
| Region | Deployment region |
| Instance type | GPU type + count |
| Commit revision | Specific model commit hash |
| Task | Model task type (usually auto-inferred) |
| Download pattern | Which model files to download |

---

### 29. Inference Engine Configuration

**Source pages:** `inference-endpoints/engines/vllm`, `inference-endpoints/engines/tgi`, `inference-endpoints/engines/sglang`, `inference-endpoints/engines/llama_cpp`, `inference-endpoints/engines/tei`, `inference-endpoints/engines/toolkit`, `inference-endpoints/engines/custom_container`, `inference-endpoints/about`

#### Main concepts

- **Inference engine**: the software that loads and runs the model. Hugging Face provides native support for:
  - **vLLM** — high-performance, memory-efficient; PagedAttention, continuous batching, speculative decoding, chunked prefill, multi-backend (NVIDIA/AMD/Neuron)
  - **TGI (Text Generation Inference)** — in maintenance mode as of Nov 2025; Rust+Python; continuous batching, Flash Attention, quantization (bitsandbytes/GPTQ), OpenAI-compatible API, watermarking, JSON schema guidance. Migration to vLLM recommended.
  - **SGLang** — fast serving framework; RadixAttention for prefix caching, zero-overhead scheduler, continuous batching, tensor/pipeline parallelism, expert parallelism, structured outputs, chunked prefill, FP8/INT4/AWQ/GPTQ quantization, multi-LoRA batching
  - **llama.cpp** — C/C++ engine for GGUF models; CPU+GPU; OpenAI-compatible API; auto-selected for GGUF repos
  - **TEI (Text Embeddings Inference)** — production-ready embeddings; dynamic batching, Flash Attention, Safetensors/ONNX, OpenTelemetry/Prometheus
  - **Inference Toolkit** — fallback for Transformers/Sentence-Transformers/Diffusers models; supports custom `handler.py`; not as production-ready as the above engines
  - **Custom container** — deploy any Docker container; model mounted at `/repository`; requires `/health` endpoint

#### vLLM configuration parameters

| Parameter | Description |
| --- | --- |
| Max Number of Sequences | Max sequences (requests) per batch |
| Max Number of Batched Tokens | Max total tokens per batch |
| Tensor Parallel Size | GPUs across which model weights are split per layer |
| Data Parallel Size | Number of independent model copies (TP × DP = total GPUs) |
| KV Cache DType | `auto`, `fp8`, `fp8_e5m2`, `fp8_e4m3` |
| Engine Arguments | Any vLLM `EngineArgs` (e.g., `enable_lora`) |

#### TGI configuration parameters

| Parameter | Description |
| --- | --- |
| Quantization | Quantization method (bitsandbytes, GPTQ, etc.) |
| Max Number of Tokens (per query) | Max tokens per request (prompt + generation) |
| Max Input Tokens (per query) | Max input/prompt tokens |
| Max Batch Prefill Tokens | Limits prefill tokens |
| Max Batch Total Tokens | Total potential tokens in a batch; with Max Number of Tokens determines concurrency |
| Zero-config mode | TGI v3 auto-configures max values based on hardware |

#### SGLang configuration parameters

| Parameter | Description |
| --- | --- |
| Max Running Request | Max concurrent requests |
| Max Prefill Tokens (per batch) | Max tokens per prefill operation |
| Chunked prefill size | Tokens per prefill chunk (`-1` = disable) |
| Tensor Parallel Size | GPUs for model sharding |
| KV Cache DType | `auto`, `fp8_e5m2`, `fp8_e4m3` |
| Server Arguments | Any SGLang server args (e.g., `schedule-policy`) |

#### llama.cpp configuration parameters

| Parameter | Description |
| --- | --- |
| Max Tokens (per Request) | Max tokens per request |
| Max Concurrent Requests | Max concurrent requests (requires proportional memory) |
| Environment variables | Any non-reserved llama.cpp env vars (reserved: `LLAMA_ARG_MODEL`, `LLAMA_ARG_HTTP_THREADS`, `LLAMA_ARG_N_GPU_LAYERS`, `LLAMA_ARG_EMBEDDINGS`, `LLAMA_ARG_HOST`, `LLAMA_ARG_PORT`, `LLAMA_ARG_NO_MMAP`, `LLAMA_ARG_CTX_SIZE`, `LLAMA_ARG_N_PARALLEL`, `LLAMA_ARG_ENDPOINT_METRICS`) |

#### TEI configuration parameters

| Parameter | Description |
| --- | --- |
| Max Tokens (per batch) | Tokens before forcing queue wait |
| Max Concurrent Requests | Max concurrent requests |
| Pooling | Override model pooling config |

#### Parallelism strategies (vLLM)

- **Tensor Parallelism (TP)**: splits model weights across GPUs within each layer. Required when model doesn't fit on one GPU. Formula: minimum TP = ceiling(model_size / single_gpu_memory).
- **Data Parallelism (DP)**: runs multiple independent model copies. Higher throughput when model fits on fewer GPUs.
- **Combined**: `TP × DP = total GPUs on instance`. Trade-off: higher DP → more throughput but less KV cache per copy (shorter context); higher TP → longer context but lower throughput.

---

### 30. Autoscaling

**Source pages:** `inference-endpoints/guides/autoscaling`, `inference-endpoints/guides/configuration`

#### Main concepts

- **Scale to zero**: endpoint goes idle after a configurable duration (default 1 hour) of inactivity. Cold start on next request. Proxy returns `503` while initializing; `X-Scale-Up-Timeout` header can hold the request until ready.
- **Replica count**: min/max replicas. Min must be 0 if scale-to-zero is enabled.
- **Autoscaling strategies**:
  - **Hardware utilization**: scale up when average GPU utilization exceeds threshold (default 80%) over 1-minute window (GPU) or 20s (CPU). Scale up every minute, scale down every 2 minutes with 300s stabilization.
  - **Pending requests**: scale up when average pending requests per replica exceeds threshold (default 1.5) over 20 seconds. Leading indicator vs. hardware metrics.

#### Configuration parameters

| Parameter | Description |
| --- | --- |
| Scale-to-zero timeout | Idle duration before scaling to 0 (default: 1 hour) |
| Min replicas | Floor (0 if scale-to-zero enabled) |
| Max replicas | Ceiling |
| Autoscaling strategy | Hardware utilization or pending requests |
| Hardware utilization threshold | Percentage (default 80%) |
| Pending requests threshold | Requests per replica (default 1.5) |
| X-Scale-Up-Timeout | HTTP header to hold request during scale-up (seconds) |

---

### 31. Authentication & Access Control

**Source pages:** `inference-endpoints/guides/configuration`, `inference-endpoints/guides/security`

#### Main concepts

- **Access levels**:
  - **Private** (default): accessible only to you/org members via HF access token
  - **Public**: anyone can access without authentication
  - **Authenticated**: any HF account holder can access via their token
- **TLS/SSL**: all endpoints encrypt data in transit
- **Hugging Face token**: fine-grained token with appropriate permissions

#### Key parameters

| Parameter | Description |
| --- | --- |
| Access level | Private, Public, Authenticated |
| Authorization header | `Authorization: Bearer hf_<token>` |

---

### 32. Network & PrivateLink

**Source pages:** `inference-endpoints/guides/private_link`, `inference-endpoints/guides/configuration`

#### Main concepts

- **AWS PrivateLink**: privately connect your VPC to an Inference Endpoint without exposing traffic to the public internet. Uses private IP addresses. Intra-region only.
- **VPC Service Name**: provided by HF after endpoint creation; used to create a VPC Interface Endpoint in AWS.
- **PrivateLink Sharing**: multiple endpoints can share the same VPC Endpoint.

#### Configuration parameters

| Parameter | Description |
| --- | --- |
| AWS PrivateLink toggle | Enable/disable PrivateLink for the endpoint |
| AWS Account ID | AWS account owning the target VPC |
| PrivateLink Sharing | Allow sharing the same PrivateLink across endpoints |

---

### 33. Observability (Analytics & Metrics)

**Source pages:** `inference-endpoints/guides/analytics`

#### Main concepts

- **Analytics dashboard**: real-time metrics per endpoint/replica over configurable timeframes.
- **Metrics tracked**:
  - HTTP requests (grouped by response class or individual status codes)
  - Pending requests (in-flight + being processed)
  - Latency distribution (p50/p90/p95/p99)
  - Running replicas (with max replicas line)
  - Replica timeline (pending → running transitions)
  - Compute: CPU usage, memory usage, GPU usage, GPU memory (VRAM) usage
- **OpenMetrics API** (beta, Team/Enterprise): export real-time metrics in OpenMetrics format for Prometheus/Grafana/Datadog integration.

#### API function

| Function | Method & Endpoint | Purpose |
| --- | --- | --- |
| Get OpenMetrics | `GET https://api.endpoints.huggingface.cloud/v2/endpoint/{namespace}/{name}/open-metrics` | Real-time metrics in OpenMetrics text format |

#### Exported metrics

| Metric | Type | Description |
| --- | --- | --- |
| `latency_distribution` | summary | Latency quantiles (p50, p90, p95, p99) |
| `http_requests` | counter | HTTP requests by status code and replica |
| `cpu_usage_percent` | gauge | CPU utilization |
| (additional) | gauge | Memory, GPU, VRAM utilization |

---

### 34. Runtime Logs

**Source pages:** `inference-endpoints/guides/logs`

#### Main concepts

- **Logs interface**: real-time runtime logs for debugging. Accessible via the Logs tab in the endpoint dashboard.
- **Log filtering**: toggleable Timestamp, Log Level, Content, Replica information.
- **Pagination**: latest 50 lines by default; "Load More" for historical data.
- **Log retention**: 30 days.

---

### 35. Security & Compliance

**Source pages:** `inference-endpoints/guides/security`

#### Main concepts

- **Data privacy**: HF does not store customer payloads or tokens. Logs retained 30 days. TLS/SSL in transit.
- **Model security**: private model repos, malware and pickle scans on Hub.
- **Compliance**: SOC2 Type 2 certified. GDPR DPA available via Enterprise Hub.
- **RBAC**: Hugging Face Hub Role-Based Access Control.
- **Resource Groups** (Enterprise): attribute inference costs to specific resource groups.

---

### 36. Pricing & Cost Calculation

**Source pages:** `inference-endpoints/support/pricing`

#### Main concepts

- **Per-minute billing**: although rates are shown per hour, actual billing is per minute. Charged for initializing and running states.
- **Cost formula**: `instance_hourly_rate × ((hours × min_replicas) + (scale_up_hours × additional_replicas))`
- **Subscription requirement**: active HF subscription + credits. Automatic recharge recommended.

#### Cost examples

**Basic** (CPU, 1 replica always running):
```
$0.067/hr × (730hr × 1) = $48.91/month
```

**Advanced** (GPU, 1–3 replicas, hourly spikes):
```
$0.5/hr × ((730hr × 1) + (182.5hr × 2)) = $547.50/month
```

---

### 37. Custom Containers

**Source pages:** `inference-endpoints/engines/custom_container`

#### Main concepts

- **Custom container**: deploy any Docker container when native engines don't support your model or you need custom inference logic. Full control over the server.
- **Model mounting**: the selected model repo is mounted at `/repository` inside the container — always load from there, not from the Hub.
- **Health endpoint**: the platform pings `/health` every second as a readiness probe. Return `503` until the model is loaded.
- **Non-root user**: recommended for security.

#### Requirements

| Requirement | Description |
| --- | --- |
| Container image | Push to Docker Hub, Amazon ECR, Azure ACR, or Google GCR |
| Architecture | `linux/amd64` (use `--platform linux/amd64` when building on ARM) |
| Exposed port | Specify in the Custom Container configuration (e.g., 8000) |
| `/health` endpoint | Must return 200 when ready, 503 when loading |
| `/repository` path | Model weights mounted here |

#### Custom container configuration parameters

| Parameter | Description |
| --- | --- |
| Image URL | e.g., `your-username/smollm-endpoint:v0.1.0` |
| Port | Container's exposed port |

---

### 38. Custom Inference Handlers (Toolkit)

**Source pages:** `inference-endpoints/engines/toolkit`

#### Main concepts

- **Inference Toolkit**: fallback engine for models not supported by vLLM/TGI/SGLang/llama.cpp. Wraps Transformers/Sentence-Transformers/Diffusers in a lightweight web server. Good for testing/demos, less production-ready.
- **Custom handler**: a `handler.py` file in the model repository implementing an `EndpointHandler` class with `__init__` and `__call__` methods.
- **`__init__(self, path="")`**: called on startup; receives path to model weights. Preload model here.
- **`__call__(self, data)`**: called per request; receives a dict with `inputs` key and optional kwargs. Returns serialized list/dict.
- **Custom dependencies**: add to `requirements.txt` in the model repo.

#### Handler interface

```python
class EndpointHandler():
    def __init__(self, path=""):
        # Preload model, tokenizer, etc.
        pass

    def __call__(self, data: Dict[str, Any]) -> List[Dict[str, Any]]:
        inputs = data.pop("inputs", data)
        # Run inference
        return result
```

#### Request body

The `data` dict always contains an `inputs` key. Additional keys are passed as kwargs for custom pre/post-processing logic.

---

## Summary: Capability Matrix

### Inference Providers (Serverless) — Task × Provider Support

| Task | HF Inference | Cerebras | Cohere | DeepInfra | Fal AI | Featherless | Fireworks | Groq | Novita | Nscale | OVHcloud | Public AI | Replicate | Scaleway | Together | WaveSpeed | Z.ai |
| --- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Chat (LLM) | ✅ | ✅ | ✅ | ✅ | | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | | ✅ | ✅ | | ✅ |
| Chat (VLM) | ✅ | | ✅ | ✅ | | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | | | | | | ✅ |
| Feature Extraction | ✅ | | | | | | | | | | | | | ✅ | ✅ | | |
| Text to Image | ✅ | | | | ✅ | | | | | ✅ | | | ✅ | | | ✅ | ✅ | |
| Text to Video | | | | | ✅ | | | | ✅ | | | | ✅ | | | ✅ | | |
| Speech to Text | ✅ | | | ✅ | ✅ | | | | | | | | ✅ | | ✅ | | |

### Inference Endpoints (Dedicated) — Engine × Model Type

| Engine | LLMs | VLMs | Embeddings | GGUF | Custom Logic |
| --- | :---: | :---: | :---: | :---: | :---: |
| vLLM | ✅ | ✅ (partial) | ✅ (partial) | | |
| TGI (maintenance) | ✅ | | | | |
| SGLang | ✅ | ✅ | ✅ (partial) | | |
| llama.cpp | ✅ | | ✅ | ✅ | |
| TEI | | | ✅ | | |
| Inference Toolkit | ✅ | ✅ | ✅ | | ✅ |
| Custom Container | ✅ | ✅ | ✅ | ✅ | ✅ |

### Authentication Across Services

| Service | Auth Method | Header |
| --- | --- | --- |
| Inference Providers (router) | HF fine-grained token | `Authorization: Bearer hf_****` |
| Inference Endpoints (inference) | HF access token | `Authorization: Bearer hf_****` |
| Inference Endpoints (management) | HF access token | `Authorization: Bearer hf_****` |
| Organization billing | HF token + org header | `X-HF-Bill-To: org-name` |
| Custom provider key | Provider API key (set in HF settings) | `Authorization: Bearer hf_****` (HF swaps auth on routing) |

### Pricing Models

| Service | Billing Model | Granularity | Markup |
| --- | --- | --- | --- |
| Inference Providers (routed) | Pay-as-you-go per request | Per token (input/output) | None (pass-through) |
| Inference Providers (custom key) | Billed by provider | Per token | None |
| Inference Endpoints | Per-minute compute | Per minute of instance uptime | N/A (infrastructure cost) |
| HF-Inference (provider) | Per compute time | Per second × hardware price | None |