---
name: venice-errors
description: Handle Venice API errors correctly. Covers the StandardError / DetailedError / ContentViolationError / X402InferencePaymentRequired body shapes, every meaningful status code (400, 401, 402, 403, 415, 422, 429, 500, 503, 504), the 402 PAYMENT-REQUIRED header used by x402 inference, 422 content-policy suggested_prompt retry pattern, 429 rate-limit headers, and an exponential-backoff retry strategy with idempotency.
---

# Venice errors & retries

Every Venice endpoint returns one of four error shapes. Knowing which shape you got tells you how to react.

## Error body shapes

### 1. `StandardError` — simple message

The default shape for 4xx/5xx. Emitted when there's nothing structured to surface.

```json
{ "error": "Unauthorized" }
```

### 2. `DetailedError` — Zod validation failure

Used for `400` on malformed request bodies. `details` is the raw Zod `_errors` tree and tells you which field failed.

```json
{
  "error": "Invalid request",
  "details": {
    "_errors": [],
    "messages": { "_errors": ["Field is required"] }
  }
}
```

Render `details` to the user so they can fix the input; don't retry — the request shape is wrong.

### 3. `ContentViolationError` — 422 on image/audio/video prompts

Returned by image/audio/video generation endpoints when the prompt trips content policy. May include a model-provided safe alternative.

```json
{
  "error": "Content policy violation",
  "suggested_prompt": "A cinematic instrumental track inspired by stormy weather and dramatic tension."
}
```

**Pattern** — retry once with `prompt = suggested_prompt` if the user consents.

### 4. `X402InferencePaymentRequired` — 402 on x402 inference calls

Returned only when the caller authenticated with **SIWE** and has insufficient credit. Discriminated by `code: "PAYMENT_REQUIRED"`.

```json
{
  "error": "Payment required",
  "code": "PAYMENT_REQUIRED",
  "message": "Insufficient x402 balance",
  "suggestedTopUpUsd": 10,
  "minimumTopUpUsd": 5,
  "supportedTokens": ["USDC"],
  "supportedChains": ["base"],
  "topUpInstructions": {
    "step1": "POST /api/v1/x402/top-up with no payment header to get payment requirements",
    "step2": "Sign a USDC transfer authorization using the x402 SDK (createPaymentHeader)",
    "step3": "POST /api/v1/x402/top-up with the signed X-402-Payment header",
    "receiverWallet": "<RECEIVER_WALLET_ADDRESS>",
    "tokenAddress": "<USDC_TOKEN_ADDRESS>",
    "tokenDecimals": 6,
    "network": "eip155:8453",
    "minimumAmountUsd": 5
  },
  "siwxChallenge": { ... SIWE template ... }
}
```

The same information is base64-JSON encoded in the `PAYMENT-REQUIRED` response header (x402 v2 spec). See [`venice-x402`](../venice-x402/SKILL.md).

## Status code map

| Status | Body | Meaning | What to do |
|---|---|---|---|
| `400 Bad Request` | `DetailedError` | Malformed input. Zod `details` identifies the field. | Fix and re-send. **Don't retry.** |
| `401 Unauthorized` | `StandardError` | Missing / invalid Bearer API key or SIWE. | Rotate credentials. **Don't retry.** |
| `402 Payment Required` | Bearer: `StandardError` (`INSUFFICIENT_BALANCE`). SIWE: `X402InferencePaymentRequired` + `PAYMENT-REQUIRED` header. | Out of DIEM/USD/wallet credit. | Bearer: top up at venice.ai. SIWE: run the x402 top-up flow. |
| `403 Forbidden` | `StandardError` | Valid auth but not entitled. Typical: trial-limited endpoint, beta model, API-key consumption cap hit, SIWE signer ≠ path wallet. | **Don't retry.** Investigate entitlements. |
| `415 Unsupported Media Type` | `StandardError` | Wrong `Content-Type` (e.g. JSON sent to a multipart endpoint, or vice versa). | Fix headers. **Don't retry.** |
| `422 Unprocessable Entity` | `ContentViolationError` (image/audio/video). | Content policy violation. | Optionally retry once with `suggested_prompt`. |
| `429 Too Many Requests` | `StandardError` | Rate limit cap tripped. Also returned by `/crypto/rpc/{network}` when credit-per-day or concurrency cap tripped. | Honor `X-RateLimit-*` headers, back off with jitter. |
| `500 Internal Server Error` | `StandardError` | Unexpected failure. | Retry with exponential backoff + idempotency key where supported. |
| `503 Service Unavailable` | `StandardError` | Upstream model / service temporarily down. | Retry with backoff. Consider a fallback model. |
| `504 Gateway Timeout` | `StandardError` | Upstream slow. Mostly on `/chat/completions` with huge contexts. | Switch to `stream: true` or shorter prompts. |

## Rate-limit headers (`429`)

Emitted on `/crypto/rpc/{network}` and other high-QPS endpoints:

| Header | Meaning |
|---|---|
| `X-RateLimit-Limit` | Per-minute request cap for your tier (paid = 100, staff = 1000 on crypto RPC). |
| `X-RateLimit-Remaining` | Requests remaining in the current 60-second window. |
| `X-RateLimit-Reset` | Unix timestamp (seconds) when the window resets. |

Inference endpoints (chat, image, audio, video) use a per-API-key tier defined via `/api_keys/rate_limits`. See [`venice-api-keys`](../venice-api-keys/SKILL.md) to pre-fetch your caps, and [`venice-billing`](../venice-billing/SKILL.md) for DIEM/USD usage.

## Response headers on `402` (x402)

| Header | Notes |
|---|---|
| `PAYMENT-REQUIRED` | Base64-JSON of the x402 v2 `accepts[]` list. Protocol-level discovery — parse even if you don't parse the JSON body. |

## Retry strategy

### Never retry

- `400` — bad input. Fix the request.
- `401` — bad auth. Fix credentials.
- `403` — not entitled. Don't hammer.
- `415` — wrong `Content-Type`.

### Retry with modification

- `402` (x402) — run top-up then retry.
- `402` (Bearer) — surface to user; top up at venice.ai.
- `422` with `suggested_prompt` — one retry with the safer prompt.

### Retry with backoff

- `429` — back off for at least `X-RateLimit-Reset - now()`. Add jitter.
- `500` / `503` / `504` — exponential backoff (e.g. 0.5s, 1s, 2s, 4s, 8s), capped at ~30s. **3–5 retries max.**
- Use `Idempotency-Key` (e.g. on `/crypto/rpc/{network}`) so retries can't double-bill state-mutating calls.

### Reference retry loop

```ts
async function callVenice<T>(fn: () => Promise<Response>): Promise<T> {
  const maxRetries = 5
  let delay = 500
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const res = await fn()
    if (res.ok) return res.json() as Promise<T>

    const body = await res.clone().json().catch(() => ({}))
    const { status } = res

    if ([400, 401, 403, 415].includes(status)) {
      throw Object.assign(new Error(body.error ?? 'Venice error'), { status, body })
    }

    if (status === 402 && body.code === 'PAYMENT_REQUIRED') {
      await topUpX402(body.suggestedTopUpUsd)
      continue
    }

    if (status === 422 && body.suggested_prompt) {
      throw Object.assign(new Error('Content policy'), { status, body })
    }

    if (status === 429) {
      const reset = Number(res.headers.get('x-ratelimit-reset')) * 1000
      const wait = Math.max(reset - Date.now(), delay) + Math.random() * 250
      await sleep(wait)
      delay *= 2
      continue
    }

    if (status >= 500 && attempt < maxRetries) {
      await sleep(delay + Math.random() * 250)
      delay *= 2
      continue
    }

    throw Object.assign(new Error(body.error ?? 'Venice error'), { status, body })
  }
  throw new Error('Exceeded max retries')
}
```

## Streaming errors

Streaming responses (`stream: true` on chat, TTS, video-queue progress) deliver mid-stream errors as SSE events:

```
data: {"error": {"type": "…", "message": "…"}}
```

Treat them as terminal — the underlying connection is closed. The HTTP status is `200` because a successful stream can't be changed mid-flight.

## Request-ID correlation

Keep the `X-Request-ID` header from every response (always present on `/crypto/rpc/*`, typically on others). Include it in support tickets — Venice keys diagnostic logs by this ID.

## Common gotchas

- A `402` from `/x402/top-up` with no `X-402-Payment` header is the **expected discovery** response, not an error. See [`venice-x402`](../venice-x402/SKILL.md).
- A `500` on `/chat/completions` with a huge file upload often means the upstream model chose to abort — reduce `max_tokens` / image size rather than blindly retrying.
- `429` on `/crypto/rpc/{network}` may mean the **24-hour credit cap** tripped, not the per-minute one. Check `customMessage`.
- `DetailedError.details` is a Zod `_errors` tree, not a flat map. Walk it recursively.
- Some endpoints (image generation) echo `X-Rate-Limit` variants — treat any header whose name starts with `X-RateLimit` as advisory.
- Don't treat an empty `stream` chunk as an error — send-keepalives look like `data: [DONE]` or empty lines.
