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

Three routes on **G-1 Studio**. All of them are scoped by `deployer_id`, which defaults to `"default"` — a single-deployer install can ignore it entirely.

!!! note "The deployer id is hashed"
    Responses carry `deployer_id_hash`, never the id you sent. You address the switch by the plaintext id; the ledger stores only its hash. That is what lets the audit trail prove *which* system was suspended without retaining the identifier as personal data.

---

### GET /v1/glad/kill-switch/status

**What it does.** Returns whether the system is suspended, and — because CA SB 942 gives you 72 hours to actually suspend after an activation is requested — whether you are still inside that window.

=== "curl"

    ```bash
    curl -s "http://localhost:8080/v1/glad/kill-switch/status?deployer_id=default" | jq
    ```

=== "Python"

    ```python
    import httpx

    st = httpx.get("http://localhost:8080/v1/glad/kill-switch/status",
                   params={"deployer_id": "default"}).json()
    print(st["status"])                       # "active" | "suspended"
    if not st["sb942_compliant"]:
        print(f"SB 942 window exceeded: {st['elapsed_hours']:.1f}h since activation")
    ```

=== "TypeScript"

    ```ts
    const st = await fetch("http://localhost:8080/v1/glad/kill-switch/status?deployer_id=default")
      .then(r => r.json())
    if (st.status === "suspended") console.warn("service is suspended")
    ```

**What comes back**

```json
{
  "deployer_id_hash": "9c1f2a7b4e0d6a18",
  "status": "active",
  "last_event_id": "KS-3F9A1C7B2E5D8046A1B2",
  "last_updated": "2026-06-10T12:00:00+00:00",
  "sb942_compliant": true,
  "elapsed_hours": null
}
```

| Field | Description |
|---|---|
| `status` | `"active"` (serving) or `"suspended"` (killed). A deployer that has never used the switch reads `"active"`. |
| `last_event_id` | The `KS-…` id of the most recent activate/deactivate/test event, or `null`. |
| `last_updated` | When the state last changed. |
| `sb942_compliant` | `false` only when an activation was recorded, the system is **still not** suspended, and more than 72 hours have passed. |
| `elapsed_hours` | Hours since the last activation event. `null` when there has never been one. |

#### Query parameters

| Parameter | Default | Description |
|---|---|---|
| `deployer_id` | `"default"` | Which deployer's switch to read. |

---

### POST /v1/glad/kill-switch/activate

**What it does.** Suspends the deployer's system and writes the event to the ledger. Takes effect immediately — there is no confirmation step and no scheduled auto-release.

=== "curl"

    ```bash
    curl -s -X POST http://localhost:8080/v1/glad/kill-switch/activate \
      -H "Content-Type: application/json" \
      -d '{
        "deployer_id":  "default",
        "reason":       "Regulatory audit in progress — service suspended per CA SB 942",
        "activated_by": "compliance-officer@acme.com"
      }' | jq
    ```

=== "Python"

    ```python
    import httpx

    def emergency_suspend(reason: str, officer: str, deployer_id: str = "default"):
        r = httpx.post("http://localhost:8080/v1/glad/kill-switch/activate", json={
            "deployer_id": deployer_id,
            "reason": reason,
            "activated_by": officer,
        }, timeout=30)
        r.raise_for_status()
        ev = r.json()
        print(f"suspended — event {ev['event_id']} at {ev['timestamp']}")
        return ev
    ```

=== "TypeScript"

    ```ts
    const ev = await fetch("http://localhost:8080/v1/glad/kill-switch/activate", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        deployer_id: "default",
        reason: "Regulatory audit in progress",
        activated_by: "compliance-officer@acme.com",
      }),
    }).then(r => r.json())
    console.log(ev.event_id, ev.new_status)   // "suspended"
    ```

**What comes back** — the recorded event:

```json
{
  "event_id": "KS-3F9A1C7B2E5D8046A1B2",
  "deployer_id_hash": "9c1f2a7b4e0d6a18",
  "event_type": "activated",
  "timestamp": "2026-06-10T12:00:00+00:00",
  "reason": "Regulatory audit in progress — service suspended per CA SB 942",
  "activated_by": "compliance-officer@acme.com",
  "new_status": "suspended"
}
```

#### Request fields

| Field | Type | Default | Description |
|---|---|---|---|
| `deployer_id` | `string` | `"default"` | Which deployer to suspend. |
| `reason` | `string` | `""` | Justification, stored verbatim. Not enforced by the API — **enforce it in your own tooling**, because a suspension with no recorded reason is an audit gap. |
| `activated_by` | `string` | `null` | Who or what activated it. |

!!! warning "No auto-release"
    Activation is permanent until something calls `deactivate`. There is no timer, no expiry and no scheduled re-enable — if you want a 72-hour window, your own scheduler has to close it.

---

### POST /v1/glad/kill-switch/deactivate

**What it does.** Resumes service. Same body as activate; the recorded `event_type` is `deactivated` and `new_status` is `active`.

=== "curl"

    ```bash
    curl -s -X POST http://localhost:8080/v1/glad/kill-switch/deactivate \
      -H "Content-Type: application/json" \
      -d '{
        "deployer_id":  "default",
        "reason":       "Audit complete. Resuming normal operation.",
        "activated_by": "compliance-officer@acme.com"
      }' | jq
    ```

=== "Python"

    ```python
    httpx.post("http://localhost:8080/v1/glad/kill-switch/deactivate", json={
        "deployer_id": "default",
        "reason": "Audit complete.",
        "activated_by": "compliance-officer@acme.com",
    }).raise_for_status()
    ```

=== "TypeScript"

    ```ts
    await fetch("http://localhost:8080/v1/glad/kill-switch/deactivate", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ deployer_id: "default", reason: "Audit complete.",
                             activated_by: "compliance-officer@acme.com" }),
    })
    ```

!!! info "`activated_by`, not `deactivated_by`"
    Both endpoints share one request model. The field is `activated_by` on deactivation too — the event's `event_type` is what distinguishes the two.

---

## Per-Application kill

The routes above suspend a **deployer** — the whole deployment. To stop one Application while the rest keep serving, use the control-plane kill instead:

```bash
curl -s -X POST http://localhost:8080/v1/glad/apps/support_bot/kill \
  -H "X-Geodesia-Admin-Key: $GEODESIA_ADMIN_TOKEN"
```

That sets the Application's status to `killed`. `pause` / `resume` are the reversible pair for planned work. See [Managing Applications](../studio/applications.md).

---

## Configuration

```yaml
kill_switch:
  enabled: true
  sb942_compliance_hours: 72   # window the SB 942 check measures against
```

Those are the only two keys. There is no reason requirement, no notification address and no maximum window in the configuration — everything else is policy you enforce around the API.

---

## Regulatory Context

| Law | Requirement | Kill Switch Coverage |
|---|---|---|
| EU AI Act Art. 14 | Human operators must be able to interrupt the system | Immediate global interrupt |
| EU AI Act Art. 14(4)(a) | Override capability at any time | Yes — suspends all inference for the deployer |
| CA SB 942 | Suspend service within 72 hours of a regulatory request | `sb942_compliant` / `elapsed_hours` on the status endpoint measure exactly this |
| ISO 42001 §8.4 | Incident response and operational continuity | Every event written to the [audit chain](audit-chain.md) |
| NIST AI RMF GOVERN 1.7 | Mechanisms for disabling AI systems | Yes — documented stop capability |
