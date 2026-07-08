# Reports & Deployer Manual

Geodesia G-1 can generate legally-formatted compliance reports as PDF or DOCX files, suitable for submission to regulatory authorities, internal auditors, or enterprise compliance teams. It also generates a **Deployer Transparency Manual** — an EU AI Act Article 13-compliant instructions-for-use document.

---

## Compliance Report

### POST /v1/glad/report

Generate a comprehensive compliance audit report covering a specified time window.

```bash
curl -X POST http://localhost:8199/v1/glad/report \
  -H "Content-Type: application/json" \
  -d '{
    "deployer_id": "acme-corp",
    "regulatory_framework": ["EU_AI_ACT", "GDPR"],
    "period_start": "2026-01-01T00:00:00Z",
    "period_end": "2026-06-30T23:59:59Z",
    "fmt": "pdf",
    "client_info": {
      "company_name": "ACME Corporation",
      "system_name": "Customer Support AI",
      "responsible_person": "Maria Rossi, Chief AI Officer",
      "contact_email": "ai-compliance@acme.com"
    }
  }' \
  --output compliance_report_h1_2026.pdf
```

#### Request Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `deployer_id` | `string` | ✅ | Which deployer's data to include |
| `regulatory_framework` | `array[string]` | ✅ | Which frameworks to report on. At least one required. See [Regulatory Coverage](../regulatory/index.md) for valid codes. |
| `period_start` | `string` | ✅ | ISO 8601 start of the reporting period |
| `period_end` | `string` | ✅ | ISO 8601 end of the reporting period |
| `fmt` | `string` | — | Output format: `"pdf"` (default) or `"docx"` |
| `client_info` | `object` | — | Deployer information for the cover page (see below) |
| `include_fria_summary` | `boolean` | — | Include a summary of active FRIA dossiers. Default `true`. |
| `include_chain_snapshot` | `boolean` | — | Include an audit chain hash snapshot. Default `true`. |
| `include_oversight_summary` | `boolean` | — | Include human oversight statistics. Default `true`. |
| `include_raw_call_log` | `boolean` | — | Include per-call detection statistics (may be large). Default `false`. |
| `include_threshold_history` | `boolean` | — | Include a log of threshold changes during the period. Default `true`. |
| `watermark_report` | `boolean` | — | Embed an HMAC watermark in the report PDF. Default `true`. |

#### `client_info` Fields

| Field | Description |
|---|---|
| `company_name` | Legal name of the deploying organization |
| `system_name` | Name of the AI system being reported on |
| `responsible_person` | Full name and title of the AI Act Article 26 deployer representative |
| `contact_email` | Compliance contact email |
| `deployment_date` | When the system was first put into production |
| `address` | Registered address of the deploying organization |
| `eu_representative` | EU representative contact (required for non-EU deployers) |

---

### Report Structure

A generated compliance report includes the following sections:

| Section | Content |
|---|---|
| Executive Summary | System overview, deployment period, compliance status summary |
| System Description | Technical description of the AI system, model specifications, detection capabilities |
| Regulatory Mapping | Per-framework compliance status table |
| Detection Statistics | Call volumes, pass/block rates, per-axis detection rates over the reporting period |
| Oversight Summary | Human review statistics, escalation rates, decision distribution |
| FRIA Status | Active FRIA dossiers — status, completion, approval dates |
| Audit Chain | Chain integrity status, entry count, chain hash |
| Threshold History | All threshold changes during the period with timestamps and responsible parties |
| Kill Switch History | Activation/deactivation events |
| Incident Log | Calls escalated to oversight due to anomalous scores |
| Recommendations | Gaps identified and recommended actions |
| Regulatory Appendix | Article-by-article compliance checklist |

---

## Deployer Transparency Manual

### POST /v1/glad/deployer-manual

Generate a **Deployer Transparency Manual** — a plain-language document explaining how the AI system works, what it can and cannot do, and how to monitor it. This satisfies **EU AI Act Article 13** (transparency obligations) and **Article 26(1)** (instructions for use).

```bash
curl -X POST http://localhost:8199/v1/glad/deployer-manual \
  -H "Content-Type: application/json" \
  -d '{
    "deployer_id": "acme-corp",
    "fmt": "pdf",
    "client_info": {
      "company_name": "ACME Corporation",
      "system_name": "Customer Support AI"
    }
  }' \
  --output deployer_manual.pdf
```

#### Request Fields

Same as the compliance report, minus the `period_start`/`period_end` fields. The manual describes the current system state rather than a historical period.

---

### Manual Contents

| Section | Content |
|---|---|
| System Identity | Name, version, provider identity, last updated |
| Intended Purpose | What the system is designed to do and the use cases it is approved for |
| Limitations | What the system cannot do; failure modes; known false positive/negative rates |
| Detection Capabilities | Plain-language description of each detection axis |
| Risk Level | EU AI Act Annex III classification |
| Human Oversight | Oversight procedures; how to enable or escalate reviews |
| Threshold Configuration | Current detection thresholds with explanation |
| Logging & Data Retention | What is logged, for how long, and under what policy |
| Rights of Affected Persons | How individuals can exercise rights under GDPR and equivalent laws |
| Incident Reporting | How to report incidents; contact information |
| Change Management | How updates to the system are notified to deployers |

---

## Provider Identity

### GET /v1/glad/provider-identity

Returns machine-readable provider identity information — the system's "identity card" for regulatory systems.

```bash
curl http://localhost:8199/v1/glad/provider-identity
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

### GET /v1/glad/threshold-prefs

Retrieve the currently stored detection threshold preferences.

```bash
curl "http://localhost:8199/v1/glad/threshold-prefs?deployer_id=acme-corp"
```

### POST /v1/glad/threshold-prefs

Update threshold preferences. Changes are logged to the audit chain.

```bash
curl -X POST http://localhost:8199/v1/glad/threshold-prefs \
  -H "Content-Type: application/json" \
  -d '{
    "deployer_id": "acme-corp",
    "prompt_safety": 0.35,
    "answer_safety": 0.50,
    "halluc": 0.617,
    "combined_halluc": 0.617,
    "changed_by": "maria.rossi@acme.com",
    "reason": "Reducing false positives on corporate knowledge base queries"
  }'
```

| Field | Type | Description |
|---|---|---|
| `deployer_id` | `string` | Deployer to update thresholds for |
| `prompt_safety` | `float` | Prompt safety threshold [0–1] |
| `answer_safety` | `float` | Answer safety threshold [0–1] |
| `halluc` | `float` | Hallucination threshold [0–1] |
| `combined_halluc` | `float` | Combined hallucination threshold [0–1] |
| `changed_by` | `string` | Identity of the person making the change (logged) |
| `reason` | `string` | Reason for the change (logged) |

---

## Data Retention

### GET /v1/glad/retention

Returns the current data retention policy and statistics.

```bash
curl http://localhost:8199/v1/glad/retention
```

```json
{
  "policy": {
    "call_log_days": 365,
    "oversight_days": 730,
    "fria_days": 3650,
    "chain_days": 3650
  },
  "statistics": {
    "oldest_call": "2026-01-01T00:00:00Z",
    "total_calls": 48291,
    "calls_eligible_for_deletion": 0,
    "storage_used_mb": 142.8
  }
}
```

Configure retention periods in [config.yaml](../configuration/index.md#retention-section).

---

## License Tokens

### POST /v1/glad/license-tokens/source-bundle

Used internally by the deployment toolchain to obtain a signed download URL for the encrypted source bundle. Requires a valid `GLAD_LICENSE_TOKEN` in the request headers.

This endpoint is intended for automated deployment scripts, not end-user applications.
