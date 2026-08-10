# Dashboard

**What it does.** Three read-only endpoints on **G-1 Studio** that answer, in ascending detail: *is the service up?*, *is the deployment healthy?*, *and is it defensible to a regulator?*

| Endpoint | Question it answers |
|---|---|
| `GET /v1/glad/health` | Is the control plane responding? |
| `GET /v1/glad/dashboard` | Which of the seven compliance subsystems are green, yellow or red — and why? |
| `GET /v1/glad/scorecard` | Per-law: compliant, partial, or a gap? |
| `GET /v1/glad/dashboard/charts` | The time series behind the charts. |

---

## Call it

=== "curl"

    ```bash
    curl -s http://localhost:8080/v1/glad/health | jq
    curl -s "http://localhost:8080/v1/glad/dashboard?deployer_id=default" | jq '{overall_health, issues, warnings}'
    curl -s "http://localhost:8080/v1/glad/scorecard" | jq '.scorecard.EU_AI_ACT'
    ```

=== "Python"

    ```python
    import httpx

    c = httpx.Client(base_url="http://localhost:8080", timeout=60)

    dash = c.get("/v1/glad/dashboard", params={"deployer_id": "default"}).json()
    print("overall:", dash["overall_health"])
    for name, sub in dash["systems"].items():
        print(f"  {name:20s} {sub['health']}")
    for issue in dash["issues"]:
        print("  ISSUE:", issue)
    for w in dash["warnings"]:
        print("  warn :", w)

    card = c.get("/v1/glad/scorecard").json()
    for law, entry in card["scorecard"].items():
        print(f"{law:16s} {entry['status']}")
    ```

=== "TypeScript"

    ```ts
    const dash = await fetch("http://localhost:8080/v1/glad/dashboard?deployer_id=default")
      .then(r => r.json())

    console.log("overall:", dash.overall_health)
    for (const [name, sub] of Object.entries<any>(dash.systems)) {
      console.log(` ${name}: ${sub.health}`)
    }
    dash.issues.forEach((i: string) => console.error("ISSUE:", i))
    ```

---

## GET /v1/glad/dashboard

**What comes back**

```json
{
  "timestamp": "2026-06-18T09:00:00+00:00",
  "deployer_id": "default",
  "overall_health": "yellow",
  "systems": {
    "kill_switch":     { "health": "green",  "status": "active", "sb942_compliant": true,
                         "elapsed_hours": null, "total_events": 3 },
    "chain_integrity": { "health": "green",  "valid": true, "total_entries": 14821,
                         "broken_at": null, "last_hash": "a8b3c1…" },
    "retention":       { "health": "green",  "total_calls": 48291, "overdue_calls": 0,
                         "policy": { "default_months": 6 } },
    "human_oversight": { "health": "yellow", "total_reviews": 142, "pending_reviews": 14,
                         "italy_132_pending_notification": 0, "by_level": { "1": 120, "2": 20, "3": 2 } },
    "watermark":       { "health": "green",  "total_disclosures": 48180, "total_calls": 48291,
                         "coverage_pct": 99.8 },
    "notifications":   { "health": "green" },
    "fria":            { "health": "green",  "active_frias": 1, "total_frias": 2 }
  },
  "issues": [],
  "warnings": ["Pending Level-3 reviews awaiting Italy 132/2025 notification."]
}
```

### How the health colour is decided

Every subsystem reports its own `health`, and the deployment's `overall_health` is the worst of them:

| Colour | Meaning | What produces it |
|---|---|---|
| 🔴 `red` | Something is **broken** — it goes in `issues` | Audit chain fails verification · the SB 942 72-hour window has been exceeded · more than 10% of calls are past their retention window |
| 🟡 `yellow` | Something needs **attention** — it goes in `warnings` | The kill switch is currently engaged · Level-3 reviews awaiting notification · overdue retention under 10% · low watermark coverage · no active FRIA while calls exist |
| 🟢 `green` | Nominal | Everything above is clear |

`issues` and `warnings` are **human-readable strings**, already written for an operator. Show them; do not try to parse them.

!!! info "The dashboard verifies the chain on every call"
    `chain_integrity` is computed by re-walking the whole HMAC chain, exactly as [`/chain/verify`](audit-chain.md) does. On a large ledger that takes seconds — poll this endpoint on a sensible interval, not on every page render.

### Query parameters

| Parameter | Default | Description |
|---|---|---|
| `deployer_id` | *(the configured provider deployer id, else `"default"`)* | Which deployment to report on. |

---

## GET /v1/glad/scorecard

**What it does.** Re-expresses the same seven subsystem health values as a per-law verdict. It runs the dashboard internally, so it costs the same chain verification.

**What comes back**

```json
{
  "generated_at": "2026-06-18T09:00:00+00:00",
  "deployer_id": "default",
  "scorecard": {
    "EU_AI_ACT": {
      "status": "partial",
      "checks": {
        "Art 12 record_keeping":  "compliant",
        "Art 14 human_oversight": "partial",
        "Art 18 retention":       "compliant",
        "Art 27 fria":            "compliant",
        "Art 50_2 watermark":     "compliant"
      }
    }
  }
}
```

`status` and every entry in `checks` is one of **`compliant`** / **`partial`** / **`gap`** — the direct mapping of green / yellow / red. A law's overall `status` is the worst of its checks.

!!! warning "This is a self-assessment, not a certification"
    The scorecard reports what *this deployment can evidence about itself*: chain intact, reviews closed, retention honoured, FRIA present, watermarks applied. It is the input to a compliance conversation, not the output of one.

### Query parameters

| Parameter | Default | Description |
|---|---|---|
| `deployer_id` | *(as above)* | Which deployment to score. |

---

## GET /v1/glad/dashboard/charts

The time series behind the dashboard's charts — passed, blocked, hallucinated and unsafe volumes over time.

| Parameter | Description |
|---|---|
| `deployer_id` | Scope to one deployer. |
| `application_id` | Scope to one Application. |

---

## Health checks

### GET /v1/glad/health

The control-plane liveness probe. No database access.

```json
{ "status": "ok", "version": "1.0.0" }
```

### GET /health

Studio's process-level health, with a public configuration snapshot attached. Mounted **outside** `/v1`, so it is reachable on Studio's own port (`:8199`) rather than through the unified port on 8080.

G1-Proxy has its own, separate health check at `GET /health` on `:8800` (`/gw/health` through the unified port) — see the [G1-Proxy API map](../g1-proxy/api-reference.md#status-health). A full liveness check probes **both**.
