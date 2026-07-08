# Response Format Reference

Every response from the Geodesia G-1 gateway includes a `geodesia` extension object. This page is the authoritative reference for all fields in that object.

---

## Gateway Response Structure

When you make a request to `POST /v1/chat/completions`, the response follows the standard OpenAI format with an additional `geodesia` key:

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1749555000,
  "model": "mistralai/Mistral-7B-v0.3",
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
  "usage": { ... },
  "geodesia": {
    "call_id": "call_abc123",
    "session_id": "sess_xyz",
    "prompt_blocked": false,
    "answer_blocked": false,
    "block_reason": null,
    "dominant_axis": null,
    "axes_available": ["halluc_context", "halluc_closedbook", "prompt_safety", "answer_safety", "jailbreak"],
    "scores": { ... },
    "thresholds": { ... },
    "watermark": { ... },
    "rag": null,
    "latency_ms": 312
  }
}
```

---

## `geodesia` Object

### Top-Level Fields

| Field | Type | Description |
|---|---|---|
| `call_id` | `string` | Unique identifier for this call. Use this to look up the call in the audit chain or oversight queue. |
| `session_id` | `string` | Session grouping identifier. |
| `prompt_blocked` | `boolean` | `true` if the input prompt was blocked before generation. When `true`, `choices[0].message.content` is `null` (or contains a block notice). |
| `answer_blocked` | `boolean` | `true` if the generated answer was withheld due to a detection threshold. |
| `block_reason` | `string` \| `null` | Plain-language reason for blocking. `null` if not blocked. |
| `dominant_axis` | `string` \| `null` | Which detection axis was the primary reason for blocking when multiple axes triggered. |
| `axes_available` | `array[string]` | List of detection axes that were actually computed. Depends on which model is loaded and whether context was provided. |
| `scores` | `object` | Per-axis detection scores. See below. |
| `thresholds` | `object` | The threshold values that were applied (after any per-request overrides). |
| `watermark` | `object` | Watermark metadata. See below. |
| `rag` | `object` \| `null` | RAG retrieval results, if RAG was triggered. See below. |
| `latency_ms` | `integer` | Total wall-clock latency for this request in milliseconds. |

---

### `scores` Object

Contains one field per available detection axis:

```json
"scores": {
  "prompt_safety": {
    "score": 0.12,
    "logit": -1.98,
    "triggered": false
  },
  "answer_safety": {
    "score": 0.08,
    "logit": -2.44,
    "triggered": false,
    "combined_score": 0.07,
    "combined_logit": -2.51,
    "combined_threshold": 0.11,
    "combined_triggered": false,
    "per_signal": { ... }
  },
  "halluc_context": {
    "score": 0.23,
    "logit": -1.19,
    "triggered": false,
    "combined_score": 0.18,
    "combined_logit": -1.52,
    "combined_threshold": 0.617,
    "combined_triggered": false,
    "per_signal": { ... },
    "n_signals": 10
  },
  "halluc_closedbook": {
    "score": 0.09,
    "triggered": false
  },
  "jailbreak": {
    "score": 0.03,
    "triggered": false
  }
}
```

#### Per-Axis Score Fields

| Field | Description |
|---|---|
| `score` | Calibrated probability [0–1]. **Use this for display and threshold comparison.** Higher = more likely the issue is present. |
| `logit` | Raw logit (uncalibrated). Present for advanced users and debugging only. |
| `triggered` | `true` if `score` exceeded the configured threshold |
| `combined_score` | For `halluc_context` and `answer_safety`: the output of the calibrated multi-signal combiner, which aggregates several internal detection signals. This is the definitive score used for blocking decisions. |
| `combined_logit` | Raw logit from the combiner |
| `combined_threshold` | The threshold used for the combined score decision |
| `combined_triggered` | Whether the combined score triggered a block decision |
| `per_signal` | Breakdown of each internal signal's contribution to the combined score |
| `n_signals` | Number of signals included in the combined score |

!!! info "Which score to use"
    Always use `combined_score` when it is present (for `halluc_context` and `answer_safety`). The `combined_score` is calibrated with AUROC > 0.97 and is more reliable than the raw `score`. Use raw `score` for axes that do not have a combiner (`halluc_closedbook`, `jailbreak`, `prompt_safety`).

---

### `thresholds` Object

The thresholds that were applied to produce the `triggered` values. These may differ from the default configuration if per-request `threshold_overrides` were provided.

```json
"thresholds": {
  "prompt_safety": 0.35,
  "answer_safety": 0.11,
  "halluc_context": 0.617,
  "halluc_closedbook": 0.50,
  "jailbreak": 0.50
}
```

---

### `watermark` Object

```json
"watermark": {
  "token": "hmac:v1:a8b3c1d4e5f6...",
  "generated_by": "Geodesia G-1",
  "call_id": "call_abc123",
  "timestamp": "2026-06-10T10:23:45Z",
  "disclosure": "This content was generated by an AI system."
}
```

| Field | Description |
|---|---|
| `token` | HMAC-SHA256 token for verification. Pass to `POST /v1/glad/watermark/verify` to confirm authorship. |
| `generated_by` | System identifier |
| `call_id` | Matches `geodesia.call_id` |
| `timestamp` | ISO 8601 generation timestamp |
| `disclosure` | Plain-language AI disclosure string (configurable in `watermark.disclosure_text`) |

---

### `rag` Object

Present when RAG retrieval was triggered for this request.

```json
"rag": {
  "retrieved_chunks": [
    {
      "text": "Paris is the capital of France.",
      "source": "geography-kb/europe.pdf",
      "score": 0.97,
      "rank": 1
    }
  ],
  "collection_id": "my-knowledge-base",
  "query_used": "capital of France",
  "n_retrieved": 3,
  "reranked": true
}
```

| Field | Description |
|---|---|
| `retrieved_chunks` | List of retrieved document chunks, sorted by relevance |
| `retrieved_chunks[].text` | The chunk text |
| `retrieved_chunks[].source` | Source document identifier |
| `retrieved_chunks[].score` | Retrieval relevance score (0–1) |
| `retrieved_chunks[].rank` | Rank position after reranking |
| `collection_id` | Which RAG collection was queried |
| `query_used` | The query sent to the vector store (may differ from the original prompt if the gateway rewrote it) |
| `n_retrieved` | Number of chunks retrieved |
| `reranked` | Whether the BGE reranker was applied |

---

## Blocked Response

When `prompt_blocked` or `answer_blocked` is `true` in blocking mode, the response `content` is replaced with a block notice:

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "[Response withheld: answer_safety threshold exceeded (score: 0.78, threshold: 0.11)]"
      },
      "finish_reason": "content_filter"
    }
  ]
}
```

In passthrough mode, the original response is delivered and `answer_blocked` is set to `true` as an annotation.

---

## Streaming Format

In streaming mode (`stream: true`), the `geodesia` object is delivered in the final chunk — the chunk with `finish_reason: "stop"` or `"content_filter"`. All preceding chunks are standard OpenAI SSE format without the `geodesia` extension.

```
data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":"Paris"},...}],...}
data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":"."},...}],...}
data: {"id":"chatcmpl-abc","choices":[{"delta":{},"finish_reason":"stop"}],"geodesia":{...}}
data: [DONE]
```
