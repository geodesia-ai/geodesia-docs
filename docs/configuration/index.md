# Configuration Reference

Geodesia G-1 uses a single `config.yaml` file for the Product Backend. The Gateway is configured via environment variables and the [GatewayConfig API](../gateway/configuration.md). This page documents every `config.yaml` section.

---

## File Location

By default, the product backend looks for `config.yaml` in the current working directory. Override with the `GLAD_CONFIG` environment variable:

```bash
GLAD_CONFIG=/etc/geodesia/config.yaml python -m uvicorn main:app ...
```

---

## Top-Level Structure

```yaml
models:          # Model checkpoint catalog
app:             # Application-level settings
training:        # Training defaults (if fine-tuning is enabled)
generation:      # Default generation parameters
audit:           # Audit logging configuration
provider:        # Provider identity
watermark:       # AI watermark settings
retention:       # Data retention policies
kill_switch:     # Kill switch settings
fria:            # FRIA defaults
human_oversight: # Human oversight settings
xai:             # Explainability settings
notifications:   # Alert and notification settings
```

---

## `models` Section

Defines the catalog of available model checkpoints. Each entry can be selected by name through the [Model Switcher API](../models/index.md).

```yaml
models:
  - name: gemma4_e2b_default
    description: "Gemma 4 E2B — full 5-axis detection"
    path: runs/api_product_model_v3
    device: cuda:0
    axes:
      - halluc_context
      - halluc_closedbook
      - prompt_safety
      - answer_safety
      - jailbreak
    default: true

  - name: ministral3_3b_safety
    description: "Ministral 3-3B — best prompt safety performance"
    path: runs/ministral3_3b_full_safety
    device: cuda:1
    axes:
      - prompt_safety
      - answer_safety
      - jailbreak
```

### Per-Model Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | ✅ | Unique name used to refer to this checkpoint |
| `description` | `string` | — | Human-readable description |
| `path` | `string` | ✅ | Path to the checkpoint directory (absolute or relative to working dir) |
| `device` | `string` | — | CUDA device to load on: `"cuda:0"`, `"cuda:1"`, `"cpu"`. Default: value of `GLAD_DEVICE` env var |
| `axes` | `array[string]` | — | Which detection axes this checkpoint supports. Defaults to all 5. |
| `default` | `boolean` | — | If `true`, this checkpoint is loaded at startup. Only one checkpoint can be default. |

---

## `app` Section

Core application settings.

```yaml
app:
  host: "127.0.0.1"
  port: 8199
  applicable_laws:
    - EU_AI_ACT
    - GDPR
    - ISO_42001
  db_path: "var/glad.sqlite3"
  log_level: "info"
  max_concurrent_requests: 8
  request_timeout_seconds: 120
  deployer_id: "default"
```

| Field | Default | Description |
|---|---|---|
| `host` | `"127.0.0.1"` | Bind address |
| `port` | `8199` | Bind port |
| `applicable_laws` | `["EU_AI_ACT"]` | Regulatory frameworks to activate. See [Regulatory Coverage](../regulatory/index.md) for valid codes. |
| `db_path` | `"var/glad.sqlite3"` | Path to the SQLite database file. Can be overridden with `GLAD_DB_PATH`. |
| `log_level` | `"info"` | Python logging level: `"debug"`, `"info"`, `"warning"`, `"error"` |
| `max_concurrent_requests` | `8` | Maximum concurrent inference requests. Higher values require more VRAM. |
| `request_timeout_seconds` | `120` | Maximum time allowed for a single inference + evaluation call |
| `deployer_id` | `"default"` | Default deployer ID used when none is specified in API calls |

---

## `generation` Section

Default generation parameters, used when the caller does not provide a `generation_config`.

```yaml
generation:
  max_new_tokens: 160
  temperature: 0.7
  top_p: 0.9
  do_sample: true
  repetition_penalty: 1.0
```

| Field | Default | Description |
|---|---|---|
| `max_new_tokens` | `160` | Maximum tokens to generate |
| `temperature` | `0.7` | Sampling temperature. Lower = more deterministic. |
| `top_p` | `0.9` | Nucleus sampling cutoff probability |
| `do_sample` | `true` | Whether to use sampling. `false` = greedy decoding |
| `repetition_penalty` | `1.0` | Penalty for repeated tokens. Values > 1 reduce repetition. |

---

## `audit` Section

Controls the append-only audit chain.

```yaml
audit:
  enabled: true
  chain_enabled: true
  log_all_calls: true
  log_blocked_only: false
  hmac_key_env: "GLAD_AUDIT_HMAC_KEY"
```

| Field | Default | Description |
|---|---|---|
| `enabled` | `true` | Whether audit logging is active |
| `chain_enabled` | `true` | Whether to use HMAC chain linking (tamper-evident). If `false`, entries are written but not linked. |
| `log_all_calls` | `true` | Write every call to the audit log |
| `log_blocked_only` | `false` | If `true`, only write blocked/flagged calls |
| `hmac_key_env` | `"GLAD_AUDIT_HMAC_KEY"` | Name of the environment variable containing the HMAC signing key. If not set, a random key is generated at startup (not suitable for cross-restart verification). |

---

## `provider` Section

Provider identity information returned by `GET /v1/glad/provider-identity` and included in all generated reports.

```yaml
provider:
  name: "Geodesia AI"
  system_name: "Geodesia G-1"
  version: "1.0.0"
  contact_email: "compliance@geodesia.ai"
  eu_representative: null        # Required for non-EU providers selling to EU
  website: "https://geodesia.ai"
```

---

## `watermark` Section

AI content watermarking configuration.

```yaml
watermark:
  enabled: true
  algorithm: "hmac_sha256"
  include_in_response: true
  disclosure_text: "This content was generated by an AI system."
  custom_disclosure: null
```

| Field | Default | Description |
|---|---|---|
| `enabled` | `true` | Whether to watermark responses |
| `algorithm` | `"hmac_sha256"` | Watermark algorithm. Only `"hmac_sha256"` is currently supported. |
| `include_in_response` | `true` | Include `geodesia.watermark` in every API response |
| `disclosure_text` | `"This content was generated by an AI system."` | Plain-language disclosure string shown in the manifest watermark |
| `custom_disclosure` | `null` | Override `disclosure_text` with a jurisdiction-specific string |

---

## `retention` Section

Data retention policies. Data older than the configured period is eligible for deletion.

```yaml
retention:
  call_log_days: 365        # Inference call records
  oversight_days: 730       # Human oversight reviews
  fria_days: 3650           # FRIA dossiers (10 years)
  chain_days: 3650          # Audit chain entries
  auto_delete: false        # Whether to automatically delete expired records
```

| Field | Default | Description |
|---|---|---|
| `call_log_days` | `365` | Retention for inference call records (days) |
| `oversight_days` | `730` | Retention for human oversight review records |
| `fria_days` | `3650` | Retention for FRIA dossiers. EU AI Act requires 10 years. |
| `chain_days` | `3650` | Retention for audit chain entries |
| `auto_delete` | `false` | Automatically delete expired records. Default is `false` — you must explicitly trigger deletion. |

---

## `kill_switch` Section

```yaml
kill_switch:
  enabled: true
  require_reason: true
  max_auto_deactivate_hours: 8760
  notification_email: null
  audit_all_events: true
```

| Field | Default | Description |
|---|---|---|
| `enabled` | `true` | Whether the kill switch is available |
| `require_reason` | `true` | Reject activation requests without a `reason` field |
| `max_auto_deactivate_hours` | `8760` | Maximum auto-deactivation window (one year) |
| `notification_email` | `null` | Email to notify on activation/deactivation. Requires `notifications` to be configured. |
| `audit_all_events` | `true` | Write every activation/deactivation to the HMAC chain |

---

## `fria` Section

FRIA defaults and auto-generation settings.

```yaml
fria:
  default_risk_level: "high"
  default_applicable_laws:
    - EU_AI_ACT
    - GDPR
  auto_prefill_from_config: true
  require_approval_before_production: false
  export_watermark: true
```

| Field | Default | Description |
|---|---|---|
| `default_risk_level` | `"high"` | Risk level pre-populated in new FRIA dossiers |
| `default_applicable_laws` | `["EU_AI_ACT"]` | Laws pre-populated in new dossiers |
| `auto_prefill_from_config` | `true` | Automatically fill technical sections from the active model and detection configuration |
| `require_approval_before_production` | `false` | If `true`, the system will warn if no approved FRIA exists |
| `export_watermark` | `true` | Embed an HMAC watermark in exported FRIA PDFs |

---

## `human_oversight` Section {#human-oversight}

```yaml
human_oversight:
  enabled: true
  required: true
  tiers:
    - level: operator
      review_window_hours: 24
    - level: ai_responsible
      review_window_hours: 72
  oversight_threshold: 0.7
  review_deadline_hours: 48
  auto_escalate: true
  notification_email: null
```

| Field | Default | Description |
|---|---|---|
| `enabled` | `true` | Whether oversight queuing is active |
| `required` | `true` | EU AI Act Art. 14 compliance flag — used in scorecard |
| `tiers` | see above | List of escalation levels with their review time windows |
| `oversight_threshold` | `0.7` | Detection score above which a call enters the oversight queue |
| `review_deadline_hours` | `48` | Time before an unreviewed call is marked "overdue" |
| `auto_escalate` | `true` | Automatically escalate overdue reviews to the next tier |
| `notification_email` | `null` | Email to notify when new calls enter the queue |

---

## `xai` Section

Explainability settings.

```yaml
xai:
  enabled: true
  mupax_n_samples: 500
  mupax_max_input_tokens: 512
  pss_n_samples: 5
  pss_temperature: 0.7
  pss_match_mode: "ngram"
  causal_max_analyze_tokens: 256
```

| Field | Default | Description |
|---|---|---|
| `enabled` | `true` | Whether XAI computation is available |
| `mupax_n_samples` | `500` | Number of Monte Carlo samples for MuPAX attribution |
| `mupax_max_input_tokens` | `512` | Maximum input length for MuPAX. Inputs longer than this are truncated from the left. |
| `pss_n_samples` | `5` | Default number of resamples for PSS |
| `pss_temperature` | `0.7` | Default PSS sampling temperature |
| `pss_match_mode` | `"ngram"` | Default PSS alignment algorithm |
| `causal_max_analyze_tokens` | `256` | Maximum tokens to analyze in causal XAI mode. Increasing this improves precision but increases compute. |

---

## `notifications` Section

Email and webhook notifications for compliance events.

```yaml
notifications:
  smtp_host: null
  smtp_port: 587
  smtp_user: null
  smtp_password_env: "GLAD_SMTP_PASSWORD"
  from_email: null
  webhook_url: null
  webhook_secret_env: "GLAD_WEBHOOK_SECRET"
  events:
    - kill_switch_activated
    - oversight_escalation
    - chain_tamper_detected
```

| Field | Default | Description |
|---|---|---|
| `smtp_host` | `null` | SMTP server hostname. If null, email notifications are disabled. |
| `smtp_port` | `587` | SMTP port |
| `smtp_user` | `null` | SMTP username |
| `smtp_password_env` | `"GLAD_SMTP_PASSWORD"` | Environment variable containing the SMTP password |
| `from_email` | `null` | Sender email address |
| `webhook_url` | `null` | HTTP POST webhook URL for event notifications. If null, webhooks are disabled. |
| `webhook_secret_env` | `"GLAD_WEBHOOK_SECRET"` | Env var containing the webhook HMAC secret for signature verification |
| `events` | see above | List of event types that trigger notifications |
