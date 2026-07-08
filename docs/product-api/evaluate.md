# Evaluate Endpoint

The evaluate endpoint generates a response from the loaded model and scores it in a single call. It is the core API for direct batch evaluation workflows where you supply a prompt and want both the generated answer and its full detection scores in one HTTP round-trip.

---

## POST /glad/evaluate

### Request Body (`EvaluateRequest`)

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `model_path` | `string` | ✅ | — | Absolute path to the model checkpoint directory. Relative paths are resolved against the working directory. |
| `prompt` | `string` | ✅ | — | The user's input prompt. Minimum length: 1 character. |
| `context` | `string` | — | `null` | Optional grounding context. When provided, the `halluc_context` (faithfulness) axis scores the answer against this text. |
| `generation_config` | `object` | — | See below | Parameters that control how the model generates the answer. |
| `session_id` | `string` | — | auto-generated | Logical grouping for a conversation. Multiple calls with the same `session_id` are linked in the audit trail. |
| `system_prompt_text` | `string` | — | `null` | The exact system or constitutional prompt prepended at generation time. Used by the MuPAX explainability engine to exclude those tokens from attribution, so explanations cover only user text and the generated answer. |
| `explain` | `boolean` | — | `false` | When `true` and the server has explainability enabled, compute per-token MuPAX attribution and include it in the response under `xai.mupax_halluc` / `xai.mupax_safety`. |
| `explain_mode` | `string` | — | `"standard"` | `"standard"` returns χ attribution per answer token. `"causal"` additionally computes a token→token causal matrix for the highest-importance flagged answer token — answering "which prompt tokens caused this specific answer token?" |
| `credit_tiers` | `array[string]` | — | `["mupax"]` when `explain=true` | Which attribution methods to run. Options: `"gradient"` (fast, deterministic), `"learned"` (learned attribution head if present), `"mupax"` (statistically robust Monte Carlo), `"pss"` (Positional Semantic Stability — consistency-based, ~N× generation cost). |
| `pss_n_samples` | `integer` | — | `5` | Number of extra generation samples for PSS attribution. Range 2–16. Each extra sample costs roughly one generation pass. |
| `pss_temperature` | `float` | — | `0.7` | Sampling temperature for PSS resamples. Must be > 0. |
| `pss_match_mode` | `string` | — | `"ngram"` | PSS alignment algorithm. Options: `"ngram"` (combines n-gram containment + entity strict match), `"strict"` (exact surface match), `"fuzzy"` (Levenshtein distance), `"entity"` (entity-only), `"claim"` (sentence-level bidirectional). |
| `threshold_overrides` | `object` | — | `null` | Runtime threshold overrides in probability space. Supported keys: `prompt_safety`, `answer_safety`, `halluc`, `combined_halluc`. |
| `bypass_prompt_block_for_generation` | `boolean` | — | `false` | When `true`, an unsafe-prompt detection does not abort generation. The model still generates a response, all scores are computed, but no content is withheld. Used by the "passthrough" review mode. |
| `enable_edfl_isr` | `boolean` | — | `false` | **Experimental.** Compute the EDFL/ISR sufficiency gate for evidence-grounded binary questions. Only applies when an evidence list is available through the agent-flow layer. Requires 3–12× more computation. |

### `generation_config` Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `max_new_tokens` | `integer` | `160` | Maximum tokens to generate. |
| `temperature` | `float` | `0.7` | Sampling temperature. |
| `top_p` | `float` | `0.9` | Nucleus sampling probability. |
| `do_sample` | `boolean` | `true` | Whether to use sampling. When `false`, generation is greedy (deterministic). |

---

### Response Body (`EvaluateResponse`)

```json
{
  "session_id": "sess_abc123",
  "call_id": "call_def456",
  "timestamp": "2026-06-10T14:23:00.123Z",
  "prompt_blocked": false,
  "response": "The capital of France is Paris.",
  "block_reason": null,
  "context_truncated": false,
  "safety": { ... },
  "hallucination": { ... },
  "model_metadata": { ... },
  "generation_diagnostics": { ... },
  "xai": null,
  "reasoning_trace": { ... }
}
```

| Field | Description |
|---|---|
| `session_id` | Session identifier (provided or auto-generated) |
| `call_id` | Unique identifier for this specific call |
| `timestamp` | ISO 8601 timestamp |
| `prompt_blocked` | `true` if the prompt was blocked before generation |
| `response` | The model's generated answer. `null` if `prompt_blocked` is `true`. |
| `block_reason` | Human-readable reason for blocking, or `null` |
| `context_truncated` | `true` if the provided context was truncated to fit the model's context window |
| `safety` | Safety detection results. See below. |
| `hallucination` | Hallucination detection results. See below. |
| `model_metadata` | Checkpoint information |
| `generation_diagnostics` | Timing and token count diagnostics |
| `xai` | Explainability results when `explain=true`. Contains `mupax_halluc`, `mupax_safety`, `mupax_halluc_causal` depending on `explain_mode` and `credit_tiers`. |
| `reasoning_trace` | Internal scoring trace for debugging |

### `safety` Object

| Field | Description |
|---|---|
| `prompt_unsafe_logit` | Raw logit score for prompt unsafety (uncalibrated) |
| `prompt_unsafe_score` | Probability [0, 1] for prompt unsafety |
| `answer_unsafe_logit` | Raw logit score for answer unsafety (uncalibrated) |
| `answer_unsafe_score` | Probability [0, 1] for answer unsafety |
| `decision` | `"safe"`, `"unsafe"`, or `"blocked"` |
| `decision_rule` | Which rule triggered the decision |
| `threshold_used` | The detection threshold that was applied |
| `combined_answer_safety_score` | Calibrated combined safety score (0–1). This is the production score used for decisions. |
| `combined_answer_safety_threshold` | The threshold for the combined score |
| `combined_answer_safety_triggered` | Whether the combined threshold was exceeded |
| `combined_answer_safety_per_signal` | Per-signal breakdown. Each signal shows `raw_logit`, `zscore`, `weight`, `contribution`. |

### `hallucination` Object

| Field | Description |
|---|---|
| `hallucination_score` | Primary hallucination probability [0, 1] |
| `decision` | `"grounded"` or `"hallucinated"` |
| `threshold_used` | The threshold applied |
| `context_provided` | Whether context was provided (affects which axes run) |
| `combined_halluc_score` | Calibrated combined hallucination score (0–1). This is the production score. |
| `combined_halluc_threshold` | The threshold for the combined score |
| `combined_halluc_triggered` | Whether the combined threshold was exceeded |
| `combined_halluc_per_signal` | Per-signal breakdown with contributions from all 10 internal signals. Each: `raw_logit`, `zscore`, `weight`, `contribution`. |
| `combined_halluc_n_signals` | Number of signals included in the combined score |
| `nsp_commission_score` | Narrative Semantic Probe — commission error score |
| `nsp_coverage_score` | NSP — coverage score |
| `nsp_assertiveness_score` | NSP — assertiveness score |
| `drift_score` | Contextual drift score across generation layers |
| `closed_book_fabrication_score` | Closed-book fabrication probability (when context is absent) |
| `closed_book_fabrication_reason` | Human-readable reason for the closed-book score |
| `edfl_isr` | EDFL/ISR sufficiency gate result (when `enable_edfl_isr=true`). Fields: `isr`, `delta_bar`, `decision` (`answer`/`abstain`/`disabled`). |

---

## GET /glad/finetune/status/{job_id}

Returns the status of an active or completed fine-tuning job.

```bash
curl http://localhost:8199/glad/finetune/status/job_abc123
```

```json
{
  "job_id": "job_abc123",
  "status": "running",
  "progress": 0.34,
  "current_step": 340,
  "total_steps": 1000,
  "last_loss": 0.0423
}
```

| Field | Description |
|---|---|
| `status` | `"queued"`, `"running"`, `"completed"`, or `"failed"` |
| `progress` | Fraction completed (0–1), present only while `"running"` |
| `current_step` | Current training step |
| `total_steps` | Total steps in the job |
| `last_loss` | Most recent training loss |
| `output_path` | Path to the output checkpoint (when `"completed"`) |
| `error` | Error message (when `"failed"`) |

---

## POST /glad/export_audit

Exports an audit bundle for one or more inference sessions.

### Request Body (`ExportAuditRequest`)

| Field | Type | Required | Description |
|---|---|---|---|
| `session_id` | `string` | ⚠️ One of these two | Export all calls from this session. |
| `call_ids` | `array[string]` | ⚠️ One of these two | Export a specific list of call IDs. |
| `client_info` | `object` | — | Deployer information for the report cover page: `company_name`, `system_name`, `deployment_date`, `responsible_person`, `contact_email`. |
| `regulatory_framework` | `array[string]` | — | List of regulatory frameworks to include in the audit report. Defaults to `["EU_AI_ACT"]`. See [Supported Laws](../regulatory/index.md) for valid codes. |
| `include_raw_scores` | `boolean` | — | Whether to include raw numeric scores in the export. Default `false`. |
| `include_compliance_bundle` | `boolean` | — | Whether to include the full compliance artifact bundle (hash chain, watermarks, FRIA reference). Default `true`. |
| `compliance_context` | `object` | — | Additional metadata to include in compliance reports. |
| `output_path` | `string` | — | Absolute file path to write the export. If omitted, the file is written to a temp directory and returned as a file download. |
