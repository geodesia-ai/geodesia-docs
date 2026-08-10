# Reports & Deployer Manual

Geodesia G-1 can generate legally-formatted compliance reports as PDF or DOCX files, suitable for submission to regulatory authorities, internal auditors, or enterprise compliance teams. It also generates a **Deployer Transparency Manual** — an EU AI Act Article 13-compliant instructions-for-use document.

---

## Compliance Report

**What it does.** Aggregates everything the deployment recorded — call statistics, per-axis detection rates, oversight activity, audit-chain state, FRIA status — into one report, mapped article by article against the legal frameworks you select.

Three endpoints, same request body:

| Endpoint | Returns | Use when |
|---|---|---|
| `POST /v1/glad/report` | Structured JSON | You want the data, or you are building your own document. |
| `POST /v1/glad/report/pdf` | The PDF inline | Small reports, interactive use. |
| `POST /v1/glad/report/pdf/async` | `{job_id, token}` | **Everything else.** A multi-framework render over a busy database takes minutes. |

---

### Call it

=== "curl"

    ```bash
    # structured data
    curl -s -X POST http://localhost:8080/v1/glad/report \
      -H "Content-Type: application/json" \
      -d '{
        "company_name":      "ACME Corporation",
        "company_country":   "IT",
        "model_description": "Customer Support AI",
        "applicable_laws":   ["EU_AI_ACT", "GDPR"],
        "from_date":         "2026-01-01",
        "to_date":           "2026-06-30",
        "contact_email":     "ai-compliance@acme.com",
        "language":          "en"
      }' | jq

    # PDF, inline
    curl -s -X POST http://localhost:8080/v1/glad/report/pdf \
      -H "Content-Type: application/json" \
      -d '{"company_name":"ACME Corporation","company_country":"IT",
           "applicable_laws":["EU_AI_ACT","GDPR"]}' \
      --output compliance_report.pdf
    ```

=== "Python"

    ```python
    import time, httpx

    c = httpx.Client(base_url="http://localhost:8080", timeout=300)
    body = {
        "company_name": "ACME Corporation",
        "company_country": "IT",
        "model_description": "Customer Support AI",
        "applicable_laws": ["EU_AI_ACT", "GDPR"],
        "from_date": "2026-01-01",
        "to_date": "2026-06-30",
        "contact_email": "ai-compliance@acme.com",
    }

    # Async is the safe path — a big render outlives most connection timeouts.
    job = c.post("/v1/glad/report/pdf/async", json=body).json()
    while True:
        st = c.get(f"/v1/glad/doc-jobs/{job['job_id']}/status",
                   params={"token": job["token"]}).json()
        if st["status"] == "error":
            raise RuntimeError(st.get("error", "render failed"))
        if st["status"] == "done":
            break
        time.sleep(2)

    pdf = c.get(f"/v1/glad/doc-jobs/{job['job_id']}/download", params={"token": job["token"]})
    open("compliance_report.pdf", "wb").write(pdf.content)
    ```

=== "TypeScript"

    ```ts
    const body = {
      company_name: "ACME Corporation",
      company_country: "IT",
      model_description: "Customer Support AI",
      applicable_laws: ["EU_AI_ACT", "GDPR"],
      from_date: "2026-01-01",
      to_date: "2026-06-30",
    }

    const job = await fetch("http://localhost:8080/v1/glad/report/pdf/async", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    }).then(r => r.json())

    for (;;) {
      const st = await fetch(
        `http://localhost:8080/v1/glad/doc-jobs/${job.job_id}/status?token=${job.token}`,
      ).then(r => r.json())
      if (st.status === "error") throw new Error(st.error)
      if (st.status === "done") break
      await new Promise(r => setTimeout(r, 2000))
    }

    const blob = await fetch(
      `http://localhost:8080/v1/glad/doc-jobs/${job.job_id}/download?token=${job.token}`,
    ).then(r => r.blob())
    ```

### What comes back

`POST /v1/glad/report` returns the report as structured data — a `report_id`, the period covered, the call statistics, the per-framework mapping, and the paths of any artifacts written. The `/pdf` variants return the file itself.

### Request fields

Every field has a default; a bare `{}` produces a report for the whole history under the deployment's configured identity.

| Field | Type | Default | Description |
|---|---|---|---|
| `company_name` | `string` | `"Geodesia S.R.L."` | Legal name on the cover page. |
| `company_country` | `string` | `"IT"` | ISO country code. |
| `model_description` | `string` | `"Geodesia G-1"` | The AI system being reported on. |
| `applicable_laws` | `array[string]` | *(configured set)* | Frameworks to map against. See [Regulatory Coverage](../regulatory/index.md) for valid codes. |
| `from_date` / `to_date` | `string` | — | Reporting window. Omit both for the full history. |
| `session_ids` | `array[string]` | — | Restrict to specific conversations. |
| `call_ids` | `array[string]` | — | Restrict to specific calls. |
| `fria_id` | `string` | — | Attach a FRIA dossier to the report. |
| `deployer_id` | `string` | `"default"` | Whose data to include. |
| `contact_email` | `string` | `"compliance@example.com"` | Compliance contact on the cover page. |
| `include_json` | `boolean` | `true` | Also write the JSON artifact alongside the document. |
| `language` | `string` | `"en"` | Document language. |

!!! warning "422 means there is nothing to report"
    The PDF is only rendered when there are **logged calls in the selected window** *and* **at least one applicable law**. Miss either and you get a `422` whose `detail` says exactly which — run a few chats first, or select a framework. It is not a render failure, and it should not be reported to the user as one.

### Report structure

| Section | Content |
|---|---|
| Executive summary | System overview, period covered, compliance status |
| System description | The AI system, its model, its detection capabilities |
| Regulatory mapping | Per-framework status, article by article |
| Detection statistics | Call volumes, pass/block rates, per-axis rates over the period |
| Oversight summary | Review counts, escalations, decision distribution |
| FRIA status | Linked dossier, its state and approval date |
| Audit chain | Integrity status, entry count, chain hash |
| Kill-switch history | Activation and deactivation events |
| Regulatory appendix | Article-by-article checklist |

---

## Deployer Transparency Manual

**What it does.** Generates a plain-language instructions-for-use document: what the system does, what it cannot do, how it is overseen, what is logged and for how long. This is the **EU AI Act Article 13** transparency artifact, and the Article 26(1) instructions the deployer is required to hold.

Unlike the compliance report it describes the **current** configuration, not a historical period — so it takes no dates.

=== "curl"

    ```bash
    curl -s -X POST http://localhost:8080/v1/glad/deployer-manual/pdf \
      -H "Content-Type: application/json" \
      -d '{
        "company_name":    "ACME Corporation",
        "company_country": "IT",
        "contact_email":   "ai-compliance@acme.com",
        "applicable_laws": ["EU_AI_ACT", "GDPR"],
        "retention_months": 6,
        "language":        "en"
      }' --output deployer_manual.pdf
    ```

=== "Python"

    ```python
    manual = c.post("/v1/glad/deployer-manual", json={
        "company_name": "ACME Corporation",
        "company_country": "IT",
        "contact_email": "ai-compliance@acme.com",
        "applicable_laws": ["EU_AI_ACT", "GDPR"],
    }).json()          # structured data

    pdf = c.post("/v1/glad/deployer-manual/pdf", json={
        "company_name": "ACME Corporation",
        "company_country": "IT",
        "contact_email": "ai-compliance@acme.com",
    })
    open("deployer_manual.pdf", "wb").write(pdf.content)
    ```

=== "TypeScript"

    ```ts
    const res = await fetch("http://localhost:8080/v1/glad/deployer-manual/pdf", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        company_name: "ACME Corporation",
        company_country: "IT",
        contact_email: "ai-compliance@acme.com",
      }),
    })
    const blob = await res.blob()
    ```

### Request fields

| Field | Type | Default | Description |
|---|---|---|---|
| `company_name` | `string` | `"Geodesia S.R.L."` | |
| `company_country` | `string` | `"IT"` | |
| `contact_email` | `string` | `"compliance@geodesia.ai"` | Incident and rights contact printed in the manual. |
| `safety_threshold` | `float` | `0.7` | Documented safety threshold. |
| `halluc_threshold` | `float` | `0.75` | Documented hallucination threshold. |
| `retention_months` | `integer` | `6` | Documented retention period. |
| `review_days` | `integer` | `5` | Documented review turnaround. |
| `applicable_laws` | `array[string]` | — | Frameworks the manual addresses. |
| `language` | `string` | `"en"` | |

!!! warning "The thresholds here are documentation, not configuration"
    `safety_threshold`, `halluc_threshold`, `retention_months` and `review_days` are printed into the manual — they do **not** change what the system enforces. Pass the values your deployment actually runs, or the manual will describe a system you are not operating. The live values come from the [Application policy](../studio/applications.md) and [config.yaml](../configuration/index.md).

Both endpoints also have an `/async` variant (`POST /v1/glad/deployer-manual/pdf/async`) that returns `{job_id, token}` — same [document-job flow](../studio/api-reference.md#document-jobs) as the compliance report.

### Manual contents

| Section | Content |
|---|---|
| System identity | Name, version, provider identity, last updated |
| Intended purpose | What the system is designed to do |
| Limitations | What it cannot do; failure modes |
| Detection capabilities | Plain-language description of each detection axis |
| Risk level | EU AI Act Annex III classification |
| Human oversight | Oversight procedures and escalation paths |
| Threshold configuration | The documented detection thresholds |
| Logging & retention | What is logged, for how long, under what policy |
| Rights of affected persons | How individuals exercise their rights |
| Incident reporting | How to report an incident, and to whom |

---

## Provider Identity

### GET /v1/glad/provider-identity

Returns machine-readable provider identity information — the system's "identity card" for regulatory systems.

```bash
curl http://localhost:8080/v1/glad/provider-identity
```

```json
{
  "provider": "Geodesia AI",
  "system_name": "Geodesia G-1",
  "system_version": "1.0.0",
  "model_family": "Geodesia G-1 Companion",
  "eu_ai_act_risk_class": "high",
  "eu_conformity_basis": ["Art. 9", "Art. 10", "Art. 11", "Art. 12", "Art. 13", "Art. 14"],
  "supported_frameworks": ["EU_AI_ACT", "GDPR", "ISO_42001", ...],
  "contact": "compliance@geodesia.ai"
}
```

---

## Threshold Preferences

The **deployer-level** threshold profile: four scalar thresholds that apply where no per-Application policy overrides them. Distinct from the nine-axis per-Application policy, which lives on the [control plane](../studio/control-plane-api.md).

### GET /v1/glad/threshold-prefs

**What it does.** Returns the stored profile, or the built-in defaults if that profile has never been written.

=== "curl"

    ```bash
    curl -s "http://localhost:8080/v1/glad/threshold-prefs?profile_id=default" | jq
    ```

=== "Python"

    ```python
    import httpx
    prefs = httpx.get("http://localhost:8080/v1/glad/threshold-prefs",
                      params={"profile_id": "default"}).json()
    print(prefs)
    ```

=== "TypeScript"

    ```ts
    const prefs = await fetch("http://localhost:8080/v1/glad/threshold-prefs?profile_id=default")
      .then(r => r.json())
    ```

**What comes back**

```json
{
  "profile_id": "default",
  "prompt_safety_threshold": 0.731,
  "answer_safety_threshold": 0.142,
  "halluc_threshold": 0.500,
  "combined_halluc_threshold": 0.782,
  "updated_at": "2026-06-18T09:00:00Z"
}
```

`updated_at` is absent until the profile has been written at least once.

### POST /v1/glad/threshold-prefs

**What it does.** A **partial** update — send only the thresholds you want to move. Unknown keys are ignored; a body with none of the four known keys returns **400**.

=== "curl"

    ```bash
    curl -s -X POST "http://localhost:8080/v1/glad/threshold-prefs?profile_id=default" \
      -H "Content-Type: application/json" \
      -d '{"prompt_safety_threshold": 0.80, "halluc_threshold": 0.617}' | jq
    ```

=== "Python"

    ```python
    updated = httpx.post("http://localhost:8080/v1/glad/threshold-prefs",
                         params={"profile_id": "default"},
                         json={"prompt_safety_threshold": 0.80,
                               "halluc_threshold": 0.617}).json()
    ```

=== "TypeScript"

    ```ts
    const updated = await fetch(
      "http://localhost:8080/v1/glad/threshold-prefs?profile_id=default",
      { method: "POST", headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ prompt_safety_threshold: 0.80, halluc_threshold: 0.617 }) },
    ).then(r => r.json())
    ```

**What comes back** — the complete profile after the merge.

| Field | Type | Description |
|---|---|---|
| `prompt_safety_threshold` | `float` | `[0,1]`. |
| `answer_safety_threshold` | `float` | `[0,1]`. |
| `halluc_threshold` | `float` | `[0,1]`. |
| `combined_halluc_threshold` | `float` | `[0,1]`. |

`profile_id` is a **query parameter**, not a body field, and defaults to `default`.

!!! warning "This is not the per-Application policy"
    These four thresholds are a deployer-wide profile. To change what a specific Application blocks — across all nine axes, with per-axis `block` / `annotate` / `off` — use `PUT /v1/glad/apps/{app_id}/policy`. See [Detection Thresholds](../reference/thresholds.md).

---

## Data Retention

### GET /v1/glad/retention/status

**What it does.** Reports the retention policy in force and how much data is inside, at, or past its window. Takes no parameters.

=== "curl"

    ```bash
    curl -s http://localhost:8080/v1/glad/retention/status | jq
    ```

=== "Python"

    ```python
    r = httpx.get("http://localhost:8080/v1/glad/retention/status").json()
    if r["overdue_calls"]:
        print(f"{r['overdue_calls']} records are past their retention window")
    ```

=== "TypeScript"

    ```ts
    const r = await fetch("http://localhost:8080/v1/glad/retention/status").then(r => r.json())
    if (r.overdue_calls) console.warn(`${r.overdue_calls} records past retention`)
    ```

**What comes back**

```json
{
  "generated_at": "2026-06-18T09:00:00+00:00",
  "total_calls": 48291,
  "active_calls": 48291,
  "overdue_calls": 0,
  "policy": { "default_months": 6 },
  "applicable_laws": ["EU_AI_ACT", "GDPR"],
  "tiers": {
    "standard": { "total": 48000, "archived": 0, "overdue": 0 },
    "extended": { "total": 291,   "archived": 0, "overdue": 0 }
  }
}
```

| Field | Description |
|---|---|
| `total_calls` | Retention events recorded. |
| `active_calls` | Still inside their window and not archived. |
| `overdue_calls` | Past their `expires_at` and not archived — what a GDPR erasure obligation would act on. |
| `policy` | The configured retention policy. |
| `applicable_laws` | The frameworks that policy is derived from. |
| `tiers` | Per-tier `{total, archived, overdue}`. |

Configure retention periods in [config.yaml](../configuration/index.md#retention-section).

---

## License Tokens

Customer licence tokens are managed under `/v1/glad/license-tokens/…` and documented in
[Licensing & Entitlements](../studio/licensing.md). Creating, listing and revoking a token requires
the `X-Geodesia-Admin-Key` header; validation (`GET`/`POST /v1/glad/license-tokens/validate`) does not.
