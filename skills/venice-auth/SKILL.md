---
name: venice-auth
description: Authenticate to the Venice API with a Bearer API key or with an x402 / SIWE wallet. Covers header formats, the SIWE message fields, TTL and nonce rules, the @venice-ai/x402-client SDK, and how to choose between the two modes.
---

# Venice Authentication

Every Venice endpoint accepts **one of two** auth schemes, declared in the OpenAPI spec as `BearerAuth` and `siwx`. Both are first-class — pick whichever fits the deployment.

## Use when

- You're making your first call to `api.venice.ai`.
- You're building a server-side integration (usually Bearer) or an agent / no-account wallet flow (x402).
- You hit `401 Authentication failed` and need to check header format.
- You're implementing SIWE signing manually instead of using the SDK.

## Option A — Bearer API key

```http
Authorization: Bearer <VENICE_API_KEY>
```

- Create keys at <https://venice.ai/settings/api> or via [`venice-api-keys`](../venice-api-keys/SKILL.md).
- Keys carry `consumptionLimits` (USD and/or DIEM caps) and `apiKeyType` (`ADMIN` or `INFERENCE`).
- Billing draws from DIEM (staked), USD balance, and bundled credits in order.
- Key types determine which endpoints are reachable — only `ADMIN` keys can manage other keys.

```bash
curl https://api.venice.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $VENICE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-3.3-70b",
    "messages": [{"role":"user","content":"hello"}]
  }'
```

Use the Bearer scheme when you have a Venice account, want usage analytics (`/billing/usage-analytics`), want to issue scoped child keys, or need DIEM / bundled credit priority.

## Option B — x402 wallet (SIWE)

Authenticate with an Ethereum wallet. No account needed. Pay per request in USDC on Base (chain ID `8453`). Balance lives under your wallet address and is consumed automatically.

### Header

```http
X-Sign-In-With-X: <base64(json)>
```

Where the decoded JSON is:

```json
{
  "address":   "0x... (checksummed)",
  "message":   "<SIWE message string from SiweMessage.prepareMessage()>",
  "signature": "0x... (hex)",
  "timestamp": 1712659200000,
  "chainId":   8453
}
```

### SIWE message fields (EIP-4361)

| Field | Value |
|---|---|
| `domain` | `outerface.venice.ai` |
| `uri` | `https://outerface.venice.ai` |
| `version` | `"1"` |
| `chainId` | `8453` |
| `address` | the wallet's checksummed address |
| `statement` | `"Sign in to Venice API"` |
| `nonce` | random 16-char hex, single-use |
| `issuedAt` / `expirationTime` | ISO-8601, recommended TTL **10 minutes** |

The header is short-lived — generate a fresh one before every burst of requests (or cache ~8 minutes, leaving safety margin).

### Manual signing (TypeScript)

```ts
import { Wallet } from 'ethers'
import { SiweMessage } from 'siwe'

const wallet = new Wallet(process.env.WALLET_KEY!)

function makeSiwxHeader() {
  const msg = new SiweMessage({
    domain: 'outerface.venice.ai',
    address: wallet.address,
    statement: 'Sign in to Venice API',
    uri: 'https://outerface.venice.ai',
    version: '1',
    chainId: 8453,
    nonce: crypto.randomUUID().replace(/-/g, '').slice(0, 16),
    issuedAt: new Date().toISOString(),
    expirationTime: new Date(Date.now() + 10 * 60_000).toISOString(),
  })
  const message = msg.prepareMessage()
  const signature = wallet.signMessageSync(message)
  return btoa(JSON.stringify({
    address: wallet.address,
    message,
    signature,
    timestamp: Date.now(),
    chainId: 8453,
  }))
}

const res = await fetch('https://api.venice.ai/api/v1/chat/completions', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-Sign-In-With-X': makeSiwxHeader(),
  },
  body: JSON.stringify({
    model: 'llama-3.3-70b',
    messages: [{ role: 'user', content: 'hello' }],
  }),
})
```

### SDK shortcut

```bash
npm install @venice-ai/x402-client
```

```ts
import { VeniceClient } from '@venice-ai/x402-client'

const venice = new VeniceClient(process.env.WALLET_KEY!)

await venice.topUp(10)            // $10 USDC on Base (first time only)
const res = await venice.chat({
  model: 'llama-3.3-70b',
  messages: [{ role: 'user', content: 'Hello!' }],
})
console.log(res.choices[0].message.content)
```

`VeniceClient` and `createAuthFetch` handle SIWE signing, header rotation, and `402` top-up prompts automatically.

### First-time top-up (wallet → credits)

```
POST /x402/top-up                # WITHOUT X-402-Payment header → returns payment requirements
→ sign a USDC transfer authorization with the x402 SDK (createPaymentHeader)
POST /x402/top-up                # WITH X-402-Payment header → credits land on your wallet address
```

See [`venice-x402`](../venice-x402/SKILL.md) for the full flow.

## Choosing between the two

| Need | Pick |
|---|---|
| Server-side dashboard with usage analytics | Bearer |
| Scoped child keys, consumption limits per app | Bearer |
| DIEM-staked users / bundled credits | Bearer |
| Serverless function that pays per call | x402 |
| Agents with an on-chain budget, no account | x402 |
| End-user wallets authing directly (browser extension, mobile wallet) | x402 |
| Team sharing — one seed, many consumers | Bearer (+ child keys) |

Both schemes can co-exist: a Pro user may generate a **Web3 API key** via `POST /api_keys/generate_web3_key` that ties an on-chain wallet to an off-chain key with an EIP-191 signature. See [`venice-api-keys`](../venice-api-keys/SKILL.md).

## Common auth errors

| Status | Likely cause |
|---|---|
| `401 Authentication failed` | bad/expired key, SIWE `expirationTime` in the past, wrong `domain`/`uri` fields, chain id != `8453` |
| `401 This model is only available to Pro users` | using x402 or an INFERENCE key on a gated model — switch to a Pro Bearer key |
| `402 PAYMENT_REQUIRED` (x402) | wallet balance too low; read `topUpInstructions` and top up via `/x402/top-up` |
| `402 INSUFFICIENT_BALANCE` (Bearer) | DIEM + USD + bundled credits are all empty; top up at venice.ai |

## Security hygiene

- Bearer keys behave like passwords — store in a secret manager, rotate on compromise, scope via `consumptionLimits`.
- SIWE requires a private key signer on the client side. For browsers, use a wallet provider (MetaMask, WalletConnect) — do **not** ship raw private keys.
- Use `expirationTime` ≤ 10 minutes and rotate nonces. Never reuse a signed `X-Sign-In-With-X` header across hours or across machines.
- Rate limits are per-key (Bearer) or per-wallet (x402). See [`venice-api-keys`](../venice-api-keys/SKILL.md) and [`venice-errors`](../venice-errors/SKILL.md).
