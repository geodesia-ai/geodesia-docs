# Chat API

**What it does.** `POST /v1/chat/completions` is the endpoint you point your existing OpenAI client at. G1-Proxy screens the prompt, forwards the request to *your* upstream LLM, scores the answer across the [nine detection axes](detection-axes.md), and hands back the ordinary OpenAI response with a `geodesia` object attached. Blocked requests come back with `finish_reason: "content_filter"`, which standard OpenAI clients already handle.

Everything the endpoint does not recognise as a Geodesia control field is **forwarded verbatim** to the upstream — that is what makes it a drop-in replacement for vLLM, SGLang or the OpenAI API itself.

Three wire formats, one pipeline:

| Endpoint | Format |
|---|---|
| `POST /v1/chat/completions` | OpenAI chat (SSE streaming) |
| `POST /api/chat` | Ollama chat (NDJSON streaming) |
| `POST /v1/completions` | OpenAI **legacy** text completion — the proxy wraps `prompt` into a one-message chat and unwraps the result |

---

## Call it

=== "curl"

    ```bash
    curl -s http://localhost:8080/gw/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{
        "model": "my-model",
        "stream": false,
        "messages": [{"role": "user", "content": "What is the capital of France?"}]
      }' | jq '{
        answer:   .choices[0].message.content,
        decision: .glad_decision,
        brake:    .geodesia.brake,
        axes:     .geodesia.axis_energy
      }'
    ```

=== "Python"

    ```python
    from openai import OpenAI

    # Point the official client at the proxy. Nothing else changes.
    client = OpenAI(base_url="http://localhost:8080/gw/v1", api_key="not-needed-locally")

    r = client.chat.completions.create(
        model="my-model",
        messages=[{"role": "user", "content": "What is the capital of France?"}],
        stream=False,
        extra_body={                       # Geodesia extension fields go here
            "context": "",
            "thinking_level": 0,
        },
    )
    print(r.choices[0].message.content)

    # The detection payload rides on the raw response, outside the typed model.
    g = r.model_extra["geodesia"]
    print(r.model_extra["glad_decision"], g["dominant_axis"])
    for axis, e in g["axis_energy"].items():
        print(f"  {axis:20s} p={e['p_detector']:.3f} thr={e['threshold']:.3f} flag={e['flag']}")
    ```

=== "TypeScript"

    ```ts
    import OpenAI from "openai"

    const client = new OpenAI({
      baseURL: "http://localhost:8080/gw/v1",
      apiKey: "not-needed-locally",
    })

    const r = (await client.chat.completions.create({
      model: "my-model",
      stream: false,
      messages: [{ role: "user", content: "What is the capital of France?" }],
      // @ts-expect-error — Geodesia extension fields are not in the OpenAI types
      thinking_level: 0,
    })) as any

    console.log(r.choices[0].message.content)
    console.log(r.glad_decision, r.geodesia.dominant_axis)
    ```

### What comes back

The standard OpenAI body, plus three top-level fields and the `geodesia` object:

```json
{
  "id": "chatcmpl-geodesia-1718000000000",
  "object": "chat.completion",
  "model": "my-model",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "The capital of France is Paris." },
      "finish_reason": "stop"
    }
  ],
  "glad_decision": "passed",
  "glad_mode": "blocking",
  "geodesia": {
    "axis_energy": {
      "halluc_context":    { "p_detector": 0.12, "flag": false, "threshold": 0.6475, "available": true },
      "halluc_closedbook": { "p_detector": 0.08, "flag": false, "threshold": 0.58,   "fact_seeking": true, "available": true },
      "prompt_safety":     { "p_detector": 0.02, "flag": false, "threshold": 0.9215, "available": true },
      "answer_safety":     { "p_detector": 0.03, "flag": false, "threshold": 0.7295, "available": true },
      "jailbreak":         { "p_detector": 0.01, "flag": false, "threshold": 0.9997, "available": true },
      "rag_jailbreak":     { "p_detector": 0.04, "flag": false, "threshold": 0.2501, "available": true },
      "profanity":         { "p_detector": 0.00, "flag": false, "threshold": 0.90,   "available": true },
      "out_of_scope":      { "p_detector": 0.01, "flag": false, "threshold": 0.90,   "available": true },
      "prompt_complexity": { "p_detector": 0.22, "flag": false, "threshold": 0.50,   "available": true }
    },
    "brake": false,
    "dominant_axis": "halluc_context"
  }
}
```

**When the request is blocked**, the content is replaced and `finish_reason` becomes `content_filter`:

```json
{
  "choices": [
    {
      "message": { "role": "assistant", "content": "[Geodesia blocked — prompt safety (input)]" },
      "finish_reason": "content_filter"
    }
  ],
  "glad_decision": "blocked",
  "glad_mode": "blocking",
  "glad_scores": { "safety_decision_rule": "prompt_safety" }
}
```

!!! warning "`out_of_scope` stays silent without a declared scope"
    "Off topic" is undefined until something says what the topic *is*. Send the Application's purpose as a `system` message, or set `policy.scope` once per Application, or that axis will score near zero on everything — see [Detection Axes](detection-axes.md#out_of_scope-off-topic-out-of-scope).

---

## Request reference

### Standard OpenAI fields

Forwarded to the upstream unchanged.

| Field | Type | Required | Description |
|---|---|---|---|
| `messages` | `array` | ✅ | Conversation history. `role` is `user`, `assistant`, `system` or `tool`; `content` is a string or a content-parts array. |
| `model` | `string` | — | Falls back to the gateway's configured `upstream_model`, or the Application's binding. |
| `stream` | `boolean` | — | `true` for SSE. |
| `max_tokens`, `temperature`, `top_p`, `n`, `stop`, `frequency_penalty`, `presence_penalty`, `seed`, `logit_bias`, `response_format`, `tools`, `tool_choice`, … | *(various)* | — | Forwarded verbatim. |
| vLLM-only sampling keys — `top_k`, `min_p`, `min_tokens`, `ignore_eos`, `stop_token_ids`, `best_of`, `repetition_penalty`, `guided_*`, … | *(various)* | — | Also forwarded verbatim to self-hosted upstreams. |

!!! info "Strict upstreams get a whitelist"
    Hosted **OpenAI** and **Azure OpenAI** reject unknown arguments with a `400`. For those two upstream types the proxy keeps only OpenAI-recognised generation parameters and drops the rest — vLLM-only sampling keys *and* any control field a client attached. Self-hosted OpenAI-compatible servers tolerate extras and keep the full drop-in parameter set.

### Geodesia extension fields

Recognised by the proxy and **never** forwarded to the upstream.

| Field | Type | Default | Description |
|---|---|---|---|
| `context` | `string` | `""` | Explicit grounding text. Scored against the answer on the `halluc_context` axis **and** injected into the upstream request so the model can use it. |
| `rag` | `object` | — | Knowledge-base retrieval for this turn: `{collection_id, top_k, rerank, verify, verify_deep}`. See [Knowledge Base](../rag/index.md). |
| `web_search` | `boolean` | `false` | Search the live web, screen every page through the detector, and ground the answer in the safe ones. Requires `GW_WEBSEARCH_ENABLED=1` (the default). See [Live Web Search](web-search.md). |
| `mode` / `glad_mode` | `string` | *(config)* | `"block"` withholds flagged content; `"passthrough"` returns it annotated. Overrides the deployment/Application enforcement for this request only. See [Enforcement Modes](enforcement-modes.md). |
| `threshold_overrides` | `object` | — | Per-axis thresholds for this request. **Only the five base axes are honoured**: `halluc_context`, `halluc_closedbook`, `prompt_safety`, `answer_safety`, `jailbreak`. Keys for the other four are silently ignored — change those in the Application policy instead. |
| `thinking_level` / `glad_thinking_level` | `integer` | `0` | Detection depth, `0`–`3` (`3` = MAX). Clamped, never rejected. Falls back to the deepest level the deployment can serve. See [Thinking Levels](thinking-levels.md). |
| `domain` / `geodesia_domain` | `string` | `"general"` | Selects the domain-conditional calibration bucket for the closed-book axis. `general` / `default` / `all` / empty all mean domain-agnostic. |
| `pass_extra` | `integer` | `1` | Extra answer samples for closed-book uncertainty. Only applied when the upstream exposes log-probabilities and the turn has no explicit context. Each extra sample costs another generation. |
| `self_consistency` | `boolean` | `false` | Shorthand for `pass_extra > 1`. |
| `self_consistency_samples` | `integer` | — | How many extra samples when `self_consistency` is on. |
| `glad_scan` / `scan` | `boolean` | `true` | Set to `false` to **bypass detection entirely** for this request — pure pass-through to the upstream, no scoring, no blocking. |
| `pii_guard` / `glad_pii_guard` | `boolean` | *(config)* | Per-request override of PII redaction. |
| `constitutional_ai` | `boolean` | *(config)* | Per-request override of the constitutional system prompt. Wins over the deployment config and the Application policy. |
| `application_id` / `app_id` | `string` | — | Route this request to a specific Application. Equivalent to the `X-Geodesia-App` header. |
| `session_id` | `string` | — | Groups turns into one conversation in the audit log. |
| `upstream_api_key` / `upstream_type` | `string` | — | Override the upstream binding for this request. |

!!! danger "`glad_scan: false` turns the guardrail off"
    It is there for health checks and for replaying traffic you have already scored. A request served with `glad_scan: false` is not screened, not scored and not blocked — and the response carries no verdict. Do not let application code set it from user input.

### How an Application is resolved

The proxy picks the Application in this order:

1. `application_id` or `app_id` on the body,
2. the `X-Geodesia-App` header,
3. the Application that owns the `g1k_…` key in `Authorization: Bearer …`,
4. otherwise `default`.

An explicit id always wins, including an explicit `"default"`. The API-key fallback exists so a raw client authenticating with nothing but its key still lands on its own Application. A **paused or killed** Application keeps its id on the audit record but is served with the global configuration.

---

## Streaming

With `stream: true` the response is SSE (OpenAI format) or NDJSON (Ollama format).

=== "curl"

    ```bash
    curl -N -s http://localhost:8080/gw/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{"model":"my-model","stream":true,
           "messages":[{"role":"user","content":"Explain TLS in two sentences."}]}'
    ```

=== "Python"

    ```python
    from openai import OpenAI

    client = OpenAI(base_url="http://localhost:8080/gw/v1", api_key="not-needed-locally")

    stream = client.chat.completions.create(
        model="my-model",
        messages=[{"role": "user", "content": "Explain TLS in two sentences."}],
        stream=True,
    )
    for chunk in stream:
        extra = chunk.model_extra or {}
        if "geodesia" in extra:                       # first chunk: input analysis; last: full analysis
            print("\n[geodesia]", extra["geodesia"])
        delta = chunk.choices[0].delta.content if chunk.choices else None
        if delta:
            print(delta, end="", flush=True)
        if chunk.choices and chunk.choices[0].finish_reason == "content_filter":
            print("\n-- halted by Geodesia --")
    ```

=== "TypeScript"

    ```ts
    const stream = await client.chat.completions.create({
      model: "my-model",
      stream: true,
      messages: [{ role: "user", content: "Explain TLS in two sentences." }],
    })

    for await (const chunk of stream) {
      const c = chunk as any
      if (c.geodesia) console.log("\n[geodesia]", c.geodesia)
      process.stdout.write(chunk.choices[0]?.delta?.content ?? "")
      if (chunk.choices[0]?.finish_reason === "content_filter") {
        console.log("\n-- halted by Geodesia --")
      }
    }
    ```

**Where the payload arrives**

1. **First chunk** — the input-region analysis (`prompt_safety`, `jailbreak`, and the other prompt axes) rides on the first empty-delta chunk, so you can show safety status before a single token of the answer.
2. **Final chunk** — or the early-stop chunk — carries the full analysis for every axis.

**The mid-stream brake.** The proxy re-reads the accumulating generation every `cadence_tokens` tokens (default 32). If an output-region axis fires before generation finishes it injects `\n\n[Geodesia: generation halted — energy barrier]`, sends a final chunk with `finish_reason: "content_filter"` and the detection payload, then `data: [DONE]`.

!!! warning "Partial flagged content is visible before the brake"
    A user may see the first tokens of a dangerous answer before it is cut. If your application must never render partial flagged content, use non-streaming mode, or buffer client-side until you see a terminal `finish_reason`.

---

## Ollama format — `POST /api/chat`

The same pipeline in Ollama's wire format. Every Geodesia extension field above works identically.

=== "curl"

    ```bash
    curl -s http://localhost:8080/gw/api/chat \
      -H "Content-Type: application/json" \
      -d '{
        "model": "llama3.2",
        "stream": false,
        "messages": [{"role": "user", "content": "What is the capital of France?"}],
        "context": "",
        "mode": "block"
      }' | jq '{answer: .message.content, decision: .glad_decision}'
    ```

=== "Python"

    ```python
    import httpx

    r = httpx.post("http://localhost:8080/gw/api/chat", json={
        "model": "llama3.2",
        "stream": False,
        "messages": [{"role": "user", "content": "What is the capital of France?"}],
        "mode": "block",
    }, timeout=120).json()
    print(r["message"]["content"], r["glad_decision"])
    ```

=== "TypeScript"

    ```ts
    const r = await fetch("http://localhost:8080/gw/api/chat", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: "llama3.2",
        stream: false,
        messages: [{ role: "user", content: "What is the capital of France?" }],
        mode: "block",
      }),
    }).then(r => r.json())
    console.log(r.message.content, r.glad_decision)
    ```

**What comes back**

```json
{
  "model": "llama3.2",
  "message": { "role": "assistant", "content": "The capital of France is Paris." },
  "done": true,
  "glad_decision": "passed",
  "glad_mode": "blocking",
  "geodesia": { "axis_energy": {}, "brake": false, "dominant_axis": "halluc_context" }
}
```

---

## Response reference

### Top-level extension fields

| Field | Values | Description |
|---|---|---|
| `glad_decision` | `"passed"` \| `"blocked"` | `"blocked"` means at least one axis fired. In passthrough mode the answer is still returned. |
| `glad_mode` | `"blocking"` \| `"passthrough"` | The enforcement mode actually applied to this response. |
| `glad_scores` | `{"safety_decision_rule": "<axis>"}` | Present only when blocked. Names the axis that triggered it. |

### The `geodesia` object

| Field | When present | Description |
|---|---|---|
| `axis_energy` | always | Per-axis `{p_detector, flag, threshold, available}`. `available: false` means the axis could not be scored on this turn — most often `halluc_closedbook` against an upstream with no log-probabilities. |
| `brake` | always | `true` when any answer-region axis fired. |
| `dominant_axis` | always | The axis with the highest score, flagged or not. |
| `routing` | complexity routing on | `{axis, score, threshold, used_complex_model, model}` — see [Token & Cost Control](cost-control.md). |
| `thinking_level` | `thinking_level` ≥ 1 requested **and** servable | The level this turn actually ran at. Absent means the standard path. |
| `thinking_escalated` | level 1 only | Whether the extra work was spent on this turn. |
| `rag` | RAG active | `{sources, n_sources, verification}`. |
| `certificate` | `GW_CERTIFICATE=on` | Signed decision certificate built from this payload's axes. |
| `input` | streaming | The input-region analysis sent in the first chunk. |

Full field-by-field treatment: [API Response Format](../reference/response-format.md).

---

## More examples

### Grounded request — hallucination scored against your own text

```bash
curl -s http://localhost:8080/gw/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-model",
    "stream": false,
    "context": "The Eiffel Tower was built between 1887 and 1889 and stands 330 metres tall.",
    "messages": [{"role":"user","content":"How tall is the Eiffel Tower?"}]
  }' | jq '{answer: .choices[0].message.content, halluc: .geodesia.axis_energy.halluc_context}'
```

### Passthrough — see what the model *would* have said

```bash
curl -s http://localhost:8080/gw/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-model",
    "stream": false,
    "mode": "passthrough",
    "messages": [{"role":"user","content":"How do I pick a lock?"}]
  }' | jq '{answer: .choices[0].message.content, decision: .glad_decision, axis: .glad_scores}'
```

### Loosen two axes for one request

```bash
curl -s http://localhost:8080/gw/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-model",
    "stream": false,
    "threshold_overrides": {"prompt_safety": 0.70, "jailbreak": 0.55},
    "messages": [{"role":"user","content":"Explain a penetration test methodology."}]
  }'
```

### Answer from a knowledge base

```bash
curl -s http://localhost:8080/gw/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "X-Geodesia-App: support_bot" \
  -d '{
    "model": "my-model",
    "stream": false,
    "rag": {"collection_id": "my-policy-docs", "top_k": 5, "verify": true},
    "messages": [{"role":"user","content":"What is our refund policy?"}]
  }' | jq '{answer: .choices[0].message.content, sources: [.geodesia.rag.sources[].title]}'
```

### Route by Application API key

```bash
curl -s http://localhost:8080/gw/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer g1k_live_9c1f2a7b4e0d" \
  -d '{"model":"my-model","stream":false,
       "messages":[{"role":"user","content":"Hello"}]}'
```

No `application_id` needed — the key identifies the Application, and its policy, thresholds, cost centre and upstream binding all apply.
