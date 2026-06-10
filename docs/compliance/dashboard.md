# Dashboard

The Compliance Dashboard provides real-time visibility into the operational status of your AI deployment. It shows call volumes, detection outcomes, and regulatory scorecard metrics.

---

## GET /v1/glad/dashboard

Returns the full dashboard payload used by the web interface.

```bash
curl http://localhost:8199/v1/glad/dashboard
```

### Query Parameters

| Parameter | Type | Description |
|---|---|---|
| `deployer_id` | `string` | Optional. Filter results to a specific deployer. Defaults to `"default"`. |

### Response

The dashboard response includes:

**Call Metrics (runtime):**

| Field | Description |
|---|---|
| `total` | Total number of inference calls in the database |
| `passed` | Calls where all detection axes were below threshold |
| `prompt_blocked` | Calls where the input prompt was blocked |
| `answer_unsafe` | Calls where the answer was flagged as unsafe |
| `answer_safe` | Calls where the answer passed safety checks |
| `hallucinated` | Calls where the answer was flagged as hallucinated |
| `grounded` | Calls where the answer was grounded (context faithfulness passed) |
| `flagged_any` | Calls flagged on any axis |

**Compliance Status:**

| Field | Description |
|---|---|
| `subsystems` | Per-component compliance status (7 subsystems) |
| `frameworks_compliant` | Number of regulatory frameworks meeting compliance criteria |
| `kill_switch_active` | Whether the kill switch is currently engaged |
| `pending_reviews` | Number of calls awaiting human review |
| `chain_integrity` | Whether the audit chain passes integrity verification |

---

## GET /v1/glad/scorecard

Returns a regulatory compliance scorecard — a structured summary of which requirements are met, which are pending, and which require attention.

```bash
curl http://localhost:8199/v1/glad/scorecard
```

### Query Parameters

| Parameter | Type | Description |
|---|---|---|
| `deployer_id` | `string` | Optional deployer filter. Defaults to `"default"`. |

### Response

The scorecard contains one entry per regulatory framework in your configuration. Each entry shows:

| Field | Description |
|---|---|
| `framework` | Framework code (e.g., `"EU_AI_ACT"`) |
| `score` | Numeric compliance score (0–100) |
| `status` | `"compliant"`, `"partial"`, or `"non_compliant"` |
| `requirements` | List of specific requirements with their individual status |
| `gaps` | Requirements that are not yet met, with recommendations |

---

## GET /v1/glad/health

A lightweight health check for the compliance backend. Does not require database access.

```bash
curl http://localhost:8199/v1/glad/health
```

```json
{"status": "ok", "version": "1.0.0"}
```

---

## GET /health (root)

The root health check also returns the active configuration snapshot:

```bash
curl http://localhost:8199/health
```

```json
{
  "status": "ok",
  "version": "1.0.0",
  "config": {
    "model_loaded": true,
    "device": "cuda:0",
    "kill_switch_active": false,
    "applicable_laws": ["EU_AI_ACT", "GDPR", "ISO_42001"]
  }
}
```
