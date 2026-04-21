---
name: venice-embeddings
description: Call POST /embeddings on Venice. Covers request shape (input, model, encoding_format, dimensions, user), OpenAI compatibility, response compression (gzip/br), and practical usage for retrieval, clustering, and RAG.
---

# Venice Embeddings

`POST /api/v1/embeddings` returns vector embeddings for strings. It's OpenAI-compatible: the request and response match `https://api.openai.com/v1/embeddings` closely enough that the OpenAI SDK works out of the box with `baseURL: "https://api.venice.ai/api/v1"`.

## Use when

- You're building retrieval / RAG / similarity search.
- You need text clustering, classification, deduplication, or reranking.
- You want Venice's E2EE / private inference property on vectors.

Text-only. For image embeddings use a multimodal chat model to produce a structured representation, or query `GET /models?type=IMAGE` for models that expose embedding outputs.

## Minimal request

```bash
curl https://api.venice.ai/api/v1/embeddings \
  -H "Authorization: Bearer $VENICE_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept-Encoding: gzip, br" \
  -d '{
    "model": "text-embedding-bge-m3",
    "input": "Why is the sky blue?"
  }'
```

```json
{
  "object": "list",
  "model": "text-embedding-bge-m3",
  "data": [
    { "object": "embedding", "index": 0, "embedding": [0.0023, -0.0093, 0.0158, ...] }
  ],
  "usage": { "prompt_tokens": 8, "total_tokens": 8 }
}
```

## Request schema

| Field | Type | Notes |
|---|---|---|
| `model` | string | **Required.** Model ID from `GET /models?type=EMBEDDING`. |
| `input` | string \| string[] \| number[] \| number[][] | **Required.** Single string, array of strings, or pre-tokenized arrays. |
| `encoding_format` | `"float"` \| `"base64"` | Default `"float"`. Use `"base64"` for ~4× payload shrinkage; decode client-side. |
| `dimensions` | integer | Optional. Truncate output to N dimensions (only supported by some models — check the model's `modelSpec`). |
| `user` | string | Accepted for OpenAI compat. Discarded by Venice. |

`input` max length depends on the model's context window (often 512–8192 tokens). For arrays, Venice returns one embedding per element, in order, with matching `index`.

## Response headers & compression

Request `Accept-Encoding: gzip, br`. The response will include `Content-Encoding` accordingly. For long batches this matters — vectors are large.

For x402 auth, `X-Balance-Remaining` reports your remaining USDC credits.

## Using the OpenAI SDK

```ts
import OpenAI from 'openai'

const client = new OpenAI({
  apiKey: process.env.VENICE_API_KEY,
  baseURL: 'https://api.venice.ai/api/v1',
})

const res = await client.embeddings.create({
  model: 'text-embedding-bge-m3',
  input: ['first doc', 'second doc'],
})

const vec0 = res.data[0].embedding
```

## Batch-embedding pattern

```ts
async function embedBatch(texts: string[], batchSize = 64) {
  const out: number[][] = []
  for (let i = 0; i < texts.length; i += batchSize) {
    const slice = texts.slice(i, i + batchSize)
    const res = await client.embeddings.create({
      model: 'text-embedding-bge-m3',
      input: slice,
      encoding_format: 'float',
    })
    for (const row of res.data) out[i + row.index] = row.embedding
  }
  return out
}
```

- Keep batches ≤ model context limit total tokens.
- On `429`, back off exponentially and halve the batch — see [`venice-errors`](../venice-errors/SKILL.md).

## Choosing a model

Query `GET /models?type=EMBEDDING` for the current catalog (IDs, dimensions, context length, pricing per input token). Common choices:

- **`text-embedding-bge-m3`** — strong multilingual, 1024-dim.
- Any model with `modelSpec.capabilities.supportsEmbeddings === true`.

Always pin the model ID — cosine distances are **not** comparable across different embedding models.

## Error handling

| Code | Meaning |
|---|---|
| `400` | Validation error. Check `details` in the response for the exact field. |
| `401` | Auth / Pro-only model. |
| `402` | Insufficient balance. Bearer → `INSUFFICIENT_BALANCE`. x402 → structured `PAYMENT_REQUIRED`. |
| `415` | Wrong `Content-Type` — must be `application/json`. |
| `429` | Rate limited. |
| `500` | Inference failed; retry with jitter. |
| `503` | Model at capacity; retry later. |

## Gotchas

- `dimensions` is not universally supported — check the model's `modelSpec.constraints.embeddingDimensions` first.
- `input` must not be empty; Venice rejects empty strings with `400`.
- Cosine similarity assumes L2-normalized vectors. Many Venice embedding models return **already-normalized** vectors — verify with `Math.hypot(...v) ≈ 1` before re-normalizing.
- For RAG, store `model` alongside the vector so you can re-embed on upgrade.
