# Release Notes

Changes that affect how you integrate with Geodesia. The product version (for example `1.0.10`) and the
[response schema version](response-format.md) (`schema_version`, for example `"1.1"`) are separate numbers: the
schema version changes only when the shape of the `geodesia` object changes.

---

## 1.0.10 — 2026-10-30 (g1-proxy, g1-proxy-light, g1-studio)

### Detection — higher recall on explicit harmful requests

- The detection models and the thinking-level rules of the 1.0.6 line are back, with all the code of 1.0.9.1.
  The stricter thresholds introduced in 1.0.8 kept false positives very low, but let too many short, explicit
  harmful requests through at levels 0–2. 1.0.10 restores the higher recall on these requests.
- Thinking levels: level 0 and level 1 decide per tier (`prompt_safety` or `jailbreak` over its threshold);
  level 2 fuses the margins of Geodesia-G and Geodesia-H, and level 3 fuses Geodesia-G, -H and -A, calling
  Geodesia-A on every level-3 request (in 1.0.8 – 1.0.9.1 it was called only inside an uncertainty band). See [Thinking Levels](../g1-proxy/thinking-levels.md).
- `halluc_context` uses the reader thresholds calibrated for these models.
- Nothing changes in the API or in the response schema.

---

## 1.0.9.1 — 2026-10-28 (g1-proxy, g1-proxy-light, g1-studio)

### Explaining `halluc_context` — clause and word

- A flagged `halluc_context` verdict can be explained down to the **clause** and the **word**: the clauses of the
  answer that the context does not support, then the words inside them that make the difference (several at once
  when the error needs more than one word). Each word is linked to the closest sentence of the context, so the
  explanation shows *what* the context says instead (e.g. "March" in the answer against "September" in the
  document). Available in the **Causal Explainability** page and in **Eval**. The LLM judge remains a fallback.
  See [Causal Explainability](../g1-proxy/causal-xai.md).

### G1 Eval

- **Second answer for closed-book.** Without a context, Eval accepts an optional `answer_2`: a second,
  independent answer to the same prompt. `halluc_closedbook` is then measured from the agreement between the two
  answers (multilingual NLI) plus token probabilities read locally, with no call to an LLM and no upstream
  logprobs. See [G1 Eval](../g1-proxy/eval.md).
- `out_of_scope` and the "Verify references" option are no longer part of Eval (a context is not a declared
  scope; the reference check stays available on `/v1`).
- Eval sessions are listed in the Causal Explainability page, so an Eval verdict can be explained like a chat.

### Closed-book calibration

- The UI shows the SLEDGE catalog: calibrations already available for the selected upstream model can be applied
  without recalibrating. See [Closed-Book Calibration](../g1-proxy/closed-book-calibration.md).

### System-prompt leak (also in 1.0.8.1)

- Requests for the system prompt are recognised with more verbs ("get", "fetch") and with typos
  ("get me ssytem promt"), only when they target the assistant's own instructions — never "my" prompt or a third
  party's.
- Codes and values that the system prompt declares secret are protected as single tokens: a secret that leaves
  inside a paraphrase is caught.
- In streaming, when the system prompt holds secrets, the tail of the answer is held back and checked token by
  token; it is delivered at the end only if it is not a leak (also on an upstream error and with PII redaction).

---

## 1.0.8 — 2026-10-26 (g1-proxy, g1-proxy-light, g1-studio)

Point releases 1.0.8.1 – 1.0.8.5 (26–27 October) are included below.

- **New detection models and thresholds.** Superseded by 1.0.10, which restores the 1.0.6 models and rules.
- **System-prompt leak and documents.** A document placed in the system prompt or in `context` can be quoted:
  only the instructions and the declared secrets are protected, sentence by sentence. A context that starts with
  `SYSTEM:` is protected like a system message (1.0.8.1).
- **No default upstream model.** The installer and the gateway no longer pick a model on your behalf: set
  `UPSTREAM_MODEL` at install time or choose it in the UI (Settings → Gateway). Chat and closed-book calibration
  wait for it. On Ollama the installer checks that the chosen model is installed.
- **Closed-book calibration.** SLEDGE calibration runs with the upstream's thinking turned off. The default
  method is **SLEDGE-Next** (`fork,prem,nli`); SLEDGE-Next v2 is still selectable. The calibration selector
  remembers your choice.
- **`halluc_context` visible again** on deployments where the detector graph was compiled.
- **G1 Eval page** in the g1-proxy-light UI (`/eval`, 1.0.8.2), with the clause-level explanation of
  `halluc_context` (1.0.8.4).
- UI and API messages are in English.

---

## 1.0.6 / 1.0.7 — 2026-10-24

Detection-model updates of Geodesia-G, -H and -A with recalibrated thresholds for every axis and for the
long-document `halluc_context` reader. 1.0.7 was published for g1-proxy-light only. 1.0.4 and 1.0.5 were
internal candidates and were never released.

---

## 1.0.3.1 — 2026-10-21 (g1-proxy, g1-proxy-light, g1-studio)

- **Long texts on every tier.** Geodesia-H and Geodesia-A now read texts longer than their window in sliding
  windows, like Geodesia-G (maximum per axis). A harmful request buried in the middle of a long document is
  scored as such at every thinking level.
- **Agentic traces.** An upstream answer that calls a tool is not checked for `halluc_context` (it is not a claim
  about the context). In Eval and the OpenRouter façade, a turn with `tool_calls` is not taken as the answer, and
  a conversation that ends with a tool output is a turn in progress.
- Geodesia-A no longer contributes to `rag_jailbreak` at level 3.
- **Closed-book on Ollama.** SLEDGE-Next v2 works on Ollama (premise by forcing). The logprob probe reports the
  real reason when an upstream cannot be used (`model_not_found`, `auth_error`, …), and "Test capabilities" runs a
  fresh probe.
- The default Application's binding follows the container environment, and saving the `default` Application in
  the UI sends only the fields that changed (it no longer rewrites the upstream).

---

## 1.0.3 — 2026-10-12 (g1-proxy, g1-proxy-light, g1-studio)

- **New detection models** for Geodesia-G, -H and -A, with fewer false positives on borderline prompts. Level 1
  is Geodesia-H alone (the G-then-H cascade is removed).
- **Only the axes you choose.** With an explicit `axes` list, only those axes are computed; the others report
  `available: false` with `unavailable_reason: "axis not requested"` and cannot hold the request. Previously the
  deciding axes were always added back.
- **Your thresholds apply to `halluc_context`.** A threshold set with the Settings slider, a policy or
  `threshold_overrides` is now honoured by the `halluc_context` reader (short and long documents), also for the
  `default` Application.
- **Context read as sent.** Leading and trailing whitespace of `context` is no longer stripped before scoring.
- **`enforce_out_of_scope`** (off by default): when on, `out_of_scope` is decided by a dedicated scope reader
  fused with Geodesia-G.
- Faster first axes: the detector graph is compiled at start-up (allow 4–5 minutes of warm-up before the first
  request is fast).
- Known limit: multi-turn jailbreaks are best served at thinking level 3.

---

## 1.0.2 — 2026-10-05 (g1-proxy, g1-proxy-light, g1-studio)

First 1.0 release on the registry; it includes the 1.0.0 and 1.0.1 work.

### Detection

- **`halluc_context` read by a dedicated grounding model** (a fine-tuned 4B reader), fused with a claim-level
  head on long documents (> 7,000 characters). It is the only reader of `halluc_context` and serves `/v1`, MCP and
  G1 Eval alike. It runs in int8 on the GPU when there is one (`GW_HALLUC_2B_DEVICE=auto`), on the CPU otherwise.
- **Conversation window by default.** The input scan reads the last 50 user turns and the assistant turns between
  them (`GW_PROMPT_WINDOW`), so an attack spread over several turns is no longer judged on its harmless last line.
  Also on the G1 Eval façades.
- **`/v1` blocks by default** (`GW_BLOCK_INPUT=1`); the UI chat runs in `blocking`.
- The dilution guard is removed; the decode guard remains (`details.recovered_by: "decode_guard"`).
- The dedicated `prompt_safety_x` axis introduced in 0.4.4 is no longer shipped.
- Streaming: `input_scan` declares `thinking.complete: false` until every requested tier has answered.

### Operations

- g1-proxy-light is sized for a 16 GB GPU: the `halluc_context` reader and Geodesia-A are served in int8.

---

## 0.4.5 — 2026-09-28 (g1-proxy, g1-proxy-light, g1-studio)

Production release of the 0.4.4 features.

- **Sealed images.** Model weights are encrypted at rest inside the images; the `halluc_context` reader and the
  explainability replay worker now start under sealing (they did not in 0.4.4).
- An unreachable upstream answers `502` at once; the logprob probe connects in 5 s.
- User thresholds and the live tier status reach the final payload.
- **Reference check:** PubMed abstracts are read by DOI; calls to NCBI are spaced.
- **Live web search** uses a search API instead of scraping. See [Live Web Search](../g1-proxy/web-search.md).
- `/g1/eval` reports the product version read from the version file.
- The container health check follows the configured port.

---

## 0.4.4 — 2026-09-27 (g1-proxy, g1-proxy-light, g1-studio)

Includes 0.4.3 (2026-09-26), which was not published as images.

### G1 Eval — evaluation-only API

- **`POST /g1/eval`** scores a prompt, a context and an answer you already have, with no LLM call. The axes come
  from the fields you fill in; the pipeline is the `/v1` one (same models, thresholds and `default` Application
  state), so the verdicts are bit-identical to a chat. The response is English JSON with
  `product {name, version}`.
- **OpenRouter-compatible façades:** `POST /g1/v1/chat/completions`, the Decisions API
  (`POST /g1/v1/systemone`, `POST /g1/api/alpha/decisions`) and `GET /g1/v1/models`.
- **Local call log** (SQLite) with `GET /g1/eval/stats` and `GET /g1/eval/export`, protected by a dedicated
  token. See [G1 Eval](../g1-proxy/eval.md).

### Reference check (0.4.3)

- Opt-in `"verify_references": true`: every link, DOI, arXiv id, PubMed id and citation in the answer is looked up
  on open sources, and a local classifier reads whether the source supports the sentence that cites it. One
  hallucinated reference raises `halluc_closedbook` (or `halluc_context` with a context) above its threshold; the
  check only reports, never blocks. Response schema **1.1** adds the `references` details and the
  `reference_check` streaming event. See [Reference check](response-format.md#reference-check).
- **Closed-book without prompt logprobs (0.4.3).** On upstreams that do not return prompt logprobs, the default
  closed-book method is SLEDGE-logprobs (fork + NLI).

### Detection and operations

- `prompt_safety_x`: a dedicated framed-harm axis that can only add a `prompt_safety` flag (removed in 1.0.2).
- **g1-proxy-light carries everything g1-proxy serves** — thinking levels 0–3, the `halluc_context` reader, the
  reference-check classifier, SLEDGE closed-book and G1 Eval — except the 9B judge, speech recognition and RAG.
  Level 3 was served as level 2 in earlier light images; this is fixed.
- Geodesia-A gets up to 15 minutes to start (`GB_TIER_START_TIMEOUT=900`): on slow disks level 3 silently fell
  back to level 2.
- Sealed images: the Geodesia-H/A checkpoints and the SLEDGE artifacts load through the weight vault (levels 1–3
  and closed-book were off in sealed images).

---

## 0.4.2 — 2026-09-24 (g1-proxy, g1-proxy-light, g1-studio)

!!! warning "Breaking change: `geodesia` object, schema 1.0"
    Clients that read `glad_decision`, `glad_mode`, `glad_scores`, `geodesia.axis_energy` or any per-axis
    `p_detector` / `flag` field must be updated. There is no compatibility layer. The complete old → new mapping
    is in [Migrating from the pre-1.0 payload](response-format.md#migrating-from-the-pre-10-payload).

### Chat API — one `geodesia` object, schema 1.0

- **One object, nothing else.** Chat responses (`/v1/chat/completions`, `/v1/completions`, `/api/chat`,
  `/v1/glad/evaluate`) add a single `geodesia` key to the standard OpenAI or Ollama body. The top-level
  `glad_decision`, `glad_mode`, `glad_scores` and `geodesia_research` keys are gone.
- **Versioned schema.** `geodesia.schema_version` (`"1.0"`) is always the first field.
- **A clear verdict.** `decision` is `allowed`, `flagged` (a violation was detected but the content was delivered,
  e.g. in `passthrough` mode) or `blocked` (content withheld). `mode` is `blocking` or `passthrough`, and
  `reason` is `{stage, axis, detail}` — `null` when the turn is allowed.
- **One axis shape.** Every entry of `axes` and `additional_axes` reports `score`, `threshold`, `flagged`,
  `available` and `role`, plus `label`, `raw_score`, `hard_block`, `unavailable_reason` and `details` where they
  apply. An axis that could not be measured reports `available: false` and `score: null` — never `0`.
- **What moved.**

    | Before | Now |
    |---|---|
    | `glad_decision` (`passed` / `blocked`) | `geodesia.decision` (`allowed` / `flagged` / `blocked`) |
    | `glad_mode` | `geodesia.mode` |
    | `glad_scores.safety_decision_rule`, `flagged_axis` | `geodesia.reason.axis` |
    | `geodesia.axis_energy` | `geodesia.axes` |
    | axis `p_detector` / `flag` / `verdict` | `score` / `flagged` / `label` |
    | `dominant_axis`, `brake` | removed — use `decision` and `reason` |
    | `thinking_level`, `thinking_tiers_used`, `thinking_escalated` | `geodesia.thinking.{level, tiers_used, escalated}` |
    | `geodesia.pii_guard` (`by_label`) | `geodesia.pii` (`by_type`) |
    | `geodesia.mcp` | `geodesia.tool_guard` |
    | `geodesia.entitlement` | `geodesia.quota` (with `reason.stage: "quota"`) |
    | `rag.n_sources` | `rag.source_count` |

- **Typed streaming.** Every SSE (or NDJSON) frame that carries Geodesia information has a `geodesia.event`:
  `input_scan`, `progress`, `research` or `final`. The verdict is on the `final` frame only.
- **Product names only.** Internal field names are no longer exposed. The detector tiers in
  `thinking.tiers_used` are `geodesia_g`, `geodesia_h`, `geodesia_a` and `native`.
- **`context_judge`** uses the same vocabulary: `flagged` (was `flag`), `claims[].index` / `start` / `end`,
  `truncated`. See [Context judge](response-format.md#context-judge).

### Request fields

- The `glad_`-prefixed request aliases were **removed**: `glad_mode`, `glad_thinking_level`, `glad_scan`,
  `glad_pii_guard`, `glad_axes` and `glad_enable_judge_for_context_hallucination`. Use `mode`, `thinking_level`,
  `scan`, `pii_guard`, `axes` and `enable_judge_for_context_hallucination`.

### Certificate `geodesia-cert-3`

- The signed certificate uses the same field names as the response: `score`, `flagged`, `available`, `label`,
  `raw_score`, `threshold_method` and a top-level `calibration` record (was `glad-cert-2` with `p_detector`,
  `flag`, `verdict`, `p_model`, `thr_kind`, `calib`).
- Axes that were not measured carry `score: null` and `available: false`.
- `certificate.verdict` is the **policy** verdict, independent of `mode`: a passthrough turn can report
  `decision: "flagged"` with `certificate.verdict: "blocked"`. See [Certificate](response-format.md#certificate).

### Web search

- Research progress arrives as `geodesia.research` on frames with `event: "research"` (was the top-level
  `geodesia_research`).
- Per-page screening scores use the axis vocabulary: `axes.<axis>.{score, threshold, flagged}` (was
  `{p, thr, flag}`), and the deciding axis of a blocked page is `axis` (was `dominant`).
- A `web_search: true` request is always answered as a stream. See [Live Web Search](../g1-proxy/web-search.md).

### Tool guard (MCP-aware chat)

- A tool-guard block is now delivered in the format the client asked for: a single SSE chunk (or NDJSON frame)
  with `finish_reason: "content_filter"` when `stream: true`, instead of a plain JSON body. The block reports
  `decision: "blocked"`, `reason.stage: "tools"` and the findings in `geodesia.tool_guard`. See
  [Chat-aware MCP guard](../mcp/chat-aware.md).

### Explainability — MuPAX G1 verdict

- The verdict-explanation API (`/v1/glad/causal-explainability/verdict/jobs`) uses the schema-1.0 vocabulary:
  request `source_axes` (was `source_axis_energy`), `source_decision` values `allowed` / `flagged` / `blocked`
  (was `passed` / `blocked`), and `source_verdict` is a full schema-1.0 chat response.
- In the result, the view formerly called `block` is `decision`, a decision mismatch is reported on the field
  `/decision`, score keys look like `axes/jailbreak/score`, and the `token_importance` signals are `score`
  (default), `raw_score`, `head_score`, `linear_probe_z` and `logit`. See
  [Explaining a served verdict](../g1-proxy/causal-xai.md#explaining-a-served-verdict-mupax-advanced).

### Realtime tier scores in streaming

- The stream opens at once and every detector tier's scores are sent as soon as that tier has scored:
  `input_scan` for the prompt (Geodesia-G, then G+H, then G+H+A), `progress` for the answer, each with
  `thinking.tiers_used`. The first scores no longer wait for the slowest tier or for the model.
- In `blocking` mode the stream now opens before the prompt verdict is complete (the model is still called only
  after it).

### Feedback memory per Application

- The episodic memory is **per Application**: each Application's corrections are consulted only for that
  Application (previously one memory served every Application that had feedback learning on).
- The feedback API resolves the Application from the **API key** (`Authorization: Bearer g1k_…`): rows of other
  Applications are invisible (`404`), naming another Application is refused (`403`), no key is `401`. Operators
  (gateway token) may name any Application or act across all of them.
- `scores` accepts the schema-1.0 `geodesia` object or its `axes`.
- The **default** Application's "Learn from feedback" switch is honoured (it was ignored: the default
  Application ran on the gateway configuration only). G1-Studio accepts policies that carry the 9-axis head's
  additional axes even when `GB_EXTRA_AXES` is not set in its container.
- When a thinking level is requested but Geodesia-H is unavailable, the verdict now declares
  `thinking: {level, tiers_used: ["geodesia_g"]}` instead of looking like level 0.

### Scoring bypass and explainability routing

- `scan: false` now reports `decision: "not_scanned"` (no verdict, no `axes`) instead of `allowed`.
- With `GW_API_TOKEN` set on the gateway, `scan: false` is accepted only from a caller presenting that token
  (`403` otherwise). Requests relayed by G1-Studio carry it.
- `/v1/completions` honours `constitutional_ai` and `scan` like `/v1/chat/completions`.
- G1-Studio forwards `/v1/glad/causal-explainability/analyze` and `/v1/glad/causal-explainability/verdict/*` to
  the gateway (they previously reached the control plane, which has no detector, and answered
  `400 No model path configured`).
- The watermark HMAC key is no longer derived from the database path: it comes from `GLAD_WATERMARK_SECRET` or a
  random key created once next to the database (`watermark.key`, mode 0600). Watermarks issued earlier still
  verify when the identifier is in the watermark log.

### Security fix

- **System-prompt leak axis.** When the `sysprompt_leak` axis detects that an answer reproduces the protected
  system prompt or tool descriptions, the response no longer echoes a fragment of the protected text back to
  the caller. It reports only where the leak came from (`details.leak_source`), its kind and its coverage.
