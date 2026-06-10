# Gateway — Introduction

The **Geodesia Gateway** is an OpenAI-compatible HTTP proxy that adds real-time AI validation to any LLM backend. It listens for chat requests, screens them, forwards them to the configured upstream model, validates the response, and returns the result — with a `geodesia` field attached containing the full detection payload.

## Key Capabilities

| Capability | Description |
|---|---|
| **Drop-in compatibility** | Accepts `POST /v1/chat/completions` (OpenAI) and `POST /api/chat` (Ollama). Any client that works with OpenAI works unchanged. |
| **5-axis detection** | Scores every request across context faithfulness, closed-book fabrication, prompt safety, answer safety, and jailbreak. |
| **Streaming support** | Fully streaming — tokens are relayed in real-time. Mid-stream braking stops a harmful response before it finishes. |
| **Enforcement modes** | `blocking` withholds flagged content. `passthrough` returns the answer but annotates the response with the detection verdict. |
| **Per-request thresholds** | Override detection thresholds on a per-call basis via `threshold_overrides`. |
| **Knowledge Base** | Built-in RAG: upload documents, retrieve relevant passages, verify citations claim-by-claim. |
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

Proxy to the upstream's `/v1/models`. Returns available model IDs.

<div class="endpoint"><span class="method method-get">GET</span><span class="path">/v1/glad/gateway/config</span></div>

Returns the current gateway configuration (API key is masked).

<div class="endpoint"><span class="method method-post">POST</span><span class="path">/v1/glad/gateway/config</span></div>

Updates one or more configuration fields at runtime. Changes take effect on the next request.

<div class="endpoint"><span class="method method-post">POST</span><span class="path">/upstream/test</span></div>

Tests a candidate upstream connection: reachability, model list, logprob support, and a sample reply — all in one call.

<div class="endpoint"><span class="method method-post">POST</span><span class="path">/calibrate</span></div>

Runs (or re-runs) the closed-book calibration against the current upstream. Streams progress. Reloads the checkpoint on completion without restarting.

<div class="endpoint"><span class="method method-post">POST</span><span class="path">/v1/glad/causal-explainability/analyze</span></div>

Computes black-box token-level causal attribution (occlusion or MuPAX) for a given prompt/response pair.

<div class="endpoint"><span class="method method-get">GET</span><span class="path">/v1/glad/documentation</span></div>

Serves the built-in user guide as Markdown.

Plus the [Knowledge Base](../rag/index.md) (`/v1/glad/rag/*`) and [Agent Flow](../agent-flow/index.md) (`/v1/agent-flow/*`) sub-routers.

## Request & Response Extension

Every standard OpenAI request body is accepted verbatim. Geodesia adds the following **optional** fields:

| Field | Type | Description |
|---|---|---|
| `context` | string | Explicit grounding context text. GLAD scores the answer against this to detect faithfulness violations. |
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
