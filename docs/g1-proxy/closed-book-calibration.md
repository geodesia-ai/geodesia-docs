# Closed-Book Calibration (SLEDGE)

The `halluc_closedbook` axis catches a model **fabricating from memory** — no context to check against,
nothing to retrieve, just a confident invention. It reads the model's own **token logprobs**, which
means it is measured on *how the model behaved while writing*, not on the words it produced.

That is also why it cannot be shipped pre-calibrated. Two models writing the same sentence have
different surprise profiles, so **each served LLM needs its own SLEDGE artifact**. Applications bound to
the same LLM share one automatically.

---

## The one prerequisite

The upstream must return **per-token logprobs** on the OpenAI-compatible endpoint. Studio shows this as
**closed-book available** next to the connection test. Without it the axis runs text-head-only and says
so — it does not silently pretend.

---

## Detector generations

The detector is built in **blocks** of features. Which blocks you can compose depends on what the
upstream gives you.

| Detector | Blocks | Features | What the extra blocks add |
|---|---|---|---|
| **classic (fork)** | `fork` | 55 | *Semantic forking* — which words the model was weighing among the alternatives it already returned |
| **SLEDGE-Next** | `fork,prem,nli` | 71 | + premise surprise + an NLI judge on the alternatives |
| **SLEDGE-Next v2** | `fork,prem2,nli` | 71 | same, with the **v2** premise block |

**The `fork` block** reads information the gateway already pays for and nobody used: the top-20
alternatives come back as (string, logprob) pairs, and the strings were being thrown away. Reading them
is worth a large margin across generators, at no extra generation cost.

**The `nli` block** runs a real NLI judge on the alternatives *embedded in the sentence*, rather than
comparing bare strings.

**The premise blocks** look at the **question**, not the answer. They catch the failure every
answer-side signal misses by construction: the model accepts an entity that does not exist and builds a
coherent world around it. There is no retrieval, so there is no surprise to detect downstream.

!!! note "Why v2 replaced v1"
    The v1 premise block turned out to key on the **length of the question**. A real paper wrapped in a
    person-phrasing got flagged; an invented one without the wrapper passed. v2 isolates the span of the
    cited entity and looks *inside* it — a memorised entity completes itself, an invented one stays
    plausible-but-surprising at every token.

---

## `prompt_logprobs` — the capability that picks the default

The premise blocks need the logprobs of the **prompt** tokens, not the generated ones. These are two
different API features, and confusing them is the easiest mistake in this panel:

| | what it covers | who returns it |
|---|---|---|
| `logprobs` / `top_logprobs` | the **generated** tokens | part of the OpenAI API — vLLM, Ollama, OpenAI |
| `prompt_logprobs` | the **question**'s tokens | a **vLLM extension** — not in the OpenAI API |

So an upstream can show **closed-book available** and still have no `prompt_logprobs`. There is no
contradiction: the first badge is about the answer, the second capability is about the question. A
server that does not know the field simply ignores it — no error, just nothing in the response.

**G-1 picks the default from what it detects:**

* `prompt_logprobs` **available** → **SLEDGE-Next v2**, because the premise block can be computed;
* **not available** → **classic (fork)**, because those 7 features would sit at zero. A detector with
  dead blocks inside is not more capable, only wider.

!!! tip "Serving the same model on vLLM is the whole fix"
    If you need v2 and your upstream is Ollama or OpenAI, serve the model with **vLLM**. The gateway
    detects `prompt_logprobs` at start-up and moves the default to SLEDGE-Next v2 on its own — nothing
    to configure. `GW_SLEDGE_BLOCKS` overrides the choice if you want to force it.

---

## Running a calibration

Studio → the application → **SLEDGE closed-book calibration**.

**Fast** is the mode you want. It reuses a baked base detector and refits only the conformal threshold,
in minutes. **Deep** retrains the detector itself and needs roughly four times the generated corpus.

The `× corpus` fraction scales the **targets** (faithful answers per length bucket), not the pool: the
run stops when the buckets are full, not when the pool is exhausted.

!!! info "The first calibration of a new model may be *degraded*, and says so"
    Fast needs a baked base built for the same blocks. If none is shipped for the chosen blocks, the run
    does **not** fail: it fits the detector on the corpus it just generated and labels the artifact
    `fast-degraded`. That model is weaker — a few hundred rows against the thousands behind a full base
    — and it is declared, not hidden. The artifact it writes then becomes the base, so the **second**
    calibration of the same model is a full Fast run.

**Stop** cancels a run at any point; the panel returns to editable immediately.

---

## Per-language thresholds

A single threshold is an English threshold. Measured across languages, one global cut produced false
positive rates from 1.1% to 27%. Ticking **per-language thresholds** calibrates a cut per language and
keeps each inside budget.

Languages that cannot reach the minimum sample size are **excluded and declared** rather than given a
threshold nobody could trust.

---

## The catalog

A calibrated head is offered to a **public, signed catalog**, so the next deployment of the same model
against the same detector head downloads it instead of paying for its own generations.

On start-up the gateway looks the model up and downloads only if something is there — and refuses an
artifact whose signature does not verify, or that was calibrated against a **different detector head**.
A `.pkl` is executable code; it never reaches disk on a bad signature.

---

## Related

* [Detection Axes](detection-axes.md) · [Grounding](grounding.md) · [Thinking Levels](thinking-levels.md)
* [Upstream Backends](backends.md) — which servers return what
