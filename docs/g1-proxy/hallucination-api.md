# Hallucination API — grounded, sufficiency and closed-book (2.0)

Three endpoints, served by the **G1V** model on vLLM ([the vLLM stack](vllm-stack.md)). They judge whether an answer is
**supported by the documents it was given** — not whether it is true in the world.

| Endpoint | Question it answers | You send |
|---|---|---|
| [`POST /g1/hallucination-context`](#hallucination-context) | Is the answer supported by the context? Which sentences and words are not? | question, context, answer |
| [`POST /g1/sufficiency`](#sufficiency) | Is the context **enough** to answer? If not, what should I search for? | question, context, answer |
| [`POST /g1/hallucination-closedbook`](#closed-book) | Do independent answers to the same question disagree, with no documents at all? | 2–8 answers |

Check that the build serves them: `GET /g1/hallucination/health` → `200 {"status":"ok"}`. A `404` means the build does not have
the routes — do not simulate the result. Authentication is the same as the [Safety API](safety-api.md#authentication).

!!! info "Also in the UI"
    Studio and the light UI have a **Hallucination** page with one tab per endpoint (Grounded, Sufficiency, Closed-book).
    See [Studio UI reference](../studio/ui-reference.md#hallucination).

## Shared rules (grounded and sufficiency)

- `context` is a string or an array of strings (each passage separately). At least one non-empty passage; up to 256 passages and 200,000 characters.
- **Total input limit: 16,384 tokens** (instructions + question + context + answer). When it does not fit, the **end of the context is cut**:
  `limits.context_truncated` is `true` and `limits.context_tokens_read` says how much was read. A truncated context means a **partial check**.
- `profile`: `balanced` (default), `low_false_positives` (fewer false alarms, fewer detections) or `high_recall` (the opposite).
- `threshold` (optional, in [0, 1]) applies to the **calibrated probability**. Setting it changes the operating point; the response adds the warning `custom_threshold`.
- The decision threshold depends on the context length: contexts of up to 7,000 characters (`context_group: "short"`, 0.5195 in the balanced profile) and longer ones (`"long"`, 0.2735).
- Sentence rules are tuned for English: other languages add the warning `claim_rules_english`.
- Errors: `400 missing_answer`, `400 empty_context`, `400 invalid_context`, `400 invalid_profile`, `400 invalid_threshold`, `413 too_many_passages`, `413 context_too_long`, `413 answer_too_long`, `413 input_too_long`, `401 unauthorized`, `503 model_unavailable`.

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
flagged sentences marked.

---

## `sufficiency`

Same request as above, with these differences: `prompt` is **required** (the queries depend on the question), `answer` is required for now, plus

| Field | Default | Meaning |
|---|---|---|
| `queries` | `true` | `false` returns only the verdict |
| `k_queries` | **10** | the **maximum** number of queries to return (1–50). **You choose it.** The model may produce fewer; the response never pads |

| Response field | Meaning |
|---|---|
| `sufficient` | **the decision**: `true` if the context is enough; `false` when at least one claim in the answer lacks support **because information is missing** |
| `insufficiency.probability` | confidence that the context is insufficient. **Not calibrated** (`insufficiency.calibrated: false`): do not present it as a calibrated probability |
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

Rules:

- The model usually produces two or three queries by itself. Asking for more (`k_queries` up to 50) adds **sampled variants**, filtered by the same
  anchoring rule, and the response carries the warning `queries_beyond_greedy_sampled`. Queries are never invented to reach `k_queries`.
- An empty `queries` with `queries_status` `timeout` or `error` does **not** mean that nothing is missing.
- Queries are **hypotheses anchored on the answer**, not evidence: they preserve the entities of the question and of the answer, and the answer's own claims are
  still unverified.

**Retrieval loop.** `sufficient == false` → run each `queries[].text` on your retriever → append the passages to `context` → regenerate →
call `hallucination-context` again. Stop after **at most two rounds** and tell the user if it is still unsupported.

---

## `closed-book`

Judges an answer **without any document**, by reading each answer against the others.

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

Measured on that bank: AUROC 0.66 (2 answers), 0.70 (3–4), 0.75 (5 or more), false positives 11–16%. **Closed-book is a signal, not a block**: disagreement between answers
does not prove that one is wrong, and two identical wrong answers are not flagged. More independent answers make it more reliable.

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
