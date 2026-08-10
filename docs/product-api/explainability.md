# Explainability API

Two interfaces, and for almost every integration only one of them matters. The **black-box endpoint** on
G1-Proxy attributes a verdict to the tokens that caused it without touching model internals — that is the
production path. The **inline `explain` flag** on the Studio-local evaluate endpoint needs a checkpoint
loaded in-process and belongs to research work.

---

## Call it

**What it does.** Send a prompt, an answer and optionally the grounding context; get back the tokens that
caused the detector's verdict, with a certificate of how strongly. Deterministic — same input, same build,
same answer.

=== "curl"

    ```bash
    curl -s -X POST http://localhost:8080/gw/v1/glad/causal-explainability/analyze \
      -H "Content-Type: application/json" \
      -d '{
        "prompt":   "According to the document, when was the Eiffel Tower built?",
        "context":  "The Eiffel Tower was constructed between 1887 and 1889.",
        "response": "The Eiffel Tower was built in 1885.",
        "method":   "dca",
        "axis":     "halluc_context"
      }' | jq '{
        axis: .detection_type,
        base: .base_score,
        mode: .xai.gradient_causal.attribution_mode,
        necessary: .xai.gradient_causal.necessary_tokens
      }'
    ```

=== "Python"

    ```python
    import httpx

    r = httpx.post(
        "http://localhost:8080/gw/v1/glad/causal-explainability/analyze",
        json={
            "prompt":   "According to the document, when was the Eiffel Tower built?",
            "context":  "The Eiffel Tower was constructed between 1887 and 1889.",
            "response": "The Eiffel Tower was built in 1885.",
            "method":   "dca",
            "axis":     "halluc_context",
        },
        timeout=360,
    )
    r.raise_for_status()
    dca = r.json()["xai"]["gradient_causal"]
    print(dca["attribution_mode"], dca["base_score"])
    for tok in dca["top_tokens"]:
        print(f"  {tok['token']!r:12s} {tok['status']:10s} effect={tok['effect']:.2f}")
    ```

=== "TypeScript"

    ```ts
    const res = await fetch(
      "http://localhost:8080/gw/v1/glad/causal-explainability/analyze",
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          prompt: "According to the document, when was the Eiffel Tower built?",
          context: "The Eiffel Tower was constructed between 1887 and 1889.",
          response: "The Eiffel Tower was built in 1885.",
          method: "dca",
          axis: "halluc_context",
        }),
      },
    )
    if (!res.ok) throw new Error(await res.text())
    const dca = (await res.json()).xai.gradient_causal
    console.log(dca.attribution_mode, dca.necessary_tokens)
    ```

### What comes back

```json
{
  "prompt": "According to the document, when was the Eiffel Tower built?",
  "full_response": "The Eiffel Tower was built in 1885.",
  "detection_type": "halluc_context",
  "base_score": 0.71,
  "xai": {
    "method": "dca",
    "gradient_causal": {
      "attribution_mode": "certified",
      "certificate_basis": "necessity",
      "base_score": 0.71,
      "sufficiency_bar": 0.639,
      "n_forward": 16,
      "top_tokens": [
        { "token": "1885", "position": 6, "region": "answer", "status": "necessary",
          "effect": 0.62, "sufficiency": 0.55, "responsibility": 1.0 }
      ],
      "necessary_tokens": ["1885"]
    }
  }
}
```

*"1885"* is the certified necessary token: remove it alone and the faithfulness score drops below the flag.

### Methods

| `method` | What it computes | Latency |
|---|---|---|
| `dca` *(default)* | Deterministic attribution on the dominant flagged axis. | ~1–3 s |
| `dca_dual` | Prompt **and** answer surfaces attributed separately. The only method that accepts an empty `response` — a prompt blocked before generation has no answer. | ~1–3 s |
| `gradient_causal` / `occlusion` | Leave-one-out occlusion. | seconds |
| `mupax_causal` | Monte-Carlo coalition estimation. Accepts `mupax_n_samples`. | minutes |

Full field-by-field reference, attribution modes and the certificate semantics:
**[Causal Explainability](../g1-proxy/causal-xai.md)**.

---

## Inline explain — Studio-local, research only

`POST /glad/evaluate` on **G-1 Studio's own port** accepts `explain: true` and returns attribution in the
response's `xai` field. It requires a research checkpoint loaded in-process, and in the packaged product
the Studio backend runs without one — so this is not the path to build on. Use the black-box endpoint above.

### Parameters

| Parameter | Description |
|---|---|
| `explain` | `true` to compute attribution. |
| `explain_mode` | `"standard"` (default) or `"causal"`. |
| `credit_tiers` | Which attribution methods to run — see below. |
| `system_prompt_text` | When given, its tokens are excluded from attribution. |

### Credit tiers

| Tier | Key | Speed | Description |
|---|---|---|---|
| 1 | `"gradient"` | ~50 ms | Deterministic prompt-token occlusion: mask one token at a time, the score change is the importance. |
| 1.5 | `"pss"` | ~N× generation | Positional Semantic Stability — see below. |
| 2 | `"mupax"` | ~0.4–2 s | Coalition estimation: random coalitions scored with the detector, fitted with one joint linear surrogate whose coefficients are the per-unit attribution. Accounts for interactions. Seeded, so it reproduces exactly. |
| 3 | `"learned"` | ~10 ms | Learned attribution head, when the checkpoint has one. Fastest; accuracy depends on training coverage. |

```json
{
  "model_path": "/app/pretrained_glad",
  "prompt": "When was the Eiffel Tower built?",
  "explain": true,
  "credit_tiers": ["mupax", "gradient"]
}
```

### `explain_mode: "causal"`

Additionally computes a **token→token causal matrix**: for the answer token with the highest attribution,
which prompt tokens are causally responsible for it. Not *what words mattered*, but *which prompt words
caused the model to write the most suspicious part of the answer*.

### Response structure

```json
"xai": {
  "mupax_halluc": {
    "detection_type": "hallucination",
    "top_tokens": [
      { "token": "Paris", "position": 7, "importance": 0.48,
        "retention_frequency": 0.71, "conditional_goodness": 0.88 }
    ],
    "threshold_W": 0.14,
    "threshold_percentile_used": 0.2,
    "n_accepted": 412,
    "n_total": 500,
    "attribution_heatmap": [0.02, 0.01, 0.48, 0.12],
    "score_function": "combined_logreg"
  },
  "mupax_halluc_causal": {
    "target_token": "1889",
    "target_position": 14,
    "causal_edges": [
      { "source_position": 4, "source_token": "built",
        "target_position": 14, "target_token": "1889",
        "raw_importance": 0.61, "normalized_importance": 0.83, "absolute_importance": 0.83 }
    ]
  }
}
```

| Per-token field | Description |
|---|---|
| `token` | Token text as decoded from the vocabulary. |
| `position` | Position in the full input sequence. |
| `importance` | Attribution value. Higher = contributed more to the detection score. |
| `retention_frequency` | Share of Monte-Carlo samples where this token appeared in above-threshold configurations. |
| `conditional_goodness` | Mean detection score when this token was present. |

| Causal-edge field | Description |
|---|---|
| `source_position` / `source_token` | The prompt token. |
| `target_position` / `target_token` | The answer token. |
| `raw_importance` | Signed importance; positive = causal contribution. |
| `normalized_importance` | Signed, normalised by the largest absolute value in the graph. |
| `absolute_importance` | Absolute normalised importance, `[0,1]`. |

---

## PSS — Positional Semantic Stability

A training-free method that asks a different question: *if I change the prompt here, does the key claim in
the answer change?* It needs no gradients and no weights — it generates N alternative answers and measures
which claims survive.

| Parameter | Env override | Default | Description |
|---|---|---|---|
| `pss_n_samples` | `GLAD_PSS_N_SAMPLES` | `5` | Extra samples to generate. Each costs one generation pass. 2 = fast/noisy, 16 = slow/robust. |
| `pss_temperature` | `GLAD_PSS_TEMPERATURE` | `0.7` | Must be > 0 — at 0 every sample would be identical. |
| `pss_match_mode` | `GLAD_PSS_MATCH_MODE` | `"ngram"` | How claims are compared: `ngram` (containment + entity match), `strict` (exact surface), `fuzzy` (Levenshtein), `entity`, `claim` (sentence-level bidirectional). |

**Reach for it when** you are explaining hallucination in long-form answers where occlusion is noisy, you
need attribution with no access to weights at all, or you are building a human review workflow and the
explanation has to be relatable — *"this claim changed when we removed that context sentence"*.
