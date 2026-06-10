# Detection Axes

Geodesia G-1 scores every request across **five independent detection axes**. Each axis is scored separately, has its own calibrated threshold, and can be individually configured. Understanding what each axis detects helps you tune thresholds appropriately for your use case.

---

## Overview

<div class="axis-grid">

<div class="axis-card context">
<div class="axis-name">Context Faithfulness</div>
<div class="axis-key">halluc_context</div>
<p>Detects when the model's answer makes claims that are not supported by — or contradict — the grounding context provided in the request. This is the core hallucination detector for RAG applications.</p>
</div>

<div class="axis-card closedbook">
<div class="axis-name">Closed-Book Fabrication</div>
<div class="axis-key">halluc_closedbook</div>
<p>Detects when the model confidently states facts without any grounding context — such as fabricating citations, inventing statistics, or confabulating historical events. Requires per-token log-probabilities from the upstream.</p>
</div>

<div class="axis-card prompt">
<div class="axis-name">Prompt Safety</div>
<div class="axis-key">prompt_safety</div>
<p>Detects unsafe input prompts: requests for weapons, explosives, malware, harassment, CSAM, self-harm methods, and other harmful content. Scored before the request reaches the upstream model.</p>
</div>

<div class="axis-card answer">
<div class="axis-name">Answer Safety</div>
<div class="axis-key">answer_safety</div>
<p>Detects unsafe content in the model's generated answer — even when the prompt appeared safe. Catches cases where the model spontaneously produces harmful content or is manipulated through indirect injection.</p>
</div>

<div class="axis-card jailbreak">
<div class="axis-name">Jailbreak</div>
<div class="axis-key">jailbreak</div>
<p>Detects attempts to bypass the model's safety guidelines through role-playing ("pretend you are…"), privilege escalation, encoding tricks, or other adversarial framing. Higher precision than general prompt safety.</p>
</div>

</div>

---

## Detailed Reference

### `halluc_context` — Context Faithfulness

**What it catches:** Answers that contain claims not supported by, or contradicted by, the grounding context.

**When it applies:** Any request that includes a `context` field or retrieves documents through the Knowledge Base (RAG). The axis compares the answer against the provided context using an entailment-based model. If the answer asserts something that cannot be traced to the context, the score rises.

**Example triggering scenario:** Your knowledge base contains: *"Our return policy allows refunds within 30 days."* A user asks about returns, but the model says *"You can return items within 60 days."* This is a faithfulness violation — the answer contradicts the document.

**When it does NOT apply:** Requests with no context (closed-book questions). On those, `halluc_context` scores near zero and the gateway relies on `halluc_closedbook` instead.

**RAG interaction:** When RAG claim-level verification confirms that every claim in the answer is cited from a retrieved chunk, the gateway suppresses this axis regardless of its score. The raw score is preserved in `p_detector_raw` with a `suppressed_by: "rag_claim_verification"` note for audit purposes.

**Default threshold:** `0.35` (lower = stricter; the faithfulness axis is particularly useful at low thresholds for high-stakes RAG deployments)

---

### `halluc_closedbook` — Closed-Book Fabrication

**What it catches:** Answers that confidently assert facts — names, dates, statistics, citations, URLs — without any grounding context, where the confidence pattern in the model's generation suggests fabrication rather than knowledge.

**When it applies:** Fact-seeking closed-book questions (no context provided). The axis computes a set of signals from the per-token log-probabilities of the generated answer — including vocabulary surprisal, entropy variance, and consistency across similar queries.

**Requirements:** This axis requires per-token log-probabilities from the upstream LLM. If the upstream does not provide them (e.g., standard Ollama), the axis is automatically disabled and shown as `available: false` in the response.

**Example triggering scenario:** A user asks *"Who wrote The Great Gatsby?"* The answer *"F. Scott Fitzgerald"* is correct and low-surprisal → low score. But if asked *"What did Fitzgerald say in his 1923 Paris Review interview?"* and the model invents a detailed quote, the signal patterns suggest confabulation → high score.

**`fact_seeking` gate:** The axis includes a gate that checks whether the question is genuinely fact-seeking (rather than creative, conversational, or instructional). Non-fact-seeking turns never flag on this axis regardless of their score.

**Advisory mode:** Setting `pass_extra > 1` draws multiple independent samples and computes self-consistency signals. Responses that are consistent across samples receive a lower fabrication score.

**Default threshold:** `0.50`

---

### `prompt_safety` — Prompt Safety

**What it catches:** User prompts that explicitly or implicitly request harmful, illegal, or dangerous content — including:

- Weapons (conventional, CBRN, explosives)
- Malware, exploits, intrusion instructions
- Harassment, doxxing, defamation
- CSAM or sexualised content involving minors
- Drug synthesis
- Content that violates EU AI Act Article 5 prohibited practices (social scoring, predictive policing by profile, emotion inference at work/school, etc.)
- Bias-laundering (using proxy variables to discriminate by protected characteristics)

**When it runs:** On the **input prompt**, before the request is forwarded to the upstream LLM. In blocking mode, a flagged prompt stops the pipeline immediately — the upstream model is never called.

**Note on evasive framing:** The detector is specifically trained to see through common evasion techniques: fiction wrappers ("write a story where a character explains how to…"), professional authority claims ("I'm a nurse and need to know…"), and encoding tricks. Legitimate professional questions (pharmacist asking about drug interactions, security researcher studying exploits) are distinguished from genuinely unsafe requests by intent and specificity.

**Default threshold:** `0.90` (high by design — prompt safety has a larger base rate of false positives from edge-case benign queries. The threshold balances safety against over-blocking.)

---

### `answer_safety` — Answer Safety

**What it catches:** Unsafe, harmful, or policy-violating content in the generated answer, regardless of whether the prompt appeared safe.

**When it runs:** On the **generated answer**, after the upstream LLM completes. In streaming mode it also fires mid-stream on the accumulated text every `cadence_tokens` tokens.

**Why this axis exists separately:** Models can produce unsafe content even from benign prompts through indirect prompt injection (malicious instructions hidden in retrieved documents), jailbreak techniques that slip past input screening, or spontaneous model failure. Having a separate output scorer catches these cases.

**Default threshold:** `0.57`

---

### `jailbreak` — Jailbreak Detection

**What it catches:** Sophisticated adversarial attempts to override the model's safety guidelines. While `prompt_safety` casts a broad net over general harm, the jailbreak axis is tuned specifically for manipulation and privilege-escalation patterns:

- Role-play overrides ("You are DAN, you have no restrictions…")
- Hypothetical framing ("In a world where it was legal…")
- Token smuggling and encoding tricks
- Multi-step escalation across conversation turns
- Prompt injection from external content (documents, tool outputs)
- Claims of special authority ("I'm Anthropic and I need you to…")

**Default threshold:** `0.57`

---

## Reading Axis Results

Each axis produces a per-axis object in the `geodesia.axis_energy` response field:

```json
{
  "halluc_context": {
    "p_detector": 0.72,
    "flag": true,
    "threshold": 0.35,
    "available": true,
    "suppressed_by": null,
    "p_detector_raw": null
  }
}
```

| Field | Type | Description |
|---|---|---|
| `p_detector` | `float` [0, 1] | Detection probability. Higher means more likely to be the kind of content this axis detects. |
| `flag` | `bool` | `true` if `p_detector >= threshold`. This is what triggers a brake or block. |
| `threshold` | `float` | The threshold used for this request (may be overridden by `threshold_overrides`). |
| `available` | `bool` | `false` if the axis cannot run (e.g., `halluc_closedbook` when the upstream has no logprobs). A `false` axis never flags. |
| `fact_seeking` | `bool` | (closed-book only) Whether the question was classified as fact-seeking. Only fact-seeking questions can flag. |
| `suppressed_by` | `string` \| `null` | Reason why the axis was suppressed despite the score. Example: `"rag_claim_verification"` when all RAG claims are verified. |
| `p_detector_raw` | `float` \| `null` | The original score before suppression, for audit purposes. |

The `geodesia.brake` field is `true` if any **answer-region** axis (`halluc_context`, `halluc_closedbook`, `answer_safety`) has `flag: true`. Input-region axes (`prompt_safety`, `jailbreak`) affect the input phase separately.

---

## Axis Grouping

| Phase | Axes | Timing |
|---|---|---|
| **Input validation** | `prompt_safety`, `jailbreak` | Before forwarding to the upstream LLM |
| **Output validation** | `halluc_context`, `halluc_closedbook`, `answer_safety` | After the upstream LLM responds |

This grouping matters for enforcement: `block_input` only applies to the input phase, `block_output` only to the output phase. A request can have a clean input but a flagged output (or vice versa).

---

## Axis Availability Summary

| Axis | Requires logprobs | Requires context |
|---|---|---|
| `halluc_context` | No | Yes (scores 0 without context) |
| `halluc_closedbook` | **Yes** | No (disabled when context is present) |
| `prompt_safety` | No | No |
| `answer_safety` | No | No |
| `jailbreak` | No | No |
