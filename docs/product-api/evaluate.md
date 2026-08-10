# Evaluate Endpoint

**What it does.** `POST /v1/glad/evaluate` on **G1-Proxy** generates an answer from your upstream LLM *and* scores it, in one HTTP round-trip. It is the batch-workflow twin of the [Chat API](../g1-proxy/chat-api.md): same pipeline, same detection, but you send a bare `prompt` instead of a message array and you get the full detection payload back rather than a chat-shaped body.

Reach for it when you are scoring a corpus, running an offline evaluation, or wiring detection into something that is not a chat client.

---

## Call it

=== "curl"

    ```bash
    curl -s http://localhost:8080/gw/v1/glad/evaluate \
      -H "Content-Type: application/json" \
      -H "X-Geodesia-App: support_bot" \
      -d '{
        "model": "my-model",
        "prompt": "How tall is the Eiffel Tower?",
        "context": "The Eiffel Tower was built between 1887 and 1889 and stands 330 metres tall."
      }' | jq '{
        answer:   .choices[0].message.content,
        decision: .glad_decision,
        axes:     .geodesia.axis_energy
      }'
    ```

=== "Python"

    ```python
    import httpx

    rows = [
        {"prompt": "How tall is the Eiffel Tower?",
         "context": "The Eiffel Tower … stands 330 metres tall."},
        {"prompt": "Summarise our refund policy.", "context": ""},
    ]

    with httpx.Client(base_url="http://localhost:8080/gw",
                      headers={"X-Geodesia-App": "support_bot"},
                      timeout=120) as c:
        for row in rows:
            r = c.post("/v1/glad/evaluate", json={"model": "my-model", **row}).json()
            axes = r["geodesia"]["axis_energy"]
            flagged = [a for a, e in axes.items() if e.get("flag")]
            print(f"{r['glad_decision']:8s} flagged={flagged or '-'}  {row['prompt'][:40]}")
    ```

=== "TypeScript"

    ```ts
    const rows = [
      { prompt: "How tall is the Eiffel Tower?", context: "…stands 330 metres tall." },
      { prompt: "Summarise our refund policy.", context: "" },
    ]

    for (const row of rows) {
      const r = await fetch("http://localhost:8080/gw/v1/glad/evaluate", {
        method: "POST",
        headers: { "Content-Type": "application/json", "X-Geodesia-App": "support_bot" },
        body: JSON.stringify({ model: "my-model", ...row }),
      }).then(r => r.json())

      const flagged = Object.entries(r.geodesia.axis_energy)
        .filter(([, e]: any) => e.flag)
        .map(([a]) => a)
      console.log(r.glad_decision, flagged, row.prompt.slice(0, 40))
    }
    ```

### What comes back

An OpenAI-shaped body carrying the full detection payload — the same structure the [Chat API](../g1-proxy/chat-api.md#what-comes-back) returns:

```json
{
  "choices": [
    { "index": 0,
      "message": { "role": "assistant", "content": "It stands 330 metres tall." },
      "finish_reason": "stop" }
  ],
  "glad_decision": "passed",
  "glad_mode": "blocking",
  "geodesia": {
    "axis_energy": {
      "halluc_context":    { "p_detector": 0.07, "flag": false, "threshold": 0.6475, "available": true },
      "halluc_closedbook": { "p_detector": 0.04, "flag": false, "threshold": 0.58,   "available": true },
      "prompt_safety":     { "p_detector": 0.01, "flag": false, "threshold": 0.9215, "available": true },
      "answer_safety":     { "p_detector": 0.02, "flag": false, "threshold": 0.7295, "available": true },
      "jailbreak":         { "p_detector": 0.00, "flag": false, "threshold": 0.9997, "available": true },
      "rag_jailbreak":     { "p_detector": 0.03, "flag": false, "threshold": 0.2501, "available": true },
      "profanity":         { "p_detector": 0.00, "flag": false, "threshold": 0.90,   "available": true },
      "out_of_scope":      { "p_detector": 0.02, "flag": false, "threshold": 0.90,   "available": true },
      "prompt_complexity": { "p_detector": 0.18, "flag": false, "threshold": 0.50,   "available": true }
    },
    "brake": false,
    "dominant_axis": "prompt_complexity"
  }
}
```

Never streamed: this endpoint always returns a single JSON body, whatever `stream` you send.

---

## Request reference

| Field | Type | Required | Description |
|---|---|---|---|
| `prompt` | `string` | ✅¹ | The input to score. |
| `messages` | `array` | ✅¹ | An OpenAI message array, if you would rather send conversation shape. **Wins over `prompt`** when both are present. |
| `model` | `string` | — | Defaults to the Application's binding, or the gateway's configured model. |
| `context` | `string` | — | Grounding text. Drives the `halluc_context` axis and is injected into the generation. |
| `rag` | `object` | — | Retrieve the context instead of supplying it: `{collection_id, top_k, rerank, verify}`. |
| `mode` / `glad_mode` | `string` | — | `block` or `passthrough` for this call. |
| `threshold_overrides` | `object` | — | Per-axis thresholds. **Only the five base axes** are honoured — see [Chat API](../g1-proxy/chat-api.md#geodesia-extension-fields). |
| `thinking_level` | `integer` | — | `0`–`3` (`3` = MAX). See [Thinking Levels](../g1-proxy/thinking-levels.md). |
| `domain` | `string` | — | Domain-conditional calibration bucket for the closed-book axis. |
| `application_id` / `app_id` | `string` | — | Same as the `X-Geodesia-App` header. |
| `pii_guard` | `boolean` | — | Per-request PII redaction override. |

¹ Send one of `prompt` or `messages`. Everything else on the body that is not a Geodesia control field is forwarded to the upstream as a generation parameter.

!!! note "`pass_extra` is fixed at 1 here"
    Unlike the chat endpoint, `/v1/glad/evaluate` always runs a single generation pass. If you want closed-book self-consistency sampling, use `POST /v1/chat/completions` with `pass_extra` or `self_consistency`.

---

## Scoring without generating

If you already have both the prompt **and** the answer and only want them scored — replaying stored traffic, evaluating another system's output, testing a threshold change — do **not** use this endpoint: it will generate a fresh answer. Use the attribution endpoint instead, which scores text you supply:

```bash
curl -s http://localhost:8080/gw/v1/glad/causal-explainability/analyze \
  -H "Content-Type: application/json" \
  -d '{
    "prompt":   "How tall is the Eiffel Tower?",
    "response": "It is 1200 metres tall and located in Berlin.",
    "context":  "The Eiffel Tower … stands 330 metres tall.",
    "method":   "dca"
  }'
```

See [Explainability](explainability.md) and [Causal Explainability](../g1-proxy/causal-xai.md).

---

## Studio-local research endpoints

G-1 Studio also exposes an older, **local-model** evaluation surface at `/glad/evaluate`, `/glad/export_audit` and `/glad/finetune`. These are mounted outside `/v1`, so the unified port on 8080 does not route to them — they are reachable only on Studio's own port (`:8199` by default) and they require a research checkpoint loaded in-process. In the packaged product the Studio backend runs **without** a GPU model, so they are not the path to build on.

| Method | Path | What it does |
|---|---|---|
| `POST` | `/glad/evaluate` | Generate + score against the locally loaded checkpoint. Body: `{model_path, prompt, context?, generation_config?, session_id?, explain?, credit_tiers?, threshold_overrides?, …}`. |
| `POST` | `/glad/export_audit` | Export an audit bundle. Body: `{session_id \| call_ids, client_info?, regulatory_framework?, include_raw_scores?, include_compliance_bundle?, output_path?}`. |
| `POST` | `/glad/finetune` | Submit a fine-tuning job. **202** with a `job_id`. |
| `GET` | `/glad/finetune/status/{job_id}` | `{job_id, status, progress?, current_step?, total_steps?, last_loss?, output_path?, error?}`. `status` is `queued` \| `running` \| `completed` \| `failed`; **404** on an unknown job. |

For anything production, use `POST /v1/glad/evaluate` on G1-Proxy — it scores against your real upstream, honours Application policy, and writes to the compliance ledger.
