# Threshold Tuning Guide

Geodesia G-1 uses **probability thresholds** to decide when a detection score is high enough to trigger a block or flag. Setting thresholds correctly is one of the most important operational decisions you will make — too low and you block legitimate content; too high and harmful content gets through.

This guide explains what each threshold controls, how to reason about the trade-off, and recommended starting points by deployment type.

---

## The Detection Decision

For each axis, the decision is:

```
score >= threshold  →  triggered (flag or block)
score <  threshold  →  pass
```

Scores are always in the range [0, 1]:

- **0** = the model is confident this content is safe/grounded
- **1** = the model is confident this content is unsafe/hallucinated

Thresholds define where you draw the line between pass and flag.

---

## Threshold Locations

Thresholds live in three places, resolved from highest to lowest precedence:

1. **Per-request `threshold_overrides`** in the API request body
2. **Database threshold preferences** set via `POST /v1/glad/threshold-prefs`
3. **Default values** compiled into the model

To set persistent thresholds for your deployment:

```bash
curl -X POST http://localhost:8199/v1/glad/threshold-prefs \
  -H "Content-Type: application/json" \
  -d '{
    "deployer_id": "acme-corp",
    "prompt_safety": 0.35,
    "answer_safety": 0.50,
    "halluc": 0.617,
    "combined_halluc": 0.617
  }'
```

---

## Per-Axis Reference

### `prompt_safety` — Input Safety

Controls when a **user prompt** is considered unsafe and blocked before generation.

| Threshold | Behavior |
|---|---|
| `0.10–0.20` | Aggressive — catches most unsafe requests but will flag ambiguous phrasing ("how do I shoot a video?") |
| `0.30–0.40` | **Recommended starting point** — good balance for general-purpose chatbots |
| `0.50–0.70` | Conservative — only blocks clear violations |
| `0.80+` | Near-permissive — use only if you have a separate safety layer upstream |

**Calibrated default:** `0.35`

**Tip:** If you see frequent false positives on technical prompts (SQL queries, security research, medical questions), raise this threshold incrementally. Use the [human oversight](../compliance/oversight.md) queue to accumulate real-world false-positive examples before tuning.

---

### `answer_safety` — Output Safety (combined)

Controls when a **generated answer** is considered unsafe and withheld. The `combined_score` from the multi-signal combiner is compared against this threshold.

| Threshold | Behavior |
|---|---|
| `0.10–0.15` | Very aggressive — catches subtle harmful content but has high false positives on corporate/grounded queries |
| `0.20–0.40` | **Recommended** for customer-facing deployments |
| `0.50–0.70` | Moderate — suitable when you also have human review active |
| `0.80+` | Near-permissive — use only in research/internal contexts |

**Calibrated default:** `0.11` (Youden optimal)

!!! warning "Known issue with default threshold"
    The default threshold of 0.11 is calibrated for maximum balanced accuracy but sits in the middle of the benign score distribution. Deployers using the system with corporate knowledge bases or RAG pipelines will see false positives. **Recommended production value: 0.50** until you have measured your specific benign score distribution.

---

### `halluc` (combined) — Hallucination Detection

Controls when an answer is considered a hallucination. This applies to both:
- **Context faithfulness** (`halluc_context`): is the answer supported by the provided context?
- **Closed-book fabrication** (`halluc_closedbook`): is the answer a confident fabrication?

The `combined_halluc_score` from the 10-signal calibrated combiner is compared against this threshold.

| Threshold | Behavior |
|---|---|
| `0.40–0.50` | Aggressive — catches subtle hallucinations but flags uncertain but correct answers |
| `0.55–0.65` | **Recommended** for RAG pipelines and fact-checking use cases |
| `0.617` | **Default** — optimal on HaluEval benchmark (AUROC 0.97) |
| `0.70–0.80` | Conservative — only blocks confident fabrications |
| `0.85+` | Near-permissive |

**Calibrated default:** `0.617`

---

### `jailbreak` — Jailbreak Detection

Controls when a prompt is classified as a jailbreak attempt (attempting to bypass safety constraints or manipulate the model's behavior).

| Threshold | Behavior |
|---|---|
| `0.30–0.40` | Aggressive — blocks some creative/roleplay prompts that look like jailbreaks |
| `0.50` | **Default** — good balance for most deployments |
| `0.60–0.70` | Conservative — only blocks explicit jailbreak patterns |

**Calibrated default:** `0.50`

---

## Recommended Configurations by Deployment Type

### Customer Support Chatbot (RAG)

```json
{
  "prompt_safety": 0.40,
  "answer_safety": 0.50,
  "halluc": 0.60,
  "combined_halluc": 0.60
}
```

Rationale: moderate input filtering with a slightly higher answer safety threshold to reduce false positives on corporate/grounded content.

---

### Public-Facing Chat (General Purpose)

```json
{
  "prompt_safety": 0.30,
  "answer_safety": 0.25,
  "halluc": 0.617,
  "combined_halluc": 0.617
}
```

Rationale: stricter safety across both prompt and answer, default hallucination threshold.

---

### Internal Knowledge Base / Enterprise

```json
{
  "prompt_safety": 0.60,
  "answer_safety": 0.70,
  "halluc": 0.55,
  "combined_halluc": 0.55
}
```

Rationale: relaxed safety thresholds (employees are vetted users), stricter hallucination to protect factual accuracy.

---

### Legal / Medical / Finance (High-Stakes)

```json
{
  "prompt_safety": 0.25,
  "answer_safety": 0.20,
  "halluc": 0.45,
  "combined_halluc": 0.45
}
```

Rationale: aggressive on all axes — in high-stakes domains it is better to flag and escalate to human review than to allow a borderline response through.

---

### Supervisory / Audit Mode (No Blocking)

Set enforcement to `"passthrough"` and keep default thresholds. The system annotates every response with scores without blocking anything.

```json
{
  "mode": "passthrough"
}
```

---

## The Three-Zone Model

Think of each axis threshold as dividing scores into three zones:

```
0.0 ────────────────────[threshold - 10%]──[threshold]──────── 1.0
         Safe zone              Borderline        Triggered zone
      (pass silently)          (flag for review)  (block or annotate)
```

The **borderline zone** (within 10% of the threshold in either direction) is where the detection is least certain. Configure [human oversight](../compliance/oversight.md) to queue borderline calls for review:

```yaml
human_oversight:
  oversight_threshold: 0.55   # 0.617 threshold - ~10%
```

This captures all borderline calls for human review without changing the block threshold.

---

## Measuring Your Threshold

The best way to calibrate thresholds for your specific deployment:

1. Run the system in **passthrough mode** for at least 1,000 calls
2. Export the call log from the dashboard
3. Review the score distribution for each axis
4. Set your threshold at the 95th percentile of benign scores (leaving 5% FPR as acceptable)
5. Validate with known-harmful examples from your domain

This approach gives you a threshold tuned to your traffic, not a generic benchmark.
