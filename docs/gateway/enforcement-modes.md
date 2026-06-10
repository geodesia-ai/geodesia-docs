# Enforcement Modes

Geodesia G-1 can operate in two enforcement modes that control what happens when a detection axis flags content. The mode can be set globally (in the gateway configuration) and overridden on a per-request basis.

---

## Modes

### Blocking Mode

In **blocking mode**, flagged content is **withheld** and replaced with a short notice. The end user never sees the problematic content.

- **Input blocking** (`block_input: true`): A prompt that triggers `prompt_safety` or `jailbreak` is refused before it reaches the upstream LLM. The model is never called.
- **Output blocking** (`block_output: true`, default): An answer that triggers `halluc_context`, `halluc_closedbook`, or `answer_safety` is replaced with a block notice. In streaming mode, generation is halted at the next `cadence_tokens` boundary.

**Block notice format:**

```
[Geodesia blocked — answer safety]
```

The notice names the detection axis that triggered the block.

**HTTP response for a blocked request:**

```json
{
  "choices": [{
    "message": {"role": "assistant", "content": "[Geodesia blocked — prompt safety (input)]"},
    "finish_reason": "content_filter"
  }],
  "glad_decision": "blocked",
  "glad_mode": "blocking",
  "glad_scores": {"safety_decision_rule": "prompt_safety"}
}
```

### Passthrough Mode

In **passthrough mode**, the **real answer is returned** even when an axis flags it. The response is annotated with the detection verdict, but no content is withheld.

Passthrough is useful when:

- You are a developer reviewing what the model actually said before making a blocking decision
- You are collecting data to calibrate thresholds
- Your application has its own downstream filtering and needs the raw answer
- You are using Geodesia for monitoring/observability only

**Annotated response in passthrough:**

```json
{
  "choices": [{
    "message": {"role": "assistant", "content": "The full answer, even if flagged."},
    "finish_reason": "stop"
  }],
  "glad_decision": "blocked",
  "glad_mode": "passthrough",
  "glad_scores": {"safety_decision_rule": "answer_safety"},
  "geodesia": {
    "brake": true,
    "axis_energy": {
      "answer_safety": {"p_detector": 0.73, "flag": true, "threshold": 0.57}
    }
  }
}
```

Key distinction: `glad_decision` is `"blocked"` (the axis fired), but `glad_mode` is `"passthrough"` (no content was withheld). Your application can inspect `glad_decision` to decide whether to show the answer.

---

## Setting the Mode

### Globally (gateway configuration)

```bash
# Enable blocking for both input and output (recommended for production)
curl -X POST http://localhost:8800/v1/glad/gateway/config \
  -d '{"block_input": true, "block_output": true}'

# Enable passthrough for monitoring/audit only
curl -X POST http://localhost:8800/v1/glad/gateway/config \
  -d '{"block_input": false, "block_output": false}'
```

Environment variables:

| Variable | Default | Effect |
|---|---|---|
| `GW_BLOCK_INPUT` | `0` | Set to `1` to enable input blocking globally |
| (no env var for output) | `true` | Output blocking is enabled by default |

### Per-request override

Use the `mode` (or `glad_mode`) field to override the global setting for one request:

```bash
# Force passthrough for this request (review mode)
curl -X POST http://localhost:8800/v1/chat/completions \
  -d '{"model":"my-model","messages":[...],"mode":"passthrough","stream":false}'

# Force blocking for this request
curl -X POST http://localhost:8800/v1/chat/completions \
  -d '{"model":"my-model","messages":[...],"mode":"block","stream":false}'
```

**Accepted values for `mode` / `glad_mode`:**

| Value | Effect |
|---|---|
| `"block"`, `"blocking"`, `"enforce"` | Block flagged content (withhold) |
| `"passthrough"`, `"monitor"`, `"annotate"`, `"observe"`, `"score"` | Return answer but annotate |
| (omitted) | Fall back to the gateway's global `block_input`/`block_output` settings |

---

## Mid-Stream Braking

For streaming requests in blocking mode, the gateway performs **mid-stream checks** every `cadence_tokens` tokens (default: every 32 tokens). If an output-region axis fires during generation:

1. The gateway emits the last complete text chunk
2. Appends: `\n\n[Geodesia: generation halted — energy barrier]`
3. Sends the final SSE chunk with `finish_reason: "content_filter"` and the full `geodesia` detection payload
4. Closes the stream with `data: [DONE]`

The user may have received some content before the brake fires. If your application must never display partially-generated flagged content, use non-streaming mode (`stream: false`) or filter on the `finish_reason` field.

```
cadence_tokens: 32   →  Check at tokens 32, 64, 96, ...
                         (Harmful content spanning tokens 50–80 detected at token 96)
```

To change the cadence:

```bash
curl -X POST http://localhost:8800/v1/glad/gateway/config \
  -d '{"cadence_tokens": 16}'   # more aggressive (higher latency)
```

---

## Input vs Output Enforcement

The two blocking flags are independent:

| Config | Effect |
|---|---|
| `block_input=true, block_output=true` | Full enforcement: block dangerous prompts AND dangerous answers |
| `block_input=false, block_output=true` | Log + annotate dangerous prompts, but never show dangerous answers |
| `block_input=true, block_output=false` | Block dangerous prompts, but show all answers (annotated) |
| `block_input=false, block_output=false` | Monitoring only: annotate everything, block nothing |

For RAG deployments where prompt safety is less of a concern (the model only answers from documents), a common setup is `block_input=false, block_output=true` to avoid over-blocking legitimate questions while still preventing the model from returning fabricated answers.

---

## Dominant Axis

When multiple axes flag simultaneously, the **dominant axis** is the one with the highest `p_detector` score among the flagged axes. This is what gets reported in `glad_scores.safety_decision_rule` and in the block notice. It is the most likely cause of the violation.

```json
"glad_scores": {"safety_decision_rule": "jailbreak"}
```

The dominant axis is also stored in `geodesia.dominant_axis`.
