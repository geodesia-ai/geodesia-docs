# Model Catalog

Geodesia G-1 ships with a catalog of pre-configured model checkpoints. Each checkpoint has been trained to operate specific detection axes and is optimized for different accuracy/latency trade-offs.

---

## Available Models

The `GET /v1/glad/models/available` endpoint returns the currently loaded model and the full catalog:

```bash
curl http://localhost:8199/v1/glad/models/available
```

```json
{
  "active_model": "gemma4_e2b_default",
  "available": [
    {
      "name": "gemma4_e2b_default",
      "description": "Gemma 4 E2B — full 5-axis detection, production default",
      "axes": ["halluc_context", "halluc_closedbook", "prompt_safety", "answer_safety", "jailbreak"],
      "loaded": true,
      "device": "cuda:0",
      "vram_gb": 9.96
    },
    {
      "name": "ministral3_3b_safety",
      "description": "Ministral 3-3B — optimized for prompt safety detection",
      "axes": ["prompt_safety", "answer_safety", "jailbreak"],
      "loaded": false,
      "device": "cuda:1",
      "vram_gb": 7.2
    }
  ]
}
```

### Response Fields

| Field | Description |
|---|---|
| `active_model` | Name of the currently active checkpoint |
| `available[].name` | Unique checkpoint name (configured in `config.yaml`) |
| `available[].description` | Human-readable description |
| `available[].axes` | Which detection axes this checkpoint supports |
| `available[].loaded` | Whether this checkpoint is currently loaded in GPU memory |
| `available[].device` | CUDA device this checkpoint runs on |
| `available[].vram_gb` | Approximate VRAM usage when loaded |

---

## Model Switching

### POST /v1/glad/models/switch

Switch the active model checkpoint at runtime. The new model is loaded before the old one is unloaded to minimize service interruption.

```bash
curl -X POST http://localhost:8199/v1/glad/models/switch \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "ministral3_3b_safety",
    "reason": "Switching to high-precision prompt safety model for legal department deployment",
    "switched_by": "ops@acme.com"
  }'
```

#### Request Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `model_name` | `string` | ✅ | Name from the model catalog |
| `reason` | `string` | — | Reason for the switch. Recorded in the audit chain. |
| `switched_by` | `string` | — | Identity of the person or process triggering the switch. |

#### Response

```json
{
  "status": "switched",
  "previous_model": "gemma4_e2b_default",
  "active_model": "ministral3_3b_safety",
  "switched_at": "2026-06-10T14:00:00Z",
  "audit_entry": "chain_xyz123"
}
```

Model switches are recorded in the audit chain with the `model_switched` event type.

---

## Axis Coverage by Model

| Model | Halluc (Context) | Halluc (Closed-Book) | Prompt Safety | Answer Safety | Jailbreak |
|---|---|---|---|---|---|
| `gemma4_e2b_default` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ministral3_3b_safety` | ❌ | ❌ | ✅ | ✅ | ✅ |
| `qwen3_1_7b` | ✅ | ❌ | ✅ | ✅ | ❌ |

When a model does not support an axis, the corresponding score in the response will be `null` and the threshold comparison will not run. The `geodesia.axes_available` field in the response lists the axes that were actually computed.

---

## Choosing a Model

**Use `gemma4_e2b_default`** when:
- You need all 5 detection axes
- You have a GPU with ≥24 GB VRAM
- You are processing customer-facing conversations where hallucination in RAG is a concern

**Use `ministral3_3b_safety`** when:
- Your primary concern is prompt safety and jailbreak prevention
- You are running on a GPU with 16 GB VRAM (full-safety model fits)
- You want the highest prompt safety AUROC (+22% vs the default model)

**Use CPU mode** (any model with `"device": "cpu"`) when:
- You have no GPU but need compliance APIs only
- You are running in batch processing mode where latency is not critical

---

## VRAM Requirements

| Model | Min VRAM | Recommended VRAM |
|---|---|---|
| `gemma4_e2b_default` | 24 GB | 48 GB (A6000) |
| `ministral3_3b_safety` | 16 GB | 24 GB |
| `qwen3_1_7b` | 8 GB | 16 GB |
| CPU mode | 0 GB (uses RAM) | ≥32 GB RAM |

---

## Model Metadata

Each loaded model exposes its configuration metadata via `GET /v1/glad/models/available`. This includes the checkpoint's training configuration, calibrated thresholds, and supported mechanisms.

Model metadata is also embedded in every compliance report and deployer manual.
