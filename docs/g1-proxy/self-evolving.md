# Self-Evolving Security

**Security is relative by definition.** What counts as an unacceptable request is not a property of a model — it is a property of *your* company, *your* sector, *your* internal policy, and it changes as your product changes. A hospital's over-refusal is a bank's minimum standard. A phrase that is an attack in a customer-support bot is the daily vocabulary of a red team.

No general-purpose safety model can encode that, and no vendor can ship it to you pre-configured. So Geodesia does not ship one fixed line: it ships a system that **learns your line from your own traffic**, under human supervision, entirely inside your perimeter.

This page is the whole loop end to end — what each part does, where it runs, what it costs, and what it deliberately refuses to do.

!!! tip "The loop can also run without a human"
    Everything below is driven by someone raising a flag. There is now a second source for the same memory: an automatic reviewer that runs on spare CPU while your machine is idle, re-scores the traffic nobody flagged against **your own description of your organisation**, and contributes only the cases where it confidently disagrees with the detector. It is off by default. See **[Personal Safety](personal-safety.md)**.

---

## The three timescales

The central design decision is that a correction does **not** go straight into the model. It lands on the timescale that matches its nature:

<div class="axis-grid">

<div class="axis-card prompt">
<div class="axis-tag">Immediate · seconds</div>
<div class="axis-name">The threshold</div>
<div class="axis-key">Policy Lens → app policy</div>
<p>The whole population sits in the wrong place. Move the line, hot-reload, effective on the next request. Simulated exactly on your own traffic <em>before</em> it is applied.</p>
</div>

<div class="axis-card context">
<div class="axis-tag">Fast · one request</div>
<div class="axis-name">Episodic memory</div>
<div class="axis-key">approved flag → exemplar bank</div>
<p>The population is fine and <em>one case</em> is not. The corrected pattern is recalled at scoring time, exactly, <strong>without retraining anything</strong>.</p>
</div>

<div class="axis-card closedbook">
<div class="axis-tag">Structural · a training run</div>
<div class="axis-name">The weights</div>
<div class="axis-key">approved corpus → re-train</div>
<p>The corrections have accumulated into a pattern worth generalising. Fold them into the detector — deliberately, when someone decides to.</p>
</div>

</div>

!!! abstract "Why the layering is the point — *memorize, don't smear the manifold*"
    The detector's geometry is validated **out of distribution**, and that validation is the product. Every ad-hoc tweak to fix one embarrassing example risks trading a measured, general capability for a local patch — and you find out months later, on traffic you cannot reproduce.

    So deployment-specific incidents are **memorised** in a non-parametric memory that sits *beside* the model and never touches its weights, and they are folded into the weights only as a deliberate, versioned act. That is what lets one deployment adapt to its own definition of unacceptable without each adaptation degrading everything else.

---

## Where each part runs

The loop spans both halves of the product, and it is worth being explicit about which process owns what:

| Stage | Runs in | Surface |
|---|---|---|
| A user or reviewer raises a flag | **G-1 Studio** (Chat, Policy Lens) or any API client | `POST /v1/glad/feedback` |
| The flag waits for a curator | **G-1 Studio** | *Feedback* workspace |
| A curator approves / rejects | **G-1 Studio** | `POST /v1/glad/feedback/{id}/review` |
| The bank is built and consulted **at scoring time** | **G-1 Proxy** (the gateway, next to the detector) | in-process, per request |
| A re-train is triggered and promoted | **G-1 Proxy** | `POST /v1/glad/feedback/retrain` |

One SQLite table (`feedback`) on the database the gateway already uses — no new datastore, and creating it never touches existing tables, so any deployment picks it up automatically.

---

## 1. Raising a flag

Every message in the chat carries a **flag** control, and it is **region-scoped**: flagging an assistant message produces `region: "answer"`, flagging a user message produces `region: "prompt"`.

The person flagging picks a **plain-language problem** — no ML jargon, ever. The system maps that choice to a *suggested* axis, which the curator can override:

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
| either | *"👍 This is benign — correct"* | `correctly_benign` | *inferred — reinforcement* |
| either | *"👍 A real threat, correctly blocked"* | `correctly_blocked` | *inferred — reinforcement* |
| either | *"Other…"* (free text) | `other` | *curator assigns* |

The flag snapshots its own context — the `prompt`, any RAG `context`, the `answer`, and the detector `scores` at flag time — so the curator sees the full picture and the corpus is self-contained. The row starts as `status: "pending"`, scoped to the active Application.

!!! tip "Praise is training data"
    `correctly_benign` and `correctly_blocked` confirm a decision that was already right: a confirmed benign becomes a **benign anchor** (keep allowing this), a confirmed threat a **danger anchor**. A loop that only ever hears about mistakes drifts, because nothing holds the correct decisions in place.

!!! note "The vocabulary is served, not hard-coded"
    `GET /v1/glad/feedback/schema` returns the axis list, the region grouping and the full `problem → axis` map. A front-end mirrors it instead of embedding it, so a checkpoint that gains an axis needs **no UI change** — the three newest axes appeared in the flag menu this way.

Corrections can also be raised from [Policy Lens](../studio/policy-lens.md) while tuning a threshold on real traffic: *Wrongly blocked* and *Should block* push into this same queue, with the axis set to the one under examination so the correction trains the right head.

---

## 2. The curator gate

Curators work the queue in the **Feedback** workspace of G-1 Studio. For each pending flag:

- **Approve** — confirm the incident and set the final **axis** and **verdict**:
    - `false_negative` — the detector *should* have flagged this and didn't. A **positive** example, and a **danger anchor** in the memory.
    - `false_positive` — the detector flagged a benign message. A **negative** example, and a **benign anchor**.
- **Reject** — noise or abuse. Rejected rows train nothing, ever.
- **Re-open** — back to `pending`.

Only **approved** rows with a resolved axis reach the export or the memory. This is the human-oversight control the [Compliance Platform](../compliance/oversight.md) records — nothing influences scoring or training without a person's explicit approval.

The queue also shows what each detector scored on the flagged axis at the time, so a curator can see *how close* the call was before ruling on it.

### The contrastive twin (optional)

The review form also accepts a **benign twin** — a superficially similar but harmless counterpart of the flagged incident — plus a cell `weight` and an `attack_family` label. With a twin present the memory reasons *contrastively*: it fires only when the query is more danger-like than the twin, which sharpens the boundary instead of merely pushing it. Omit them and the memory behaves as a plain episodic bank. This is what the [v2 bank](#the-v2-bank-contrastive-safety-memory) consumes.

### One-human-in-the-loop mode

`GW_FEEDBACK_AUTOAPPROVE=on` lets a flag whose plain-language problem resolves to an axis go **straight into the engine** with no curator. `other` and unresolvable flags still queue.

!!! danger "Understand what you are turning on"
    Auto-approve removes the only gate between an end user and the live scoring behaviour. It is right for a closed pilot where the flaggers *are* the security team. It is wrong for anything user-facing — an annoyed user who flags every refusal becomes a training signal. On a regulated deployment, leaving this `off` is part of your human-oversight evidence.

---

## 3. The fast loop — episodic memory

The approved corpus becomes a live, **non-parametric memory** consulted at scoring time. It is **opt-in and off by default**: with it disabled, no bank is built or consulted and detection is **byte-identical**.

At scoring time the detector embeds the current input onto its manifold and compares it with the stored exemplars for the axes of that region:

- a close match to a **false-negative** exemplar **raises** the axis probability — *we missed this before, never again*;
- a close match to a **false-positive** exemplar **suppresses** it — *a known benign pattern we over-flagged*;
- **a danger match always wins over a benign one** — the memory never suppresses something that also resembles a known dangerous case.

The match is a cosine similarity on the unit sphere with a **high floor** (`τ`, default `0.88`). That is deliberate: this is **exact-pattern recall**, not fuzzy generalisation. Generalising is the slow loop's job, and confusing the two is how a memory turns into an unaudited second model.

**Per model, per tenant.** The bank is built from the shared corpus using G1-Hummingbird's own embedding function. A row whose axis the model does not have is simply skipped. Banks are rebuilt only when the corpus version changes, so a request pays nothing while the corpus is stable.

**Per Application.** `policy.feedback_learning` opts a single Application in without flipping the global default, so one tenant can learn from its own corrections while another stays byte-identical.

When a bank match moves a score, the affected axis carries an `exemplar_match` annotation in the response — the contribution is always auditable:

```json
{
  "axis_energy": {
    "jailbreak": {
      "p_detector": 0.94,
      "threshold": 0.9997,
      "flag": true,
      "exemplar_match": { "verdict": "false_negative", "sim": 0.93 }
    }
  }
}
```

### The v2 bank (Contrastive Safety Memory)

An opt-in second-generation memory that ships **alongside** v1 — the live path stays byte-identical until you switch it on. Each thing it changes exists because of a specific way v1 fails:

| v1 weakness | What v2 does instead |
|---|---|
| A one-sided raise/suppress nudge drifts into over-refusal | **Contrastive cells**: the danger is stored paired with a benign twin, and the cell fires only if the query is more danger-like than the twin by a margin |
| Nearest-neighbour on a single exemplar is fragile to one mislabelled row | A **vote over k neighbours** (distance-weighted), plus a **credibility gate** — how much the neighbourhood agrees with the winning class |
| `τ = 0.88` is a magic number with no false-positive guarantee | A **split-conformal threshold per (region, axis)**, calibrated on benign serving traffic, with a finite-sample bound on the false-positive lift |
| Rewriting `p_detector` and then re-flagging against the old conformal threshold breaks that threshold's guarantee | v2 is a **separate OR-term**: it raises its own `flag` and **never rewrites** `p_detector` |
| One pooled vector loses compositional intent | **Per-span matching** (MaxSim over spans) plus a lexical surface-form channel |

Each cell keeps the originating `feedback_id`, which is what makes provenance, audit and **GDPR deletion of a single incident** possible.

### Configuration

| Variable | Default | Description |
|---|---|---|
| `GW_FEEDBACK_BANK` | `off` | Master switch. `on` builds and consults the per-model bank; `off` → never built, zero overhead, byte-identical detection. |
| `GW_BANK_V2` | `off` | Use the Contrastive Safety Memory instead of v1. Requires `GW_FEEDBACK_BANK=on`. |
| `GW_FEEDBACK_BANK_TAU` | `0.88` | v1 cosine floor below which an exemplar is ignored. Higher = stricter exact-pattern recall. |
| `GW_FEEDBACK_BANK_GAIN` | `1.0` | v1: how hard a perfect match pushes the probability. |
| `GW_BANK_ALPHA` | `0.01` | v2 conformal false-positive-lift budget per region (Bonferroni-split across axes). |
| `GW_BANK_K` | `5` | v2 neighbours in the distance-weighted vote. |
| `GW_BANK_CRED_MIN` | `0.6` | v2 minimum neighbourhood agreement required to trust a match. |
| `GW_BANK_CONTRAST_MIN` | `0.05` | v2 margin by which the danger cell must beat its benign twin. |
| `GW_BANK_BMAX` | `2000` | v2 per-(region, axis) memory cap; least-useful cells are evicted first. |

```bash
# gateway with the contrastive memory on
GW_FEEDBACK_BANK=on GW_BANK_V2=on \
  python -m glad_minimal.gateway.geodesia_gateway --host 0.0.0.0 --port 8800 ...
```

---

## 4. The slow loop — into the weights

The approved corpus downloads as **JSONL**, one training example per line, ready to aggregate across Applications (and, for a vendor, across customers):

```bash
curl -s "http://localhost:8800/v1/glad/feedback/export?status=approved&application_id=acme" \
  -o feedback_approved_acme.jsonl
```

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

`label` is derived from the verdict: a `false_negative` on a danger axis → `1`, a `false_positive` → `0`.

### Triggering it

```bash
# fast loop — rebuild the episodic memory from the approved corpus, right now
curl -s http://localhost:8800/v1/glad/feedback/retrain \
  -H "Content-Type: application/json" -d '{"mode": "memory", "application_id": "acme"}'

# slow loop — export the corpus and launch the configured trainer in the background
curl -s http://localhost:8800/v1/glad/feedback/retrain \
  -H "Content-Type: application/json" -d '{"mode": "weights"}'
```

```json
{ "job_id": "rt_5f1c9a2e7b04", "mode": "weights", "n_rows": 214,
  "status": "queued", "created_at": "2026-08-05T10:02:11+00:00" }
```

| `mode` | What happens | Cost |
|---|---|---|
| `memory` (default) | Bumps the bank version so the gateway rebuilds the [episodic memory](#3-the-fast-loop-episodic-memory) on the next request. | Instant, no GPU, no subprocess |
| `weights` | Exports the approved corpus to JSONL and launches the configured trainer as a background subprocess. | A real training run |

One `weights` job runs at a time (a second returns `409`). Poll it — the log tail comes with the status:

```bash
curl -s "http://localhost:8800/v1/glad/feedback/retrain/status?job_id=rt_5f1c9a2e7b04"
```

With **no trainer configured** the corpus is still exported and the job returns `status: "prepared"` with the path. You never lose the export because the training command was not wired yet.

### Promotion

When a `weights` job finishes successfully and auto-promotion is on (**the default**), the gateway points the live detector at the freshly trained checkpoint. If the served checkpoint path is a symlink, the swap is **atomic** and survives the restart; if it is not, the monitor **hot-reloads in process** — the new model serves immediately, with no downtime, until the next real restart.

| Variable | Default | Description |
|---|---|---|
| `GW_RETRAIN_CMD` | *(unset)* | Trainer command, with `{corpus}` and `{out}` placeholders. Unset ⇒ export-only (`prepared`). |
| `GW_RETRAIN_CWD` | *(unset)* | Working directory for the trainer subprocess. |
| `GW_RETRAIN_DIR` | `<db dir>/retrain` | Where corpora, logs and outputs are written. |
| `GW_RETRAIN_AUTOPROMOTE` | `1` | Promote the new checkpoint automatically on success. |
| `GW_RETRAIN_RESTART` | `1` | Restart the service after repointing. `0` ⇒ hot-reload only. |
| `GW_RETRAIN_OUT_CKPT` | *(newest `*.pt`)* | Explicit checkpoint template, e.g. `{out}/model.pt`. |

!!! danger "Auto-promotion is a real deployment change"
    With `GW_RETRAIN_AUTOPROMOTE` on, a completed job **replaces the live detector**. The UI is *warned*, not asked — there is no yes/no gate.

    That is the right behaviour for a self-improving deployment you control end to end. It is the wrong behaviour when a human is supposed to sign off on every model change. **Set it to `0` for regulated deployments** and promote deliberately — and record which way you set it, because "how does the model change in production" is a question your [FRIA](../compliance/fria.md) has to answer.

---

## What the loop does not do

Being explicit about the limits is what makes the rest trustworthy.

- **It does not train on your traffic by default.** Nothing is learned from a request unless a human flagged it *and* a curator approved it. Ordinary traffic contributes nothing.
- **It does not learn silently.** The bank is off by default; when on, every score it moves is annotated in the response, and every threshold change is a versioned policy write in the audit trail.
- **It does not send anything anywhere.** Flags, queue, memory, export and re-train are all local to the deployment's own database and GPU. Feedback is scoped per Application — one tenant never sees another's corpus.
- **It does not generalise from one example.** The fast loop is exact-pattern recall with a high similarity floor, by design. If you want generalisation, that is the slow loop, and it is a decision someone makes.
- **It does not silently reshape the detector's validated geometry.** That is the whole reason there are three timescales instead of one.

---

## REST API

All routes are mounted under **`/v1/glad/feedback`** on **G1-Proxy** — reached as `/gw/v1/glad/feedback/…` through the unified port. The Application is resolved from `application_id` in the body/query or the `X-Geodesia-App` header (default `default`).

### Create a flag

**What it does.** Records that a served turn was judged wrong, in plain language. It lands as `status: "pending"` for a curator, unless `GW_FEEDBACK_AUTOAPPROVE=on` and the problem maps cleanly to an axis — then it goes straight into the engine.

=== "curl"

    ```bash
    curl -s http://localhost:8080/gw/v1/glad/feedback \
      -H "Content-Type: application/json" \
      -H "X-Geodesia-App: acme" \
      -d '{
        "region":     "answer",
        "problem":    "fabricated",
        "note":       "invented a citation",
        "prompt":     "Who won the 1923 Paris Review prize?",
        "answer":     "The 1923 Paris Review prize went to …",
        "message_id": "msg_42",
        "session_id": "sess_7"
      }'
    ```

=== "Python"

    ```python
    import httpx

    c = httpx.Client(base_url="http://localhost:8080/gw",
                     headers={"X-Geodesia-App": "acme"}, timeout=30)

    # Read the vocabulary instead of hard-coding it — axes can be added per deployment.
    schema = c.get("/v1/glad/feedback/schema").json()
    print([p["key"] for p in schema["problems"]])

    flag = c.post("/v1/glad/feedback", json={
        "region":  "answer",
        "problem": "fabricated",
        "note":    "invented a citation",
        "prompt":  "Who won the 1923 Paris Review prize?",
        "answer":  "The 1923 Paris Review prize went to …",
        "session_id": "sess_7",
    }).json()
    print(flag["id"], flag["status"])
    ```

=== "TypeScript"

    ```ts
    const H = { "Content-Type": "application/json", "X-Geodesia-App": "acme" }
    const base = "http://localhost:8080/gw"

    const schema = await fetch(`${base}/v1/glad/feedback/schema`, { headers: H }).then(r => r.json())
    console.log(schema.problems.map((p: any) => p.key))

    const flag = await fetch(`${base}/v1/glad/feedback`, {
      method: "POST",
      headers: H,
      body: JSON.stringify({
        region: "answer",
        problem: "fabricated",
        note: "invented a citation",
        prompt: "Who won the 1923 Paris Review prize?",
        answer: "The 1923 Paris Review prize went to …",
        session_id: "sess_7",
      }),
    }).then(r => r.json())
    console.log(flag.id, flag.status)
    ```

**What comes back** — the stored row, with its generated id and current `status`.

#### Request fields

| Field | Type | Required | Description |
|---|---|---|---|
| `region` | `string` | ✅ | `prompt` or `answer` — which side of the turn was wrong. |
| `problem` | `string` | — | A plain-language problem key from `/schema`. Mapped to an axis server-side. |
| `axis` | `string` | — | Name the axis directly. **Overrides** the problem→axis map. For API callers who know the vocabulary. |
| `verdict` | `string` | — | `false_negative` (should have fired) or `false_positive` (fired wrongly). |
| `note` | `string` | — | Free text for the curator. |
| `prompt` / `context` / `answer` | `string` | — | The turn itself. Supply them so the correction can be replayed and, later, trained on. |
| `message_id` / `session_id` | `string` | — | Link back to the served turn. |
| `application_id` | `string` | — | Same as the `X-Geodesia-App` header. |
| `scores` | `object` | — | The detection payload the turn was served with, so the curator sees what the detector thought at the time. |

!!! tip "Read `/schema`, don't hard-code axes"
    `GET /v1/glad/feedback/schema` returns `{axes, prompt_axes, answer_axes, problems, problem_to_axis, verdicts, regions}`. A deployment can ship extra axes; a client that reads the schema keeps working, one with a hard-coded list quietly drops them.

### Review a flag (curator)

**What it does.** Approves, rejects or re-opens a flag. Approving is what makes a correction real: it names the axis, states which way the detector was wrong, and — optionally — records a **contrastive benign twin**, the near-identical harmless case that must *not* flip.

=== "curl"

    ```bash
    curl -s http://localhost:8080/gw/v1/glad/feedback/fb_9c1f2a7b4e0d6a18/review \
      -H "Content-Type: application/json" \
      -d '{
        "status":   "approved",
        "axis":     "halluc_closedbook",
        "verdict":  "false_negative",
        "reviewer": "anna@acme.com"
      }'
    ```

=== "Python"

    ```python
    c.post("/v1/glad/feedback/fb_9c1f2a7b4e0d6a18/review", json={
        "status": "approved",
        "axis": "halluc_closedbook",
        "verdict": "false_negative",
        "reviewer": "anna@acme.com",
        # contrastive twin — the benign case that must NOT flip
        "twin_prompt": "Who won the 1923 Nobel Prize in Literature?",
        "twin_answer": "W. B. Yeats.",
        "attack_family": "fabricated_award",
        "weight": 1.0,
    })
    ```

=== "TypeScript"

    ```ts
    await fetch(`${base}/v1/glad/feedback/fb_9c1f2a7b4e0d6a18/review`, {
      method: "POST",
      headers: H,
      body: JSON.stringify({
        status: "approved",
        axis: "halluc_closedbook",
        verdict: "false_negative",
        reviewer: "anna@acme.com",
      }),
    })
    ```

| Field | Type | Required | Description |
|---|---|---|---|
| `status` | `string` | ✅ | `approved` \| `rejected` \| `pending`. |
| `axis` | `string` | — | The axis this correction belongs to. |
| `verdict` | `string` | — | `false_negative` \| `false_positive`. |
| `reviewer` | `string` | — | Who decided. |
| `note` | `string` | — | Curator note. |
| `weight` | `float` | — | How strongly this exemplar should count. |
| `twin_prompt` / `twin_answer` | `string` | — | The contrastive benign twin. |
| `attack_family` | `string` | — | Groups related corrections. |

### Push a correction into the engine

**What it does.** `memory` refreshes the episodic exemplar bank and takes effect on the **next request** — no restart, no training. `weights` exports the approved corpus and launches the configured trainer as a subprocess; poll the returned job.

=== "curl"

    ```bash
    # instant: refresh the exemplar bank
    curl -s -X POST http://localhost:8080/gw/v1/glad/feedback/retrain \
      -H "Content-Type: application/json" \
      -d '{"mode": "memory", "application_id": "acme"}'

    # heavy: export corpus + launch the trainer
    JOB=$(curl -s -X POST http://localhost:8080/gw/v1/glad/feedback/retrain \
      -H "Content-Type: application/json" \
      -d '{"mode": "weights"}' | jq -r .job_id)

    curl -s "http://localhost:8080/gw/v1/glad/feedback/retrain/status?job_id=$JOB" | jq
    ```

=== "Python"

    ```python
    import time

    job = c.post("/v1/glad/feedback/retrain", json={"mode": "weights"}).json()
    while True:
        st = c.get("/v1/glad/feedback/retrain/status", params={"job_id": job["job_id"]}).json()
        print(st["status"], st.get("log_tail", "")[-200:])
        if st["status"] in ("completed", "failed"):
            break
        time.sleep(10)
    ```

=== "TypeScript"

    ```ts
    const job = await fetch(`${base}/v1/glad/feedback/retrain`, {
      method: "POST", headers: H, body: JSON.stringify({ mode: "weights" }),
    }).then(r => r.json())

    for (;;) {
      const st = await fetch(
        `${base}/v1/glad/feedback/retrain/status?job_id=${encodeURIComponent(job.job_id)}`,
        { headers: H },
      ).then(r => r.json())
      if (st.status === "completed" || st.status === "failed") break
      await new Promise(r => setTimeout(r, 10_000))
    }
    ```

### Full route list

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v1/glad/feedback/schema` | Axis vocabulary + plain-language `problem → axis` map. |
| `POST` | `/v1/glad/feedback` | Create a flag. |
| `GET` | `/v1/glad/feedback` | List / filter the queue: `status`, `application_id`, `axis`, `region`, `limit` (≤ 1000), `offset`. |
| `GET` | `/v1/glad/feedback/stats` | Pending / approved / rejected / total counts. |
| `POST` | `/v1/glad/feedback/{id}/review` | Curator action. |
| `DELETE` | `/v1/glad/feedback/{id}` | Drop a row. |
| `GET` | `/v1/glad/feedback/export` | The decided corpus as JSONL. Defaults to `status=approved`. |
| `GET` | `/v1/glad/feedback/bank/status` | Exemplar-bank version + approved count. |
| `POST` | `/v1/glad/feedback/retrain` | `{mode: "memory" \| "weights", application_id?}`. |
| `GET` | `/v1/glad/feedback/retrain/status?job_id=…` | Job state + log tail. |
| `GET` | `/v1/glad/feedback/retrain/jobs` | All re-train jobs. |
| `GET` | `/v1/glad/feedback/auto/status` | Idle-judge state: installed, running, queue depths, disagreements, promotions. |
| `GET` `PUT` | `/v1/glad/feedback/auto/config` | Read / patch the idle-judge configuration. |
| `GET` | `/v1/glad/feedback/auto/prompt-preview?axis=…` | **Read-only** — the exact prompt the judge will see. |
| `GET` | `/v1/glad/feedback/auto/items?state=…&limit=…` | What the judge queued, scored or promoted. |

---

## See also

- [Policy Lens](../studio/policy-lens.md) — the global lever, with the counterfactual computed before you apply it
- [Detection Axes](detection-axes.md) — the nine axes a correction can be attributed to
- [Human Oversight](../compliance/oversight.md) — how curator approval becomes compliance evidence
