# G1 Eval — score texts you already have

**G1 Eval** evaluates a prompt, a context and an answer that you **already have**. Nothing is sent to an LLM:
there is no upstream call and no pass-through. You get back one JSON object with a score, a threshold and a
verdict for every axis the texts make measurable.

Use it to:

- score a corpus of past conversations (CRM cases, support tickets, agent traces) offline;
- check an answer produced by a system you do not route through the proxy;
- reproduce a single case end to end: paste the request, the document the agent read and the reply it sent.

G1 Eval is served by **G1-Proxy** and by **G1-Proxy Light** on the same port as the chat API.

!!! info "Same pipeline as the chat API, same numbers"
    `/g1/eval` does not have a scorer of its own. It turns your texts into a `/v1/chat/completions` request for
    the **`default` Application** and runs it through the very same code. The only difference is that your
    `answer` takes the place of the LLM's reply. Same loaded models, fusion banks, thresholds and feedback memory,
    so the same texts give the same scores on both entries, on every axis and at every thinking level.

---

## Request

`POST /g1/eval` with a JSON body.

| Field | Type | Meaning |
|---|---|---|
| `prompt` | string | The user (or agent) request. |
| `context` | string | The document the answer should be grounded in: a knowledge-base article, a CRM record, retrieved passages, tool output. |
| `answer` | string | The answer to evaluate. |
| `answer_2` | string | Optional. A **second, independent** answer to the same `prompt`. Without a `context`, it makes closed-book hallucination measurable by [consistency](#closed-book-hallucination-with-a-second-answer). |
| `thinking_level` | `0`–`3` | Which detector tiers run. Default `2`. See [Thinking Levels](thinking-levels.md). |
| `verify_references` | bool | Check links, DOIs, arXiv ids, PubMed ids and citations in the answer on open sources. Off by default. See [Reference check](../reference/response-format.md#reference-check). |
| `reference_relevance_threshold` | number 0–1 | Optional. Relevance threshold of the reference check (default `0.08`). |
| `session_id` | string | Optional. Groups the call into a conversation in [Causal Explainability](#in-the-ui). The Eval page sends one automatically (`eval-…`). |

At least `prompt` or `answer` must be non-empty, otherwise the call returns `422 nothing_to_evaluate`.

G1 Eval has **no Application of its own**. There is no Application API key and no app selection: it always runs
with the configuration of the `default` Application. It also does not accept `threshold_overrides`, `mode` or
`axes` — the thresholds are those of the `default` Application's policy, and the axes are chosen by which
fields you fill.

### Which axes are computed

| Fields filled | Axes |
|---|---|
| `prompt` | `prompt_safety`, `jailbreak`, `rag_jailbreak` (read on the prompt itself), `profanity`, `prompt_complexity` |
| `prompt` + `context` | the prompt axes; `rag_jailbreak` reads the **context**; `prompt_safety` and `jailbreak` are also read **on the context** and reported in `context_axes` |
| `answer` | `answer_safety` |
| `answer` + `context` | `answer_safety`, `halluc_context` |
| `prompt` + `answer` | the prompt axes, `answer_safety`, `halluc_closedbook` |
| `answer` + `answer_2` (no `context`) | `answer_safety`, `halluc_closedbook` measured by consistency |
| all three | every axis above that applies |

`out_of_scope` is **not** an Eval axis: in Eval the context is a document, not a declared scope.

`halluc_closedbook` normally needs the token log-probabilities of the model that **generated** the answer, and a
pasted answer does not carry them. In Eval it is therefore measured in one of two ways:

- with `answer_2`, by [consistency between the two answers](#closed-book-hallucination-with-a-second-answer);
- with `verify_references: true`, through the [reference check](../reference/response-format.md#reference-check).

With neither, the axis is reported `available: false`, with its reason in `unavailable_reason`. An axis that was
not measured never reports a score of `0`.

---

## Example

=== "curl"

    ```bash
    curl -s http://localhost:8800/g1/eval \
      -H "Content-Type: application/json" \
      -d '{
        "prompt": "Summarize what this customer asked and what we committed to.",
        "context": "Call on Oct 20: the customer asked about multi-currency reporting and EU data residency. No pricing, discount or SLA was discussed.",
        "answer": "They asked about multi-currency and EU data residency. We also agreed a 20% discount and a 99.99% SLA.",
        "thinking_level": 2
      }' | jq
    ```

=== "Python"

    ```python
    import requests

    r = requests.post("http://localhost:8800/g1/eval", json={
        "prompt": "Summarize what this customer asked and what we committed to.",
        "context": "Call on Oct 20: the customer asked about multi-currency reporting and EU data residency. "
                   "No pricing, discount or SLA was discussed.",
        "answer": "They asked about multi-currency and EU data residency. "
                  "We also agreed a 20% discount and a 99.99% SLA.",
        "thinking_level": 2,
    }, timeout=300)
    r.raise_for_status()
    ev = r.json()
    print(ev["decision"], ev["flagged_axes"])
    ```

On a G-1 Studio deployment that fronts the proxy on port 8080, use `http://localhost:8080/gw/g1/eval`
(the `/gw` prefix, as for every proxy path — see the [API map](api-reference.md)).

### Response

```json
{
  "schema_version": "g1-eval-1.0",
  "product": {"name": "g1-proxy-light", "version": "1.0.10"},
  "product_version": "v1.0.10",
  "decision": "flagged",
  "flagged_axes": ["halluc_context"],
  "advisory_flags": [],
  "inputs": {"prompt": true, "context": true, "answer": true},
  "thinking_level": {"requested": 2, "tiers_used": ["geodesia_g", "geodesia_h"]},
  "axes": {
    "prompt_safety":   {"score": 0.012, "threshold": 0.6377, "flagged": false, "available": true, "role": "enforce"},
    "jailbreak":       {"score": 0.003, "threshold": 0.9864, "flagged": false, "available": true, "role": "enforce"},
    "rag_jailbreak":   {"score": 0.001, "threshold": 0.5,    "flagged": false, "available": true, "role": "enforce",
                        "details": {"note": "read on the supplied context"}},
    "answer_safety":   {"score": 0.004, "threshold": 0.5,    "flagged": false, "available": true, "role": "enforce"},
    "halluc_context":  {"score": 0.947, "threshold": 0.8164, "flagged": true,  "available": true, "role": "enforce"},
    "profanity":       {"score": 0.002, "threshold": 0.5,    "flagged": false, "available": true, "role": "advisory"},
    "prompt_complexity": {"score": 0.21, "threshold": 0.5,   "flagged": false, "available": true, "role": "classifier",
                          "label": "simple"}
  },
  "context_axes": {
    "prompt_safety": {"score": 0.004, "threshold": 0.6377, "flagged": false, "available": true, "role": "enforce"},
    "jailbreak":     {"score": 0.001, "threshold": 0.9864, "flagged": false, "available": true, "role": "enforce"}
  },
  "latency_ms": 412.7
}
```

The values above are illustrative. Read thresholds from the response, never from this page: they depend on the
release and on the `default` Application's policy.

| Field | Meaning |
|---|---|
| `schema_version` | `g1-eval-1.0`. Always the first key. |
| `product`, `product_version` | Which deployable answered (`g1-proxy` or `g1-proxy-light`) and its version, also as a tag (`v1.0.10`). Read from the version file in the image, so it always matches `GET /version`. |
| `decision` | `flagged` when at least one **enforce** axis is flagged (on the texts or on the context), otherwise `allowed`. Eval never blocks anything: it only reports. |
| `flagged_axes` | The flagged enforce axes. An axis flagged on the context appears as `context.<axis>`. |
| `advisory_flags` | Flagged **advisory** axes (for example `profanity`). They do not change `decision`. |
| `inputs` | Which fields were non-empty (`answer_2` appears only when it was used). |
| `thinking_level` | The level requested and the tiers that actually ran. If a tier could not start (for example level 3 on a GPU without enough memory), `tiers_used` says so — the call does not fail. |
| `axes` | One object per axis, with the **same shape as the chat API** (`score`, `threshold`, `flagged`, `available`, `role`, plus `label`, `raw_score`, `unavailable_reason`, `details` where they apply). See [Response Format](../reference/response-format.md). |
| `context_axes` | `prompt_safety` and `jailbreak` read on the `context` (only when both `prompt` and `context` are given): an instruction hidden in a document shows up here. |
| `reference_check` | Present when `verify_references` was requested: the per-reference result. |
| `latency_ms` | Server-side time of the evaluation. |

Axes that Eval computes differently from the chat API carry a `details.note` (for example `rag_jailbreak` read on
the prompt when there is no context). `prompt_safety.details.framed_harm_score` is the reading for harm requested
inside a frame (fiction, persona, hypothetical); it can only raise the flag.

### Errors

| Status | `error.code` | When |
|---|---|---|
| `400` | `invalid_json` | The body is not a JSON object. |
| `422` | `nothing_to_evaluate` | Neither `prompt` nor `answer` is filled. |

---

## Closed-book hallucination with a second answer

Without a context, the usual closed-book measure reads the **generator's own** token log-probabilities, which a
pasted answer does not have. Eval offers another way: send a second answer, generated independently for the same
prompt, in `answer_2`.

```bash
curl -s http://localhost:8800/g1/eval -H "Content-Type: application/json" -d '{"prompt": "Who designed the Eiffel Tower and when was it completed?", "answer": "It was designed by Gustave Eiffel'"'"'s company and completed in 1889.", "answer_2": "Maurice Koechlin and Émile Nouguier designed it for Gustave Eiffel'"'"'s firm; it was finished in 1889."}' | jq '.axes.halluc_closedbook'
```

```json
{
  "score": 0.31, "threshold": 0.85, "flagged": false, "available": true, "role": "enforce",
  "details": {
    "method": "consistency between two answers (NLI) + …",
    "clauses": [
      {"text": "It was designed by Gustave Eiffel's company", "start": 0, "end": 43,
       "contradiction": 0.07, "atoms_missing": []}
    ]
  }
}
```

The score combines two readings:

- **consistency**: a multilingual natural-language-inference model compares the two answers clause by clause.
  Facts that a model makes up tend to change from one generation to the next; facts it knows tend to stay.
- **a re-read of question and answer** by the grounding reader that is already loaded for `halluc_context`,
  which provides token-level probabilities in place of the generator's.

`details.clauses` lists up to 8 clauses of `answer`, most contradicted first, with the numbers and names
(`atoms_missing`) that the second answer does not contain. `answer_2` is used only when there is **no** context:
with a context, grounded hallucination (`halluc_context`) is the measure that applies.

!!! tip "How to get the second answer"
    Ask the same model the same question again, in a fresh conversation, with sampling on (temperature > 0).
    Two answers that are copies of each other carry no information; two answers from different models
    measure disagreement between models, not hallucination of the first one.

---

## In the UI

The configuration UI has an **Eval** page (G1-Proxy Light serves it itself; with G-1 Studio it is in the Studio
UI). It has four fields — Prompt, Context, Answer, Second answer — a thinking-level selector (L0–L3) and
**Evaluate**. The result shows the verdict, the tiers that ran, the latency and one row per axis with its score,
threshold and role. **Load example** fills in a CRM case with an invented discount and SLA.

When both a context and an answer are filled and `halluc_context` was scored, the page also offers
**Explain by clause**: the answer is split into clauses, and each clause gets its own score against the same
threshold, the verdict *supported / not supported*, and the closest sentence of the context. The row score shown
above is the served `halluc_context` value; the clauses are its breakdown. The same breakdown is available over
HTTP (it requires `Authorization: Bearer <GW_API_TOKEN>` when that token is set):

```bash
curl -s http://localhost:8800/v1/glad/causal-explainability/halluc-clauses \
  -H "Content-Type: application/json" \
  -d '{"context": "…", "answer": "…", "prompt": "…"}' | jq '{row_score, threshold, clauses: [.clauses[] | {text, score, flagged}]}'
```

Every evaluation run from the Eval page is also saved as a conversation (session id `eval-…`, one per browser
tab) in **Causal Explainability**. From there you can open it and run a word-level explanation of
`halluc_context`: the flagged clauses are reduced word by word to find which words make the claim unsupported, and
a graph links them to the closest sentence of the context. See [Causal Explainability](causal-xai.md).

API calls without a `session_id` are not added to Causal Explainability.

---

## OpenRouter-compatible endpoints

The same evaluation is also exposed in two wire formats that model marketplaces understand. Every one of these
runs exactly the `/g1/eval` evaluation.

| Method | Path | Contract |
|---|---|---|
| `GET` | `/g1/v1/models` | The model card: id (default `geodesia/g1-eval`), modalities `text` → `text`, `decisions`; prices; context length; capacity; data centers; `compliance.zdr`. |
| `POST` | `/g1/v1/chat/completions` | OpenAI chat. The evaluation JSON comes back as `choices[0].message.content`. |
| `POST` | `/g1/v1/systemone` | Decisions API (`{state, model, questions}` → `{model, answers, usage}`). |
| `POST` | `/g1/api/alpha/decisions` | Same Decisions contract, on the alpha path. |

These paths live under `/g1/v1` because `/v1/chat/completions` is the reverse proxy's own route.

### Chat completions

How `messages` maps onto the Eval fields:

| Message | Becomes |
|---|---|
| the last `user` message (with the earlier user turns of the conversation window, as the chat proxy reads them) | `prompt` |
| `system`, `developer`, `context`, `tool`, `document` messages, and a top-level `context` field | `context` |
| an `assistant` message after the last `user` message | `answer` |

In an **agent trace**, an assistant message that carries `tool_calls` is an action, not an answer, and is never
evaluated as one. A conversation that ends with a `tool` result is a turn still in progress: it has no answer yet,
so `halluc_context` is not scored. The final assistant message after the tool results is the answer.

Top-level `thinking_level`, `verify_references` and `reference_relevance_threshold` pass through. `stream: true`
returns SSE with keep-alive comments while scoring. Sampling parameters are ignored, since nothing is generated.
`usage` is an estimate (about 4 characters per token).

```bash
curl -s http://localhost:8800/g1/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "geodesia/g1-eval", "messages": [
        {"role": "system", "content": "Refund policy: refunds within 30 days with receipt."},
        {"role": "user", "content": "Can I get a refund after 45 days?"},
        {"role": "assistant", "content": "Yes, refunds are accepted up to 60 days."}]}' \
  | jq -r '.choices[0].message.content' | jq '{decision, flagged_axes}'
```

### Decisions API

One `noul` question per axis. Name the question after the axis (`prompt_safety`, `jailbreak`, `rag_jailbreak`,
`profanity`, `prompt_complexity`, `answer_safety`, `halluc_context`, `halluc_closedbook`), or give it any name and
set `instructions: {"axis": "<axis>"}`. `state` is a string (the prompt), an object `{prompt, context, answer}`, or a
list of chat messages.

```bash
curl -s http://localhost:8800/g1/v1/systemone \
  -H "Content-Type: application/json" \
  -d '{"model": "geodesia/g1-eval",
       "state": {"context": "Refunds within 30 days with receipt.", "answer": "Refunds are accepted up to 60 days."},
       "questions": {"halluc_context": {"type": "noul"}}}'
# {"model": "geodesia/g1-eval", "answers": {"halluc_context": {"type": "noul", "noul": 0.93}}, "usage": {...}}
```

`noul` is the axis score **re-centred on the axis' own threshold**: `0.5` is the threshold, and `noul > 0.5` means
the axis is flagged. Raw axis scores cannot be compared with each other, because every axis has its own threshold;
re-centred, they all read the same way.

A question type other than `noul`, an unknown axis, or an axis the given fields cannot measure returns `422` —
never an invented `0.5`.

### Access, capacity and the model card

| Variable | Default | Effect |
|---|---|---|
| `GW_EVAL_API_KEYS` | unset | Comma-separated Bearer keys required on the `/g1/v1/*` and `/g1/api/*` endpoints. Unset = open. |
| `GW_EVAL_MODEL_ID` | `geodesia/g1-eval` | Model id. A request for another model returns `404`. |
| `GW_EVAL_MAX_CONCURRENCY` | `4` | Concurrent evaluations. Above it the endpoints answer an immediate `429`. |
| `GW_EVAL_PRICE_REQUEST` / `_PROMPT` / `_COMPLETION` | `0` | USD prices published in the model card. |
| `GW_EVAL_CONTEXT_LENGTH` | `32768` | Declared context length. |
| `GW_EVAL_DATACENTERS` | `US` | Comma-separated ISO country codes. |

`compliance.zdr` in the model card is `false` while the [call log](#call-log) stores request texts.

!!! warning "`/g1/eval` itself has no key"
    `GW_EVAL_API_KEYS` protects the marketplace endpoints only. `POST /g1/eval` is open, like the rest of the
    data plane on a single-tenant install. If the port is reachable from outside, put the proxy behind your
    reverse proxy and restrict `/g1/` there.

---

## Call log

Every `/g1/eval` call (and every call through the endpoints above) is written to a local SQLite file on the
persistent volume. Statistics are always stored; the texts and the full response are stored too unless you turn
that off.

| Variable | Default | Effect |
|---|---|---|
| `GW_EVAL_LOG` | `1` | `0` disables the log entirely. |
| `GW_EVAL_LOG_DB` | `g1_eval.sqlite3` next to `GW_DB_PATH` | Path of the SQLite file. In the shipped images: `/app/var/g1_eval.sqlite3`. |
| `GW_EVAL_CAPTURE_BODY` | `1` | `0` keeps statistics only: no prompt, context, answer or response. |
| `GW_EVAL_EXPORT_TOKEN` | unset | Enables the two HTTP endpoints below. Unset = HTTP export off. |
| `GW_EVAL_CLIENT_SALT` | built-in | Salt for the client hash. |

!!! danger "Stored texts are customer data"
    With `GW_EVAL_CAPTURE_BODY=1` (the default), every prompt, context and answer sent to Eval is kept on disk.
    Decide explicitly whether that is acceptable for each installation. The client IP is never stored: only a
    salted hash prefix, enough to count distinct callers.

Writing the log never blocks or fails a call: if the write fails, the call is served anyway and the error is
printed in the container log.

### Each row

| Column | Content |
|---|---|
| `id`, `ts` | Unique id and UTC time (`2026-11-05T09:30:12Z`). |
| `product`, `product_version`, `schema_version` | Who answered, and with which schema. |
| `thinking_level`, `verify_references` | The request options. |
| `has_prompt`, `has_context`, `has_answer` | Which fields were filled. |
| `decision`, `flagged_axes`, `axes` | The verdict, and `{score, threshold, flagged}` per axis. |
| `latency_ms`, `status` | Server time and HTTP status. |
| `client`, `user_agent` | Salted hash of the client IP, and the user agent. |
| `request_body`, `response_body` | The texts and the full response; `null` when `GW_EVAL_CAPTURE_BODY=0`. |

### Downloading the log over HTTP

Two endpoints read the log. They need their **own** token, `GW_EVAL_EXPORT_TOKEN`. It is deliberately not the
gateway token and not your licence token: the export carries customer texts, and a gateway left without
`GW_API_TOKEN` would otherwise let anyone read them.

| Method | Path | Returns |
|---|---|---|
| `GET` | `/g1/eval/stats?since=` | JSON: calls, distinct clients, latency mean/p50/p95, and counts by day, decision, thinking level, input combination and flagged axis. |
| `GET` | `/g1/eval/export?since=&limit=` | NDJSON, one row per line, oldest first. |

Authenticate with `Authorization: Bearer <token>` or `X-Export-Token: <token>`.

`since` is compared with `ts` as text, so write it in the same UTC form: a date (`2026-11-01`) or a full
timestamp (`2026-11-01T08:00:00Z`). `limit` caps the number of rows; `0` or absent means all.

| Status | `error.code` | Meaning |
|---|---|---|
| `403` | `export_disabled` | `GW_EVAL_EXPORT_TOKEN` is not set in the container. |
| `401` | `unauthorized` | Wrong or missing token. |
| `404` | `log_disabled` | `GW_EVAL_LOG=0`. |

**1. Set the token.** Generate one and add it to the proxy service in your `docker-compose.yml` (for an install
made with `install.sh`, in the installation folder, `./geodesia-g1` by default):

```bash
openssl rand -hex 24
```

```yaml
services:
  g1-proxy-light:            # or g1-proxy
    environment:
      GW_EVAL_EXPORT_TOKEN: "paste-the-generated-value-here"
```

```bash
sudo docker compose up -d --force-recreate
```

**2. Download.** Once `/health` answers again:

```bash
TOK="paste-the-generated-value-here"
curl -s -H "Authorization: Bearer $TOK" "http://127.0.0.1:8800/g1/eval/stats?since=2026-11-01" | jq
curl -s -H "Authorization: Bearer $TOK" "http://127.0.0.1:8800/g1/eval/export?since=2026-11-01" > eval.ndjson
```

!!! warning "Line continuations"
    A `\` continues a shell command only as the **last character** of a line. Written in the middle of a line
    (`… \ "http://…"`) it turns the next argument into a URL that starts with a space, and curl rejects it.
    Keep the command on one line, as above, or put the `\` at the very end.

**Without HTTP.** The file can always be copied from the container, with or without the token:

```bash
sudo docker compose cp g1-proxy-light:/app/var/g1_eval.sqlite3 .
sqlite3 g1_eval.sqlite3 "SELECT ts, decision, flagged_axes FROM eval_calls ORDER BY ts DESC LIMIT 20"
```

---

## See also

- [Chat API](chat-api.md) — the same pipeline, with an LLM behind it.
- [Detection Axes](detection-axes.md) — what each axis measures.
- [Thinking Levels](thinking-levels.md) — what levels 0–3 run.
- [Response Format](../reference/response-format.md) — the per-axis object and the reference check.
- [Causal Explainability](causal-xai.md) — explanations of a verdict, down to the word.
