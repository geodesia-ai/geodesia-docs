# Human Feedback Loop

Geodesia G-1 turns everyday chat usage into a **continuous improvement flywheel**. When a user spots a bad answer — a hallucination that slipped through, or a benign message that was wrongly flagged — they raise a one-click **feedback flag** directly in the chat. A reviewer (the *curator*) triages the flag in a queue, and the approved incidents feed two loops:

- a **fast loop** — an opt-in, non-parametric [episodic exemplar bank](#episodic-exemplar-bank-fast-loop) that the live detector consults at scoring time, giving exact recall of the corrected pattern **without retraining**;
- a **slow loop** — a labelled **JSONL corpus** you export and aggregate for the next detector fine-tune, [triggerable from the API](#triggering-a-re-train).

Corrections can also be raised from [Policy Lens](../studio/policy-lens.md) while tuning a threshold on real traffic — same queue, same pipeline.

!!! abstract "Why it matters"
    The detector geometry is validated out-of-distribution and must not be disturbed by ad-hoc tweaks. The feedback loop lets a deployment *memorize* rare, deployment-specific incidents — a particular jailbreak phrasing, a domain term the model over-flags — without smearing that validated manifold. **Memorize, don't retrain** for the cases that matter today; fold them into the weights later, deliberately.

![Diagram](../assets/diagrams/gateway-feedback.svg){: .diagram }

The whole system is **additive and model-agnostic**: flags live in one extra SQLite table on the database the gateway already uses (no new datastore), and the canonical incident record is shared by both detectors — **GLAD-Hummingbird** (all nine axes) and **GLAD-Tapestry** (the five it has). A flag whose axis a given model does not have is simply skipped when that model builds its bank.

---

## Flagging from chat

Every message in the chat carries a small **flag** control. A flag is **region-scoped**:

- flag an **assistant** message → `region: "answer"`
- flag a **user** message → `region: "prompt"`

The user then picks a **plain-language problem** (no ML jargon). The system maps that choice to a *suggested* detection axis, which the curator can later override:

| Region | Plain-language problem | `problem` key | Suggested axis |
|---|---|---|---|
| `answer` | *"It is made up / not true"* | `fabricated` | `halluc_closedbook` |
| `answer` | *"Contradicts the sources / the document"* | `contradicts_sources` | `halluc_context` |
| `answer` | *"Contains dangerous or harmful content"* | `dangerous_answer` | `answer_safety` |
| `prompt` | *"Dangerous request not blocked"* | `dangerous_request` | `prompt_safety` |
| `prompt` | *"Attempt to bypass the rules"* | `jailbreak_attempt` | `jailbreak` |
| `prompt` | *"Document with hidden instructions"* | `hidden_instructions` | `rag_jailbreak` |
| `prompt` | *"Contains profanity / offensive language"* | `profane_content` | `profanity` |
| `prompt` | *"Off-topic / out of scope"* | `off_topic` | `out_of_scope` |
| `prompt` | *"Request is too complex / ambiguous"* | `too_complex` | `prompt_complexity` |
| either | *"It was wrongly blocked (this is benign)"* | `wrongly_blocked` | *inferred from the score snapshot* |
| either | *"👍 This is benign — correct"* | `correctly_benign` | *inferred — positive reinforcement* |
| either | *"👍 A real threat, correctly blocked"* | `correctly_blocked` | *inferred — positive reinforcement* |
| either | *"Other…"* (free text) | `other` | *curator assigns* |

!!! tip "Praise is training data too"
    `correctly_benign` and `correctly_blocked` are **positive reinforcement**: they confirm a decision that was already right. A confirmed benign becomes a benign anchor (keep allowing this), a confirmed threat becomes a danger anchor. A loop that only ever hears about mistakes drifts, because nothing holds the correct decisions in place.

The vocabulary above is **served, not hard-coded**: `GET /v1/glad/feedback/schema` returns the axis list, the region grouping and the full `problem → axis` map, so a front-end stays in sync as axes are added and no UI code changes when the served checkpoint gains an axis.

The flag snapshots the surrounding context — the `prompt`, any RAG `context`, the `answer`, and the detector `scores` at flag time — so the curator has the full picture and the corpus is self-contained. The new row starts as `status: "pending"` and is scoped to the active Application (resolved from `application_id` in the body or the `X-Geodesia-App` header, falling back to `default`).

---

## The review queue

Curators work the queue in the **Feedback** view of the web UI (or directly via the [API](#rest-api)). For each pending flag a curator can:

- **Approve** — confirm the incident, set the final **axis** and the **verdict**:
    - `false_negative` — the detector *should* have flagged this and didn't (a miss). Becomes a **positive** training example.
    - `false_positive` — the detector flagged a benign message (an over-flag). Becomes a **negative** example.
- **Reject** — noise or abuse. Rejected rows train nothing.
- **Re-open** — move a row back to `pending`.

Only **approved** rows with a resolved axis feed the export and the exemplar bank. The default verdict for an approved flag is `false_negative` (something slipped through), and the default axis is the one suggested from the user's problem — the curator overrides either as needed.

### Contrastive safety memory (optional)

The review form also accepts the fields of the **v2 Contrastive Safety Memory (Membrane)** — a benign *twin* of the flagged incident plus a cell `weight` and an `attack_family` label. When present these let the exemplar bank reason contrastively (a benign counterpart right next to the dangerous one), sharpening the boundary instead of just pushing it. These fields are optional; omit them and the bank behaves as a plain episodic memory.

---

## Episodic exemplar bank (fast loop)

The approved corpus can be turned into a live, **non-parametric memory** that nudges detection at scoring time. It is **opt-in** and **off by default** — when disabled, no bank is built or consulted and detection is **byte-identical** to today.

At scoring time the detector embeds the current input onto its manifold (`q_sphere`) and compares it against the stored exemplars for the axes of that region:

- a close match to a **false-negative** exemplar **raises** the axis probability (*we missed this before — never again*);
- a close match to a **false-positive** exemplar **suppresses** it (*a known benign pattern we over-flagged*);
- a danger match always wins over a benign one — the bank never suppresses something that also resembles a known dangerous case.

The match is a cosine similarity on the unit sphere with a high floor (`τ`, default `0.88`): this is **exact-pattern recall**, not fuzzy generalization (that is the slow-loop fine-tune's job). Each detector builds its **own** bank from the same shared corpus using its own embedding function, and a bank is rebuilt only when the corpus version changes.

When a bank match moves a score, the affected axis carries an `exemplar_match` annotation in the response so the contribution is auditable:

```json
{
  "axis_energy": {
    "jailbreak": {
      "p_detector": 0.94,
      "threshold": 0.5,
      "flag": true,
      "exemplar_match": { "verdict": "false_negative", "sim": 0.93 }
    }
  }
}
```

### Enabling the bank

| Variable | Default | Description |
|---|---|---|
| `GW_FEEDBACK_BANK` | `off` | Master switch. `on` builds and consults the per-model bank from approved feedback. `off` → never built, zero overhead, byte-identical detection. |
| `GW_BANK_V2` | `off` | Use the **Contrastive Safety Memory** bank (kNN-LM-style vote over dangerous/benign twins). Requires `GW_FEEDBACK_BANK=on`. |
| `GW_FEEDBACK_BANK_TAU` | `0.88` | Cosine-similarity floor below which an exemplar is ignored. Higher = stricter exact-pattern recall. |
| `GW_FEEDBACK_BANK_GAIN` | `1.0` | How hard a perfect match pushes the probability (`1.0` = all the way to 0/1 at `sim == 1`, `weight == 1`). |

```bash
# start the gateway with the episodic exemplar bank enabled
GW_FEEDBACK_BANK=on \
  python -m glad_minimal.gateway.geodesia_gateway --host 0.0.0.0 --port 8800 ...
```

!!! note "Storage"
    Feedback lives in the `feedback` table of the gateway's existing SQLite database (`var/glad.sqlite3` by default, or `GW_DB_PATH`). Creating it never touches existing tables, so any deployment picks it up automatically.

---

## Export for fine-tuning (slow loop)

The approved corpus downloads as **JSONL** — one training example per line — ready to aggregate across Applications (and, for a vendor, across customers) for the next detector fine-tune:

```bash
curl -s "http://localhost:8800/v1/glad/feedback/export?status=approved&application_id=acme" \
  -o feedback_approved_acme.jsonl
```

Each line is self-contained:

```json
{
  "id": "fb_9c1f2a7b4e0d6a18",
  "region": "prompt",
  "axis": "jailbreak",
  "label": 1,
  "verdict": "false_negative",
  "prompt": "...",
  "context": "",
  "answer": "",
  "problem": "jailbreak_attempt",
  "note": "slipped past the input screen",
  "application_id": "acme",
  "created_at": "2026-06-26T10:14:33+00:00",
  "source": "human_feedback"
}
```

`label` is derived from the verdict: a `false_negative` on a danger axis → `1` (positive), a `false_positive` → `0` (negative).

---

## REST API

All routes are mounted under **`/v1/glad/feedback`** on the gateway, alongside the rest of the control plane. The active Application is resolved from `application_id` in the body/query or the `X-Geodesia-App` header (default `default`).

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/glad/feedback/schema` | Axis vocabulary + plain-language `problem → axis` map (keeps the UI in sync with the backend). |
| `POST` | `/v1/glad/feedback` | Create a flag from chat (`status=pending`). |
| `GET` | `/v1/glad/feedback` | List / filter the queue (`status`, `application_id`, `axis`, `region`, `limit`, `offset`). |
| `GET` | `/v1/glad/feedback/stats` | Pending / approved / rejected counts. |
| `POST` | `/v1/glad/feedback/{id}/review` | Curator action — approve (axis + verdict), reject, or re-open. |
| `DELETE` | `/v1/glad/feedback/{id}` | Drop a row. |
| `GET` | `/v1/glad/feedback/export` | Download the approved corpus as JSONL. |
| `GET` | `/v1/glad/feedback/bank/status` | Exemplar-bank version + approved count. |
| `POST` | `/v1/glad/feedback/retrain` | Trigger a re-train from the approved corpus — `mode: "memory"` (instant bank refresh) or `"weights"` (background trainer). |
| `GET` | `/v1/glad/feedback/retrain/status?job_id=…` | Job state + a tail of its log. |
| `GET` | `/v1/glad/feedback/retrain/jobs` | All re-train jobs. |

### Create a flag

```bash
curl -s http://localhost:8800/v1/glad/feedback \
  -H "Content-Type: application/json" \
  -H "X-Geodesia-App: acme" \
  -d '{
    "region": "answer",
    "problem": "fabricated",
    "note": "invented a citation",
    "prompt": "Who won the 1923 Paris Review prize?",
    "answer": "The 1923 Paris Review prize went to ...",
    "message_id": "msg_42",
    "session_id": "sess_7"
  }'
```

### Review a flag (curator)

```bash
curl -s http://localhost:8800/v1/glad/feedback/fb_9c1f2a7b4e0d6a18/review \
  -H "Content-Type: application/json" \
  -d '{
    "status": "approved",
    "axis": "halluc_closedbook",
    "verdict": "false_negative",
    "reviewer": "anna@acme.com"
  }'
```

To record a contrastive benign twin at review time, add `weight`, `twin_prompt` / `twin_answer`, and `attack_family` (consumed by the [v2 bank](#contrastive-safety-memory-optional)).

---

## Triggering a re-train

The two loops can be driven from the API — the fast one instantly, the slow one as a background job.

```bash
# fast loop: rebuild the episodic memory from the approved corpus, right now
curl -s http://localhost:8800/v1/glad/feedback/retrain \
  -H "Content-Type: application/json" \
  -d '{"mode": "memory", "application_id": "acme"}'
```

```bash
# slow loop: export the corpus and launch the configured trainer in the background
curl -s http://localhost:8800/v1/glad/feedback/retrain \
  -H "Content-Type: application/json" \
  -d '{"mode": "weights"}'
```

```json
{ "job_id": "rt_5f1c9a2e7b04", "mode": "weights", "n_rows": 214,
  "status": "queued", "created_at": "2026-08-05T10:02:11+00:00" }
```

Poll it, log tail included:

```bash
curl -s "http://localhost:8800/v1/glad/feedback/retrain/status?job_id=rt_5f1c9a2e7b04"
```

| `mode` | What happens | Cost |
|---|---|---|
| `memory` (default) | Bumps the bank version so the gateway rebuilds the [exemplar bank](#episodic-exemplar-bank-fast-loop) from the approved corpus on the next request. | Instant, no GPU, no subprocess |
| `weights` | Exports the approved corpus to JSONL and launches the configured trainer as a background subprocess. | A real training run |

One `weights` job runs at a time (a second returns `409`). With **no trainer configured** the corpus is still exported and the job returns `status: "prepared"` with the path — you never lose the export just because the training command was not wired yet.

| Variable | Default | Description |
|---|---|---|
| `GW_RETRAIN_CMD` | *(unset)* | The trainer command, with `{corpus}` and `{out}` placeholders. Unset ⇒ export-only (`prepared`). |
| `GW_RETRAIN_CWD` | *(unset)* | Working directory for the trainer subprocess. |
| `GW_RETRAIN_DIR` | `<db dir>/retrain` | Where corpora, logs and outputs are written. |
| `GW_RETRAIN_AUTOPROMOTE` | `1` | On a successful `weights` job, point the live detector at the new checkpoint and restart. Set `0` to keep promotion manual. |
| `GW_FEEDBACK_AUTOAPPROVE` | `off` | `on` ⇒ a flag whose plain-language problem resolves to an axis goes straight into the engine with no curator. `other` and unresolvable flags still wait in the queue. |

!!! danger "Auto-promotion is a real deployment change"
    With `GW_RETRAIN_AUTOPROMOTE` on (the default), a completed `weights` job **replaces the live detector and restarts the service**. That is the right behaviour for a self-improving deployment you control end to end; it is the wrong behaviour if a human is supposed to sign off on every model change. Turn it off for regulated deployments and promote deliberately — the choice itself is worth recording in your [FRIA](../compliance/fria.md).

---

## Security that evolves with the deployment

Everything above exists because **safety is relative**. What must be blocked is a function of the company, the sector and the internal policy — not of the model — and it moves as the product moves. A detector calibrated for the general case is a starting point, never the answer.

Three levers, three timescales, all inside your deployment:

| Lever | Mechanism | Effect |
|---|---|---|
| **Global** | Threshold change from [Policy Lens](../studio/policy-lens.md) | Immediate, simulated exactly on your own traffic *before* it is applied |
| **Surgical** | Approved correction → exemplar bank | The corrected pattern is recalled at scoring time, without retraining |
| **Structural** | Approved corpus → `retrain` → weights | The corrections are folded into the detector, deliberately |

The layering is the point: the detector's geometry is validated out-of-distribution and must not be disturbed by ad-hoc tweaks, so deployment-specific incidents are **memorised** rather than trained in — and folded into the weights only when someone decides to. That is what lets one deployment adapt to its own definition of unacceptable without each adaptation degrading everything else.

---

## Privacy & governance

- Feedback is **scoped per Application** — one tenant never sees another's flags or corpus.
- Nothing leaves the deployment: flags, the queue, the bank, and the export are all local to the gateway's own database.
- Approval is a **human-in-the-loop** gate — only a curator's explicit approval lets an incident influence scoring or training, which is exactly the human-oversight control the [Compliance Platform](../compliance/oversight.md) records.
