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
**closed-book available** next to the connection test. Without them the axis reports `available: false`
("upstream does not expose per-token logprobs") — it does not silently pretend. Two routes still measure it
without logprobs; see [Without generator logprobs](#without-generator-logprobs).

---

## Detector generations

The detector is built in **blocks** of features. Which blocks you can compose depends on what the
upstream gives you.

| Detector | Blocks | Features | What the extra blocks add |
|---|---|---|---|
| **classic (fork)** | `fork` | 55 | *Semantic forking* — which words the model was weighing among the alternatives it already returned |
| **SLEDGE-logprobs** | `fork,nli` | 64 | + an NLI judge on the alternatives. Everything that can be computed from the **answer's** logprobs alone |
| **SLEDGE-Next** | `fork,prem,nli` | 71 | + premise surprise + an NLI judge on the alternatives |
| **SLEDGE-Next v2** | `fork,prem2,nli` | 71 | same, with the **v2** premise block |

!!! note "Versions"
    SLEDGE-logprobs ships since 0.4.3 (up to 0.4.2 an upstream without `prompt_logprobs` got **classic
    (fork)**). Since 1.0.3.1 the premise blocks also work on Ollama, by forcing (below). Since 1.0.8 the
    default is **SLEDGE-Next**; SLEDGE-Next v2 stays selectable.

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

**Premise by forcing (Ollama).** Ollama does not return `prompt_logprobs`, but it can score a given text:
the gateway reads the question token by token through the raw generate endpoint with `top_logprobs`
(teacher forcing — one short call per question token). At start-up it probes whether the upstream supports
this, and records it next to the capability (`premise_fallback: "teacher-forced"`). When it works, the premise
blocks are real on Ollama too.

**G-1 picks the default from what it detects:**

* `prompt_logprobs` **available**, or the premise readable **by forcing** → **SLEDGE-Next** (`fork,prem,nli`).
  SLEDGE-Next v2 (`fork,prem2,nli`) is offered next to it in the calibration panel;
* **neither** → **SLEDGE-logprobs**: the `fork` and `nli` blocks, fitted **without** the premise columns. This
  is the default for OpenAI and most hosted APIs.

When the probe finds no `prompt_logprobs`, this default wins even over the detector currently served.
The reason is that a SLEDGE-Next artifact served against such an upstream runs with its premise columns
at zero. Those columns then contribute nothing, so nothing crashes and the threshold still holds its
false-positive budget. But every other weight was fitted next to a live premise block, so the ranking is
worse than a detector fitted on the blocks it can actually see. Until the probe has run, the served
detector decides.

!!! tip "Forcing a generation"
    `GW_SLEDGE_BLOCKS` overrides the choice, for example `GW_SLEDGE_BLOCKS=fork,prem2,nli` for SLEDGE-Next v2
    or `GW_SLEDGE_BLOCKS=fork,nli` for SLEDGE-logprobs. `GW_SLEDGE_UI_BLOCKS` changes only what the calibration
    panel proposes. If your upstream is a hosted API with neither capability, serving the same model with
    **vLLM** (or Ollama) makes the premise blocks available — the gateway detects it at start-up, nothing to
    configure.

---

## Running a calibration

Studio → the application → **SLEDGE closed-book calibration**.

The model is queried with **reasoning switched off** (`reasoning_effort: "none"` and
`chat_template_kwargs.enable_thinking: false`; a provider that rejects one of the two parameters is retried
without it). A thinking model otherwise spends its tokens reasoning and the calibration corpus fills with
truncated answers. `CBSG_THINKING=on` restores the old behaviour.

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

**Browsing it.** The calibration panel lists what the catalog holds **for the detector head this deployment
serves** — model, generation, blocks, mode, measured false-positive rate, publication date — marks the entry
that matches the served model and the ones already installed, and installs an entry with one click, without
calibrating. The same is available on the API:

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/sledge/catalog` | `{enabled, url, head_sha, served_model, entries[{model, generation, blocks, mode, published_at, measured_fpr, n_faithful, matches_served_model, installed}]}` |
| `POST` | `/v1/glad/sledge/catalog/install` | Body `{model}`. Downloads and verifies the artifact, registers it, and reports whether it is now active for the served model. `404` if the model is not published for this head, `409` if the catalog is disabled, `502` if the signature, head or download fails. |

Installed artifacts are written next to the served registry (on G1-Proxy Light, the data volume), so they
survive an image update.

On start-up the gateway looks the model up and downloads only if something is there — and refuses an
artifact whose signature does not verify, or that was calibrated against a **different detector head**.
A `.pkl` is executable code; it never reaches disk on a bad signature.

A catalog artifact replaces a local one only when its generation ranks higher, and the rank depends on
the upstream. When `prompt_logprobs` is available the order is classic < SLEDGE-logprobs < SLEDGE-Next <
SLEDGE-Next v2. When it is not, the two SLEDGE-Next generations drop below SLEDGE-logprobs, so a
SLEDGE-Next v2 in the catalog never replaces a SLEDGE-logprobs artifact on such an upstream. A generation
the gateway does not recognise is never downloaded and never overwritten.

---

## Without generator logprobs

Many hosted APIs return no logprobs at all. Two routes still put a number on `halluc_closedbook`:

**Reference check** (chat API, MCP, G1 Eval). With `verify_references: true`, the links, DOIs, arXiv and PubMed
ids and citations in the answer are verified on open sources. A reference that does not exist, points to another
work or is not pertinent makes the axis available and flags it (`details.method: "reference_check"`). See
[Reference check](../reference/response-format.md#reference-check).

**A second answer** ([G1 Eval](eval.md)). Send `answer_2` — a second, independent answer of the same model to the
same question — next to `answer`, with no context. The axis is then measured from two readings:

- **consistency between the two answers**: a multilingual NLI model checks, clause by clause, whether each answer
  entails the other, where they contradict each other, and which figures and names appear in one only. A model
  that knows the fact says the same thing twice; a model that invents usually does not;
- **borrowed logprobs**: the grounding model already loaded for `halluc_context` re-reads the question and the
  answer and supplies the per-token probabilities the upstream did not, on which the same SLEDGE feature
  functions are computed.

A logistic head over both readings, with a conformal threshold, gives the score; `details.clauses` lists the
clauses that contradict the other answer most. It ships with one calibration (named in
`details.calibrated_on`) and needs no per-model SLEDGE artifact. The NLI model loads on the first request that
carries `answer_2`.

---

## Related

* [Detection Axes](detection-axes.md) · [Grounding](grounding.md) · [Thinking Levels](thinking-levels.md)
* [Upstream Backends](backends.md) — which servers return what
