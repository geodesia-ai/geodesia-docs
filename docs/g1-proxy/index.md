# G1-Proxy — Introduction

!!! tip "Looking for the full endpoint list?"
    Every route G1-Proxy exposes, in one page: **[Complete API Map](api-reference.md)**.

**G1-Proxy** is an OpenAI-compatible HTTP proxy that adds real-time AI validation to any LLM backend. It listens for chat requests, screens them, forwards them to the configured upstream model, validates the response, and returns the result — with a `geodesia` field attached containing the full detection payload.

## Call it

Point any OpenAI client at the proxy. Nothing else in your code changes — the answer comes back in the
shape you already handle, with the verdict attached.

=== "curl"

    ```bash
    curl -s http://localhost:8080/gw/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{"model":"my-model","stream":false,
           "messages":[{"role":"user","content":"What is the capital of France?"}]}' \
      | jq '{answer: .choices[0].message.content, decision: .glad_decision}'
    ```

=== "Python"

    ```python
    from openai import OpenAI

    client = OpenAI(base_url="http://localhost:8080/gw/v1", api_key="not-needed-locally")
    r = client.chat.completions.create(
        model="my-model",
        messages=[{"role": "user", "content": "What is the capital of France?"}],
    )
    print(r.choices[0].message.content)
    print(r.model_extra["glad_decision"], r.model_extra["geodesia"]["dominant_axis"])
    ```

=== "TypeScript"

    ```ts
    import OpenAI from "openai"

    const client = new OpenAI({ baseURL: "http://localhost:8080/gw/v1", apiKey: "not-needed-locally" })
    const r = (await client.chat.completions.create({
      model: "my-model",
      messages: [{ role: "user", content: "What is the capital of France?" }],
    })) as any

    console.log(r.choices[0].message.content, r.glad_decision)
    ```

Full request/response contract: **[Chat API](chat-api.md)**. Every route: **[Complete API Map](api-reference.md)**.

## Key Capabilities

| Capability | Description |
|---|---|
| **Drop-in compatibility** | Accepts `POST /v1/chat/completions` (OpenAI) and `POST /api/chat` (Ollama). Any client that works with OpenAI works unchanged. |
| **9-axis detection** | Scores every request across context faithfulness, closed-book fabrication, prompt safety, answer safety, jailbreak, `rag_jailbreak`, profanity, out-of-scope and prompt complexity — in a single forward pass. See [Detection Axes](detection-axes.md). |
| **Streaming support** | Fully streaming — tokens are relayed in real-time. Mid-stream braking stops a harmful response before it finishes. |
| **Enforcement modes** | `blocking` withholds flagged content. `passthrough` returns the answer but annotates the response with the detection verdict. |
| **Per-request thresholds** | Override detection thresholds on a per-call basis via `threshold_overrides`. |
| **Knowledge Base** | Built-in RAG: upload documents, retrieve relevant passages, verify citations claim-by-claim. |
| **Live Web Search** | Optional `web_search` per request: searches the live web, screens every page through the G1-Hummingbird firewall, and grounds the answer in the safe pages. Tavily key → reliable results; DuckDuckGo key-less fallback. See [Live Web Search](web-search.md). |
| **Causal XAI** | Token-level attribution — entirely black-box, no access to model internals or GPU memory required. |
| **Config persistence** | Backend selection, model, and thresholds are written to a JSON file. Setup survives restarts and container rebuilds. |
| **Compliance logging** | Every request is written to the shared SQLite audit database so the compliance dashboard stays current. |

## Endpoint Summary

<div class="endpoint"><span class="method method-post">POST</span><span class="path">/v1/chat/completions</span></div>

OpenAI-compatible chat endpoint. Both streaming and non-streaming. Extended with `geodesia` detection payload.

<div class="endpoint"><span class="method method-post">POST</span><span class="path">/api/chat</span></div>

Ollama-compatible chat endpoint. Mirrors the OpenAI endpoint in behaviour.

<div class="endpoint"><span class="method method-get">GET</span><span class="path">/health</span></div>

Health check. Returns upstream type, logprob capability, and active axis count (4 or 5).

<div class="endpoint"><span class="method method-get">GET</span><span class="path">/v1/models</span></div>

Returns Geodesia's own model descriptor — id, `capabilities` (e.g. `hallucination_detection`, `tool_calling`), and the detected upstream base model — so an OpenAI-compatible client can discover what the deployment supports without a separate probe. See [Tool calling](chat-api.md#tool-calling-tools-tool_choice).

<div class="endpoint"><span class="method method-get">GET</span><span class="path">/v1/glad/gateway/config</span></div>

Returns the current gateway configuration (API key is masked).

<div class="endpoint"><span class="method method-post">POST</span><span class="path">/v1/glad/gateway/config</span></div>

Updates one or more configuration fields at runtime. Changes take effect on the next request.

<div class="endpoint"><span class="method method-post">POST</span><span class="path">/upstream/test</span></div>

Tests a candidate upstream connection: reachability, model list, logprob support, and a sample reply — all in one call.

<div class="endpoint"><span class="method method-post">POST</span><span class="path">/v1/glad/causal-explainability/analyze</span></div>

Computes black-box token-level causal attribution (occlusion or MuPAX LLM) for a given prompt/response pair.

<div class="endpoint"><span class="method method-get">GET</span><span class="path">/v1/glad/documentation</span></div>

Serves the built-in user guide as Markdown.

Plus the [Knowledge Base](../rag/index.md) (`/v1/glad/rag/*`) sub-router.

## Request & Response Extension

Every standard OpenAI request body is accepted verbatim. Geodesia adds the following **optional** fields:

| Field | Type | Description |
|---|---|---|
| `context` | string | Explicit grounding context text. G1-Hummingbird scores the answer against this to detect faithfulness violations. |
| `mode` / `glad_mode` | `"block"` \| `"passthrough"` | Per-request enforcement mode. Overrides the gateway's configured `block_input` / `block_output`. |
| `threshold_overrides` | object | Per-axis detection thresholds (probability 0–1) that override the calibrated defaults for this request only. |
| `rag` | object | RAG configuration: `collection_id`, `top_k`, `rerank`, `verify`. See [Knowledge Base](../rag/index.md). |
| `pass_extra` | integer | Number of extra generation samples for the closed-book uncertainty estimate. Default `1` (no extra). |
| `self_consistency` | boolean | Enable self-consistency sampling for closed-book uncertainty. |
| `self_consistency_samples` | integer | Number of samples when `self_consistency` is `true`. |

The response carries the standard OpenAI structure with three extra top-level fields:

| Field | Value |
|---|---|
| `glad_decision` | `"passed"` or `"blocked"` |
| `glad_mode` | `"blocking"` or `"passthrough"` — the mode that was actually used |
| `geodesia` | Full detection payload. See [API Response Format](../reference/response-format.md). |
