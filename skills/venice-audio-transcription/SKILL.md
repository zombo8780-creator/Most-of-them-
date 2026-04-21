---
name: venice-audio-transcription
description: Transcribe audio files to text via POST /audio/transcriptions. Covers supported models (Parakeet, Whisper, Wizper, Scribe, xAI STT), supported formats (wav/flac/m4a/aac/mp4/mp3/ogg/webm), response formats (json/text), timestamps, and language hints. OpenAI-compatible multipart.
---

# Venice Transcription (`/audio/transcriptions`)

`POST /api/v1/audio/transcriptions` takes an audio file and returns text. It's OpenAI-compatible with `multipart/form-data` — the OpenAI SDK's `audio.transcriptions.create()` works unchanged.

## Use when

- You need STT (speech-to-text) for voice notes, meetings, podcasts, short audio.
- You need timestamps for subtitles / chapters.
- You want to pick between fast local-style models (Parakeet) and large multilingual ones (Whisper, Wizper, Scribe).

For long video / YouTube transcription, see [`venice-video`](../venice-video/SKILL.md)'s `/video/transcriptions` (takes a public video URL directly).

## Minimal request

```bash
curl https://api.venice.ai/api/v1/audio/transcriptions \
  -H "Authorization: Bearer $VENICE_API_KEY" \
  -F "file=@./meeting.m4a" \
  -F "model=nvidia/parakeet-tdt-0.6b-v3" \
  -F "response_format=json" \
  -F "timestamps=false"
```

```json
{ "text": "Alright everyone, let's kick off the meeting..." }
```

With `timestamps=true`, `json` format also returns segment/word timings (schema is model-specific).

## Request (`multipart/form-data`)

| Field | Type | Default | Notes |
|---|---|---|---|
| `file` | binary | — | **Required.** Audio file. Supported: `wav`, `wave`, `flac`, `m4a`, `aac`, `mp4`, `mp3`, `ogg`, `webm`. Base64 is **not** accepted — upload as a real file. |
| `model` | enum | `nvidia/parakeet-tdt-0.6b-v3` | See models below. |
| `response_format` | `json` / `text` | `json` | `text` returns `text/plain` body. |
| `timestamps` | bool | `false` | Include segment/word timestamps (JSON only). |
| `language` | string | — | ISO 639-1 hint (e.g. `en`, `ja`). Only Whisper-family models honor it; others auto-detect. |

## Models

| Model ID | Notes |
|---|---|
| `nvidia/parakeet-tdt-0.6b-v3` | Default. Fast, English-first, great for real-time-ish flows. |
| `openai/whisper-large-v3` | Large multilingual, honors `language` hint. |
| `fal-ai/wizper` | Whisper variant, competitive on quality/latency tradeoff. |
| `elevenlabs/scribe-v2` | ElevenLabs Scribe, strong on noisy audio. |
| `stt-xai-v1` | xAI Speech-to-Text. |

`GET /models?type=asr` returns the current catalog. ASR pricing is `pricing.per_audio_second.usd` — cost scales with audio duration.

## OpenAI SDK

```ts
import OpenAI from 'openai'
import fs from 'node:fs'

const client = new OpenAI({
  apiKey: process.env.VENICE_API_KEY,
  baseURL: 'https://api.venice.ai/api/v1',
})

const out = await client.audio.transcriptions.create({
  file: fs.createReadStream('meeting.m4a'),
  model: 'openai/whisper-large-v3',
  response_format: 'json',
  language: 'en',
  // @ts-expect-error — Venice-specific extra, passes through multipart
  timestamps: true,
})

console.log(out.text)
```

## Batch / long files

Venice doesn't expose native chunking. For files > ~30 min, split client-side on silence with `ffmpeg` or `pydub`, transcribe each chunk, then concatenate with offset timestamps.

```bash
ffmpeg -i long.mp3 -f segment -segment_time 600 -c copy chunk_%03d.mp3
```

## Errors

| Code | Meaning |
|---|---|
| `400` | Bad params, unsupported audio format, empty file. |
| `401` | Auth / Pro-only. |
| `402` | Insufficient balance. |
| `413` | File too large. Split and retry. |
| `415` | Wrong `Content-Type` — must be `multipart/form-data`. |
| `422` | Content policy violation on detected text. |
| `429` | Rate limited. |
| `500` / `503` | Transient; retry with jitter. |

## Gotchas

- `file` must be uploaded as a real multipart file part. JSON + base64 is **not** supported here.
- `timestamps: true` + `response_format: text` is a mismatch — Venice emits JSON regardless when timestamps are requested; set `response_format: json` explicitly.
- `language` is Whisper-specific. Parakeet / Scribe ignore it and auto-detect.
- Peak concurrency limits apply — on `429`, back off; big batches should throttle to ~5 parallel requests.
- Content classifiers run on the **transcript**. A profane or policy-violating source audio can trip `422` during transcription.
