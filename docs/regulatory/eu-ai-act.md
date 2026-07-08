# EU AI Act — Article Mapping

The **EU Artificial Intelligence Act** (Regulation EU 2024/1689) entered into force on 1 August 2024. This page maps each relevant article to the specific Geodesia G-1 feature or API that satisfies it.

---

## Applicability

If you are deploying an AI system that falls under **Annex III** of the EU AI Act — which includes systems used in education, employment, essential services, law enforcement, biometrics, migration, or the administration of justice — you are a **deployer of a high-risk AI system** and are subject to Articles 25–30 and 70–73.

If you are simply embedding a general-purpose LLM (like Mistral or Gemma) into a product, the obligations depend on whether the **specific application** meets Annex III criteria.

Geodesia G-1 was built assuming the most demanding scenario: Annex III high-risk deployment.

---

## Chapter-by-Chapter Coverage

### Chapter II — Prohibited AI Practices (Articles 5–6)

Article 5 prohibits certain AI applications (social scoring, real-time biometric surveillance in public spaces, subliminal manipulation). Geodesia G-1 does not facilitate these use cases. The **jailbreak detection axis** (`jailbreak`) can help prevent your system from being used to attempt such applications.

---

### Chapter III — High-Risk AI Systems

#### Article 9 — Risk Management System

> Deployers must establish a risk management system throughout the AI lifecycle.

| Requirement | Geodesia G-1 Feature |
|---|---|
| Identify and analyze known and foreseeable risks | [Detection axes](../gateway/detection-axes.md) — 5-axis risk monitoring |
| Evaluate risks from deployment context | [FRIA](../compliance/fria.md) — deployment context section |
| Implement risk management measures | [Thresholds](../reference/thresholds.md) — configurable detection thresholds |
| Testing with real-world data | [Human oversight decisions](../compliance/oversight.md) — ground-truth feedback loop |

---

#### Article 10 — Data and Data Governance

> Data used to train and test the AI system must meet quality criteria.

Geodesia G-1 does not train models in production. For custom fine-tuning workflows, the system logs all training-related operations. The detection engine's input data (user prompts and context) is logged in the audit chain for governance review.

---

#### Article 11 — Technical Documentation

> Providers must maintain technical documentation before market placement.

The [Deployer Manual](../compliance/reports.md#deployer-transparency-manual) (`POST /v1/glad/deployer-manual`) generates Article 13/11-compliant documentation automatically from the live system configuration.

---

#### Article 12 — Record-Keeping

> High-risk AI systems must log automatically to enable post-market monitoring.

| Requirement | Geodesia G-1 Feature |
|---|---|
| Automatic logging | Every inference call written to the database automatically |
| Log integrity | [HMAC audit chain](../compliance/audit-chain.md) — tamper-evident |
| Log retention | [Retention policy](../compliance/reports.md#data-retention) — configurable per data type |
| Log accessibility | `GET /v1/glad/chain/entries` — query and export |

---

#### Article 13 — Transparency and Provision of Information

> High-risk AI systems must be transparent to deployers.

| Requirement | Geodesia G-1 Feature |
|---|---|
| Clear instructions for use | `POST /v1/glad/deployer-manual` |
| Capabilities and limitations | Deployer manual — limitations section |
| AI-generated content disclosure | [Watermark](../compliance/watermark.md) — manifest + latent |
| Provider identity | `GET /v1/glad/provider-identity` |

---

#### Article 14 — Human Oversight

> High-risk AI systems must have human oversight capabilities.

| Requirement | Geodesia G-1 Feature |
|---|---|
| Ability to interrupt the system | [Kill Switch](../compliance/kill-switch.md) — immediate suspension |
| Ability to override outputs | `POST /v1/glad/oversight/decide` with `decision: "overridden_allow"` |
| Monitoring for anomalies | [Dashboard](../compliance/dashboard.md) — real-time metrics |
| Human interpretation of outputs | [Explainability](../product-api/explainability.md) — per-token attribution |
| Escalation chain | [Oversight](../compliance/oversight.md) — tiered escalation |

---

### Chapter V — General-Purpose AI Models (Articles 51–56)

If you are using a **general-purpose AI model** (GPAI) such as a public LLM, you are a deployer and must comply with the GPAI provisions. Geodesia G-1 provides:

- Watermarking for synthetic content (Art. 50)
- Monitoring and logging (Art. 72)
- Incident reporting infrastructure (Art. 73)

---

### Chapter VI — Measures in Support of Innovation

No restrictions from Geodesia G-1's side. The system supports regulatory sandboxes by providing complete audit trails that can be submitted to national authorities.

---

### Chapter VII — Governance (Articles 57–75)

#### Article 50 — Transparency Obligations for Certain AI Systems

> AI-generated content must be disclosed.

Geodesia G-1 satisfies this with:
- **Manifest watermark** — `geodesia.watermark.disclosure` in every response
- **Latent watermark** — HMAC-SHA256 token verifiable at `POST /v1/glad/watermark/verify`
- **Configurable disclosure text** — set `watermark.disclosure_text` in config.yaml

---

#### Article 72 — Post-Market Monitoring

> Providers and deployers must establish post-market monitoring systems.

| Requirement | Geodesia G-1 Feature |
|---|---|
| Continuous monitoring | [Dashboard](../compliance/dashboard.md) — real-time detection statistics |
| Serious incident detection | [Human oversight](../compliance/oversight.md) — anomaly escalation |
| Performance tracking | `GET /v1/glad/scorecard` — per-framework compliance tracking |

---

#### Article 73 — Reporting of Serious Incidents

> Providers must report serious incidents to national authorities within 15 days.

Geodesia G-1 does not automatically submit to authorities (no access to national authority endpoints). However:
- All serious incidents are flagged in the [audit chain](../compliance/audit-chain.md)
- The [compliance report](../compliance/reports.md) can be generated on demand for incident submission
- The `POST /v1/glad/report` endpoint generates the required technical documentation

---

### Article 26 — Obligations of Deployers

> Deployers must use AI systems in accordance with instructions and maintain human oversight.

| Obligation | Geodesia G-1 Support |
|---|---|
| Use only for intended purpose | `purpose` field in FRIA; deployer manual |
| Monitor operation | Dashboard + scorecard |
| Maintain oversight | Oversight queue + escalation chain |
| Report malfunctions | Incident log via audit chain |
| Inform provider of serious risks | Notification configuration |

---

### Article 27 — FRIA

> Deployers of high-risk AI systems in certain domains must conduct a FRIA before deployment.

Full FRIA lifecycle: [FRIA documentation](../compliance/fria.md).

```
POST /v1/glad/fria          → create
PUT /v1/glad/fria/{id}      → edit
POST /v1/glad/fria/{id}/approve  → approve
GET /v1/glad/fria/{id}/export?fmt=pdf  → export
```

---

## Quick Compliance Checklist

Use this checklist before going to production with a high-risk AI system:

- [ ] Create a FRIA dossier (`POST /v1/glad/fria`)
- [ ] Complete all FRIA sections and attach evidence
- [ ] Get FRIA approved by the AI Responsible (`POST /v1/glad/fria/{id}/approve`)
- [ ] Generate and review the Deployer Manual (`POST /v1/glad/deployer-manual`)
- [ ] Configure applicable laws in `config.yaml`
- [ ] Set detection thresholds (`POST /v1/glad/threshold-prefs`)
- [ ] Enable human oversight (`human_oversight.enabled: true`)
- [ ] Verify audit chain integrity (`GET /v1/glad/chain/verify`)
- [ ] Test the kill switch in staging (`POST /v1/glad/kill-switch/activate` + deactivate)
- [ ] Review the compliance scorecard (`GET /v1/glad/scorecard`)
- [ ] Run a compliance report (`POST /v1/glad/report`)
