# Threshold Tuning Guide

A threshold turns a probability into a decision, which makes it the single most consequential number in
the deployment — and the one most often set by intuition. This page says where each threshold lives, what
the calibrated defaults are and what they were measured against, and how to move one without guessing.

---

## Where thresholds live

Resolved from highest to lowest precedence:

1. **Per-request `threshold_overrides`** in the API request body
2. **The Application's policy** — `policy.thresholds`, set from Studio, from [Policy Lens](../studio/policy-lens.md), or via `PUT /v1/glad/apps/{id}/policy`
3. **The serving calibration** shipped with the detector checkpoint

The per-Application policy is the one you normally use, because it is versioned, audited, and hot-reloaded:

```bash
curl -s -X PUT http://localhost:8080/v1/glad/apps/support_bot/policy \
  -H "Content-Type: application/json" \
  -d '{"policy": {"thresholds": {"prompt_safety": 0.88, "out_of_scope": 0.85}}}'
```

Send only the axes you are changing — the update is a merge and the response echoes the resulting policy with a new `config_version`.

---

## The calibrated defaults

These are the serving-calibrated values for the 9-axis head. They are a **starting point produced by a specific calibration on a specific checkpoint**, not universal constants.

| Axis | Default | Enforcement | How it was set |
|---|---|---|---|
| `prompt_safety` | `0.9215` | `block` | Joint 2 % false-positive budget with `jailbreak`, on a **multilingual** benign pool |
| `jailbreak` | `0.9997` | `block` | Same joint budget; the axis is extremely confident on real attacks, so the operating point sits far out on the tail |
| `rag_jailbreak` | `0.2501` | `block` | Aggressive by design — benign retrieved context almost never contains imperatives aimed at the model |
| `halluc_context` | `0.6475` | `annotate` | Dev split |
| `halluc_closedbook` | `0.58` | `annotate` | Advisory — the SLEDGE conformal calibration actually decides this axis at serving time |
| `answer_safety` | `0.7295` | `annotate` | Dev split |
| `profanity` | `0.90` | `annotate` | Conservative: only fires when the detector is very sure |
| `out_of_scope` | `0.90` | `annotate` | Conservative: refusing a customer is expensive |
| `prompt_complexity` | `0.50` | `off` | The **training boundary** of a binary classifier — not a false-positive budget |

!!! danger "Two things these numbers depend on"
    **The checkpoint.** Thresholds do not transfer across detector builds. A new build means re-running the calibration; an Application created earlier keeps the thresholds stored in its own policy.

    **The language mix.** The safety pair is calibrated on a multilingual benign pool for a reason: a threshold calibrated on an English-only pool produced a **13 % false-positive rate on Italian traffic**. Multilingual attack detection is largely a *calibration* problem, not a capability problem — the ranking (AUROC 0.85–0.99) can be excellent while recall sits at 16–64 % simply because the scores are shifted. If your traffic is dominated by a language your calibration pool does not represent, re-calibrate; do not nudge the number.

---

## How to set a threshold properly

The only defensible method is to measure on **your** traffic. In order of preference:

### 1. Policy Lens (recommended)

[Policy Lens](../studio/policy-lens.md) re-decides every stored request under a candidate threshold, exactly — the threshold is applied after scoring, so this is a recomputation over scores already in your call log, not a simulation.

1. Run the axis in `annotate` for a few days so real traffic accumulates.
2. Open Policy Lens, pick the axis, drag the slider.
3. Read **would unblock** / **would newly block** — and read the actual messages that move.
4. Check **reviewer-confirmed good moves**: how many of the moves your own reviewers already flagged as correct.
5. Apply. It hot-reloads on the next request and bumps `config_version`.

### 2. Offline, from the score distribution

If you prefer to compute it yourself, pull the traffic and work on the numbers:

```bash
curl -s "http://localhost:8080/v1/glad/apps/support_bot/messages?limit=2000" \
  | jq '[.items[] | select(.axes.prompt_safety != null) | .axes.prompt_safety] | sort'
```

Set the threshold at the percentile of **benign** scores matching the false-positive rate you can afford (95th percentile ⇒ ~5 % FPR), then validate against known-bad examples from your own domain. A benchmark FPR is not your FPR.

### 3. Passthrough observation

Set enforcement to `annotate` (or run the request with `mode: "passthrough"`), let 1 000+ calls accumulate, export from the dashboard, and repeat step 2. Nothing is blocked while you measure.

---

## The detection decision

For each axis, the decision is:

```
score >= threshold  →  flagged
score <  threshold  →  pass
```

Scores are always in the range [0, 1]: **0** = the detector is confident the content is safe/grounded/in-scope; **1** = confident it is not.

A flag is not automatically a block. What a flag *does* depends on the axis's **enforcement mode** — `block`, `annotate`, or `off` — configured per Application. See [Detection Axes](../g1-proxy/detection-axes.md#guardrails-vs-operational-axes).

---

## Choosing by what the error costs you

Thresholds are a business decision dressed as a number. The useful question is never "what is the right value" but "which mistake can I afford".

| Axis | A false positive costs you | A false negative costs you | Lean |
|---|---|---|---|
| `prompt_safety` / `jailbreak` | An over-refused customer; support load; churn | An unsafe answer, and an incident to explain | Strict for public/consumer surfaces, relaxed for vetted internal users |
| `rag_jailbreak` | A retrieved document rejected | An agent obeying an injected instruction | Strict — the base rate of imperatives in benign context is very low |
| `halluc_context` | A correct answer annotated as unsupported | A confidently wrong answer shipped to a customer | Strict in legal / medical / finance |
| `halluc_closedbook` | An "uncertain but correct" answer flagged | A fabricated citation, name or statistic | Strict for fact-seeking products, relaxed for creative ones |
| `answer_safety` | A benign answer withheld | Harmful content delivered | Strict on public surfaces |
| `profanity` | A frustrated but legitimate customer moderated | Offensive language passing into your logs and your brand | Relaxed for support (people swear), strict for public forums |
| `out_of_scope` | **A customer you should have served is refused** | You pay for an answer outside your remit | Conservative (`0.90`+), and only promote it to `block` once Policy Lens shows the refused list contains nothing you wanted |
| `prompt_complexity` | You pay premium rates for an easy question | A hard question gets a weak answer | Start at `0.50` and watch the Model-A/Model-B split in [Cost & FinOps](../studio/cost.md) |

---

## Starting points by deployment type

Use these as *first* values, then tune with Policy Lens. Only the axes you would actually move are listed; everything else keeps its calibrated default.

### Customer support (RAG, public)

```json
{ "thresholds": { "halluc_context": 0.55, "answer_safety": 0.65,
                  "profanity": 0.95, "out_of_scope": 0.85 },
  "enforcement": { "out_of_scope": "block", "answer_safety": "block" } }
```

Grounding matters more than tone (customers swear; that is not a safety event), and off-topic traffic is worth refusing outright — it is the cheapest token you will ever save.

### Internal knowledge base (vetted users)

```json
{ "thresholds": { "prompt_safety": 0.96, "jailbreak": 0.9999,
                  "halluc_context": 0.50 },
  "enforcement": { "profanity": "off" } }
```

Relaxed safety (employees are identified), stricter hallucination — the risk here is a confident wrong answer being trusted, not an attacker.

### Legal / medical / finance (high-stakes)

```json
{ "thresholds": { "prompt_safety": 0.85, "halluc_context": 0.40,
                  "halluc_closedbook": 0.45, "answer_safety": 0.60 },
  "enforcement": { "halluc_context": "block" } }
```

Aggressive everywhere: in high-stakes domains, flagging and escalating to [human oversight](../compliance/oversight.md) beats letting a borderline answer through.

### Supervisory / audit mode (no blocking)

Set every axis to `annotate` (or send `mode: "passthrough"`). Every response is scored and logged, nothing is withheld. This is the right first week of any deployment.

---

## The three-zone model

Think of each threshold as dividing scores into three zones:

```
0.0 ──────────────────[threshold − 10%]──[threshold]──────── 1.0
        Safe zone            Borderline        Flagged zone
     (pass silently)       (queue for review)  (block or annotate)
```

The **borderline zone** is where detection is least certain and where human review is worth the money. Configure [human oversight](../compliance/oversight.md) to queue borderline calls without changing the block threshold:

```json
"human_oversight": { "auto_trigger": true, "safety_threshold": 0.70, "halluc_threshold": 0.75 }
```

Reviewed borderline calls become corrections, corrections feed the [self-evolving loop](../g1-proxy/self-evolving.md), and the loop is what lets you move the threshold next month on evidence instead of instinct.

---

## See also

- [Detection Axes](../g1-proxy/detection-axes.md) — what each axis scores and how enforcement is grouped
- [Policy Lens](../studio/policy-lens.md) — the counterfactual simulator these thresholds deserve
- [Token & Cost Control](../g1-proxy/cost-control.md) — the two axes where the threshold is a spending decision
- [Self-Evolving Security](../g1-proxy/self-evolving.md) — correcting individual cases instead of moving the line
