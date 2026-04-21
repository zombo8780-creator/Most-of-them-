---
name: venice-responses
description: Use Venice's Alpha POST /responses endpoint - an OpenAI-compatible Responses API with typed output blocks (reasoning, message, function_call, web_search_call). Covers request shape, streaming, differences from /chat/completions, and limitations (stateless, no E2EE, no venice_parameters).
---

# Venice Responses API (Alpha)

`POST /api/v1/responses` is Venice's OpenAI-compatible Responses endpoint. It returns a **structured, typed output array** instead of a single `message.content` string — ideal for agents that need to separate reasoning, messages, tool calls, and built-in tool events.

> **Alpha.** Restricted access during Alpha. Request access before relying on it in production. Schemas may change.

## Use when

- You need the OpenAI Responses-style response shape (`output[]` with typed `type: "reasoning" | "message" | "function_call" | "web_search_call"` blocks) for a client library that expects it.
- You want clean separation of reasoning vs message vs tool-call output.
- You want streaming via SSE with typed events.

Otherwise use [`venice-chat`](../venice-chat/SKILL.md) — it has more features, more models, and full Venice parameters.

## Limitations vs `/chat/completions`

| Limitation | Detail |
|---|---|
| **Stateless** | No conversation persistence across requests. Send the full history each call. |
| **No E2EE** | E2EE-capable models reject `/responses`. Use `/chat/completions` with E2EE headers for encrypted inference. |
| **No `venice_parameters`** | Use model feature suffixes (e.g. `:web_search=on`) or OpenAI-native params instead. |
| **Alpha cohort only** | `401` if your key isn't whitelisted for Alpha. |

## Authentication

Same as the rest of the API — either `Authorization: Bearer <key>` or `X-Sign-In-With-X: <SIWE>`. See [`venice-auth`](../venice-auth/SKILL.md).

## Minimal request

```bash
curl https://api.venice.ai/api/v1/responses \
  -H "Authorization: Bearer $VENICE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-3.3-70b",
    "input": "Explain why the sky is blue in one paragraph."
  }'
```

`input` accepts:
- a plain string, or
- an array of typed input items (similar to `chat/completions` message parts) for multi-turn or multimodal history.

## Response shape

```json
{
  "id": "resp_abc123",
  "object": "response",
  "created_at": 1735689600,
  "model": "llama-3.3-70b",
  "status": "completed",
  "output": [
    {
      "type": "reasoning",
      "id": "rs_1",
      "summary": ["I considered Rayleigh scattering..."],
      "encrypted_content": "..."
    },
    {
      "type": "message",
      "id": "msg_1",
      "status": "completed",
      "role": "assistant",
      "content": [{
        "type": "output_text",
        "text": "The sky is blue because...",
        "annotations": [{
          "type": "url_citation",
          "url": "https://example.com/rayleigh",
          "title": "Rayleigh scattering",
          "start_index": 42,
          "end_index": 99
        }]
      }]
    },
    {
      "type": "function_call",
      "id": "fc_1",
      "call_id": "call_abc",
      "name": "get_weather",
      "arguments": "{\"city\":\"Paris\"}",
      "status": "completed"
    },
    {
      "type": "web_search_call",
      "id": "ws_1",
      "status": "completed"
    }
  ],
  "usage": {
    "input_tokens": 20,
    "input_tokens_details": {"cached_tokens": 0},
    "output_tokens": 80,
    "output_tokens_details": {"reasoning_tokens": 40},
    "total_tokens": 100
  }
}
```

Top-level `status` ∈ `completed` | `failed` | `in_progress` | `cancelled`. On `failed`, `error.code` and `error.message` are populated.

## Output block types

| `type` | Purpose |
|---|---|
| `reasoning` | Thought process from reasoning models. `summary[]` holds human-readable text; `encrypted_content` holds opaque signatures — round-trip verbatim for multi-turn tool calls. |
| `message` | Main text output. `content[].type === "output_text"`, plus `annotations[]` for `url_citation` entries from web search. |
| `function_call` | Tool call: `name`, stringified-JSON `arguments`, `call_id`. |
| `web_search_call` | Sentinel showing the built-in web_search tool fired; use alongside `url_citation` annotations on messages. |

Match tool outputs back by `call_id` when continuing the turn.

## Common request fields (OpenAI-compatible)

| Field | Notes |
|---|---|
| `model` | Required. Model ID, trait, or compatibility mapping. Feature suffixes allowed. |
| `input` | Required. String or input-items array. |
| `instructions` | System/developer instruction string (Responses equivalent of a leading system message). |
| `tools` | Array of `{type:"function",function:{...}}`, `{type:"web_search"}`, `{type:"x_search"}`, `{type:"code_interpreter"}`, `{type:"file_search"}`, or `{type:"computer_use_preview"}` — availability depends on the model. |
| `tool_choice` | `"auto"` / `"required"` / `"none"` / `{type:"function",function:{"name":"..."}}`. |
| `parallel_tool_calls` | Boolean. |
| `reasoning.effort` / `reasoning.summary` | Reasoning controls for thinking models. |
| `response_format` | `{type:"json_schema", json_schema:{...}}` for structured output. |
| `stream` | Boolean. SSE response with typed events (`response.created`, `response.output_item.added`, `response.output_text.delta`, `response.completed`, …). |
| `metadata` | Key/value pairs. |

## Streaming

With `stream: true`, the response is an SSE stream of typed events. Typical flow:

```
event: response.created
event: response.output_item.added        # type=reasoning
event: response.reasoning_summary.delta
event: response.output_item.added        # type=message
event: response.output_text.delta
event: response.output_text.delta
event: response.output_item.done
event: response.completed
```

Consume events in order and reconstruct `output[]` client-side; the shape on `response.completed` matches the non-streamed response exactly.

## Authentication & error responses

- `401` — auth failed **or** this model is only available to Pro users (Alpha cohort / gated models).
- `402` — insufficient balance. Bearer → `INSUFFICIENT_BALANCE`. x402 → `PAYMENT_REQUIRED` with `topUpInstructions` and `siwxChallenge`.
- `429` — rate-limited.
- `500` — inference failed.

`X-Balance-Remaining` is on 200 responses when using x402 auth; `PAYMENT-REQUIRED` header on 402.

## Migration notes

- Port `messages` → pass as `input` (string or typed array).
- `venice_parameters.character_slug` → **not supported**; inline the character's system prompt via `instructions`.
- `venice_parameters.enable_web_search` → append `:web_search=on` to the model ID **or** add `{"type":"web_search"}` to `tools`.
- `venice_parameters.strip_thinking_response` → append `:strip_thinking_response=true` to model ID.
- Full E2EE flow → stay on `/chat/completions`.
