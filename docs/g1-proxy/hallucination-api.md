# Hallucination API — grounded, sufficiency and closed-book (2.1)

Three endpoints, served by the **G1V** and **dpo9** models on vLLM, in FP8 ([the vLLM stack](vllm-stack.md)). They judge whether an answer is
**supported by the documents it was given** — not whether it is true in the world.

**What is new in 2.1**

- **Contexts up to 1,048,576 tokens (1M)** in `hallucination-context` and `sufficiency`. A long context is read in overlapping windows and every claim is checked against the passages that matter for it ([Long contexts](#long-contexts-up-to-1m-tokens)).
- **`sufficiency` works without an answer**: send only the question and the context and get the verdict and the search queries before any answer exists ([sufficiency](#sufficiency)).
- **Faster.** A grounded check answers in about 0.3 s and a sufficiency check with its search queries in about 0.5 s on typical inputs (one RTX A6000, FP8); sufficiency with queries used to take several seconds.
- Passages can carry a **title and a label** (`{text, title, label}`), and `sufficiency` also returns the **missing facts** in plain words.

| Endpoint | Question it answers | You send |
|---|---|---|
| [`POST /g1/hallucination-context`](#hallucination-context) | Is the answer supported by the context? Which sentences and words are not? | question, context, answer |
| [`POST /g1/sufficiency`](#sufficiency) | Is the context **enough** to answer? If not, what should I search for? | question, context (the answer is optional) |
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
| `sufficiency` | decide whether to answer, retrieve more, or ask the user; and get the queries for the missing information | **Yes, as a routing decision** (answer / retrieve again / abstain). The false-alarm rate and the recall are published, up to 256k-token contexts ([Published operating points](#published-operating-points)) |
| `hallucination-closedbook` | monitor and sample: raise a flag for review, log it, route to a second answer | **No. It is a signal.** It fires on clean answers often enough that it should flag, not block |

The caveats on this page are there so that you can set the operating point deliberately: every number carries its measurement
condition, and where something has not been measured the page says so.

## What is behind the endpoints

The endpoints are served by **trained models**, not by a prompt around a general-purpose LLM: **G1V**, a fine-tuned reader that
produces the verdict, the flagged sentences and the word weights (one instance serves all three), and **dpo9**, a 2-billion-parameter
model that writes the search queries and the missing facts of `sufficiency`. The grounded thresholds are calibrated and depend on the context
length; the sufficiency decision uses a single measured threshold. Both models run in FP8 in your own deployment ([the vLLM stack](vllm-stack.md)) and
fit together with the safety models in a 24 GB GPU.

## Against a general-purpose LLM used as a judge

A common alternative is to ask a general-purpose LLM (here: a model of 27–100 billion parameters, run locally) whether an
answer is supported by the context. The table lists what that setup does not guarantee and what `hallucination-context` does.

On hallucination detection with context, **G1V is more than three times more accurate than an LLM used as a judge, on average across the benchmarks evaluated.**

| Point | LLM as a judge | `hallucination-context` (G1V) |
|---|---|---|
| **Coverage of the claims** | One verdict on the whole answer. There is no guarantee that every claim was examined: an error in the middle of a long answer can pass, and you cannot tell which part weighed on the verdict. | **Every claim is checked on its own against the whole context**, so none is skipped by construction, and you know which claim is flagged and where. A sentence with fewer than two content words is not scored on its own; the verdict of the whole answer still covers it ([Short sentences](#short-sentences)). |
| **Repeatability** | The answer can change from one run to the next and the JSON format can break. Even at temperature 0, how requests are batched together can change the result. | A **numeric score per claim**. It also varies slightly with batching: up to 0.025 depending on the padding. For this reason the reader is kept stable, and the verdict changes in 1–2% of cases on vLLM. |
| **Threshold** | A yes/no with no calibration: you do not know how many false alarms it produces. | A **continuous score** with a threshold chosen at a declared false-positive rate (for sufficiency, the measured false-alarm rate and recall in [Published operating points](#published-operating-points)). |
| **Explanation** | Usually one free-form sentence. | The **flagged claims and words, with their position** in your answer. |

**Checking claim by claim with an LLM.** An LLM can also do the check claim by claim, with several calls: extract the
claims, then make one call per claim. That multiplies the cost and the time by the number of claims, because each call is
a call to the large model; the result is **not repeatable**; and the quality of the extraction step is **not verified**.
G1V does the same claim-level check with a **small, calibrated model, in a repeatable way**, in one request.

## Health and readiness

`GET /g1/hallucination/health` → `200 {"status":"ok"}` when the verification service and its G1V and dpo9 servers answer.

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

## Shared rules (grounded and sufficiency)

- `context` is a string or an array of passages: each passage is a string or an object `{text, title, label}` (title and label are optional). At least one non-empty passage; up to 256 passages.
- **Context up to 1,048,576 tokens (1M).** A context that does not fit one 16,384-token reading is read in overlapping windows ([Long contexts](#long-contexts-up-to-1m-tokens)). The question and the answer together are read in one 16,384-token window.
- `profile`: `balanced` (default), `low_false_positives` (fewer false alarms, fewer detections) or `high_recall` (the opposite).
- `threshold` (optional, in [0, 1]) applies to the **calibrated probability**. Setting it changes the operating point; the response adds the warning `custom_threshold`.
- The decision threshold of `hallucination-context` depends on the context length: contexts of up to 7,000 characters (`context_group: "short"`, 0.5195 in the balanced profile) and longer ones (`"long"`, 0.2735).
- Errors: `400 missing_answer` (`hallucination-context`), `400 empty_context`, `400 invalid_context`, `400 invalid_profile`, `400 invalid_threshold`, `413 too_many_passages`, `413 context_too_long` (more than 1M tokens), `413 answer_too_long`, `413 input_too_long`, `401 unauthorized`, `503 model_unavailable`.

### Long contexts, up to 1M tokens

Send the whole context: up to **1,048,576 tokens** are read, with no cut and no rewriting on your side.

- A context that fits one 16,384-token reading is read at once, as before.
- A longer context is read in **windows of 16,384 tokens with a 2,048-token overlap**. For each sentence of the answer the checker keeps the **four windows most relevant to it** (ranked by BM25; all of them when there are five or fewer) plus a window built from the passages that match it best, and the sentence gets the lowest score among them; the verdict of the whole answer combines the answer-level reading with its weakest sentence.
- `sufficiency` reads the question against the same windows.
- The response carries a `reading` block: `band` (`short`, `long_single_window`, `long_windows`), `windows`, `readings` and `estimated_seconds`. The warning `long_context_windows` marks a request read in windows.

| Context | Result |
|---|---|
| up to 16,384 tokens | one reading, about a third of a second for a typical answer |
| 64k and 256k tokens | grounded verdict: AUROC within 0.01 of the single reading of the same answers (RAGTruth test +0.005, FaithBench −0.010 at 256k); sufficiency: see [Published operating points](#published-operating-points) |
| up to 1M tokens | grounded verdict: AUROC within 0.002 of the 256k reading on 50 answers lengthened with real passages; a context of roughly 50,000 tokens is checked in about 25 seconds (one RTX A6000) |
| more than 1M tokens | `413 context_too_long` |

Cost grows with the number of sentences and windows (`reading.estimated_seconds` tells you before you decide to send more). `limits.context_tokens_read` says how many tokens were read, and `limits.context_truncated` is `false` when the whole context was read.

Limits at a glance:

| Limit | What happens | Where you see it |
|---|---|---|
| more than 1,048,576 tokens of context, or more than 256 passages | the request is **rejected before reading** | `413 context_too_long`, `413 too_many_passages` |
| question + answer longer than one 16,384-token window | the request is rejected | `413 input_too_long` |

---

## `hallucination-context`

| Field | Type | Required | Meaning |
|---|---|---|---|
| `prompt` | string | no (recommended) | the user question; `messages` is also accepted (last user turn) |
| `context` | string \| array | yes | the passages the answer must rely on: strings or `{text, title, label}` objects; up to 1M tokens |
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
| `reading` | how the context was read: `band`, `windows`, `readings`, `estimated_seconds` ([Long contexts](#long-contexts-up-to-1m-tokens)) |
| `limits`, `warnings`, `models`, `latency_ms` | `context_tokens_read`, `sentences_omitted`, …; typical warnings `long_context_windows`, `custom_threshold`, `claim_rules_english`. Keep `warnings` with your logs |

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
 "reading": {"band": "short", "windows": null, "readings": 3}, "limits": {"context_truncated": false}, "warnings": []}
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

Given a question and the retrieved context, the service says whether the context is **sufficient** to answer the question and,
if it is not, says **what is missing** and proposes search queries for it. It **does not retrieve anything** and contains
no retriever: your own retriever consumes the queries.

**You do not need an answer.** Send the question and the context alone to decide, *before* generating, whether to answer, retrieve more or ask the user. If you send an answer too, the same call also returns the grounded verdict on it.

### What the sufficiency endpoint gives you

- **A verdict and a next step in one call.** It tells you whether the context is enough for the question, what is missing, and up to three search queries for it.
- **Question and context only.** The verdict is decided from the question read against the context, so it is available as soon as retrieval ends. Send the answer when you have it and you also get `hallucinated` and the unsupported sentences.
- **Contexts up to 1M tokens** ([Long contexts](#long-contexts-up-to-1m-tokens)).
- **Fast.** About 0.5 s with the search queries on typical inputs (one RTX A6000, FP8); the verdict alone is faster still.
- **Queries that fit your own retriever.** They are plain text, anchored on your question, so any search backend can consume them; nothing is retrieved on your behalf and nothing is sent to an external service.
- **Missing facts in plain words** (`missing_facts`), to show to the user or to log.
- **Five European languages:** English, Italian, German, French and Spanish ([Languages](#languages)).
- **Self-hosted.** The models run in your own deployment, in FP8, on one 24 GB GPU together with the safety models ([the vLLM stack](vllm-stack.md)).

Same request as above, with these differences: `prompt` is **required** (the verdict is decided from the question), `answer` is **optional**, plus

| Field | Default | Meaning |
|---|---|---|
| `queries` | `true` | `false` returns only the verdict |
| `k_queries` | **3** | the **maximum** number of queries to return (1–3 are produced; larger values are accepted and return the three). The model may produce fewer; the response never pads |

| Response field | Meaning |
|---|---|
| `sufficient` | **the decision**: `true` if the context is enough to answer the question; `false` when the question cannot be answered from it |
| `insufficiency.probability`, `insufficiency.threshold` | the score the decision is made on, and its threshold (**0.402**). `insufficiency.calibrated: false`: compare it with the threshold, do not present it as a probability |
| `missing_facts` | what the context lacks, in plain words (only when `sufficient` is `false`) |
| `queries` | search queries for **your own retriever**: `{text, language, anchored, origin}`, at most `k_queries`; `origin` is `dpo9` |
| `queries_found`, `queries_returned` | produced vs returned |
| `queries_status` | `generated`, `not_needed` (sufficient), `writer_found_none`, `timeout`, `error`, `off` |
| `hallucinated`, `missing` | with an `answer`: the grounded verdict and the answer sentences the context does not support (`{sentence_id, text, probability}`). Without an answer `hallucinated` is `null` and `missing` is empty |
| `reading`, `limits` | as in [Long contexts](#long-contexts-up-to-1m-tokens), plus `query_passages_omitted`, `query_input_tokens` |

```bash
curl -sS http://localhost:8800/g1/sufficiency \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" -d '{
  "prompt": "How many visitors did the Eiffel Tower receive in 2019?",
  "context": [{"title": "Eiffel Tower", "label": "History", "text": "The Eiffel Tower was completed in 1889 for the Exposition Universelle in Paris. It is 330 metres tall."}]}'
```
```json
{"sufficient": false,
 "insufficiency": {"probability": 0.7522, "threshold": 0.402, "calibrated": false},
 "missing_facts": ["Number of visitors to the Eiffel Tower in 2019"],
 "queries": [{"text": "Eiffel Tower visitors 2019", "language": "en", "anchored": true, "origin": "dpo9"}],
 "queries_found": 1, "queries_returned": 1, "queries_status": "generated", "hallucinated": null, "missing": []}
```

### How the model reads the passages

The model reads **each passage with its title and label in front**. Send them as `{text, title, label}`: a question that cites "Article 19" or "section 4.2" usually finds that reference in the retriever's metadata, not in the passage text. A plain string works too.

| Passage form | Status |
|---|---|
| string (`"…"`) | accepted |
| object `{text, title?, label?}` | accepted: the service puts the title and the label in front of the text |

On questions that cite an article or a section number, false alarms drop from 61% without title and label to 2.6% with them, so send them when your retriever has them.

### Published operating points

Sufficiency is decided with **one threshold (0.402)** on the score of the question read against the context. Measured on two development benchmarks of questions over real documents (BT: 2,361 rows; BR: 897 rows; passages with title and label), with answers or without:

| Benchmark | False alarms (sufficient contexts flagged) | Recall of insufficient contexts | F1 |
|---|---|---|---|
| BT | 3.4% | 97.8% | 0.956 |
| BR | 1.7% | 96.0% | 0.963 |

The same decision on **long contexts**, built by adding real distractor passages in the same language (300 BT and 299 BR items):

| Context | BT: false alarms / recall | BR: false alarms / recall |
|---|---|---|
| original length | 3.5% / 99.3% | 1.7% / 96.0% |
| 64k tokens | 2.8% / 97.7% | 1.5% / 94.7% |
| 256k tokens | 3.2% / 97.3% | 1.3% / 94.0% |

AUROC stays at 0.994–0.997 from the original length to 256k tokens. Recall measures the labelled "insufficient" contexts; in a blind read of 60 cases about 8% of those labels were actually answerable, so recall is a conservative figure.

To choose your own operating point, send `threshold` in the request, or calibrate on a few hundred labelled examples from your own domain ([checklist](#how-to-validate-on-your-data)).

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
[the false-alarm rate and the recall of the sufficiency decision](#published-operating-points), with the conditions of the
measurement.

In short: a sufficiency verdict with published operating points **plus** queries for the missing information, in five
European languages, self-hosted. The figures of other tools above are as published by their authors, on their own
benchmarks, and are **not comparable** with the measurements on this page.

### Query suggestions for missing information

When the context is insufficient, **dpo9** writes up to three search queries that describe the missing information, and the list of missing facts.

- Queries are anchored on the question: they keep its entities, and they are written in the language of the question.
- They are hypotheses for your retriever: run them, then check what comes back.
- An empty `queries` with `queries_status` `timeout` or `error` means the writer did not finish, not that nothing is missing.

**Retrieval loop.** `sufficient == false` → run each `queries[].text` on your retriever → append the passages to `context` → call `sufficiency` again → when `sufficient` is `true`, generate the answer and call `hallucination-context`. Stop after **at most two rounds** and tell the user if the context is still insufficient. The context can grow up to 1M tokens, so you can keep every retrieved passage.

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
- Today each response carries `schema_version` (`"2.1"`) and the `models` block.
- A **detector version string in the response is to be added.** Until it is present, pin the image tag you deploy (`glad-vllm:2.1.0`) and record it with your results.
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
