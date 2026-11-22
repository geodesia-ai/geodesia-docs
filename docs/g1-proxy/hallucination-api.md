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

## Which endpoint can gate, and which only informs

| Endpoint | Use it to | Can it block on its own? |
|---|---|---|
| `hallucination-context` | gate or flag an answer against the passages it was given | **Yes**, once you have chosen its threshold on your data (see [How to validate on your data](#how-to-validate-on-your-data)). Its measured behaviour is in [Measured accuracy](#measured-accuracy-g1v-against-other-detectors) |
| `sufficiency` | decide whether to answer, retrieve more, or ask the user; and get the queries for the missing information | **Yes, as a routing decision** (answer / retrieve again / abstain). Operating points at stated false-positive rates are published ([Published operating points](#published-operating-points)); the served `sufficient` verdict does not use those thresholds yet (see the note there) |
| `hallucination-closedbook` | monitor and sample: raise a flag for review, log it, route to a second answer | **No. It is a signal.** It fires on clean answers often enough that it should flag, not block |

The caveats on this page are there so that you can set the operating point deliberately: every number carries its measurement
condition, and where something has not been measured the page says so.

## What is behind the endpoints

The endpoints are served by **trained models**, not by a prompt around a general-purpose LLM: **G1V**, a fine-tuned reader that
produces the verdict, the flagged sentences and the word weights, and a generation adapter on the same model that writes
the search queries of `sufficiency`. The grounded thresholds are calibrated and depend on the context
length; the sufficiency operating points were fixed on a measured set at stated false-positive rates. Both run in your own
deployment ([the vLLM stack](vllm-stack.md)).

## Against a general-purpose LLM used as a judge

A common alternative is to ask a general-purpose LLM (here: a model of 27–100 billion parameters, run locally) whether an
answer is supported by the context. The table lists what that setup does not guarantee and what `hallucination-context` does.

On hallucination detection with context, **G1V is more than three times more accurate than an LLM used as a judge, on average across the benchmarks evaluated.**

| Point | LLM as a judge | `hallucination-context` (G1V) |
|---|---|---|
| **Coverage of the claims** | One verdict on the whole answer. There is no guarantee that every claim was examined: an error in the middle of a long answer can pass, and you cannot tell which part weighed on the verdict. | **Every claim is checked on its own against the whole context**, so none is skipped by construction, and you know which claim is flagged and where. A sentence with fewer than two content words is not scored on its own; the verdict of the whole answer still covers it ([Short sentences](#short-sentences)). |
| **Repeatability** | The answer can change from one run to the next and the JSON format can break. Even at temperature 0, how requests are batched together can change the result. | A **numeric score per claim**. It also varies slightly with batching: up to 0.025 depending on the padding. For this reason the reader is kept stable, and the verdict changes in 1–2% of cases on vLLM. |
| **Threshold** | A yes/no with no calibration: you do not know how many false alarms it produces. | A **continuous score** with a threshold chosen at a declared false-positive rate (for sufficiency, the 1–10% table in [Published operating points](#published-operating-points)). |
| **Explanation** | Usually one free-form sentence. | The **flagged claims and words, with their position** in your answer. |

**Checking claim by claim with an LLM.** An LLM can also do the check claim by claim, with several calls: extract the
claims, then make one call per claim. That multiplies the cost and the time by the number of claims, because each call is
a call to the large model; the result is **not repeatable**; and the quality of the extraction step is **not verified**.
G1V does the same claim-level check with a **small, calibrated model, in a repeatable way**, in one request.

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

!!! info "Longer contexts in the next release"
    The next release will support a context of **256k tokens**. The limits on this page describe the current build (16,384 tokens).

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

### Measured accuracy: G1V against other detectors

**Design.** G1V was compared with Jev, Vectara HHEM 2.3, Granite Guardian 3.3 and LettuceDetect (small and large) on
**10,121 answers across 11 public benchmarks**, under identical row-level testing conditions. The test sets are out-of-distribution for G1V except where noted as in-domain; their
out-of-distribution status **for the competing models is not guaranteed**.

**AUROC by benchmark** (full-response faithfulness; higher is better):

| Benchmark | Responses | G1V | Jev | Vectara HHEM 2.3 | Granite Guardian 3.3 | LettuceDetect mmBERT | LettuceDetect Qwen 2B |
|---|---|---|---|---|---|---|---|
| ANLI-R3 | 1,200 | 0.897 | 0.885 | 0.777 | 0.809 | 0.523 | 0.668 |
| CovidFact | 316 | 0.928 | 0.908 | 0.872 | 0.897 | 0.728 | 0.776 |
| FaithBench | 750 | 0.707 | 0.726 | 0.610 | 0.611 | 0.642 | 0.668 |
| FaithDial | 800 | 0.967 | 0.899 | 0.813 | 0.866 | 0.764 | 0.712 |
| FinQA | 1,657 | 0.823 | 0.680 | 0.423 | 0.423 | 0.406 | 0.457 |
| PlaceboBench | 482 | 0.788 | 0.656 | 0.543 | 0.493 | 0.587 | 0.608 |
| RAGTruth++ | 408 | 0.867 | 0.864 | 0.725 | 0.688 | 0.714 | 0.708 |
| RAGTruth test | 2,700 | 0.926 | 0.940 | 0.873 | 0.844 | 0.880 | 0.905 |
| SciFact | 209 | 0.963 | 0.954 | 0.862 | 0.908 | 0.789 | 0.892 |
| HalluDial | 799 | 0.789 | 0.776 | 0.687 | 0.702 | 0.728 | 0.700 |
| TabFact | 800 | 0.971 | 0.955 | 0.704 | 0.787 | 0.616 | 0.717 |
| **Benchmark average** | 10,121 | **0.875** | 0.840 | 0.717 | 0.730 | 0.671 | 0.710 |

G1V has a higher AUROC than Vectara HHEM 2.3 on all 11 benchmarks and a higher numerical AUROC than Jev on 9 of 11
(Jev is higher on FaithBench and RAGTruth test). Some of these differences are small (for example RAGTruth++, 0.867 against 0.864);
the paired bootstrap intervals are in the underlying report and are not reproduced here.

**Macro-average over the same 11 benchmarks, all configurations.** AUROC does not depend on a threshold; F1,
recall and false-positive rate **depend on the operating threshold calibrated for each model**. F1 is for the hallucination class.

| Model configuration | Mean AUROC | Mean F1 | Mean recall | Mean false positives |
|---|---|---|---|---|
| G1V | 0.875 | 0.678 | 0.652 | 14.0% |
| Jev | 0.840 | 0.673 | 0.711 | 26.0% |
| Granite Guardian 3.3 8B | 0.730 | 0.593 | 0.701 | 42.2% |
| Granite 4.0 Micro (unfaithful only) | 0.724 | 0.520 | 0.539 | 27.0% |
| Granite 4.0 Micro (unfaithful or partial) | 0.721 | 0.549 | 0.578 | 30.2% |
| Vectara HHEM 2.3 | 0.717 | 0.590 | 0.649 | 33.8% |
| LettuceDetect v2 Qwen 2B | 0.710 | 0.578 | 0.771 | 63.8% |
| Granite 4.1 3B (unfaithful only) | 0.692 | 0.481 | 0.458 | 23.6% |
| Granite 4.1 3B (unfaithful or partial) | 0.687 | 0.514 | 0.523 | 29.4% |
| LettuceDetect v2 mmBERT | 0.671 | 0.454 | 0.439 | 24.8% |
| Granite 4.1 8B (unfaithful or partial) | 0.664 | 0.505 | 0.609 | 29.3% |
| Granite 4.1 8B (unfaithful only) | 0.662 | 0.459 | 0.410 | 16.7% |

The pooled false-positive rate and the per-benchmark false-positive rates are different aggregates; the table reports the macro-average of the per-benchmark rates.

**PlaceboBench against Claude Opus 5.5.** On an identical **balanced sample of 450 answers**. Claude Opus 5.5 received the
**full document**; G1V reads **up to 16,384 tokens**. Each model is at its own operating point.

| Model | AUROC | Hallucination F1 | Precision | Recall | False positives |
|---|---|---|---|---|---|
| G1V | 0.790 | 0.487 | 0.904 | 0.333 | 3.6% |
| Claude Opus 5.5 | 0.782 | 0.623 | 0.809 | 0.507 | 12.0% |
| Jev | 0.650 | 0.284 | 0.780 | 0.173 | 4.9% |

G1V has the highest AUROC, the highest precision and the lowest false-positive rate of the three on this sample; Claude
Opus 5.5 has the higher recall and F1. The AUROC difference between G1V and Claude Opus 5.5 (0.790 against 0.782) is small, and the balanced
sample (450) is not the 482-answer PlaceboBench set of the table above (0.788). G1V's recall of 0.333 at this operating
point means most hallucinated answers on this benchmark are **not** flagged.

---

## `sufficiency`

Given a question and the retrieved context, the service says whether the context is **sufficient** to support the answer and,
if it is not, proposes search queries that describe the missing information. It **does not retrieve anything** and contains
no retriever: your own retriever consumes the queries.

### What the sufficiency endpoint gives you

- **A verdict and a next step in one call.** It tells you whether the context is enough, which sentences of the answer the context does not support, and proposes search queries for the missing information.
- **Queries that fit your own retriever.** They are plain text, anchored on your question and on the answer, so any search backend can consume them; nothing is retrieved on your behalf and nothing is sent to an external service.
- **Published operating points.** A table at fixed false-positive targets (1% to 10%) with confidence intervals, so you can choose the false-block rate you can tolerate. The conditions and the limits of the measurement are stated next to it.
- **Five European languages:** English, Italian, German, French and Spanish ([Languages](#languages)).
- **Self-hosted.** The models run in your own deployment, in FP8, on one GPU ([the vLLM stack](vllm-stack.md)).

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

### Zero-shot against Lacuna v1 (pilot sets)

Context sufficiency measured **without an answer** — does the context contain enough information to answer the
question — as AUROC on **pilot out-of-distribution development sets**:

| Dataset | G1V zero-shot | Lacuna v1 |
|---|---|---|
| Housing QA | 0.951 | 0.882 |
| PubMedQA | 0.951 | 0.843 |
| SQuAD2 | 0.935 | 0.925 |

These are **pilot development sets**, not a held-out test, and only AUROC is reported (no thresholds, recall or false-positive rate).
They are not the dedicated transfer benchmark mentioned above.

### Where this fits among related tools

Sufficiency is the part of this page that fewer tools offer: **a verdict on whether the context is enough, together with
search queries aimed at what is missing**. Specialised hallucination-detection vendors are not yet covered.

| Tool | Judges whether the context is enough? | Proposes queries for what is missing? |
|---|---|---|
| Amazon Bedrock, Azure and Google grounding checks | Not found: they check the faithfulness of the answer | Not found |
| NVIDIA NeMo Guardrails, TruLens, Anthropic, OpenAI, Contextual AI | Not found | Not found |
| IBM Granite RAG Library (Apache 2.0, runs locally) | Yes: "Answerability", 90.6 on MT-RAG against 82.5 for GPT-4o, as published by IBM; English | Not found (its Query Clarification asks the user a question) |
| Weaviate Query Agent (Weaviate Cloud only) | Partly: it judges the answer, not the context on its own | Partly: it lists the missing information, not stated as a search query |
| Cohere | Not found | Generates queries from the question alone, before the documents are seen |
| LlamaIndex and LangGraph CRAG | Per-document relevance, yes or no | Generic rewriting of the question, not aimed at the gap |
| Ragas | Offline only; needs the correct answer | Not found |
| Google "Sufficient Context" | Research, not a product: Gemini 1.5 Pro autorater, F1 0.935 on 115 examples, as published | Not found |

Two further observations from the same review: none of the three cloud grounding checks publishes an accuracy figure for
its check, and we found no threshold guidance tied to a stated false-positive rate. Geodesia publishes
[operating points at stated false-positive rates](#published-operating-points), with the conditions and limits of the
measurement.

In short: a sufficiency verdict with published operating points **plus** queries for the missing information, in five
European languages, self-hosted. The figures of other tools above are as published by their authors, on their own
benchmarks, and are **not comparable** with the measurements on this page.

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
| `calibrated`, `use` | the threshold depends on the number of answers (2 → 0.9923, 3–4 → 0.9888, 5 or more → 0.9859); `use` states that the result is a signal, not an automatic block |
| `warnings` | none specific to closed-book |

Accuracy depends on the stability of the LLM that produced the answers, and it improves with the number of independent answers.

- **How to use it.** Log the flag, sample the flagged answers for review, or route them to a second generation. Do not use it as the only reason to block an answer.
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

- **Pin your own operating point.** Every response returns the `threshold` it used. Send `threshold` in the request to fix the operating point yourself: a change to the default thresholds then does not move yours (the response adds the warning `custom_threshold`).
- Today each response carries `schema_version` (`"2.0"`) and the `models` block.
- A **detector version string in the response is to be added.** Until it is present, pin the image tag you deploy (`glad-vllm:2.0.0`) and record it with your results.
- Changes to the detector, its thresholds or the tables on this page will be listed in the [release notes](../reference/release-notes.md); re-run your own validation after any such change.

## How to validate on your data

Do this before any of these endpoints blocks production traffic.

1. **Label a few hundred of your own examples** for the check you will gate on (supported or not, sufficient or not), with the same prompts, contexts and answers your application produces.
2. **Pick the threshold for your tolerable false-block rate**: measure the false-block rate on your sufficient and supported examples at several thresholds, and choose the lowest false-block rate you can accept. The table above is a starting point, not a substitute.
3. **Run in audit-only (shadow) mode first**: call the endpoint, log the verdict and `limits`, do not act on it, and review the flagged cases with a person.
4. **Compare against baselines on the same labelled sample**, at the same false-block rate: (a) no check, or the check you use today; (b) a cross-encoder or NLI entailment checker; (c) a plain prompt to a general-purpose LLM that asks whether the answer is supported. Measure precision and recall at the false-block rate you tolerate, not accuracy at a default threshold.
   **Adopt the endpoint where it beats these at your false-block rate**, or where its other properties (self-hosted, queries for the missing information, sentence and word detail) are what you need. 300–500 labelled pairs is a practical size to start with; at that size the confidence interval on recall at a 1–2% false-positive rate is wide, so read small differences with care.
5. **Handle the non-answers**: `limits.context_truncated: true`, `503`, `404`, timeouts. Decide in advance what each one does (fail closed for anything that protects users).
6. **Re-run the validation** after a change of model version, of document domain, or of language.

## Known limitations

- It judges support by the **passages you sent**, not truth in the world; there is no external fact-checking.
- **Truncated requests** (16,384 tokens) give unreliable verdicts; treat them as not measurable. The next release will support a 256k-token context.
- **Languages**: English, Italian, German, French and Spanish only; per-language accuracy and the error profile of sentence splitting outside English are **not yet published**. Polish, Dutch and Portuguese are unsupported; Russian and Chinese are best effort.
- **Sufficiency**: the table is **calibration, not a held-out test**; recall is a conservative lower bound (about 8% of "insufficient" labels were answerable in a blind read of 60 cases); per-profile precision, recall and FPR, the P95 latency and the accuracy on new document domains (beyond the pilot sets above) are **not yet measured**.
- `insufficiency.probability` is **not a calibrated probability**.
- **Query suggestions** are anchored hypotheses; retrieval-gain figures are **not published**; the service does not retrieve.
- **Closed-book** is a signal whose accuracy depends on the stability of the LLM that produced the answers; only the first answer is judged; it costs 2–8 times the generation cost.
- **Words** carry relative weights, not independent probabilities; sentences shorter than two content words are not scored on their own.
- Passage objects `{text, title?, label?}` and the **detector version string** in responses are **to be added**.
