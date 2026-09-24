# Response Format Reference — `geodesia` object, schema 1.0

Every chat response from **G1-Proxy** and **G1-Proxy Light** is a standard OpenAI (or Ollama) response with
exactly **one** extra top-level key: `geodesia`. Nothing else is added. Everything Geodesia measured and decided
for the turn lives inside that object.

This page is the authoritative reference for the object. It applies to:

| Endpoint | Format |
|---|---|
| `POST /v1/chat/completions` | OpenAI chat completion / SSE chunks |
| `POST /v1/completions` | OpenAI legacy text completion |
| `POST /api/chat` | Ollama chat (JSON / NDJSON) |
| `POST /v1/glad/evaluate` | OpenAI chat completion |

!!! info "Versioning"
    `schema_version` is the version of this **API schema**, not of the product. It is always the first key of the
    object. It changes only when the shape of the object changes: a minor increase (`1.1`) adds optional fields; a
    major increase (`2.0`) removes or renames fields. Product releases (g1-proxy 0.4.2, …) do not change it.

---

## Example — answer allowed

```json
{
  "id": "chatcmpl-geodesia-1790234386650",
  "object": "chat.completion",
  "created": 1790234393,
  "model": "ministral3",
  "choices": [{
    "index": 0,
    "message": { "role": "assistant", "content": "The capital of France is **Paris**." },
    "logprobs": null,
    "finish_reason": "stop"
  }],
  "usage": { "prompt_tokens": 1826, "completion_tokens": 23, "total_tokens": 1849 },
  "geodesia": {
    "schema_version": "1.0",
    "event": "final",
    "decision": "allowed",
    "mode": "blocking",
    "reason": null,
    "axes": {
      "halluc_context": {
        "score": null, "threshold": 0.7551, "flagged": false, "available": false,
        "role": "enforce", "unavailable_reason": "no context supplied"
      },
      "halluc_closedbook": {
        "score": 0.4687, "threshold": 0.8555, "flagged": false, "available": true, "role": "advisory",
        "details": {
          "mean_surprisal": 0.369,
          "method": "sledge(head⊕logprob+conformal)",
          "fact_seeking": true,
          "abstained": false,
          "token_surprisal": [
            { "index": 0, "text": "The", "surprisal": 0.217, "start": 0, "end": 3 },
            { "index": 1, "text": " capital", "surprisal": 0.004, "start": 3, "end": 11 }
          ],
          "uncertain_span": { "text": "anything else", "start": 19, "end": 21, "surprisal": 1.093 }
        }
      },
      "prompt_safety": { "score": 0.0076, "threshold": 0.6377, "flagged": false, "available": true, "role": "enforce" },
      "answer_safety": { "score": 0.021, "threshold": 0.7953, "flagged": false, "available": true, "role": "enforce" },
      "jailbreak": {
        "score": 0.3333, "threshold": 0.9864, "flagged": false, "available": true, "role": "enforce",
        "details": { "head_score": 0.00059, "linear_probe_z": -9.6043 }
      },
      "rag_jailbreak": {
        "score": null, "threshold": 0.5768, "flagged": false, "available": false,
        "role": "advisory", "unavailable_reason": "no context supplied"
      }
    },
    "additional_axes": {
      "profanity": { "score": 0.026, "threshold": 0.7, "flagged": false, "available": true, "role": "advisory" },
      "out_of_scope": { "score": 0.0678, "threshold": 0.9534, "flagged": false, "available": true, "role": "advisory" },
      "prompt_complexity": {
        "score": 0.0155, "threshold": 0.5, "flagged": false, "available": true,
        "role": "classifier", "label": "simple"
      }
    },
    "grounding": {
      "available": true, "verdict": "grounded", "score": 0.7261, "risk": 0.4687,
      "margin": -0.4521, "regime": "closed_book", "axis": "halluc_closedbook"
    },
    "pii": {
      "enabled": true,
      "input": { "count": 0, "by_type": {} },
      "output": { "count": 0, "by_type": {} }
    }
  }
}
```

## Example — prompt blocked

The answer is withheld: `finish_reason` is `content_filter` and the message carries the block notice.

```json
{
  "id": "geodesia-block-1790232549895",
  "object": "chat.completion",
  "created": 1790232549,
  "model": "ministral3",
  "choices": [{
    "index": 0,
    "message": { "role": "assistant", "content": "[Geodesia blocked — jailbreak (input)]" },
    "finish_reason": "content_filter"
  }],
  "geodesia": {
    "schema_version": "1.0",
    "event": "final",
    "decision": "blocked",
    "mode": "blocking",
    "reason": { "stage": "input", "axis": "jailbreak", "detail": null },
    "axes": {
      "prompt_safety": { "score": 0.9143, "threshold": 0.6377, "flagged": true, "available": true, "role": "enforce" },
      "jailbreak": {
        "score": 0.9981, "threshold": 0.9864, "flagged": true, "available": true, "role": "enforce",
        "details": { "head_score": 0.993179, "linear_probe_z": 4.8373 }
      }
    }
  }
}
```

(Other axes omitted for brevity; a real response lists every axis the deployment serves.)

## Example — flagged in passthrough mode

With `"mode": "passthrough"` the real answer is returned and the violation is reported.

```json
"geodesia": {
  "schema_version": "1.0",
  "event": "final",
  "decision": "flagged",
  "mode": "passthrough",
  "reason": { "stage": "input", "axis": "jailbreak", "detail": null },
  "axes": { "…": "…" }
}
```

---

## Top-level fields

| Field | Type | Present | Description |
|---|---|---|---|
| `schema_version` | string | always | Version of this schema (`"1.0"`). Always the first key. |
| `event` | string | always | What this object reports: `final`, `input_scan`, `progress` or `research`. See [Streaming](#streaming). |
| `decision` | string | `final` only | `allowed`, `flagged` or `blocked`. See [Decision](#decision). |
| `mode` | string | `final` only | Enforcement mode applied to this request: `blocking` or `passthrough`. |
| `reason` | object \| null | `final` only | Why the decision is not `allowed`; `null` when it is. See [Reason](#reason). |
| `axes` | object | when scored | Primary detection axes, keyed by axis name. See [Axis object](#axis-object). |
| `additional_axes` | object | when scored | Annotation-only axes (never block), same shape as `axes`. |
| `grounding` | object | when scored | Fused grounding metric. See [Grounding](#grounding). |
| `thinking` | object | thinking level ≥ 1 | Which detector tiers produced the verdict. See [Thinking](#thinking). |
| `pii` | object | PII guard on | Personal data removed from the traffic, counts only. See [PII](#pii). |
| `rag` | object | knowledge base used | Retrieval sources and claim verification. See [RAG](#rag). |
| `routing` | object | complexity routing on | Which model answered and why. See [Routing](#routing). |
| `context_judge` | object | claim judge on | Per-claim grounding verdicts (advisory, uncalibrated). See [Context judge](#context-judge). |
| `certificate` | object | certificates on | Signed verdict certificate. See [Certificate](#certificate). |
| `reasoning_budget` | object | reasoning upstream | `max_tokens` raised so a reasoning model can still answer. |
| `token_saving` | object | `token_saving` switch on | `{enabled, cached_prompt_tokens?, prompt_tokens?, cache_write_prompt_tokens?, prompt_cache_key?, upstream_closed_early?, closed_reason?}`. `cached_prompt_tokens` is what the provider served from its prompt cache, present only when the provider reports it. `prompt_cache_key` is `client` or `gateway` (who set it), never the value. See [Token & Cost Control](../g1-proxy/cost-control.md#token-saving-spend-less-without-changing-what-the-model-reads). |
| `quota` | object | quota exceeded | Plan or budget limit that rejected the request. |
| `tool_guard` | object | tool guard blocked | MCP/tool-call findings that blocked the request. |
| `research` | object | `research` events | One web-search progress event. |

Optional sections are **omitted** when they do not apply; they are never sent as empty placeholders.

## Decision

| `decision` | Content returned? | Meaning |
|---|---|---|
| `allowed` | yes | No enforcing axis flagged. |
| `flagged` | yes | A policy violation was detected but the content was delivered: the request ran in `passthrough` mode, or (streaming) the violation was found only after the answer had been streamed. |
| `blocked` | no | Content was withheld. `finish_reason` is `content_filter` and the message carries a `[Geodesia blocked — …]` notice. |

A client that only needs "was this turn a violation?" can test `decision != "allowed"`.

## Reason

```json
"reason": { "stage": "input", "axis": "jailbreak", "detail": null }
```

| Field | Values | Description |
|---|---|---|
| `stage` | `input`, `output`, `tools`, `quota` | Where the decision was taken: the prompt, the generated answer, the tool/MCP pre-flight, or the plan/budget gate. |
| `axis` | axis name \| null | The axis that decided (highest-scoring flagged axis). `null` for `tools` and `quota`. |
| `detail` | string \| null | Extra context, e.g. `"system prompt leak"`, `"halted mid-stream"`, or the tool/quota message. |

## Axis object

Every entry of `axes` and `additional_axes` has the same shape.

| Field | Type | Description |
|---|---|---|
| `score` | number \| null | Risk score in [0, 1] that the decision is based on. `null` when the axis could not be measured on this turn — **never `0`**. |
| `threshold` | number \| null | Calibrated decision threshold. The axis is flagged when `score` exceeds it. |
| `flagged` | boolean | The axis is over its threshold. For `classifier` axes this means "on the far side of the boundary", not "unsafe". |
| `available` | boolean | `false` when the axis had nothing to read (no context, no logprobs, no answer yet). |
| `role` | string | `enforce` (flag withholds content in blocking mode), `advisory` (flag is a warning, never a block), `classifier` (a label, not a risk). |
| `label` | string | `classifier` axes only: the class, e.g. `simple` / `complex`. |
| `raw_score` | number | Present when the displayed `score` was aligned to the final decision (fusion, guards, context verification); the model's own score before that step. |
| `hard_block` | boolean | Present and `true` when an `advisory` axis withheld content anyway (e.g. closed-book hallucination above its extreme-confidence ceiling, or a verified system-prompt leak). |
| `unavailable_reason` | string | Present when `available` is `false`: human-readable reason. |
| `details` | object | Optional axis-specific evidence, see below. |

### Axes

| Axis | Group | Role | Reads | Description |
|---|---|---|---|---|
| `prompt_safety` | axes | enforce | prompt | Harmful request. |
| `jailbreak` | axes | enforce | prompt | Jailbreak / instruction override / system-prompt extraction. |
| `answer_safety` | axes | enforce | answer | Harmful answer content. |
| `halluc_context` | axes | enforce | answer + context | Answer not supported by the supplied context (RAG faithfulness). |
| `halluc_closedbook` | axes | advisory | answer + generator logprobs | Likely fabricated fact without context. Needs an upstream that exposes logprobs. |
| `rag_jailbreak` | axes | advisory | context | Prompt injection hidden in retrieved/supplied context. |
| `halluc_context_joint` | axes | advisory | answer + context | Joint claim–evidence verifier (when deployed). |
| `sysprompt_leak` | axes | enforce | answer | Verbatim or fragmentary reproduction of the protected system prompt/tool descriptions. Present when system-prompt protection is on. |
| `profanity` | additional_axes | advisory | prompt | Profane language. |
| `out_of_scope` | additional_axes | advisory | prompt + declared scope | Request outside the application's declared scope. |
| `prompt_complexity` | additional_axes | classifier | prompt | `simple` / `complex`; drives Model A/B routing. |

### `details` fields

| Field | Axis | Description |
|---|---|---|
| `method` | `halluc_closedbook` | Closed-book scoring method used. |
| `mean_surprisal` | `halluc_closedbook` | Mean per-token surprisal of the answer (nats). |
| `fact_seeking` | `halluc_closedbook` | The prompt asks for a fact (the axis gates on it). |
| `abstained` | `halluc_closedbook` | The model declined to answer. |
| `token_surprisal` | `halluc_closedbook` | Per-token surprisal: `[{index, text, surprisal, start, end}]` (`start`/`end` are token offsets). Only on the final event. |
| `uncertain_span` | `halluc_closedbook` | The most uncertain span: `{text, start, end, surprisal}`. |
| `context_support` | `halluc_context` | Lexical support of the answer by the context, in [0, 1]. |
| `suppressed_by` | hallucination axes | Why a flag was withdrawn, e.g. `context_support`, `rag_claim_verification`, `benign_tech_command`. |
| `head_score` | `jailbreak` | Score of the base detector head before the linear-probe combination. |
| `linear_probe_z` | `jailbreak` | Logit of the linear intent probe combined with the head. |
| `pooled_score` | prompt axes | Score before an input guard recovered a diluted or encoded attack. |
| `recovered_by` | prompt axes | `dilution_guard` or `decode_guard`: the guard that recovered the attack. |
| `flag_kept_from_level_0` | any | At thinking level ≥ 1 the fused score alone would not flag; the flag comes from the level-0 verdict (fusion can only add flags, never remove them). |
| `verification_status` | `halluc_context_joint` | Verifier status. |
| `leak_source` | `sysprompt_leak` | Which protected field leaked, e.g. `messages.system`, `tools.<name>.description`. |
| `leak_kind` | `sysprompt_leak` | `whole_field`, `verbatim` or `fragments`. |
| `leak_coverage` | `sysprompt_leak` | Fraction of the protected field reproduced. |
| `shadow_mode` | `sysprompt_leak` | `true` when the leak is only reported, not enforced. |

The protected text itself is never echoed back.

## Grounding

A single grounding number fusing `halluc_context` and `halluc_closedbook` on each axis's own threshold.

| Field | Description |
|---|---|
| `available` | `false` when neither axis could be measured. |
| `verdict` | `grounded`, `unsupported` or `not_measurable`. |
| `score` | [0, 1]; 1 = fully grounded, **0.5 = the calibrated operating point**. Below 0.5 an axis crossed its threshold. |
| `risk` | Raw probability of the deciding axis. |
| `margin` | Normalised distance from the threshold (negative = safe side). |
| `regime` | `context`, `closed_book`, `both` or `not_measurable`. |
| `axis` | Axis that produced the number. |
| `deciding_axis` | Axis whose flag produced `unsupported` (present only then). |
| `advisory_score` | Score of the advisory (out-of-regime) axis when both were measured. |
| `reasons` | Present when not measurable. |

## Thinking

Present when the request ran at thinking level ≥ 1.

```json
"thinking": { "level": 2, "tiers_used": ["geodesia_g", "geodesia_h"], "escalated": true }
```

`tiers_used` lists the detector tiers that **actually** contributed (`geodesia_g`, `geodesia_h`, `geodesia_a`,
`native`); it can be fewer than requested. `escalated` appears at level 1 only (cascade).

## PII

```json
"pii": { "enabled": true, "input": { "count": 1, "by_type": { "EMAIL": 1 } }, "output": { "count": 0, "by_type": {} } }
```

Counts per entity type only; detected values are never included.

## RAG

`{ "collection_id", "sources": [...], "source_count", "verification" }` — present when the request used a knowledge
base collection (`rag` request field). `verification` carries the claim-level check of the answer against the
retrieved sources.

## Routing

`{ "enabled", "used_complex_model", "axis", "score", "threshold", "model" }` — present when the application uses
complexity routing. `axis_missing: true` means the complexity axis was unavailable and Model A answered.

## Context judge

Present when `enable_judge_for_context_hallucination` is on. Advisory and uncalibrated: it never changes
`decision`, `axes` or `grounding`.

```json
"context_judge": {
  "available": true, "calibrated": false, "score": 0.82, "threshold": 0.5, "flagged": true,
  "verdict_counts": { "supported": 3, "unsupported": 1 },
  "claims_total": 4, "claims_judged": 4, "truncated": false, "model": "Qwen3.5-9B", "seconds": 6.4,
  "claims": [
    { "index": 3, "start": 120, "end": 171, "verdict": "unsupported", "quote": null,
      "quote_verified": false, "confidence": 0.9, "risk": 0.82 }
  ]
}
```

`verdict` per claim is `supported`, `contradicted`, `unsupported`, `no_claim` or `unjudged`. When the judge cannot
run, `available` is `false`, `score` is `null` and `reason` explains why.

## Certificate

Present when certificates are enabled (`GW_CERTIFICATE=on`). A self-contained, optionally HMAC-signed record of the
verdict, with the same axis vocabulary as above.

```json
"certificate": {
  "version": "geodesia-cert-3",
  "verdict": "allowed",
  "alpha": 0.05,
  "brake_alpha": 0.025,
  "axes": {
    "jailbreak": {
      "score": 0.3333, "available": true, "threshold": 0.9864, "flagged": false,
      "role": "enforce", "tier": "primary", "alpha": 0.05,
      "fpr_bound": "P(benign_score > threshold) <= 0.05", "threshold_method": "split-conformal"
    }
  },
  "calibration": { "model": null, "bank_version": 0 },
  "additional_axes": ["out_of_scope", "profanity", "prompt_complexity"],
  "sig": "hmac-sha256:…"
}
```

`verdict` is the **policy** verdict: `blocked` when an axis that withholds content in blocking mode flagged,
independently of the request's `mode`. In `passthrough` a turn can therefore carry `decision: "flagged"` with
`certificate.verdict: "blocked"` — the certificate attests what the policy would do, `decision` what the gateway
did. `sig` is `null` when no signing key is configured.

---

## Streaming

With `"stream": true` the response is a sequence of SSE chunks. Each chunk that carries Geodesia information has a
`geodesia` object with an `event`:

| `event` | When | Contains |
|---|---|---|
| `input_scan` | As soon as the prompt is scored (before or while the answer streams) | `axes` / `additional_axes` for the prompt axes. No decision. |
| `research` | Web search only, before the answer | `research`: one progress event (`search_started`, `page_found`, `page_read`, `page_blocked`, `page_skipped`, `search_done`, `search_error`). `page_read` / `page_blocked` carry `axes: {<axis>: {score, threshold, flagged}}`; `page_blocked` also `reason` and `axis` (the deciding axis). |
| `progress` | Periodic re-scoring during generation | `axes` with the current scores. No decision. |
| `final` | Last chunk (with `finish_reason`) | The complete verdict, same shape as the non-streaming object. |

Rules for clients:

- Take the verdict **only** from the `final` event. `input_scan` and `progress` are informational.
- If the answer is halted mid-stream, the `final` event has `decision: "blocked"`, `reason.stage: "output"` and the
  chunk's `finish_reason` is `content_filter`.
- A violation found after the answer was fully streamed is reported as `decision: "flagged"` (the text was already
  delivered).
- A request with `web_search: true` is always answered as a stream (`research` events first), whatever `stream` says.
- A quota refusal (`reason.stage: "quota"`) ends with `finish_reason: "stop"`; content and tool-guard blocks end with
  `content_filter`. Every block is delivered in the format the client asked for (SSE / NDJSON when streaming).

```text
data: {"id":"chatcmpl-geodesia-…","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":null}],
       "geodesia":{"schema_version":"1.0","event":"input_scan","axes":{"prompt_safety":{…},"jailbreak":{…}}}}
data: {"id":"chatcmpl-geodesia-…","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"Waves"},"finish_reason":null}]}
…
data: {"id":"chatcmpl-geodesia-…","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}],
       "geodesia":{"schema_version":"1.0","event":"final","decision":"allowed","mode":"blocking","reason":null,"axes":{…}}}
data: [DONE]
```

Ollama streaming (`/api/chat`, NDJSON) carries the same objects on its frames.

---

## Request fields

Geodesia-specific request fields (all optional, never forwarded to the upstream model):

| Field | Type | Description |
|---|---|---|
| `mode` | string | `block` / `blocking` or `passthrough`. Overrides the deployment default for this request. |
| `thinking_level` | integer | 0–3. Default 2. See [Thinking Levels](../g1-proxy/thinking-levels.md). |
| `context` | string | Grounding document for `halluc_context` (also injected into the upstream prompt). |
| `rag` | object | `{ "collection_id": "…" }` — retrieve context from a knowledge base. |
| `pii_guard` | boolean | Turn the PII guard on/off for this request. |
| `scan` | boolean | `false` bypasses all scoring for this request (internal tools only). |
| `axes` | array \| string | Restrict the reported axes. Enforcing axes are always scored. |
| `threshold_overrides` | object | Per-axis thresholds, e.g. `{ "jailbreak": 0.9 }`. |
| `domain` | string | Closed-book calibration domain (`general`, `legal`, …). |
| `web_search` | boolean | Live web search with per-page screening. |
| `enable_judge_for_context_hallucination` | boolean | Per-claim judge (`context_judge`). |
| `application_id` / `session_id` | string | Application routing and conversation grouping. |

---

## Migrating from the pre-1.0 payload

Before schema 1.0 the gateway added several top-level keys and used internal names. They are gone; there is no
compatibility layer.

| Before | Schema 1.0 |
|---|---|
| top-level `glad_decision` (`passed` / `blocked`) | `geodesia.decision` (`allowed` / `flagged` / `blocked`) |
| top-level `glad_mode` | `geodesia.mode` |
| top-level `glad_scores.safety_decision_rule`, `geodesia.flagged_axis` | `geodesia.reason.axis` |
| top-level `geodesia_research` (stream) | `geodesia.research` with `event: "research"` |
| `geodesia.axis_energy` | `geodesia.axes` |
| axis `p_detector` / `flag` / `verdict` | `score` / `flagged` / `label` |
| axis `p_model`, `p_detector_raw` | `raw_score` |
| axis `active` | `available` |
| axis `p_pozzi`, `lin_z` | `details.head_score`, `details.linear_probe_z` |
| axis `combinato`, `guardia_*`, `p_v5`/`p_v6`/`p_v7`, `sledge_*`, `prem_*`, `tier` | removed (internal) |
| `closedbook_method`, `lsc_span`, `token_surprisal[].i/.s` | `details.method`, `details.uncertain_span`, `details.token_surprisal[].index/.surprisal` |
| `geodesia.dominant_axis`, `geodesia.brake`, `geodesia.input` | removed (use `decision` and `reason`) |
| axis `tier` | removed: the group (`axes` / `additional_axes`) says it; still present inside the certificate |
| `energy_unit`, `energy_dHmax_joule`, axis `delta_E_joule` / `p_energy` (debug payload, `GW_CLEAN_PAYLOAD=0`) | removed |
| `dilution_guard` / `decode_guard` blocks, axis `dilution_recovered` / `decode_recovered` | `details.pooled_score` + `details.recovered_by` |
| `context_judge.flag`, `claims[].i`, `claims[].span`, `claims_truncated`, `advisory`, `score_kind` | `context_judge.flagged`, `claims[].index`, `claims[].start`/`end`, `truncated` (advisory is implied) |
| research page `axes.{p, thr, flag}`, `dominant` | `axes.{score, threshold, flagged}`, `axis` |
| `geodesia.sysprompt_leak` | `geodesia.axes.sysprompt_leak` (the protected text is no longer echoed) |
| `geodesia.thinking_level`, `thinking_tiers_used`, `thinking_escalated` | `geodesia.thinking.level`, `.tiers_used`, `.escalated` |
| tier names `glad_g`, `glad_h`, `glad_a`, `tier1_native` | `geodesia_g`, `geodesia_h`, `geodesia_a`, `native` |
| `geodesia.pii_guard` (`by_label`) | `geodesia.pii` (`by_type`) |
| `geodesia.ragionamento` | `geodesia.reasoning_budget` |
| `geodesia.entitlement` + `glad_decision: "quota_exceeded"` | `geodesia.quota` + `decision: "blocked"`, `reason.stage: "quota"` |
| `geodesia.mcp` | `geodesia.tool_guard` |
| `rag.n_sources` | `rag.source_count` |
| certificate `glad-cert-2` (`p_detector`, `flag`, `verdict`, `p_model`, `thr_kind`, `calib`) | `geodesia-cert-3` (`score`, `flagged`, `label`, `raw_score`, `threshold_method`, `calibration`) |
| request `glad_mode`, `glad_thinking_level`, `glad_scan`, `glad_pii_guard`, `glad_axes`, `glad_enable_judge_for_context_hallucination` | `mode`, `thinking_level`, `scan`, `pii_guard`, `axes`, `enable_judge_for_context_hallucination` |
