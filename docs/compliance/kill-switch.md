# Kill Switch

The Kill Switch provides an **immediate, system-wide suspension** mechanism for the Geodesia G-1 service. When activated, all inference calls are rejected with a clear compliance message. It is designed to satisfy the "stop button" requirements in EU AI Act Article 14 and the 72-hour compliance deactivation window in California SB 942.

---

## What Happens When the Kill Switch Is Active

When the kill switch is engaged:

1. The gateway immediately rejects all new inference requests with HTTP `503` and a compliance message
2. All queued calls are drained without processing
3. The kill switch activation event is written to the HMAC audit chain
4. A kill switch notification is sent to the configured contact email (if enabled)
5. All existing sessions are invalidated

The kill switch does **not** affect:
- Compliance API endpoints (oversight, FRIA, reports, audit chain)
- Health check endpoints
- Admin endpoints for deactivating the kill switch

---

## Endpoints

### GET /v1/glad/kill-switch/status

Returns the current kill switch state.

```bash
curl http://localhost:8199/v1/glad/kill-switch/status
```

#### Response

```json
{
  "active": false,
  "last_activated_at": null,
  "last_deactivated_at": "2026-06-09T08:00:00Z",
  "activated_by": null,
  "reason": null,
  "regulatory_framework": null,
  "auto_deactivate_at": null
}
```

| Field | Description |
|---|---|
| `active` | Whether the kill switch is currently engaged |
| `last_activated_at` | ISO 8601 timestamp of last activation |
| `last_deactivated_at` | ISO 8601 timestamp of last deactivation |
| `activated_by` | Identity of the person or system that activated the switch |
| `reason` | The reason provided at activation |
| `regulatory_framework` | The regulatory framework cited at activation (e.g., `"CA_SB_942"`) |
| `auto_deactivate_at` | ISO 8601 timestamp for scheduled automatic deactivation, if set |

---

### POST /v1/glad/kill-switch/activate

Engage the kill switch.

```bash
curl -X POST http://localhost:8199/v1/glad/kill-switch/activate \
  -H "Content-Type: application/json" \
  -d '{
    "reason": "Regulatory audit in progress — service suspended as required by CA SB 942",
    "activated_by": "compliance-officer@acme.com",
    "regulatory_framework": "CA_SB_942",
    "auto_deactivate_hours": 72
  }'
```

#### Request Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | `string` | ✅ | Required justification for activation. Stored permanently in the audit chain. |
| `activated_by` | `string` | ✅ | Identifier of the person or system activating the switch (email, user ID, or service name). |
| `regulatory_framework` | `string` | — | The legal basis for activation (e.g., `"CA_SB_942"`, `"EU_AI_ACT"`). |
| `auto_deactivate_hours` | `integer` | — | If provided, the kill switch automatically deactivates after this many hours. Range: 1–8760 (one year). The CA SB 942 compliance window is 72 hours. |
| `notify` | `boolean` | — | Whether to send a notification to the configured compliance email. Default `true`. |

#### Response

```json
{
  "status": "activated",
  "activated_at": "2026-06-10T12:00:00Z",
  "auto_deactivate_at": "2026-06-13T12:00:00Z",
  "audit_chain_entry": "ks_event_abc123"
}
```

---

### POST /v1/glad/kill-switch/deactivate

Disengage the kill switch and resume normal operation.

```bash
curl -X POST http://localhost:8199/v1/glad/kill-switch/deactivate \
  -H "Content-Type: application/json" \
  -d '{
    "reason": "Audit complete. Service resuming normal operation.",
    "deactivated_by": "compliance-officer@acme.com"
  }'
```

#### Request Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `reason` | `string` | ✅ | Justification for deactivation. |
| `deactivated_by` | `string` | ✅ | Identity of the person or system deactivating. |

---

## Kill Switch Configuration

Configure default kill switch behavior in [config.yaml](../configuration/index.md):

```yaml
kill_switch:
  enabled: true
  require_reason: true            # Reject activation requests without a reason
  max_auto_deactivate_hours: 8760 # Maximum allowed auto-deactivation window
  notification_email: "compliance@acme.com"
  audit_all_events: true          # Write every activation/deactivation to the HMAC chain
```

---

## Regulatory Context

| Law | Requirement | Kill Switch Coverage |
|---|---|---|
| EU AI Act Art. 14 | Human operators must be able to interrupt the system | Immediate global interrupt |
| EU AI Act Art. 14(4)(a) | Override capability at any time | Yes — deactivates all inference |
| CA SB 942 | Suspend service within 72 hours on regulatory request | `auto_deactivate_hours: 72` |
| ISO 42001 §8.4 | Incident response and operational continuity | Audit-logged activation events |
| NIST AI RMF GOVERN 1.7 | Mechanisms for disabling AI systems | Yes — documented stop capability |

---

## Integration: Programmatic Activation

The kill switch can be activated by your monitoring or incident response pipeline:

```python
import httpx

async def emergency_suspend(reason: str, officer: str):
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            "http://localhost:8199/v1/glad/kill-switch/activate",
            json={
                "reason": reason,
                "activated_by": officer,
                "regulatory_framework": "CA_SB_942",
                "auto_deactivate_hours": 72,
            }
        )
        resp.raise_for_status()
        data = resp.json()
        print(f"Kill switch activated. Auto-deactivation at: {data['auto_deactivate_at']}")
```
