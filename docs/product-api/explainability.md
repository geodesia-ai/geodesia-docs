# Explainability API

Geodesia G-1 provides two explainability interfaces: the **inline explain** flag on the evaluate endpoint (for LLM-internal attribution), and the **causal explainability endpoint** on the gateway (for black-box attribution). This page documents both.

---

## Inline Explain (Evaluate Endpoint)

When you call `POST /glad/evaluate` with `explain: true`, attribution scores are computed alongside the regular detection scores and returned in the `xai` field of the response.

### Parameters

| Parameter | Description |
|---|---|
| `explain` | Set to `true` to enable XAI computation |
| `explain_mode` | `"standard"` (default) or `"causal"` |
| `credit_tiers` | Which attribution methods to run (see below) |
| `system_prompt_text` | If provided, tokens belonging to the system prompt are excluded from attribution |

### Credit Tiers

The `credit_tiers` array specifies which attribution methods to run. Methods can be combined:

| Tier | Key | Speed | Description |
|---|---|---|---|
| Tier 1 | `"gradient"` | Fast (~50ms) | Deterministic prompt-token occlusion: each prompt token is masked one at a time and the change in detection score is the importance. Deterministic and reproducible. |
| Tier 1.5 | `"pss"`, `"tier1_5"`, `"stability"` | Slow (~N× generation) | Positional Semantic Stability: generates N alternative answers and measures how much each prompt token affects whether specific output claims appear. Training-free. Controlled by `pss_n_samples`, `pss_temperature`, `pss_match_mode`. |
| Tier 2 | `"mupax"` | Medium (~0.4–2s) | Monte Carlo Perturbation Attribution: statistically robust attribution via kernel SHAP. The production default. |
| Tier 3 | `"learned"` | Fast (~10ms) | Learned attribution head (if the checkpoint includes one). Fastest, but accuracy depends on training data coverage. |

**Example — run MuPAX and gradient together:**
```json
{
  "model_path": "/app/pretrained_glad",
  "prompt": "When was the Eiffel Tower built?",
  "explain": true,
  "credit_tiers": ["mupax", "gradient"]
}
```

### `explain_mode: "causal"`

In causal mode, the system additionally computes a **token→token causal matrix**: for the answer token with the highest attribution score, it identifies which prompt tokens are causally responsible for its generation.

This answers: *"Not just what words were important — but specifically which prompt words caused the model to write the most suspicious part of the answer."*

### XAI Response Structure

```json
"xai": {
  "mupax_halluc": {
    "detection_type": "hallucination",
    "top_tokens": [
      {
        "token": "Paris",
        "position": 7,
        "importance": 0.48,
        "retention_frequency": 0.71,
        "conditional_goodness": 0.88
      }
    ],
    "threshold_W": 0.14,
    "threshold_percentile_used": 0.2,
    "n_accepted": 412,
    "n_total": 500,
    "attribution_heatmap": [0.02, 0.01, 0.48, 0.12, ...],
    "score_function": "combined_logreg"
  },
  "mupax_safety": {
    "detection_type": "safety",
    ...
  },
  "mupax_halluc_causal": {
    "detection_type": "hallucination_causal",
    "target_token": "1889",
    "target_position": 14,
    "prompt_tokens": [...],
    "answer_tokens": [...],
    "causal_edges": [
      {
        "source_position": 4,
        "source_token": "built",
        "target_position": 14,
        "target_token": "1889",
        "raw_importance": 0.61,
        "normalized_importance": 0.83,
        "absolute_importance": 0.83
      }
    ]
  }
}
```

### Per-Token Attribution Fields

| Field | Description |
|---|---|
| `token` | The token text as decoded from the vocabulary |
| `position` | Token position in the full input sequence |
| `importance` | χ attribution value. Higher means this token contributed more to the detection score. |
| `retention_frequency` | Proportion of Monte Carlo samples in which this token appeared in configurations with above-threshold scores |
| `conditional_goodness` | Mean detection score when this token was present |

### Causal Edge Fields

| Field | Description |
|---|---|
| `source_position` | Position of the prompt token |
| `source_token` | The prompt token text |
| `target_position` | Position of the answer token |
| `target_token` | The answer token text |
| `raw_importance` | Signed χ importance (positive = causal contribution) |
| `normalized_importance` | Signed χ normalized by the maximum absolute χ in the graph |
| `absolute_importance` | Absolute normalized importance [0, 1] |

---

## Causal XAI via Gateway

For the companion gateway deployment (where Geodesia runs against an external LLM without access to model internals), black-box attribution is available at:

<div class="endpoint"><span class="method method-post">POST</span><span class="path">/v1/glad/causal-explainability/analyze</span></div>

See [Causal XAI](../gateway/causal-xai.md) for full documentation.

---

## PSS: Positional Semantic Stability

Positional Semantic Stability (PSS, Tier 1.5) is a training-free attribution method that asks: *"If I change the prompt in this position, does the key claim in the answer change?"*

Unlike gradient-based methods, PSS does not need access to model gradients. It works by generating N alternative answers with small prompt variations and measuring stability.

### PSS Configuration Parameters

| Parameter | Env override | Default | Description |
|---|---|---|---|
| `pss_n_samples` | `GLAD_PSS_N_SAMPLES` | `5` | Number of additional samples to generate. Each sample costs one generation pass. 2 = fast/noisy; 16 = slow/robust. |
| `pss_temperature` | `GLAD_PSS_TEMPERATURE` | `0.7` | Sampling temperature for PSS resamples. Must be > 0 (temperature 0 would make all samples identical). |
| `pss_match_mode` | `GLAD_PSS_MATCH_MODE` | `"ngram"` | How to compare claims across samples. Options: `"ngram"` (n-gram containment + entity match), `"strict"` (exact surface), `"fuzzy"` (Levenshtein), `"entity"` (named entities only), `"claim"` (sentence-level bidirectional). |

### When to use PSS

- You are explaining hallucination in long-form answers where gradient methods are noisy
- You need attribution that works without any model weights (fully black-box)
- You are building a human review workflow and need the explanation to be relatable ("this claim changed when we removed that specific context sentence")
