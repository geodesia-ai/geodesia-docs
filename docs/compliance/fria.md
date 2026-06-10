# FRIA — Fundamental Rights Impact Assessment

The Fundamental Rights Impact Assessment (FRIA) is mandated by **EU AI Act Article 27** for deployers of high-risk AI systems. Geodesia G-1 provides a full dossier management API: create, edit, review, approve, archive, and export FRIA documents as legally-ready PDF or DOCX bundles.

---

## What Is a FRIA?

A FRIA is a structured document that assesses the impact of deploying an AI system on the fundamental rights of individuals affected by it. It must be completed **before deployment** of high-risk AI systems and kept current throughout the system's operational life.

The FRIA dossier in Geodesia G-1 covers:

- The AI system being deployed and its purpose
- Categories of individuals affected
- Fundamental rights that could be impacted
- Probability and severity of each impact
- Mitigating measures taken
- Residual risk assessment
- Human oversight procedures
- Responsible persons and sign-off

---

## Endpoints

### POST /v1/glad/fria

Create a new FRIA dossier.

```bash
curl -X POST http://localhost:8199/v1/glad/fria \
  -H "Content-Type: application/json" \
  -d '{
    "deployer_id": "acme-corp",
    "system_name": "Customer Support AI",
    "purpose": "Automated first-level customer support and query routing",
    "use_case_category": "customer_service",
    "risk_level": "high",
    "applicable_laws": ["EU_AI_ACT", "GDPR"]
  }'
```

#### Request Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `deployer_id` | `string` | ✅ | Unique identifier for your organization or deployment unit. Used to scope queries. |
| `system_name` | `string` | ✅ | Human-readable name of the AI system being assessed. |
| `purpose` | `string` | ✅ | Description of the system's intended purpose and use case. |
| `use_case_category` | `string` | — | Category from the EU AI Act Annex III list (e.g., `"biometric"`, `"critical_infrastructure"`, `"education"`, `"employment"`, `"essential_services"`, `"law_enforcement"`, `"migration"`, `"justice"`, `"customer_service"`, `"other_high_risk"`). |
| `risk_level` | `string` | — | `"low"`, `"limited"`, `"high"`, or `"unacceptable"`. Determines which sections of the dossier are required. |
| `affected_categories` | `array[string]` | — | Categories of individuals who may be affected (e.g., `["customers", "employees", "minors"]`). |
| `applicable_laws` | `array[string]` | — | Which legal frameworks this FRIA covers. Valid values: see [Regulatory Coverage](../regulatory/index.md). |
| `description` | `string` | — | Detailed description of how the system works and its deployment context. |
| `data_types` | `array[string]` | — | Types of personal data processed (e.g., `["name", "email", "conversation_history"]`). |
| `responsible_person` | `string` | — | Full name and title of the person responsible for this assessment. |
| `contact_email` | `string` | — | Contact email for the FRIA dossier. |

#### Response

```json
{
  "id": "fria_abc123",
  "deployer_id": "acme-corp",
  "system_name": "Customer Support AI",
  "status": "draft",
  "created_at": "2026-06-10T10:00:00Z",
  "updated_at": "2026-06-10T10:00:00Z",
  "sections": { ... },
  "completion_percentage": 12
}
```

| Field | Description |
|---|---|
| `id` | Unique dossier identifier. Use this in all subsequent calls. |
| `status` | Dossier lifecycle status. See [Status Values](#status-values). |
| `sections` | Prefilled content for each FRIA section based on the system's active detection configuration. |
| `completion_percentage` | How much of the dossier is filled in (0–100). |

---

### GET /v1/glad/fria

List all FRIA dossiers for a deployer.

```bash
curl "http://localhost:8199/v1/glad/fria?deployer_id=acme-corp"
```

#### Query Parameters

| Parameter | Description |
|---|---|
| `deployer_id` | Filter to a specific deployer. |
| `status` | Filter by lifecycle status: `"draft"`, `"submitted"`, `"approved"`, `"archived"`. |
| `limit` | Maximum results to return. Default 20. |
| `offset` | Pagination offset. Default 0. |

---

### GET /v1/glad/fria/{id}

Retrieve a single FRIA dossier by ID.

```bash
curl http://localhost:8199/v1/glad/fria/fria_abc123
```

---

### PUT /v1/glad/fria/{id}

Update a FRIA dossier. Accepts any subset of the original creation fields plus section-specific content.

```bash
curl -X PUT http://localhost:8199/v1/glad/fria/fria_abc123 \
  -H "Content-Type: application/json" \
  -d '{
    "sections": {
      "impact_assessment": {
        "right_to_privacy": {
          "probability": "medium",
          "severity": "low",
          "mitigation": "Data is pseudonymized and not stored beyond the session."
        }
      }
    }
  }'
```

---

### POST /v1/glad/fria/{id}/approve

Move a dossier from `"submitted"` to `"approved"`. Records the approver identity and timestamp.

```bash
curl -X POST http://localhost:8199/v1/glad/fria/fria_abc123/approve \
  -H "Content-Type: application/json" \
  -d '{"approver": "Maria Rossi", "notes": "Approved after legal review on 2026-06-10"}'
```

---

### POST /v1/glad/fria/{id}/archive

Archive a dossier. Archived dossiers remain queryable but cannot be modified.

```bash
curl -X POST http://localhost:8199/v1/glad/fria/fria_abc123/archive
```

---

### POST /v1/glad/fria/{id}/evidence

Attach supporting evidence to a FRIA dossier. Evidence is referenced by the audit chain.

```bash
curl -X POST http://localhost:8199/v1/glad/fria/fria_abc123/evidence \
  -H "Content-Type: application/json" \
  -d '{
    "type": "technical_documentation",
    "title": "Model Accuracy Report Q2 2026",
    "content": "...",
    "file_ref": "reports/accuracy_q2_2026.pdf"
  }'
```

---

### POST /v1/glad/fria/{id}/runtime-evidence

Attach runtime evidence — detection statistics computed from live calls.

```bash
curl -X POST "http://localhost:8199/v1/glad/fria/fria_abc123/runtime-evidence?deployer_id=acme-corp"
```

This call queries the call log database for the deployer and automatically populates:
- Total calls processed
- Blocked prompt count and rate
- Hallucination detection rate
- Safety flag rate
- Average detection scores
- Time range covered

---

### GET /v1/glad/fria/{id}/export

Export the FRIA dossier as a formatted document.

```bash
# Export as PDF
curl "http://localhost:8199/v1/glad/fria/fria_abc123/export?fmt=pdf" \
  --output fria_report.pdf

# Export as DOCX
curl "http://localhost:8199/v1/glad/fria/fria_abc123/export?fmt=docx" \
  --output fria_report.docx

# Export as JSON
curl "http://localhost:8199/v1/glad/fria/fria_abc123/export?fmt=json" \
  --output fria_data.json
```

#### Query Parameters

| Parameter | Default | Description |
|---|---|---|
| `fmt` | `"pdf"` | Output format: `"pdf"`, `"docx"`, or `"json"`. |
| `include_evidence` | `true` | Include attached evidence documents in the bundle. |
| `include_runtime_stats` | `true` | Include automatically computed runtime statistics. |

The exported PDF and DOCX documents include:
- Cover page with system identity, deployer info, dates, and status
- Summary of the AI system and its risk classification
- All impact assessment sections
- Mitigation measures
- Approval signature block (if approved)
- Attached evidence index
- Runtime statistics (call counts, detection rates)
- Regulatory mapping appendix

---

### GET /v1/glad/fria/auto-prefill

Pre-fills a FRIA template using the system's active configuration — including loaded model capabilities, active detection axes, thresholds, and recent runtime data. Useful for starting a new dossier with accurate technical details.

```bash
curl "http://localhost:8199/v1/glad/fria/auto-prefill?deployer_id=acme-corp"
```

---

## Status Values

| Status | Description |
|---|---|
| `draft` | Dossier is in progress and can be freely edited. |
| `submitted` | Dossier has been submitted for review. Editing is restricted to designated reviewers. |
| `approved` | Dossier has been reviewed and approved. Immutable except for evidence attachments. |
| `archived` | Dossier has been retired. Read-only. |

---

## FRIA Section Reference

A complete FRIA dossier consists of the following sections:

| Section | Description |
|---|---|
| `system_description` | What the AI system does, how it makes decisions, and its intended audience |
| `deployment_context` | Where and how the system is deployed; organizational context |
| `affected_persons` | Categories of individuals subject to the system's decisions or outputs |
| `data_processing` | Types of personal data processed; data flows; retention |
| `impact_assessment` | Per-right assessment: probability × severity matrix for each fundamental right |
| `mitigation_measures` | Technical and procedural safeguards for each identified risk |
| `residual_risk` | Risks remaining after mitigation; acceptable risk justification |
| `human_oversight_plan` | How human reviewers are integrated; escalation paths |
| `testing_validation` | Evidence of testing and accuracy validation |
| `monitoring_plan` | How the system will be monitored post-deployment |
| `review_schedule` | How often the FRIA will be reviewed and updated |
| `sign_off` | Responsible person name, role, organization, date |

---

## EU AI Act Article Mapping

| FRIA Section | Article |
|---|---|
| System description | Art. 9 (Risk management), Art. 11 (Technical documentation) |
| Deployment context | Art. 26 (Deployer obligations) |
| Affected persons | Art. 27 §1(a) |
| Data processing | Art. 10 (Data governance), GDPR Chapter II |
| Impact assessment | Art. 27 §1(b) |
| Mitigation measures | Art. 27 §1(d) |
| Human oversight plan | Art. 14 (Human oversight) |
| Sign-off | Art. 27 §2 |
