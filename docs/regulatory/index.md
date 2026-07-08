# Regulatory Coverage

Geodesia G-1 is designed to help you operate AI systems in compliance with **13 regulatory frameworks** across 6 jurisdictions. Each framework maps to specific technical capabilities in the system.

---

## Supported Frameworks

| Code | Framework | Jurisdiction | Status |
|---|---|---|---|
| `EU_AI_ACT` | EU AI Act | European Union | ✅ Supported |
| `GDPR` | General Data Protection Regulation | European Union | ✅ Supported |
| `ISO_42001` | ISO/IEC 42001:2023 AI Management Systems | International | ✅ Supported |
| `NIST_AI_RMF` | NIST AI Risk Management Framework 1.0 | United States | ✅ Supported |
| `CA_SB_942` | California AI Transparency Act (SB 942) | California, USA | ✅ Supported |
| `ITALY_132_2025` | Italian AI Law 132/2025 | Italy | ✅ Supported |
| `UK_DUAA_2025` | UK Data Use and Access Act 2025 | United Kingdom | ✅ Supported |
| `BRAZIL_2338` | Brazil AI Regulation 2338/2023 | Brazil | ✅ Supported |
| `CANADA_AIDA` | Artificial Intelligence and Data Act | Canada | ✅ Supported |
| `CHINA_GB45654` | China GB/T 45654 AI Standard | China | ✅ Supported |
| `COLORADO_SB21_169` | Colorado AI Law SB21-169 | Colorado, USA | ✅ Supported |
| `NYC_LL144` | New York City Local Law 144 | New York City, USA | ✅ Supported |
| `SOC2` | SOC 2 Type II | United States | ✅ Supported |

---

## Configuring Applicable Laws

Specify which frameworks apply to your deployment in [config.yaml](../configuration/index.md):

```yaml
app:
  applicable_laws:
    - EU_AI_ACT
    - GDPR
    - ISO_42001
    - CA_SB_942
```

The compliance dashboard, scorecard, FRIA prefill, and generated reports all adapt to the configured frameworks.

---

## Capability Matrix

| Capability | EU AI Act | GDPR | ISO 42001 | NIST RMF | CA SB 942 | Other |
|---|---|---|---|---|---|---|
| FRIA / Impact Assessment | ✅ Art. 27 | ✅ Art. 35 (DPIA) | ✅ §8.4 | ✅ GOVERN | — | varies |
| Human Oversight | ✅ Art. 14 | — | ✅ §8.5 | ✅ MANAGE | — | varies |
| Kill Switch | ✅ Art. 14 | — | — | ✅ GOVERN | ✅ 72h | varies |
| Audit Chain / Logging | ✅ Art. 12 | ✅ Art. 30 | ✅ §9.1 | ✅ MEASURE | ✅ | ✅ |
| AI Watermark | ✅ Art. 50 | — | — | — | ✅ | Italy, UK |
| Compliance Reports | ✅ Art. 11 | — | ✅ §9.3 | ✅ GOVERN | — | varies |
| Deployer Manual | ✅ Art. 13 | — | — | — | — | — |
| Provider Identity | ✅ Art. 13 | — | — | — | — | — |
| Data Retention Policy | ✅ Art. 12 | ✅ Art. 5(1)(e) | ✅ §7.5 | — | — | varies |
| Detection / Safety | ✅ Art. 9 | — | ✅ §6.1 | ✅ MAP | ✅ | varies |

---

## Framework Descriptions

### EU AI Act

The **EU Artificial Intelligence Act** (Regulation (EU) 2024/1689) is the world's first comprehensive AI legislation. It classifies AI systems into four risk tiers:

- **Unacceptable risk** — prohibited (e.g., social scoring, real-time biometric surveillance in public)
- **High risk** — subject to strict requirements before market placement (Annex III use cases)
- **Limited risk** — transparency obligations only
- **Minimal risk** — no obligations

Geodesia G-1 provides tools for high-risk system compliance. See [EU AI Act](eu-ai-act.md) for article-by-article coverage.

---

### GDPR

The **General Data Protection Regulation** (Regulation (EU) 2016/679) governs the processing of personal data in the EU. For AI systems, key requirements include:

- Lawful basis for automated processing
- Data minimization
- Retention limits
- Right to explanation for automated decisions

Geodesia G-1's logging, retention, and audit features support GDPR Article 30 (records of processing) and Article 22 (automated decision-making) compliance.

---

### ISO/IEC 42001:2023

ISO 42001 is the international management system standard for AI governance. It requires organizations to establish, implement, maintain, and continually improve an AI management system. Geodesia G-1 provides the technical controls documented in the standard's Annex A.

---

### NIST AI Risk Management Framework 1.0

The **NIST AI RMF** provides a voluntary framework for managing AI risks across four functions: GOVERN, MAP, MEASURE, and MANAGE. Geodesia G-1 maps to all four functions, with the dashboard and scorecard supporting MEASURE requirements, and the kill switch and oversight supporting GOVERN/MANAGE.

---

### California SB 942 (AI Transparency Act)

California's **Artificial Intelligence Transparency Act** (effective January 1, 2026) requires providers of AI systems to:

- Watermark or tag AI-generated content
- Provide a publicly accessible AI detection tool
- Suspend service within 72 hours on government request

Geodesia G-1 satisfies all three requirements. See the [Watermark](../compliance/watermark.md) and [Kill Switch](../compliance/kill-switch.md) pages.

---

### Italy 132/2025

Italy's **Decree-Law 132/2025** on artificial intelligence introduces requirements for AI content disclosure, mandatory labeling of AI-generated images/audio/video, and sectoral restrictions. Geodesia G-1's manifest watermark and disclosure field satisfy the content marking requirements.

---

### UK Data Use and Access Act 2025 (DUAA)

The UK's **DUAA 2025** establishes transparency and accountability requirements for AI systems used in the UK. Geodesia G-1's deployer manual and provider identity endpoints satisfy the transparency obligations.

---

### Brazil 2338/2023

Brazil's **AI Regulation Bill 2338/2023** establishes a risk-based regulatory framework similar to the EU AI Act, with requirements for high-risk AI systems including impact assessments, human oversight, and transparency. Geodesia G-1's FRIA and oversight modules satisfy these requirements.

---

### Canada AIDA

Canada's **Artificial Intelligence and Data Act** (AIDA, part of Bill C-27) requires high-impact AI systems to implement impact assessments, monitoring, and incident reporting. Geodesia G-1 supports these requirements through its FRIA, monitoring dashboard, and audit chain.

---

### China GB/T 45654

China's **GB/T 45654** national standard for generative AI services requires content filtering, logging, and safety mechanisms. Geodesia G-1's detection axes and audit chain satisfy these requirements.

---

### Colorado SB21-169

Colorado's **SB21-169** (Artificial Intelligence Act) targets consequential AI decisions in insurance, finance, employment, and housing. It requires impact assessments and disclosure obligations.

---

### NYC Local Law 144

New York City's **Local Law 144** requires automated employment decision tools to undergo annual bias audits and provide candidates with disclosure. For deployments used in hiring contexts, Geodesia G-1's detection capabilities and audit documentation support compliance.

---

### SOC 2

SOC 2 is a voluntary auditing standard for service organizations that defines controls for security, availability, processing integrity, confidentiality, and privacy. Geodesia G-1's append-only audit chain, access logging, and kill switch satisfy the CC7 (system operations) and A1 (availability) controls.
