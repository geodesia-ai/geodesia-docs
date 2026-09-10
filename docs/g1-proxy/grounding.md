# Grounding — one number for "is this answer supported?"

Two axes answer the same question from opposite sides. `halluc_context` asks whether the answer is
supported by the **context it was given**; `halluc_closedbook` asks whether the model is **inventing
from memory** when there is no context at all. Reading them separately means deciding, per response,
which one applies.

The **`grounding`** block does that for you. Every response carries it:

```json
"grounding": {
  "score": 0.83,
  "verdict": "grounded",
  "axis": "halluc_context",
  "advisory_score": 0.61,
  "available": true
}
```

---

## The scale

`score` is in **[0, 1]**, and the middle is not arbitrary:

| value | meaning |
|---|---|
| **1.0** | perfectly anchored |
| **0.5** | exactly on the calibrated threshold |
| **0.0** | entirely unsupported |

It is the **margin against that axis' own threshold**, rescaled — so 0.5 always means "on the line",
whatever the axis and whatever the calibration. Comparing raw probabilities across axes does not work;
comparing margins does.

## Which axis decides

`axis` names the one that was **in its regime**:

* grounding context present → `halluc_context`;
* no context, but logprobs available → `halluc_closedbook`.

The other is reported as **`advisory_score`**, for information. It did not decide.

---

## When it cannot be measured

```json
"grounding": { "score": null, "verdict": null, "available": false }
```

!!! danger "`null`, never `0`"
    With neither grounding context nor logprobs, there is nothing to measure. The block says so with
    `null` and `available: false`.

    It is deliberately **not** `0`. A zero would be read as "completely unsupported" — a strong claim —
    when the truth is "not measured". Silence dressed as a score is worse than no score, and *not
    measured* is never the same as *clean*.

---

## Reading it in practice

* **Gate on the score** if you want one number: `< 0.5` means the deciding axis crossed its threshold.
* **Never treat `null` as a pass.** Branch on `available` first.
* `verdict` follows the flags actually served, so it stays consistent with enforcement.

---

## Related

* [Detection Axes](detection-axes.md) — the nine axes
* [Closed-Book Calibration](closed-book-calibration.md) — why the closed-book side is per-model
* [Response Format](../reference/response-format.md) · [Thresholds](../reference/thresholds.md)
