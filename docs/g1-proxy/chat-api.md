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
        decision: .geodesia.decision,
        reason:   .geodesia.reason,
        scores:   (.geodesia.axes | map_values(.score))
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

    # The verdict rides on the raw response, outside the typed model.
    g = r.model_extra["geodesia"]
    print(g["decision"], (g["reason"] or {}).get("axis"))   # "allowed" / "flagged" / "blocked"
    for axis, a in g["axes"].items():
        if not a["available"]:                               # not measured is never "clean"
            print(f"  {axis:20s} n/a ({a.get('unavailable_reason')})")
            continue
        print(f"  {axis:20s} score={a['score']:.3f} thr={a['threshold']:.3f} flagged={a['flagged']}")
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
    const g = r.geodesia
    console.log(g.decision, g.reason?.axis ?? null)          // "allowed" / "flagged" / "blocked"
    console.log(g.axes.jailbreak.score, g.axes.jailbreak.threshold, g.axes.jailbreak.flagged)
    ```

### What comes back

The standard OpenAI body with exactly **one** extra top-level key, `geodesia`. Everything Geodesia measured and decided for the turn lives inside it (abridged from a live response; `details` of `halluc_closedbook` and the `certificate` omitted):

```json
{
  "id": "chatcmpl-geodesia-1790234386650",
  "object": "chat.completion",
  "created": 1790234393,
  "model": "ministral3",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "The capital of France is **Paris**." },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 1826, "completion_tokens": 23, "total_tokens": 1849 },
  "geodesia": {
    "schema_version": "1.0",
    "event": "final",
    "decision": "allowed",
    "mode": "blocking",
    "reason": null,
    "axes": {
      "halluc_context":    { "score": null, "threshold": 0.7551, "flagged": false, "available": false,
                             "role": "enforce", "unavailable_reason": "no context supplied" },
      "halluc_closedbook": { "score": 0.4687, "threshold": 0.8555, "flagged": false, "available": true, "role": "advisory" },
      "prompt_safety":     { "score": 0.0076, "threshold": 0.6377, "flagged": false, "available": true, "role": "enforce" },
      "answer_safety":     { "score": 0.021,  "threshold": 0.7953, "flagged": false, "available": true, "role": "enforce" },
      "jailbreak":         { "score": 0.3333, "threshold": 0.9864, "flagged": false, "available": true, "role": "enforce",
                             "details": { "head_score": 0.00059, "linear_probe_z": -9.6043 } },
      "rag_jailbreak":     { "score": null, "threshold": 0.5768, "flagged": false, "available": false,
                             "role": "advisory", "unavailable_reason": "no context supplied" }
    },
    "additional_axes": {
      "profanity":         { "score": 0.026,  "threshold": 0.7,    "flagged": false, "available": true, "role": "advisory" },
      "out_of_scope":      { "score": 0.0678, "threshold": 0.9534, "flagged": false, "available": true, "role": "advisory" },
      "prompt_complexity": { "score": 0.0155, "threshold": 0.5,    "flagged": false, "available": true,
                             "role": "classifier", "label": "simple" }
    },
    "grounding": {
      "available": true, "verdict": "grounded", "score": 0.7261, "risk": 0.4687,
      "margin": -0.4521, "regime": "closed_book", "axis": "halluc_closedbook"
    },
    "pii": { "enabled": true, "input": { "count": 0, "by_type": {} }, "output": { "count": 0, "by_type": {} } }
  }
}
```

An axis that could not be measured on this turn reports `available: false` and `score: null` — **never `0`**. Treat it as "not measured", not as "clean".

**When the request is blocked**, the content is replaced, `finish_reason` becomes `content_filter`, and `reason` says where and why:

```json
{
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "[Geodesia blocked — jailbreak (input)]" },
      "finish_reason": "content_filter"
    }
  ],
  "geodesia": {
    "schema_version": "1.0",
    "event": "final",
    "decision": "blocked",
    "mode": "blocking",
    "reason": { "stage": "input", "axis": "jailbreak", "detail": null },
    "axes": {
      "prompt_safety": { "score": 0.9143, "threshold": 0.6377, "flagged": true, "available": true, "role": "enforce" },
      "jailbreak":     { "score": 0.9981, "threshold": 0.9864, "flagged": true, "available": true, "role": "enforce",
                         "details": { "head_score": 0.993179, "linear_probe_z": 4.8373 } }
    }
  }
}
```

(Other axes omitted for brevity; a real response lists every axis the deployment serves.) In `passthrough` mode the same request returns the real answer with `decision: "flagged"` and the same `reason`. A client that only needs "was this turn a violation?" can test `decision != "allowed"`.

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
| `web_search` | `boolean` | `false` | Search the live web, screen every page through the detector, and ground the answer in the safe ones. Requires `GW_WEBSEARCH_ENABLED=1` (the default). A web-search request is **always answered as an SSE stream**, whatever `stream` says. See [Live Web Search](web-search.md). |
| `mode` | `string` | *(config)* | `"block"` (or `"blocking"`) withholds flagged content; `"passthrough"` returns it with `decision: "flagged"`. Overrides the deployment/Application enforcement for this request only. See [Enforcement Modes](enforcement-modes.md). |
| `threshold_overrides` | `object` | — | Per-axis thresholds for this request. **Only the five base axes are honoured**: `halluc_context`, `halluc_closedbook`, `prompt_safety`, `answer_safety`, `jailbreak`. Keys for the other axes are silently ignored — change those in the Application policy instead. |
| `thinking_level` | `integer` | `2` | Detection depth, `0`–`3` (`3` = MAX). Clamped, never rejected. Falls back to the deepest level the deployment can serve; `geodesia.thinking` reports what actually ran. The deployment default is set with `GW_DEFAULT_THINKING_LEVEL`. See [Thinking Levels](thinking-levels.md). |
| `axes` | `array` \| `string` | *(all)* | Restrict the axes reported for this request (a list, or a comma/space-separated string). Axes that can withhold content are always scored regardless. |
| `domain` / `geodesia_domain` | `string` | `"general"` | Selects the domain-conditional calibration bucket for the closed-book axis. `general` / `default` / `all` / empty all mean domain-agnostic. |
| `pass_extra` | `integer` | `1` | Extra answer samples for closed-book uncertainty. Only applied when the upstream exposes log-probabilities and the turn has no explicit context. Each extra sample costs another generation. |
| `self_consistency` | `boolean` | `false` | Shorthand for `pass_extra > 1`. |
| `self_consistency_samples` | `integer` | — | How many extra samples when `self_consistency` is on. |
| `scan` | `boolean` | `true` | Set to `false` to **bypass detection entirely** for this request — pure pass-through to the upstream, no scoring, no blocking. |
| `pii_guard` | `boolean` | *(config)* | Per-request override of PII redaction. When the guard is on, the response carries `geodesia.pii` (counts per entity type, never the values). |
| `enable_judge_for_context_hallucination` | `boolean` | `false` | Run the per-claim context judge on this turn; the result appears as `geodesia.context_judge` (advisory, uncalibrated). Costs extra GPU seconds. |
| `constitutional_ai` | `boolean` | *(config)* | Per-request override of the constitutional system prompt. Wins over the deployment config and the Application policy. |
| `application_id` / `app_id` | `string` | — | Route this request to a specific Application. Equivalent to the `X-Geodesia-App` header. |
| `session_id` | `string` | — | Groups turns into one conversation in the audit log. |
| `upstream_api_key` / `upstream_type` | `string` | — | Override the upstream binding for this request. |

!!! info "The `glad_*` request aliases were removed"
    The `glad_mode`, `glad_thinking_level`, `glad_scan`, `glad_pii_guard`, `glad_axes` and `glad_enable_judge_for_context_hallucination` aliases are no longer recognised. Use the unprefixed names above. See [Migrating from the pre-1.0 payload](../reference/response-format.md#migrating-from-the-pre-10-payload).

!!! danger "`scan: false` turns the guardrail off"
    It is there for health checks and for replaying traffic you have already scored. A request served with `scan: false` is not screened, not scored and not blocked — and the response carries no verdict. Do not let application code set it from user input.

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
    verdict = None
    for chunk in stream:
        g = (chunk.model_extra or {}).get("geodesia")
        if g and g["event"] == "final":               # the only event that carries the verdict
            verdict = g
        delta = chunk.choices[0].delta.content if chunk.choices else None
        if delta:
            print(delta, end="", flush=True)

    print("\n[geodesia]", verdict["decision"], verdict["reason"])
    if verdict["decision"] == "blocked":             # halted mid-stream: finish_reason was content_filter
        print("-- halted by Geodesia --")
    ```

=== "TypeScript"

    ```ts
    const stream = await client.chat.completions.create({
      model: "my-model",
      stream: true,
      messages: [{ role: "user", content: "Explain TLS in two sentences." }],
    })

    let verdict: any = null
    for await (const chunk of stream) {
      const g = (chunk as any).geodesia
      if (g?.event === "final") verdict = g        // the only event that carries the verdict
      process.stdout.write(chunk.choices[0]?.delta?.content ?? "")
    }
    console.log("\n[geodesia]", verdict?.decision, verdict?.reason)
    if (verdict?.decision === "blocked") console.log("-- halted by Geodesia --")
    ```

**Where the payload arrives**

Every chunk that carries Geodesia information has a `geodesia` object whose `event` says what it reports:

| `event` | When | Contains |
|---|---|---|
| `input_scan` | First, on an empty-delta chunk, as soon as the prompt is scored | `axes` / `additional_axes` for the prompt axes (`prompt_safety`, `jailbreak`, …) — so you can show safety status before the first answer token. No decision. |
| `research` | Web search only, before the answer | `research`: one progress event. See [Live Web Search](web-search.md). |
| `progress` | Periodic re-scoring during generation | `axes` with the current scores. No decision. |
| `final` | Last chunk, the one with `finish_reason` | The complete verdict — same shape as the non-streaming `geodesia` object. |

Take the verdict **only** from the `final` event; `input_scan` and `progress` are informational. A violation found after the answer has already been fully streamed is reported as `decision: "flagged"`, because the text was already delivered.

```text
data: {"id":"chatcmpl-geodesia-1790234386649","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":null}],
       "geodesia":{"schema_version":"1.0","event":"input_scan","axes":{"prompt_safety":{…},"jailbreak":{…}},"additional_axes":{…}}}
data: {"id":"chatcmpl-geodesia-1790234386649","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"Ocean’s breath, …"},"finish_reason":null}]}
data: {"id":"chatcmpl-geodesia-1790234386649","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}],
       "geodesia":{"schema_version":"1.0","event":"final","decision":"allowed","mode":"blocking","reason":null,"axes":{…},…}}
data: [DONE]
```

**The mid-stream brake.** The proxy re-reads the accumulating generation every `cadence_tokens` tokens (default 32). If an output-region axis fires before generation finishes it injects `\n\n[Geodesia: generation halted — energy barrier]`, sends a `final` chunk with `finish_reason: "content_filter"`, `decision: "blocked"` and `reason: {"stage": "output", "axis": "<axis>", "detail": "halted mid-stream"}`, then `data: [DONE]`.

!!! warning "Partial flagged content is visible before the brake"
    A user may see the first tokens of a dangerous answer before it is cut. If your application must never render partial flagged content, use non-streaming mode, or buffer client-side until you see a terminal `finish_reason`.

---

## Tool calling — `tools` / `tool_choice`

`tools` and `tool_choice` are forwarded to the upstream unchanged (see [Request reference](#standard-openai-fields) above) — the proxy does not rewrite your tool schemas. What it *does* do is translate whatever the upstream's serving stack returns for a tool call back into standard OpenAI shape: an accumulating `delta.tool_calls` on each streaming chunk, `message.tool_calls` on the non-streaming response, and `finish_reason: "tool_calls"`. Any provider-native tool-call syntax the underlying model would otherwise leak into plain text (e.g. a bracketed marker or an XML-ish block, depending on the serving stack) never reaches the client as content.

This requires the upstream's own serving stack to have tool-call parsing turned on — for example vLLM's `--enable-auto-tool-choice --tool-call-parser <family>` (the parser family must match the model's chat template). Check `GET /v1/models` — a `"tool_calling"` entry in `capabilities` confirms the deployment is configured for it; if it's absent, `tools` is still forwarded but the model's raw tool-call output may come back as plain `content` instead of `tool_calls`.

=== "curl"

    ```bash
    curl -N -s http://localhost:8080/gw/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{
        "model": "my-model",
        "stream": false,
        "tool_choice": "auto",
        "tools": [{
          "type": "function",
          "function": {
            "name": "web_search",
            "description": "Search the web and return ranked results with URLs.",
            "parameters": {
              "type": "object",
              "properties": {
                "query": { "type": "string" },
                "max_results": { "type": "integer" }
              },
              "required": ["query"]
            }
          }
        }],
        "messages": [{"role": "user", "content": "Search the web for the current prime minister of Italy."}]
      }'
    ```

=== "Python"

    ```python
    from openai import OpenAI

    client = OpenAI(base_url="http://localhost:8080/gw/v1", api_key="not-needed-locally")

    r = client.chat.completions.create(
        model="my-model",
        tool_choice="auto",
        tools=[{
            "type": "function",
            "function": {
                "name": "web_search",
                "description": "Search the web and return ranked results with URLs.",
                "parameters": {
                    "type": "object",
                    "properties": {"query": {"type": "string"}, "max_results": {"type": "integer"}},
                    "required": ["query"],
                },
            },
        }],
        messages=[{"role": "user", "content": "Search the web for the current prime minister of Italy."}],
    )
    call = r.choices[0].message.tool_calls[0]
    print(call.function.name, call.function.arguments)   # web_search {"query": "...", "max_results": ...}
    ```

**What comes back** (non-streaming):

```json
{
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [{
        "id": "call_abc123",
        "type": "function",
        "function": { "name": "web_search", "arguments": "{\"query\": \"...\", \"max_results\": 5}" }
      }]
    },
    "finish_reason": "tool_calls"
  }]
}
```

Streaming carries the same call as incremental `delta.tool_calls` fragments — `id`/`type`/`function.name` on the first fragment for a given `index`, `function.arguments` concatenated across the rest — exactly the shape any OpenAI-compatible SDK already knows how to accumulate. **Parallel tool calls** (the model requesting more than one function in the same turn) come back as separate entries keyed by `index`, each with its own `id`.

### Completing the round trip

After you execute the tool, send the conversation back with the assistant's `tool_calls` from the previous turn and one `role: "tool"` result message per call — the proxy forwards both unchanged to the upstream, same as any OpenAI-compatible endpoint:

```json
{
  "model": "my-model",
  "tools": [ /* same schema as before */ ],
  "messages": [
    {"role": "user", "content": "Search the web for the current prime minister of Italy."},
    {"role": "assistant", "content": null, "tool_calls": [
      {"id": "call_abc123", "type": "function",
       "function": {"name": "web_search", "arguments": "{\"query\": \"prime minister of Italy\"}"}}
    ]},
    {"role": "tool", "tool_call_id": "call_abc123", "content": "Giorgia Meloni is the Prime Minister of Italy."}
  ]
}
```

!!! info "Tool-result messages count as grounding context"
    A `role: "tool"` message is treated the same way as `context` for the `halluc_context` axis — the final answer is scored for faithfulness against what the tool actually returned, not just against the user's original question.

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
      }' | jq '{answer: .message.content, decision: .geodesia.decision, reason: .geodesia.reason}'
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
    print(r["message"]["content"], r["geodesia"]["decision"])
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
    console.log(r.message.content, r.geodesia.decision)
    ```

**What comes back**

```json
{
  "model": "llama3.2",
  "message": { "role": "assistant", "content": "The capital of France is Paris." },
  "done": true,
  "geodesia": {
    "schema_version": "1.0",
    "event": "final",
    "decision": "allowed",
    "mode": "blocking",
    "reason": null,
    "axes": { "…": "…" }
  }
}
```

The `geodesia` object is identical to the OpenAI format. When streaming (NDJSON), the same `input_scan` / `progress` / `final` objects ride on the Ollama frames.

---

## Response reference

The proxy adds a single top-level key, `geodesia` (schema version `1.0`), to the standard OpenAI or Ollama body. Its main fields:

| Field | When present | Description |
|---|---|---|
| `schema_version` | always | `"1.0"`. Always the first key. Changes only when the shape of the object changes. |
| `event` | always | `final` on a non-streaming response; `input_scan`, `progress`, `research` or `final` on streaming chunks. |
| `decision` | `final` | `allowed`, `flagged` (violation detected, content delivered — passthrough mode, or found after the stream ended) or `blocked` (content withheld, `finish_reason: "content_filter"`). |
| `mode` | `final` | The enforcement mode actually applied: `blocking` or `passthrough`. |
| `reason` | `final` | `null` when allowed; otherwise `{stage, axis, detail}` — `stage` is `input`, `output`, `tools` or `quota`, `axis` the axis that decided. |
| `axes` | when scored | Primary axes, each `{score, threshold, flagged, available, role, …}`. `available: false` (with `score: null` and an `unavailable_reason`) means the axis could not be scored on this turn — most often `halluc_closedbook` against an upstream with no log-probabilities. |
| `additional_axes` | when scored | Annotation-only axes (`profanity`, `out_of_scope`, `prompt_complexity`), same shape. They never block. |
| `grounding` | when scored | One fused grounding number across `halluc_context` and `halluc_closedbook`. See [Grounding Score](grounding.md). |
| `thinking` | thinking level ≥ 1 | `{level, tiers_used, escalated}` — the level this turn actually ran at and the detector tiers that contributed. See [Thinking Levels](thinking-levels.md). |
| `pii` | PII guard on | `{enabled, input: {count, by_type}, output: {count, by_type}}` — counts only. |
| `routing` | complexity routing on | `{enabled, used_complex_model, axis, score, threshold, model}` — see [Token & Cost Control](cost-control.md). |
| `token_saving` | `token_saving` switch on | `{enabled, cached_prompt_tokens?, prompt_tokens?, cache_write_prompt_tokens?, prompt_cache_key?, upstream_closed_early?, closed_reason?}` — see [Token & Cost Control](cost-control.md#token-saving-spend-less-without-changing-what-the-model-reads). |
| `rag` | RAG active | `{collection_id, sources, source_count, verification}`. |
| `context_judge` | claim judge on | Per-claim grounding verdicts (advisory, uncalibrated): `{available, calibrated, score, threshold, flagged, verdict_counts, claims, …}`. See [Context judge](../reference/response-format.md#context-judge). |
| `certificate` | `GW_CERTIFICATE=on` | Signed decision certificate (`geodesia-cert-3`) built from this response's axes. Its `verdict` is the policy verdict, independent of `mode`: a passthrough turn can carry `decision: "flagged"` with `certificate.verdict: "blocked"`. See [Certificate](../reference/response-format.md#certificate). |

Optional sections are omitted when they do not apply. Full field-by-field treatment, including `reasoning_budget`, `quota`, `tool_guard` and every axis `details` field: [API Response Format](../reference/response-format.md).

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
  }' | jq '{answer: .choices[0].message.content, halluc: .geodesia.axes.halluc_context, grounding: .geodesia.grounding}'
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
  }' | jq '{answer: .choices[0].message.content, decision: .geodesia.decision, reason: .geodesia.reason}'
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
