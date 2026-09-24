# Enforcement Modes

A flagged axis has to *do* something. **Blocking** withholds the content; **passthrough** returns it with
the verdict attached. The choice is a deployment default, overridable per Application and per request —
and it is what separates a guardrail from a monitor.

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

Use the `mode` field to override the global setting for one request:

```bash
# Force passthrough for this request (review mode)
curl -X POST http://localhost:8800/v1/chat/completions \
  -d '{"model":"my-model","messages":[...],"mode":"passthrough","stream":false}'

# Force blocking for this request
curl -X POST http://localhost:8800/v1/chat/completions \
  -d '{"model":"my-model","messages":[...],"mode":"block","stream":false}'
```

**Accepted values for `mode`:**

| Value | Effect |
|---|---|
| `"block"`, `"blocking"`, `"enforce"` | Block flagged content (withhold) |
| `"passthrough"`, `"monitor"`, `"annotate"`, `"observe"`, `"score"` | Return answer but annotate |
| (omitted) | Fall back to the gateway's global `block_input`/`block_output` settings |

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
  "geodesia": {
    "schema_version": "1.0",
    "event": "final",
    "decision": "blocked",
    "mode": "blocking",
    "reason": {"stage": "input", "axis": "prompt_safety", "detail": null},
    "axes": {
      "prompt_safety": {"score": 0.9143, "threshold": 0.6377, "flagged": true, "available": true, "role": "enforce"}
    }
  }
}
```

(Other axes omitted for brevity.)

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
  "geodesia": {
    "schema_version": "1.0",
    "event": "final",
    "decision": "flagged",
    "mode": "passthrough",
    "reason": {"stage": "output", "axis": "answer_safety", "detail": null},
    "axes": {
      "answer_safety": {"score": 0.73, "threshold": 0.57, "flagged": true, "available": true, "role": "enforce"}
    }
  }
}
```

Key distinction: `decision` is `"flagged"` (an enforcing axis fired, but the content was delivered) and `mode` is
`"passthrough"`. In blocking mode the same turn would report `"blocked"` and withhold the answer. To ask "was this
turn a violation?" regardless of mode, test `geodesia.decision in ("flagged", "blocked")`:

```python
g = resp["geodesia"]
if g["decision"] in ("flagged", "blocked"):
    print("violation on", g["reason"]["stage"], "axis", g["reason"]["axis"])
```

See the [Response Format reference](../reference/response-format.md#decision) for the full decision table.

---

## Mid-Stream Braking

For streaming requests in blocking mode, the gateway performs **mid-stream checks** every `cadence_tokens` tokens (default: every 32 tokens). If an output-region axis fires during generation:

1. The gateway emits the last complete text chunk
2. Appends: `\n\n[Geodesia: generation halted — energy barrier]`
3. Sends the final SSE chunk with `finish_reason: "content_filter"` and the `geodesia` object with `event: "final"`, `decision: "blocked"` and `reason: {"stage": "output", "detail": "halted mid-stream"}`
4. Closes the stream with `data: [DONE]`

The user may have received some content before the brake fires. If your application must never display partially-generated flagged content, use non-streaming mode (`stream: false`) or filter on the `finish_reason` field. Take the verdict only from the `final` event; `input_scan` and `progress` events are informational (see [Streaming](../reference/response-format.md#streaming)).

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

## Deciding Axis

When multiple axes flag simultaneously, the **deciding axis** is the one with the highest `score` among the flagged axes. It is reported in `geodesia.reason.axis` and named in the block notice. It is the most likely cause of the violation.

```json
"reason": {"stage": "input", "axis": "jailbreak", "detail": null}
```

`reason.stage` says where the decision was taken (`input`, `output`, `tools` or `quota`); `reason` is `null` when the decision is `allowed`.
