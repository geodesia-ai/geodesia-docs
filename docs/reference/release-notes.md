# Release Notes

Changes that affect how you integrate with Geodesia. The product version (for example `0.4.2`) and the
[response schema version](response-format.md) (`schema_version`, for example `"1.0"`) are separate numbers: the
schema version changes only when the shape of the `geodesia` object changes.

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

### Security fix

- **System-prompt leak axis.** When the `sysprompt_leak` axis detects that an answer reproduces the protected
  system prompt or tool descriptions, the response no longer echoes a fragment of the protected text back to
  the caller. It reports only where the leak came from (`details.leak_source`), its kind and its coverage.
