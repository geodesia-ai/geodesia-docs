# Chat API

The gateway exposes two chat endpoints: one in the **OpenAI** wire format and one in the **Ollama** wire format. Both accept the same extended set of parameters and return the same structure — just wrapped in the respective format.

---

## POST /v1/chat/completions

OpenAI-compatible chat endpoint. Drop-in replacement for `https://api.openai.com/v1/chat/completions`.

### Request Body

All standard OpenAI fields are accepted verbatim and forwarded to the upstream. Geodesia adds the following optional fields:

#### Standard OpenAI Fields (forwarded)

| Field | Type | Required | Description |
|---|---|---|---|
| `messages` | `array` | ✅ | Conversation history. Each element has `role` (`user`, `assistant`, `system`, `tool`) and `content` (string or content-parts array). |
| `model` | `string` | ✅ | Model name. If omitted, defaults to the gateway's `upstream_model`. |
| `stream` | `boolean` | — | `true` for Server-Sent Events (SSE) streaming, `false` for a single JSON response. Default `true` in the gateway. |
| `max_tokens` | `integer` | — | Maximum tokens to generate. The gateway clamps values that exceed the model's context window rather than raising an error. |
| `temperature` | `float` | — | Sampling temperature (0 = deterministic, 1 = creative). |
| `top_p` | `float` | — | Nucleus sampling probability (0–1). |
| `n` | `integer` | — | Number of completions to generate. Forwarded directly to the upstream. |
| `stop` | `string` \| `array` | — | Stop sequences. Forwarded directly. |
| `frequency_penalty` | `float` | — | Forwarded directly. |
| `presence_penalty` | `float` | — | Forwarded directly. |

#### Geodesia Extensions

| Field | Type | Default | Description |
|---|---|---|---|
| `context` | `string` | `""` | Explicit grounding context. Geodesia scores the answer against this text using the faithfulness detection axis (`halluc_context`). It is also injected into the upstream request so the model can use it when answering. |
| `mode` / `glad_mode` | `string` | (gateway config) | Per-request enforcement mode. `"block"` withholds flagged content. `"passthrough"` returns the answer but annotates it. Overrides the gateway's configured `block_input`/`block_output` for this request only. See [Enforcement Modes](enforcement-modes.md). |
| `threshold_overrides` | `object` | — | Per-axis detection probability thresholds for this request only. Example: `{"prompt_safety": 0.75, "jailbreak": 0.60}`. Supported keys: `halluc_context`, `halluc_closedbook`, `prompt_safety`, `answer_safety`, `jailbreak`. These take precedence over calibrated defaults and the gateway's configured `thresholds`. |
| `rag` | `object` | — | Knowledge base configuration for this request. See [Knowledge Base](../rag/index.md). Fields: `collection_id` (string, required), `top_k` (int, default 5), `rerank` (bool, default true), `verify` (bool, default true), `verify_deep` (bool, default true). |
| `pass_extra` | `integer` | `1` | Number of independent answer samples to draw for closed-book uncertainty estimation. Values > 1 cause the gateway to re-sample the answer `N` times at temperature 0.8 and compute consistency signals. Only applied when the upstream supports log-probabilities and the turn has no explicit context. Use sparingly — each extra sample doubles the generation cost. |
| `self_consistency` | `boolean` | `false` | Shorthand for `pass_extra > 1`. When `true`, the gateway draws `self_consistency_samples` extra samples. |
| `self_consistency_samples` | `integer` | — | Number of extra samples when `self_consistency` is `true`. |

### Response Body

The standard OpenAI response is returned unchanged, with three additional top-level fields:

```json
{
  "id": "chatcmpl-geodesia-1718000000000",
  "object": "chat.completion",
  "model": "my-model",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "The capital of France is Paris."
      },
      "finish_reason": "stop"
    }
  ],
  "glad_decision": "passed",
  "glad_mode": "blocking",
  "geodesia": {
    "axis_energy": {
      "halluc_context": { "p_detector": 0.12, "flag": false, "threshold": 0.35, "available": true },
      "halluc_closedbook": { "p_detector": 0.08, "flag": false, "threshold": 0.50, "fact_seeking": true, "available": true },
      "prompt_safety": { "p_detector": 0.02, "flag": false, "threshold": 0.90, "available": true },
      "answer_safety": { "p_detector": 0.03, "flag": false, "threshold": 0.57, "available": true },
      "jailbreak": { "p_detector": 0.01, "flag": false, "threshold": 0.57, "available": true }
    },
    "brake": false,
    "dominant_axis": "halluc_context"
  }
}
```

#### Top-level Extension Fields

| Field | Values | Description |
|---|---|---|
| `glad_decision` | `"passed"` \| `"blocked"` | Overall decision. `"blocked"` means at least one axis was flagged (the answer may still be returned in passthrough mode). |
| `glad_mode` | `"blocking"` \| `"passthrough"` | Enforcement mode that was applied to this response. |
| `glad_scores` | `{"safety_decision_rule": "axis_name"}` | Present only when `glad_decision` is `"blocked"`. Identifies which axis triggered the block. |

#### The `geodesia` Object

See [API Response Format](../reference/response-format.md) for the full structure. In brief:

| Field | Description |
|---|---|
| `axis_energy` | Per-axis detection results (score, flag, threshold, availability) |
| `brake` | `true` if any answer-region axis is flagged |
| `dominant_axis` | The axis with the highest detection score |
| `energy_dHmax_joule` | (internal diagnostic; not used in standard integrations) |
| `rag` | Present when RAG is active; contains `sources`, `n_sources`, and `verification` results |
| `input` | Present when streaming — the input-region analysis sent in the first SSE chunk |

#### When the request is blocked

If blocking mode is on and an axis fires, the `content` field is replaced with a block notice:

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "[Geodesia blocked — prompt safety (input)]"
      },
      "finish_reason": "content_filter"
    }
  ],
  "glad_decision": "blocked",
  "glad_mode": "blocking"
}
```

The `finish_reason` is `"content_filter"` (compatible with standard OpenAI client handling).

---

## POST /api/chat

Ollama-compatible chat endpoint. Identical behaviour to `/v1/chat/completions` but uses Ollama's request/response wire format.

### Request Body

```json
{
  "model": "llama3.2",
  "stream": true,
  "messages": [
    {"role": "user", "content": "What is the capital of France?"}
  ],
  "context": "",
  "mode": "block",
  "threshold_overrides": {}
}
```

All Geodesia extension fields (`context`, `mode`, `threshold_overrides`, `rag`, `pass_extra`) are supported identically.

### Response Body

```json
{
  "model": "llama3.2",
  "message": {
    "role": "assistant",
    "content": "The capital of France is Paris."
  },
  "done": true,
  "glad_decision": "passed",
  "glad_mode": "blocking",
  "geodesia": { ... }
}
```

---

## Streaming

When `stream: true`, responses are delivered as Server-Sent Events (SSE) for OpenAI format or NDJSON for Ollama format.

### Early detection in streams

For streaming requests, Geodesia evaluates the accumulating generation every `cadence_tokens` tokens (default 32). If an output-region axis fires before the generation is complete:

1. The gateway injects a mid-stream break notice: `\n\n[Geodesia: generation halted — energy barrier]`
2. It sends the final SSE chunk with `finish_reason: "content_filter"` and the detection payload in the `geodesia` field
3. It sends `data: [DONE]` and closes the stream

This means a user may see the beginning of a response before it is cut off. If your application should never show partial flagged content, use non-streaming mode or apply client-side filtering on `finish_reason`.

### Detection payload in streams

For streaming responses, the detection payload arrives in two places:

1. **First chunk** — the `input` analysis (prompt-region axes: `prompt_safety`, `jailbreak`) is attached to the very first empty-delta chunk so you can display safety status immediately.
2. **Final chunk** (or the early-stop chunk) — the full output analysis for all 5 axes is attached to the last chunk.

---

## Working Examples

### Basic request — blocking mode

```bash
curl -s http://localhost:8800/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-model",
    "stream": false,
    "messages": [{"role":"user","content":"What is the capital of France?"}]
  }' | jq '{answer: .choices[0].message.content, decision: .glad_decision, brake: .geodesia.brake}'
```

### Grounded request with context

```bash
curl -s http://localhost:8800/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-model",
    "stream": false,
    "context": "The Eiffel Tower was built between 1887 and 1889 and stands 330 metres tall.",
    "messages": [{"role":"user","content":"How tall is the Eiffel Tower?"}]
  }' | jq '{answer: .choices[0].message.content, halluc_context: .geodesia.axis_energy.halluc_context}'
```

### Passthrough mode — review what the model would have said

```bash
curl -s http://localhost:8800/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-model",
    "stream": false,
    "mode": "passthrough",
    "messages": [{"role":"user","content":"How do I pick a lock?"}]
  }' | jq '{answer: .choices[0].message.content, decision: .glad_decision, axis: .glad_scores}'
```

### Lower the safety threshold for a specific request

```bash
curl -s http://localhost:8800/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-model",
    "stream": false,
    "threshold_overrides": {"prompt_safety": 0.70, "jailbreak": 0.55},
    "messages": [{"role":"user","content":"Explain a penetration test methodology."}]
  }'
```

### Request with RAG knowledge base

```bash
curl -s http://localhost:8800/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-model",
    "stream": false,
    "rag": {"collection_id": "my-policy-docs", "top_k": 5, "verify": true},
    "messages": [{"role":"user","content":"What is our refund policy?"}]
  }' | jq '{answer: .choices[0].message.content, sources: .geodesia.rag.sources[].title}'
```
