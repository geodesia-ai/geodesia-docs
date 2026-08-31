# G1-Proxy — Complete API Map

Every HTTP surface **G1-Proxy** exposes, in one table. G1-Proxy is the **data plane**: chat, detection, RAG, web search, audio, feedback and everything that has to happen while a request is in flight. Anything about governance, applications, cost or compliance reporting lives on the other service — see the [G-1 Studio API map](../studio/api-reference.md).

!!! info "Where to send the request"
    In the packaged product a **single port (8080)** fronts both services:

    | Reaching | Prefix | Example |
    |---|---|---|
    | **G1-Proxy** | `/gw` + the path below | `http://localhost:8080/gw/v1/chat/completions` |
    | **G-1 Studio** | the path as-is | `http://localhost:8080/v1/glad/apps` |

    In a split deployment the proxy also listens directly on **`:8800`** with **no `/gw` prefix** — `http://localhost:8800/v1/chat/completions`. Paths in this page are written **without** the prefix; add `/gw` when going through port 8080.

!!! warning "Mutating endpoints can require a token"
    When `GW_API_TOKEN` is set, the endpoints marked **🔒** below require `Authorization: Bearer <token>` (or `X-Gateway-Token: <token>`) and return `401` otherwise. When it is unset — the default for a single-tenant local install — no auth is enforced anywhere. Set it on any shared or multi-tenant host: without it, config override and upstream re-pointing are unauthenticated.

---

## One call, to check you are pointed at the right service

```bash
curl -s http://localhost:8080/gw/health | jq
# {"ok": true, "upstream_type": "ollama", "upstream": "http://localhost:11434",
#  "logprobs": true, "axes": 9,
#  "axes_available": ["halluc_context","halluc_closedbook","prompt_safety","answer_safety",
#                     "jailbreak","rag_jailbreak","profanity","out_of_scope","prompt_complexity"],
#  "axes_gated": [], "calibration": {"model": "…", "status": "calibrated"}}
```

If that answers, everything below is reachable. If it 404s you are talking to Studio — drop the `/gw`
and see the [Studio API map](../studio/api-reference.md) instead.

## Inference

| Method | Path | What it does | Docs |
|---|---|---|---|
| `POST` | `/v1/chat/completions` | OpenAI-compatible chat. Screens the prompt, forwards to the upstream LLM, scores the answer, returns the OpenAI body plus `glad_decision` / `geodesia`. Supports SSE streaming with a mid-stream brake. | [Chat API](chat-api.md) |
| `POST` | `/api/chat` | The same pipeline in the **Ollama** wire format (NDJSON streaming). | [Chat API](chat-api.md#ollama-format-post-apichat) |
| `POST` | `/v1/completions` | OpenAI **legacy** text completion. The proxy wraps `prompt` into a one-message chat, runs the full pipeline, and unwraps the result back into the legacy shape. | [Chat API](chat-api.md) |
| `POST` | `/v1/glad/evaluate` | Generate **and** score in one call, returning every internal detection metric rather than a chat-shaped body. Use it for batch scoring and for evaluating a corpus. | [Evaluate](../product-api/evaluate.md) |

All four accept the same [Geodesia extension fields](chat-api.md#geodesia-extension-fields) (`context`, `rag`, `mode`, `threshold_overrides`, `thinking_level`, `web_search`, …) and forward every other body key to the upstream verbatim, which is what makes the proxy a drop-in for vLLM.

---

## Status & health

| Method | Path | Returns |
|---|---|---|
| `GET` | `/health` | `{ok, upstream_type, upstream, internal_vllm, logprobs, axes, axes_available, axes_gated, calibration}`. `axes` is how many axes the **served checkpoint** actually scores (nine on the shipped head) and `axes_available` names them. `axes_gated` lists the ones that cannot be scored right now — `["halluc_closedbook"]` when the upstream exposes no token log-probabilities, `[]` otherwise. |
| `GET` | `/version` | The proxy's component version, read hot from `G1_PROXY_VERSION.json`. |
| `GET` | `/v1/glad/documentation` | `docs/USER_GUIDE.md` as `text/markdown`. `404` if the file is not in the image. Backs the in-product Documentation page on a proxy-only deployment. |
| `GET` | `/v1/glad/mcp/status` | MCP layer state: `{enabled, chat_aware, servers, guard, actions, domain_allowlist, egress_tools, interceptors}`. |
| `GET` | `/v1/glad/pii/status` | PII guard state, including `library.available` and `effective` — an enabled switch on an image without the library redacts nothing, and the UI must be able to say so. |
| `GET` | `/v1/glad/audio/status` | Voice-guard state: ASR model, language, cadence, `deps_installed`, the axes scored on speech, and the baked-in models you can switch to. |

---

## Runtime configuration

| Method | Path | | What it does |
|---|---|---|---|
| `GET` | `/v1/glad/gateway/config` | | The live gateway config. `upstream_api_key` is returned as `***`, never in clear. |
| `POST` | `/v1/glad/gateway/config` | 🔒 | Patch the live config; applies immediately and is persisted to `GW_CONFIG_FILE`. Sending `"***"` as the API key leaves the stored one untouched. |
| `POST` | `/upstream/test` | | Connect to a candidate backend, list its models, probe log-probability support, and return a sample reply with latency. Body: `{url, type, api_key?, model?}`. Also returns `closed_book_available` and the same `axes` / `axes_available` / `axes_gated` block as `/health`. This is the **Test connection** button. |
| `POST` | `/test-capabilities` | 🔒 | Run the detector on built-in safe/unsafe sample pairs, one per axis, and report which axes actually separate. **No upstream call** — it tests the detector, not the LLM. |

!!! danger "Keys the config endpoint refuses"
    `POST /v1/glad/gateway/config` silently drops `system_prompt`, `internal_vllm_cmd` and `internal_vllm_url`, and rejects `upstream_type: "internal"` with a `400`. Those feed a subprocess launch and the constitutional prompt; they are settable by env/CLI only, deliberately, so a remote caller cannot reach them.

---

## Calibration & licensing

| Method | Path | | What it does |
|---|---|---|---|
| `GET` | `/v1/glad/gateway/calibration` | | Closed-book recalibration state for the current upstream model: `idle` \| `running` \| `calibrated` \| `error`, plus a log tail and progress. |
| `GET` | `/v1/glad/gateway/calibration/log` | | The **full** log of the last run as plain text, persisted to disk so it survives a restart. `404` before the first run. |
| `POST` | `/calibrate` | 🔒 | Run the closed-book calibration now, **streaming the progress log** as `text/plain`. On success the fresh checkpoint is reloaded without a restart. Body: `{mode: "quick"\|"full"}` or `{fraction: 0.25}`. |
| `POST` | `/v1/glad/gateway/recalibrate` | 🔒 | Same job, non-streaming, ignoring the per-model "already done" sentinel. Returns `{ok, calibration}`. |
| `POST` | `/v1/glad/gateway/reload-sledge` | | Hot-reload the per-model calibration registry into the live detector — no model reload, no restart. Called by the calibration driver once a new artifact is registered. |
| `GET` | `/v1/glad/gateway/entitlements` | | Active plan and today's usage: tier, chats used/remaining, model cap, expiry. |
| `POST` | `/v1/glad/gateway/license` | | Install a vendor-signed licence (`{license\|token\|key}`, raw JSON or base64). Returns the new plan; `400` on an invalid signature. **No auth by design** — the Ed25519 signature *is* the authorization. |
| `DELETE` | `/v1/glad/gateway/license` | | Remove the licence, dropping back to the free tier. |
| `GET` | `/v1/glad/gateway/update-check` | | Online version check against `$GEODESIA_DL_BASE/latest.json`. Cached; `?force=1` refreshes now. |
| `GET` | `/v1/glad/gateway/notifications` | | User-facing notices for the UI banner (update available, licence expiring within 14 days). |

Both `/calibrate` and `/v1/glad/gateway/recalibrate` return **`400`** when the upstream exposes no token log-probabilities — closed-book calibration has nothing to fit without them.

There is also a second, model-recalibration control plane mounted on **both** services under `/v1/glad/calibration/*` — see [the Studio map](../studio/api-reference.md#model-recalibration).

---

## Knowledge base (RAG)

Prefix `/v1/glad/rag`. Every route honours the **`X-Geodesia-App`** header, which scopes collections to one Application. Full guide: [Knowledge Base](../rag/index.md).

| Method | Path | What it does |
|---|---|---|
| `GET` | `/status` | Whether the RAG stack is available, plus per-app collection counts. |
| `GET` | `/collections` | List collections visible to this Application. |
| `POST` | `/collections` | Create one. Body: `{name}`. |
| `GET` | `/collections/{collection_id}` | One collection with its documents. |
| `DELETE` | `/collections/{collection_id}` | Delete the collection and its index. |
| `POST` | `/collections/{collection_id}/documents` | **Multipart** upload (`file`, optional `name`). Returns **`202`** immediately and parses/embeds in the background — do not hold the connection open. |
| `DELETE` | `/collections/{collection_id}/documents/{doc_id}` | Remove a document and its chunks. |
| `POST` | `/collections/{collection_id}/query` | Retrieve passages. Body: `{query, top_k?, rerank?}` → `{context, sources, n_sources}`. |
| `GET` | `/ingest/progress` | Poll the background ingest: `{stage, …}`, where `stage` reaches `done` or `error`. This is how you know an upload finished. |

!!! tip "Upload is a two-step dance"
    `POST …/documents` → `202`, then poll `GET /v1/glad/rag/ingest/progress` until `stage === "done"`. A large PDF can take minutes; the `202` is not a completion.

---

## Live web search

Prefix `/v1/glad/websearch`. See [Live Web Search](web-search.md).

| Method | Path | What it does |
|---|---|---|
| `GET` | `/config` | `{enabled, provider, has_key, key_hint, key_source, env_locked}`. The key is **never** returned in clear — only a masked hint. |
| `POST` | `/config` | Store the provider API key out-of-band (`{api_key}`), written `0600` outside the image. `env_locked: true` means an environment variable wins and this call is a no-op. |

Web search is requested **per chat turn** with `"web_search": true` on the chat body — there is no separate search endpoint.

---

## Realtime voice guard

See [Audio Input](audio-input.md).

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/audio/status` | Configuration + whether the ASR dependencies are actually installed. |
| `WS` | `/v1/glad/audio/stream` | **WebSocket.** Client streams binary PCM float32 @ 16 kHz mono; the server emits `{decision: "pass"\|"warn"\|"block", axes, committed, is_final}` on cadence and closes on an early block. Text control frames: `{"type":"finish"}` commits and scores the final utterance, `{"type":"reset"}` starts a new one. Closes immediately with `{"type":"error","reason":"audio_disabled"}` when the voice guard is off. |
| `POST` | `/v1/glad/audio/utterance` | Single-clip path: POST a raw 16-bit PCM @ 16 kHz mono WAV as `application/octet-stream` → `{committed, decision, axes}`. This is the **Test microphone** button. |

---

## Explainability

| Method | Path | | What it does |
|---|---|---|---|
| `POST` | `/v1/glad/causal-explainability/analyze` | 🔒 | Black-box token attribution over the detector — no upstream-model internals, no gradients. Body: `{prompt, response\|full_response, context?, method?, axis?}`. `422` when neither prompt nor context is given, or when `response` is missing for any method other than `dca_dual`. |

| `method` | What it computes | Typical latency |
|---|---|---|
| `dca` *(default)* | Deterministic causal attribution on the dominant flagged axis. | ~1–3 s |
| `dca_dual` | Prompt **and** answer surfaces attributed separately — "which prompt tokens caused the block" *and* "which answer tokens caused the flag". The only method that accepts an empty `response` (a prompt blocked before generation has no answer). | ~1–3 s |
| `gradient_causal` | Leave-one-out occlusion. | seconds |
| `mupax_causal` | Monte-Carlo MuPAX. Accepts `mupax_samples`. | up to several minutes |

Pass `axis` to pin the heatmap to one axis; omit it and the attributor picks the dominant flagged axis itself. Full treatment: [Causal Explainability](causal-xai.md).

---

## Human feedback loop

Prefix `/v1/glad/feedback`. Honours `X-Geodesia-App`. Full guide: [Human Feedback Loop](feedback.md).

| Method | Path | What it does |
|---|---|---|
| `GET` | `/schema` | The vocabulary the queue accepts: `{axes, problem_to_axis, verdicts, regions}`. Read this instead of hard-coding axis names. |
| `POST` | `/` | File a correction on a served turn. |
| `GET` | `/` | The review queue. Filters: `status`, `application_id`, `axis`, `region`, `limit` (≤ 1000), `offset`. |
| `GET` | `/stats` | `{pending, approved, rejected, total}`, optionally per Application. |
| `POST` | `/{feedback_id}/review` | A curator's decision: `{status, axis?, verdict?, reviewer?, note?, weight?, twin_prompt?, twin_answer?, attack_family?}`. |
| `DELETE` | `/{feedback_id}` | Drop an entry. |
| `GET` | `/export` | Export decided entries as a training corpus. Defaults to `status=approved`. |
| `GET` | `/bank/status` | `{bank_version, approved}` — what the live exemplar bank currently holds. |
| `POST` | `/retrain` | `{mode: "memory"\|"weights", application_id?}`. `memory` refreshes the exemplar bank instantly; `weights` exports a corpus and launches the configured trainer. Returns a `job_id`. |
| `GET` | `/retrain/status?job_id=…` | Poll one job. |
| `GET` | `/retrain/jobs` | All retrain jobs. |

### Automatic feedback (idle judge)

Off until switched on. While the box is idle, a CPU judge re-scores served traffic and writes confident disagreements into the same bank a curator feeds by hand.

| Method | Path | What it does |
|---|---|---|
| `GET` | `/auto/status` | Installed? running? queue depths, disagreements, promotions, idle seconds. |
| `GET` | `/auto/config` | Current config plus `axes_available`. |
| `PUT` | `/auto/config` | Patch the config. |
| `GET` | `/auto/prompt-preview?axis=…` | **Read-only.** The exact prompt the judge will see. You configure the description of your organisation; the audit protocol around it is not a setting. |
| `GET` | `/auto/items?state=…&limit=…` | Inspect what the judge queued, scored or promoted. |

---

## Not on G1-Proxy

Frequently looked for here, actually on [G-1 Studio](../studio/api-reference.md):

| You want | It lives at |
|---|---|
| Applications, policies, API keys, cost | `/v1/glad/apps/…`, `/v1/glad/orgs/…` |
| Compliance dashboard, FRIA, kill switch, oversight, audit chain | `/v1/glad/dashboard`, `/v1/glad/fria`, `/v1/glad/kill-switch/…`, `/v1/glad/oversight/…`, `/v1/glad/chain/…` |
| Reports, deployer manual, legal frameworks | `/v1/glad/report`, `/v1/glad/deployer-manual`, `/v1/glad/legal/…` |
| Customer licence tokens | `/v1/glad/license-tokens/…` |
| Model catalogue and switching | `/v1/glad/models/available`, `/v1/glad/models/switch` |
| Stored chat history | `/v1/glad/chat-sessions`, `/v1/glad/chat-messages` |
