# Hallucination API — grounded, sufficiency and closed-book (2.0)

Three endpoints, served by the **G1V** model on vLLM ([the vLLM stack](vllm-stack.md)). They judge whether an answer is
**supported by the documents it was given** — not whether it is true in the world.

| Endpoint | Question it answers | You send |
|---|---|---|
| [`POST /g1/hallucination-context`](#hallucination-context) | Is the answer supported by the context? Which sentences and words are not? | question, context, answer |
| [`POST /g1/sufficiency`](#sufficiency) | Is the context **enough** to answer? If not, what should I search for? | question, context, answer |
| [`POST /g1/hallucination-closedbook`](#closed-book) | Do independent answers to the same question disagree, with no documents at all? | 2–8 answers |

Authentication is the same as the [Safety API](safety-api.md#authentication).

!!! info "Also in the UI"
    Studio and the light UI have a **Hallucination** page with one tab per endpoint (Grounded, Sufficiency, Closed-book).
    See [Studio UI reference](../studio/ui-reference.md#hallucination).

## What this is and is not

These endpoints return a **signal** about one thing: whether the claims in an answer are supported by the passages you
sent. They are meant as one component of a gate on RAG answers, next to your own checks.

- They are **not a source of ground truth**. A supported answer can still be wrong if the documents are wrong.
- They perform **no external fact-checking**: nothing outside the request is read, and no web or knowledge base is queried.
- They do not retrieve documents. `sufficiency` proposes search queries; your own retriever runs them.
- Every verdict depends on a threshold. Choose and validate it on your own data before you block traffic ([How to validate on your data](#how-to-validate-on-your-data)).

## Health and readiness

`GET /g1/hallucination/health` → `200 {"status":"ok"}` when the G1V servers answer.

| Status | Meaning | What to do |
|---|---|---|
| `200` | ready | route traffic |
| `503` (`degraded`) | the route exists but a model server does not answer | not ready: wait, do not send checks, do not treat a missing check as a pass |
| `404` | the **feature is not enabled in this build** — it does **not** mean the service is unhealthy | this build cannot do the check: do not simulate the result |

Recommended readiness probe: call the route until it returns `200` before sending traffic; keep probing afterwards, and
stop routing on `503`. The first start can take 10–40 minutes (see [the vLLM stack](vllm-stack.md#requirements)).

## Languages

| | Languages |
|---|---|
| **Supported** | English, Italian, German, French, Spanish |
| **Not supported** | Polish, Dutch, Portuguese: no training data and no evaluation; behaviour is undefined, **do not rely on it** |
| **Best effort, no guarantees** | Russian, Chinese: not part of the supported set |

**Per-language accuracy is not yet published.** A per-language breakdown will be added when it has been measured.

Sentence splitting and the claim rules behind the `sentences` list are tuned for English. For the other four supported
languages sentence splitting is a **separate path** and its error profile is **not yet published**; those requests carry
the warning `claim_rules_english`. Where a sentence cannot be isolated, rely on the verdict of the whole answer.

## Shared rules (grounded and sufficiency)

- `context` is a string or an array of strings (each passage separately). At least one non-empty passage; up to 256 passages and 200,000 characters.
- **Input limit: 16,384 tokens** (instructions + question + context + answer). When it does not fit, the end of the context is cut — see the **warning below**.
- `profile`: `balanced` (default), `low_false_positives` (fewer false alarms, fewer detections) or `high_recall` (the opposite).
- `threshold` (optional, in [0, 1]) applies to the **calibrated probability**. Setting it changes the operating point; the response adds the warning `custom_threshold`.
- The decision threshold of `hallucination-context` depends on the context length: contexts of up to 7,000 characters (`context_group: "short"`, 0.5195 in the balanced profile) and longer ones (`"long"`, 0.2735).
- Errors: `400 missing_answer`, `400 empty_context`, `400 invalid_context`, `400 invalid_profile`, `400 invalid_threshold`, `413 too_many_passages`, `413 context_too_long`, `413 answer_too_long`, `413 input_too_long`, `401 unauthorized`, `503 model_unavailable`.

### Truncation

!!! warning "A truncated request has no reliable verdict"
    The checker reads **at most 16,384 input tokens** and cuts the context **from the end**. When that happens,
    `limits.context_truncated` is `true`, `limits.context_tokens_read` says how much was read, and the response carries
    the warning `context_truncated`.

    **A verdict on a truncated request is unreliable**: the evidence for a claim may sit in the part that was cut, so a
    **supported answer can be marked unsupported**. A clean-looking verdict (`hallucinated: false`, `sufficient: true`)
    on a truncated request must not be read as a pass.

    Recommended handling:

    1. Read `limits.context_truncated` on **every** response.
    2. If it is `true`, treat the request as **not measurable** and **fail closed**: block, hold for review, or retry with a shorter context.
    3. Shorten or select the passages before the call (retrieve fewer, rank and keep the best) so that the request fits.
    4. Never extract a pass verdict from a truncated request.

How the limits work together:

| Limit | What happens | Where you see it |
|---|---|---|
| 16,384 tokens (what the model reads) | the request is accepted, the **end of the context is cut** | `limits.context_truncated: true`, `limits.context_tokens_read`, warning `context_truncated` |
| 200,000 characters of context, 256 passages | the request is **rejected before reading** | `413 context_too_long`, `413 too_many_passages` |

The **token limit is the one that binds first** for ordinary text: 16,384 tokens correspond to far fewer than 200,000
characters, so a long context is cut by the token limit and the character limit is only a request-size guard. The exact
number of characters that fit depends on the tokenizer and the language and is not fixed.

On the calibration set of the sufficiency verdict (see below) **no context was truncated** (median full-context length
about 7,400 tokens). Real RAG contexts can be longer than that set, so do not assume the same behaviour in production.

---

## `hallucination-context`

| Field | Type | Required | Meaning |
|---|---|---|---|
| `prompt` | string | no (recommended) | the user question; `messages` is also accepted (last user turn) |
| `context` | string \| string[] | yes | the passages the answer must rely on |
| `answer` | string | yes | the answer to verify |
| `token_level` | boolean | no, `false` | also return the unsupported **words** |
| `profile`, `threshold` | | no | see above |

| Response field | Meaning |
|---|---|
| `hallucinated` | **the decision** for the whole answer |
| `probability` | raw probability that the answer contains an unsupported or contradicted claim; `calibrated_probability` is its calibrated form |
| `threshold`, `threshold_applies_to`, `profile`, `context_group` | what it was compared with |
| `sentences` | the **flagged** sentences: `{id, text, start, end, probability, flagged, type}`. `type` is **`unsupported`** (the context does not say it) or **`contradicted`** (the context says the opposite). Offsets are Unicode code points in your `answer` |
| `sentences_total` | how many sentences the answer was split into |
| `words` | only with `token_level: true`; see below |
| `limits`, `warnings`, `models`, `latency_ms` | `context_truncated`, `sentences_omitted`, …; typical warnings `context_truncated`, `custom_threshold`, `claim_rules_english` |

```bash
curl -sS http://localhost:8800/g1/hallucination-context \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" -d '{
  "prompt": "When was the Eiffel Tower completed and how tall is it?",
  "context": ["The Eiffel Tower was completed in 1889 for the World Fair in Paris. It stands 330 metres tall and was designed by the engineering firm of Gustave Eiffel."],
  "answer": "The Eiffel Tower was completed in 1925 for the World Fair in Paris. It stands 330 metres tall and was designed by Gustave Eiffel.",
  "token_level": true}'
```
```json
{"hallucinated": true, "probability": 0.999604, "threshold": 0.519517, "profile": "balanced", "context_group": "short",
 "sentences": [{"id": 0, "text": "The Eiffel Tower was completed in 1925 for the World Fair in Paris.", "start": 0, "end": 67,
                "probability": 0.999651, "flagged": true, "type": "contradicted"}],
 "words": [{"text": "completed", "start": 21, "end": 30, "probability": 0.07735, "sentence_id": 0},
           {"text": "1925", "start": 34, "end": 38, "probability": 0.262916, "sentence_id": 0}],
 "limits": {"context_truncated": false}, "warnings": []}
```
The answer is flagged; the second sentence is supported and is **not** listed.

### Words

`words` lists the words of each flagged sentence with a weight. **Rank by `probability`** (here `1925` is the highest): the weights are a
**relative** signal inside the sentence, not independent probabilities that every listed word is wrong. In the UI the words are shaded by their
weight relative to the other words of the same sentence.

### Short sentences

A sentence with fewer than two content words (for example `It was completed in 1925.`) is **not scored on its own**. The verdict of the whole
answer (`hallucinated`) still covers it, so `sentences` can be empty even when `hallucinated` is `true`. Show the verdict, and tell the user that
no single sentence could be isolated.

### What to do with a verdict

`hallucinated: true` → do not show the answer as it is. Regenerate with the passages and an instruction to cite them, or show the answer with the
flagged sentences marked. A verdict on a [truncated request](#truncation) is not a verdict.

---

## `sufficiency`

Given a question and the retrieved context, the service says whether the context is **sufficient** to support the answer and,
if it is not, proposes search queries that describe the missing information. It **does not retrieve anything** and contains
no retriever: your own retriever consumes the queries.

Same request as above, with these differences: `prompt` is **required** (the queries depend on the question), `answer` is required for now, plus

| Field | Default | Meaning |
|---|---|---|
| `queries` | `true` | `false` returns only the verdict |
| `k_queries` | **10** | the **maximum** number of queries to return (1–50). **You choose it.** The model may produce fewer; the response never pads |

| Response field | Meaning |
|---|---|
| `sufficient` | **the decision**: `true` if the context is enough; `false` when at least one claim in the answer lacks support **because information is missing** |
| `insufficiency.probability` | confidence that the context is insufficient. **Not a calibrated probability** (`insufficiency.calibrated: false`): use it only against a threshold you have validated, never present it as a probability |
| `missing` | the answer sentences the context does not support: `{sentence_id, text, probability}` |
| `queries` | search queries for **your own retriever**: `{text, language, anchored, origin}`, at most `k_queries` |
| `queries_found`, `queries_returned` | produced vs returned |
| `queries_status` | `generated`, `not_needed` (sufficient), `need_question`, `timeout`, `error`, `off` |
| `limits` | as above plus `query_passages_omitted`, `query_input_tokens` |

```json
{"sufficient": false,
 "insufficiency": {"probability": 0.9834, "calibrated": false},
 "missing": [{"sentence_id": 0, "text": "Acme was founded by Wile in 1949.", "probability": 0.9994}],
 "queries": [{"text": "who founded Acme", "language": "en", "anchored": true, "origin": "greedy"},
             {"text": "Acme founding date", "language": "en", "anchored": true, "origin": "greedy"},
             {"text": "Acme company founder", "language": "en", "anchored": true, "origin": "greedy"}],
 "queries_found": 3, "queries_returned": 3, "queries_status": "generated"}
```

### How the model reads the passages

The model reads **each passage with its title and label in front**, in the form `title\nlabel\ntext`. This is part of how
the verdict was calibrated, so give titles when you have them.

| Passage form | Status |
|---|---|
| string (`"…"`) | accepted today |
| object `{text, title?, label?}` | **to be added**. The 2.0.0 build as served today accepts strings only and answers `400 invalid_context` for objects. Until then put the title and label in the passage text yourself, in the form `title\nlabel\ntext` |

### Published operating points

The verdict was measured and calibrated on a set built from **real documents**: **3,169 questions**, each in three
versions — the **full context**, the context with the **key passage removed**, and the context with the key passage
**replaced by an unrelated one**.

- **Positives** (context is insufficient) = the removed-passage version.
- **Negatives** = the full and the replaced versions: **6,338** rows.
- A **false positive** is a sufficient context flagged as insufficient.

The thresholds were chosen **on this set**, cross-validated by document group (5×20), so these are
**calibration numbers, not a held-out test**. AUROC **0.982** (95% CI 0.978–0.986).

| False-positive rate target | Threshold | Recall of insufficient contexts (95% CI) | Cross-validated FPR |
|---|---|---|---|
| 1% | 0.920 | 0.444 (0.413–0.481) | 1.01% |
| 2% | 0.825 | 0.664 (0.634–0.696) | 2.04% |
| 3% | 0.679 | 0.847 (0.826–0.867) | 3.03% |
| 5% | 0.339 | 0.961 (0.951–0.970) | 5.03% |
| 10% | 0.175 | 0.990 (0.985–0.994) | 10.0% |

All rows: calibration set above, cross-validated by document group (5×20), thresholds applied to the sufficiency score.

**Development set** (299 held-out questions, **descriptive only, thresholds not tuned on it**): AUROC **0.994**
(0.989–0.998). At the 1% threshold: observed FPR 0.0%, recall 0.408 (0.347–0.497). At the 2% threshold: observed FPR
0.17%, recall 0.642.

!!! note "Relation to the served build"
    In the 2.0.0 build as served today, the `sufficient` field is decided by claim-level support (a sentence of the answer
    that the context does not support because information is missing), **not** by the thresholds in this table. The table
    is the reference for the sufficiency score it was measured on; wiring those thresholds into the served verdict is not
    done yet, so the profile names do not select these thresholds today.

Read these numbers with the following in mind:

- **Recall is a conservative lower bound.** In a blind read of 60 cases, about **8%** of the "insufficient" labels were actually answerable.
- **Profiles.** `low_false_positives`, `balanced` and `high_recall` are named operating points; their **precision, recall and false-positive rate on the sufficiency verdict are not yet measured**. Use the table above as the reference, and do not read profile names as guarantees.
- **Calibrate on your own sample** (a few hundred labelled examples) before you gate anything ([checklist](#how-to-validate-on-your-data)).
- **Latency (P95) is not yet measured.**
- **New document domains.** Accuracy on other domains is being measured on a dedicated transfer benchmark (software specifications, API descriptions, coding-agent traces, contracts, regulations, patents) and will be added when it is finished. **Transfer is not claimed.**
- **Languages.** Per-language accuracy is not yet published ([Languages](#languages)).
- The numbers hold for requests that were **not truncated** ([Truncation](#truncation)).

### Query suggestions for missing information

When the context is insufficient the service proposes search queries that describe the missing information.

- The model usually produces two or three queries by itself. Asking for more (`k_queries` up to 50) adds **sampled variants**, filtered by the same anchoring rule, and the response carries the warning `queries_beyond_greedy_sampled`. Queries are never invented to reach `k_queries`.
- Queries are **hypotheses anchored on the answer**, not evidence: they preserve the entities of the question and of the answer, and the answer's own claims are still unverified.
- An empty `queries` with `queries_status` `timeout` or `error` does **not** mean that nothing is missing.
- **Retrieval-gain figures are not published**: how much the queries improve retrieval has not been independently confirmed, so there is no number to quote.

**Retrieval loop.** `sufficient == false` → run each `queries[].text` on your retriever → append the passages to `context` → regenerate →
call `hallucination-context` again. Stop after **at most two rounds** and tell the user if it is still unsupported. Check
[`limits.context_truncated`](#truncation) after every round: adding passages is what makes a request exceed the limit.

---

## `closed-book`

Judges an answer **without any document**, by reading each answer against the others. It is a **signal, not an automatic block**.

| Field | Required | Meaning |
|---|---|---|
| `answers` | yes | 2 to 8 strings. **`answers[0]` is the answer being judged**; the others are independent answers to the same question (sample your LLM several times) |
| `prompt` | no | the question |

| Response field | Meaning |
|---|---|
| `hallucinated`, `probability`, `threshold` | the decision is **for `answers[0]` only** |
| `answers` | one object per input: `{index, role: "judged" \| "reference", probability, flagged}` — the references are read too (an odd one out shows up) but only the first is decided |
| `n_answers`, `method` | how many answers were read and how |
| `calibrated`, `validation` | the threshold depends on the number of answers (2 → 0.9923, 3–4 → 0.9888, 5 or more → 0.9859); `validation` reports false-positive rate, recall and AUROC measured on a held-out bank of 411 rows in 9 languages |
| `warnings` | `closedbook_low_auroc` while the held-out AUROC is below 0.80 |

Measured on that held-out bank (411 rows, 9 languages): AUROC **0.66** with 2 answers, 0.70 with 3–4, **0.75** with 5 or more; false-positive rate **11–16%**.

- **Signal, not an automatic block.** Disagreement between answers does not prove that one is wrong, and two identical wrong answers are not flagged. More independent answers make it more reliable.
- **Only the first answer receives the verdict.** The others are read as references.
- **Cost.** Consistency needs 2–8 answers, so it costs **2–8 times the generation cost** of a single answer, plus the check itself.

```bash
curl -sS http://localhost:8800/g1/hallucination-closedbook \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" -d '{
  "prompt": "What is the capital of Australia?",
  "answers": ["The capital of Australia is Sydney.", "Canberra is the capital of Australia.", "Australia'"'"'s capital city is Canberra."]}'
```

Errors: `400 invalid_answers`, `400 not_enough_answers`, `413 too_many_answers`, `401 unauthorized`, `503 model_unavailable`.

## Through `/g1/eval`

[`/g1/eval`](eval.md) returns the same verdicts as axes: `halluc_context` (context + answer) and `halluc_closedbook` (the `answers` field), with the per-sentence
details in `axes.<name>.details`. Use the dedicated endpoints when you need the words, the queries or the sufficiency verdict.

## Versioning

Behaviour is **tied to a named model version**: the thresholds and numbers on this page belong to the model that was
measured, and a different model version can change them.

- Today each response carries `schema_version` (`"2.0"`) and the `models` block.
- A **detector version string in the response is to be added.** Until it is present, pin the image tag you deploy (`glad-vllm:2.0.0`) and record it with your results.
- Changes to the detector, its thresholds or the tables on this page will be listed in the [release notes](../reference/release-notes.md); re-run your own validation after any such change.

## How to validate on your data

Do this before any of these endpoints blocks production traffic.

1. **Label a few hundred of your own examples** for the check you will gate on (supported or not, sufficient or not), with the same prompts, contexts and answers your application produces.
2. **Pick the threshold for your tolerable false-block rate**: measure the false-block rate on your sufficient and supported examples at several thresholds, and choose the lowest false-block rate you can accept. The table above is a starting point, not a substitute.
3. **Run in audit-only (shadow) mode first**: call the endpoint, log the verdict and `limits`, do not act on it, and review the flagged cases with a person.
4. **Compare against your own baseline** (no check, or the check you use today) on the same traffic.
5. **Handle the non-answers**: `limits.context_truncated: true`, `503`, `404`, timeouts. Decide in advance what each one does (fail closed for anything that protects users).
6. **Re-run the validation** after a change of model version, of document domain, or of language.

## Known limitations

- It judges support by the **passages you sent**, not truth in the world; there is no external fact-checking.
- **Truncated requests** (16,384 tokens) give unreliable verdicts; treat them as not measurable.
- **Languages**: English, Italian, German, French and Spanish only; per-language accuracy and the error profile of sentence splitting outside English are **not yet published**. Polish, Dutch and Portuguese are unsupported; Russian and Chinese are best effort.
- **Sufficiency**: the table is **calibration, not a held-out test**; recall is a conservative lower bound (about 8% of "insufficient" labels were answerable in a blind read of 60 cases); per-profile precision, recall and FPR, the P95 latency and the accuracy on new document domains are **not yet measured**.
- `insufficiency.probability` is **not a calibrated probability**.
- **Query suggestions** are anchored hypotheses; retrieval-gain figures are **not published**; the service does not retrieve.
- **Closed-book** is a weaker signal (AUROC 0.66–0.75, false positives 11–16% on the held-out bank); only the first answer is judged; it costs 2–8 times the generation cost.
- **Words** carry relative weights, not independent probabilities; sentences shorter than two content words are not scored on their own.
- Passage objects `{text, title?, label?}` and the **detector version string** in responses are **to be added**.
