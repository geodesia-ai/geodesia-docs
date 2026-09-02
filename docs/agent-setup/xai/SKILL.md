---
name: geodesia-g1-xai
description: |
  Read and render Geodesia G-1's causal explainability output — the χ (chi) attribution values returned by
  glad.explain and /v1/glad/causal-explainability/analyze. Use when a G-1 axis flagged something and you
  need to say WHY at the token level, when comparing the dca / occlusion / mupax methods, when a user asks
  which words drove a block, or when rendering attribution as a heatmap over the original text. Covers what
  χ, importance, effect, sufficiency, responsibility and the necessary/relevant/irrelevant partition each
  mean, which of them are comparable to each other, the two failure modes that make token attribution lie,
  and a copy-pasteable diverging heatmap renderer. Never present an explanation as a decision.
---

# G-1 Causal Explainability (χ)

`glad.explain` answers a different question from `glad.analyze`. Analyze says *whether* and *how much*.
Explain says **which units of the input carried the score**, and it is a report — never a verdict.

---

## The one rule that outranks the rest

**An explanation reports; it does not decide.** The panel shows attribution for whatever axis you asked
about, including axes sitting at 2% that flagged nothing. Rendering that as "BLOCKED" manufactures a
verdict out of the floor. Read the verdict from `verdict` / `flagged_axes`; use χ only to explain a
decision that was already taken.

The second rule follows from a measured failure: **never show a token without its surrounding text.**
Bare token lists invite rationalising around tokenizer boundaries — in the example below MuPAX returns
`"previ"`, a fragment of *previous*. Shown alone, a reader invents a meaning for it. Shown in place, it
is obviously half a word. Render attribution **over the original string**, always.

---

## The two output shapes

### Single axis (`method: "dca" | "occlusion" | "mupax"`)

```jsonc
{
  "method": "mupax",
  "axis": "jailbreak",
  "verdict": { "jailbreak": { "p_detector": 0.9333, "threshold": 0.3259, "flag": true } },
  "flagged_axes": ["jailbreak", "out_of_scope"],
  "top_tokens": [
    { "token": "ignore", "position": 0, "importance": 0.128,  "effect": 0.128,  "chi": 0.128 },
    { "token": "all",    "position": 1, "importance": 0.0775, "effect": 0.0775, "chi": 0.0775 },
    { "token": "previ",  "position": 2, "importance": 0.016,  "effect": 0.016,  "chi": 0.016 }
  ],
  "attribution": { "score_function": "companion_p[jailbreak]", "base_score": 0.9333, "…": "…" },
  "causal_edges": [],
  "summary": "axis 'jailbreak' driven mainly by: 'ignore', 'all', …"
}
```

### Every flagged axis at once (`all_flagged_axes: true`)

```jsonc
{
  "method": "dca_multi_axis",
  "flagged_axes": ["jailbreak", "out_of_scope"],
  "by_axis": {
    "jailbreak": {
      "method": "dca_joint", "score_function": "companion_p[jailbreak]",
      "base_score": 0.9333, "deterministic": true, "n_forward": 23, "rho": 0.9,
      "interaction_order_used": 1,
      "top_tokens": [ /* … */ ],
      "necessary_tokens": ["all"],
      "relevant_tokens": ["Ignore", "previous", "instructions.", "obey"],
      "irrelevant_positions": [4, 5, 8, 9, 11],
      "responsibility": { "1": 1.0 }
    },
    "out_of_scope": { /* … */ }
  },
  "summary": "jailbreak: 'all'; out_of_scope: 'instructions.', 'all', 'ignore'"
}
```

**Ask for `all_flagged_axes: true` whenever more than one axis fired.** A response can be flagged on two
axes for entirely different tokens — above, `jailbreak` rests on *all* while `out_of_scope` rests on
*instructions.* Explaining only the dominant axis hides the second reason for the block.

---

## What each number means

### `chi` (χ) — MuPAX only

χ is the **regression coefficient** of a unit in a rank-contrast design: G-1 scores many masked variants
of the input, then fits a linear model whose coefficients are the per-unit attribution. Properties that
follow, and that you may rely on:

* **Signed.** Positive χ pushed the score *toward* detection; negative pushed it *away*. A negative χ is
  not "unimportant" — it is exculpatory, and a heatmap must show it as a distinct direction, not as a
  faded version of positive.
* **Additive and in score units.** χ values are commensurable with each other and roughly sum toward
  `base_score`. You may rank them, sum them, and compare two tokens' magnitudes directly.
* **Robust to interactions**, because the design varies many units at once rather than one at a time.
* In the MuPAX response `importance == effect == chi` — the same number under three names, kept for
  backward compatibility. Read `chi`.

Cost: `n_samples` forward passes. It is the expensive method.

### `importance` / `sufficiency` — DCA

DCA (deterministic causal attribution) reports something else, and **its numbers are not χ**:

* **`sufficiency`** ∈ [0, 1] — across the masked runs, the fraction in which keeping this unit was enough
  to hold the score above threshold. `importance` mirrors it.
* **`effect`** — the marginal change in score, which **can be negative even for a top token**. In the DCA
  example, `previous` has `sufficiency 0.8901` and `effect −0.0416`: highly sufficient, slightly
  score-lowering on its own. That is not a contradiction — sufficiency is about *holding the decision*,
  effect is about *moving the number*. Do not average them and do not plot them on the same scale.
* **`deterministic: true`** — same input, same explanation, every time. That is why DCA is the default:
  an explanation that changes between runs cannot be filed as evidence.

### The partition — the part most worth showing a human

DCA sorts every unit into three buckets, and this is usually a better answer than a ranked list:

| Field | Meaning | How to present it |
|---|---|---|
| `necessary_tokens` | remove it and the flag **goes away** | the actual cause — lead with this |
| `relevant_tokens` | contributes, but the flag survives without it | supporting evidence |
| `irrelevant_positions` | positions that changed nothing | say "not implicated" — never render as low risk |

`responsibility` maps a unit index to its Chockler–Halpern responsibility: `1.0` = solely responsible,
`0.5` = one of two sufficient causes, `0.333` = one of three. A low responsibility means the cause is
**distributed**, not that it is weak.

### Diagnostics worth surfacing

`base_score` (the score being explained — check it matches the verdict you are showing), `n_forward` (how
many passes it cost), `rho`, and `interaction_order_used` (`1` = single units; higher = the explanation
needed combinations, which is itself a finding about the attack).

---

## Choosing a method

| Method | Determinism | Cost | Gives you | Use when |
|---|---|---|---|---|
| `dca` (default) | deterministic | ~20–40 passes | sufficiency + the necessary/relevant partition + responsibility | an audit trail, a certificate, anything a human will file |
| `occlusion` | deterministic | one pass per unit | a fast marginal delta | interactive UI, long inputs, a first look |
| `mupax` | sampled | `n_samples` passes | signed, additive χ | you need magnitudes that add up, or interactions matter |

Default to `dca`. Reach for `mupax` when the question is *how much did each word contribute*, and for
`occlusion` when the question is *roughly where do I look* and latency matters.

---

## Rendering: a heatmap over the original text

Attribution is a **magnitude spread over positions in a string**, so the form is a heatmap on the text
itself, not a bar chart of tokens. Two rules decide the colour, and they differ per method:

* **MuPAX χ is signed → diverging.** Two poles with a **neutral grey midpoint at exactly zero**: warm for
  "pushed toward the flag", cool for "pushed away". Never a single ramp — a single ramp renders an
  exculpatory token as a weak incriminating one.
* **DCA sufficiency is unsigned [0, 1] → sequential.** One hue, light → dark, lightest at zero.

Colour never carries the meaning alone: every tinted span also exposes its numeric value (`title`
attribute below), and the necessary/relevant partition is stated in words.

```html
<style>
.xai { color-scheme: light; --surf:#fcfcfb; --ink:#0b0b0b; --mute:#52514e;
  /* diverging: cool (exculpatory) — neutral — warm (incriminating). Equal steps per arm. */
  --c4:#2a78d6; --c3:#5598e7; --c2:#9ec5f4; --c1:#cde2fb;
  --n0:#f0efec;
  --w1:#fbd9d8; --w2:#f5b0af; --w3:#ee7c7b; --w4:#e34948;
  background:var(--surf); color:var(--ink);
  font:15px/2.1 ui-monospace,SFMono-Regular,Menlo,monospace; padding:1rem; border-radius:8px }
@media (prefers-color-scheme:dark){ :root:where(:not([data-theme="light"])) .xai{
  color-scheme:dark; --surf:#1a1a19; --ink:#fff; --mute:#c3c2b7;
  --c4:#3987e5; --c3:#2a78d6; --c2:#1c5cab; --c1:#104281;
  --n0:#383835;
  --w1:#7a2a2a; --w2:#a83a3a; --w3:#cc5252; --w4:#e66767; } }
:root[data-theme="dark"] .xai{ color-scheme:dark; --surf:#1a1a19; --ink:#fff; --mute:#c3c2b7;
  --c4:#3987e5; --c3:#2a78d6; --c2:#1c5cab; --c1:#104281; --n0:#383835;
  --w1:#7a2a2a; --w2:#a83a3a; --w3:#cc5252; --w4:#e66767; }
.xai mark{ background:var(--n0); color:inherit; padding:.15em .1em; border-radius:3px; cursor:help }
.xai mark.nec{ outline:2px solid currentColor; outline-offset:1px }   /* necessary: not colour-alone */
.xai .legend{ font:12px/1.6 system-ui,sans-serif; color:var(--mute); margin-top:.75rem }
.xai .sw{ display:inline-block; width:14px; height:10px; border-radius:2px; vertical-align:-1px }
</style>

<div class="xai" id="xai"></div>
<script>
// step(): χ → one of nine slots. The midpoint is EXACTLY zero, so an exculpatory token can never
// render as a weak incriminating one. `scale` is max|χ| over the response, so the ramp is relative
// to this explanation and a uniformly weak one does not look uniformly damning.
function step(chi, scale){
  if (!scale) return 'n0';
  const t = Math.max(-1, Math.min(1, chi / scale));
  if (Math.abs(t) < 0.06) return 'n0';
  const q = Math.min(4, Math.ceil(Math.abs(t) * 4));
  return (t > 0 ? 'w' : 'c') + q;
}
function renderXAI(text, tokens, necessary){
  const scale = Math.max(...tokens.map(t => Math.abs(t.chi ?? t.effect ?? 0)), 0);
  const nec = new Set(necessary || []);
  let html = '', cursor = 0;
  // Anchor each token by SEARCHING the original string: never re-join tokens, or "previ" becomes a word.
  for (const t of tokens){
    const i = text.indexOf(t.token, cursor);
    if (i < 0) continue;                                   // fragment we cannot place → skip, don't guess
    const v = t.chi ?? t.effect ?? 0;
    html += esc(text.slice(cursor, i));
    html += `<mark class="${nec.has(t.token) ? 'nec' : ''}" style="background:var(--${step(v, scale)})"`
         +  ` title="${esc(t.token)} · χ ${v >= 0 ? '+' : ''}${v.toFixed(4)}` 
         +  `${nec.has(t.token) ? ' · NECESSARY: removing it clears the flag' : ''}">`
         +  `${esc(text.slice(i, i + t.token.length))}</mark>`;
    cursor = i + t.token.length;
  }
  html += esc(text.slice(cursor));
  document.getElementById('xai').innerHTML = html +
    `<div class="legend"><span class="sw" style="background:var(--c4)"></span> pushed away from the flag`
    + ` &nbsp; <span class="sw" style="background:var(--n0)"></span> no effect`
    + ` &nbsp; <span class="sw" style="background:var(--w4)"></span> pushed toward it`
    + ` &nbsp; · outlined = necessary · hover any span for its χ</div>`;
}
function esc(s){ return s.replace(/[&<>"]/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c])); }
</script>
```

Call it with the original string — not a re-join of the tokens:

```js
renderXAI(originalPrompt, res.top_tokens, res.necessary_tokens);
```

### In a terminal

When there is no browser, keep the same two rules — position in the real string, and value shown as a
number — and drop the colour rather than faking it:

```python
def heat(text, tokens, necessary=()):
    """Underline the original text with χ magnitude. Colour is optional; the number is not."""
    scale = max((abs(t.get("chi", t.get("effect", 0))) for t in tokens), default=0) or 1
    bar, cur, out = [" "] * len(text), 0, []
    for t in tokens:
        i = text.find(t["token"], cur)
        if i < 0:
            continue                      # a fragment we cannot place: skip it, never guess a span
        v = t.get("chi", t.get("effect", 0))
        mark = "▁▂▃▄▅▆▇█"[min(7, int(abs(v) / scale * 7))]
        for j in range(i, i + len(t["token"])):
            bar[j] = mark
        out.append((t["token"], v, t["token"] in necessary))
        cur = i + len(t["token"])
    print(text)
    print("".join(bar))
    for tok, v, nec in sorted(out, key=lambda r: -abs(r[1]))[:8]:
        print(f"  {v:+.4f}  {tok!r}{'   ← necessary' if nec else ''}")
```

---

## Two ways this output lies, and how to not be fooled

**1. Tokenizer fragments.** `"previ"` is not a word the user wrote. Anchor every span by searching the
original string (as both renderers above do) and **drop fragments you cannot place** rather than
inventing a boundary. A fragment presented as a token got 4 of 10 readers to a wrong conclusion; the same
attribution shown in place got 0 of 10.

**2. A floor rendered as a finding.** Attribution exists for axes that never flagged. If
`verdict[axis].flag` is false, say so above the heatmap — "this axis did not fire; the shading shows
where its score came from anyway" — or do not draw it. And an axis with `available: false` was **not
measured**: it has no explanation at all, and a blank heatmap for it must be labelled "not measured",
never rendered as clean.

---

## Where to call it

* **MCP** — `glad.explain` with `prompt` / `context` / `generated`, `method`, optional `axis`,
  `all_flagged_axes`. Put the flagged content in the region it came from: `generated` for an answer,
  `prompt` or `context` for a tool description or a fetched resource.
* **HTTP** — `POST /v1/glad/causal-explainability/analyze` on the gateway.

Explaining is not free — `dca` costs tens of forward passes. Call it **on a flag**, not on every request.

Full documentation: <https://geodesia-ai.github.io/geodesia-docs/>
