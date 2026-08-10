# Thinking Levels

**Thinking level** is a per-request dial that trades a little latency for a stricter, more careful verdict from **G1-Hummingbird**. It is a single integer on the request body — `thinking_level` — and it changes *how hard the detector thinks* about the turn, not *what* it reports: the response shape, the axis names and the thresholds are identical at every level.

Level `0` is the default and is what you get if you never send the field at all.

!!! abstract "TL;DR"
    Leave it at `0` for ordinary traffic. Raise it to `1` when you want a stricter opinion but only on the calls that are genuinely borderline. Use `2` for high-stakes turns. Use `3` — **MAX** — when correctness matters more than latency and you want the strictest verdict the product can produce.

---

## Call it

=== "curl"

    ```bash
    curl -s http://localhost:8080/gw/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{
        "model": "my-model",
        "stream": false,
        "thinking_level": 3,
        "messages": [{"role": "user", "content": "Summarise the contract clause and tell me if it is enforceable."}]
      }' | jq '{decision: .glad_decision, level: .geodesia.thinking_level, axes: .geodesia.axis_energy}'
    ```

=== "Python"

    ```python
    import httpx

    r = httpx.post(
        "http://localhost:8080/gw/v1/chat/completions",
        json={
            "model": "my-model",
            "stream": False,
            "thinking_level": 3,          # 0 (default) … 3 (MAX)
            "messages": [{"role": "user", "content": "Summarise the contract clause…"}],
        },
        timeout=120,
    )
    body = r.json()
    print(body["glad_decision"], body["geodesia"]["thinking_level"])
    ```

=== "TypeScript"

    ```ts
    const res = await fetch("http://localhost:8080/gw/v1/chat/completions", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: "my-model",
        stream: false,
        thinking_level: 3,               // 0 (default) … 3 (MAX)
        messages: [{ role: "user", content: "Summarise the contract clause…" }],
      }),
    })
    const body = await res.json()
    console.log(body.glad_decision, body.geodesia.thinking_level)
    ```

### What comes back

The usual OpenAI-shaped response, with the `geodesia` object carrying two extra fields:

```json
{
  "glad_decision": "passed",
  "geodesia": {
    "thinking_level": 3,
    "thinking_escalated": true,
    "axis_energy": {
      "prompt_safety": { "p_detector": 0.02, "flag": false, "threshold": 0.9215, "available": true },
      "halluc_context": { "p_detector": 0.41, "flag": false, "threshold": 0.6475, "available": true }
    }
  }
}
```

| Field | When present | Meaning |
|---|---|---|
| `thinking_level` | only when a level ≥ 1 was requested *and* this deployment could serve it | The level this turn actually ran at. **Absent** means the turn was served on the standard path — either you did not ask for a level, or the deployment cannot serve the one you asked for. |
| `thinking_escalated` | level 1 only | `true` when the turn was uncertain enough to do the extra work; `false` when the standard path was confident and the extra work was skipped. |
| `thinking_tiers_used` | levels ≥ 1 | Internal diagnostic — opaque identifiers for the internal stages that contributed. Not part of the stable contract; do not branch on it. |

Nothing else about the payload changes. A fused axis still reports one `p_detector`, one `threshold` and one `flag` — the level affects how that number was reached, not how you read it.

---

## The four levels

| Level | Name | Extra work | When to use it |
|---|---|---|---|
| `0` | **Standard** *(default)* | none | Ordinary traffic. Lowest latency; the calibrated detector you get with no configuration at all. |
| `1` | **Careful** | only on turns the detector is *unsure* about | Broad quality lift for near-zero average cost. Confident calls behave exactly like level 0. |
| `2` | **High** | on every turn | High-stakes traffic where you would rather pay the latency on every request than miss a borderline call. |
| `3` | **MAX** | on every turn, maximum depth | The strictest verdict the product produces. Legal, medical, financial, agentic tool-use — anywhere a miss is expensive. |

### Why level 1 is nearly free

Level 1 only does the extra work when at least one axis lands inside a *gray band* around its own calibrated threshold — a direct measure of "this call is genuinely uncertain". Requests the detector is already confident about (in either direction) are served on the level-0 path, so level 1's **average** latency sits far closer to level 0's than to level 2's. Whether escalation happened for a given turn is reported back as `thinking_escalated`.

Levels 2 and 3 escalate unconditionally, so the extra work is always paid there and `thinking_escalated` is not reported.

### Level 3 is additive

Level 3 does not *replace* level 2 — it adds depth on top of it. If the level-3 capability cannot answer for a given turn (pack absent, worker unavailable, an axis it has no opinion on), that turn is still served at the deepest stage that *did* answer. A level-3 request never degrades below what a level-2 request would have produced.

---

## Enabling levels above 0

Two things have to line up — a deployment-level capability, and a per-request level.

1. **Deployment** — the extended-thinking capability packs must be present on the machine serving the proxy. Point `GW_GLADH_CKPT` at the pack that unlocks levels **1–2**, and `GW_GLADA_CKPT` at the pack that unlocks level **3**. Both are lazily loaded: a pack that is configured but never requested costs nothing.
2. **Per request** — send `thinking_level` on the body (or pick the level in the Studio chat panel's **Thinking level** dropdown).

```bash
# proxy started with levels 1-3 available
GW_GLADH_CKPT=/app/runs/glad_bert/glad_bert_gladh_v1_psjbasft_live.pt \
GW_GLADA_CKPT=/app/runs/glad_bert/glad_a_agentdog_v1.pt \
GW_FUSION_BANK=/app/runs/glad_bert/fusion_bank_thinking_v3_gladh_v1.json \
  python -m glad_minimal.gateway.geodesia_gateway --host 0.0.0.0 --port 8800
```

!!! info "Graceful degradation, never a hard error"
    If a requested level is not available on this deployment — pack missing, path unreadable, calibration artifact absent — the turn is served at the deepest level that *is* available, in the worst case level 0. A client asking for level 3 against a level-0 deployment gets a correct answer, not a 4xx. The only way to tell from the client side is `geodesia.thinking_level`: if it is **absent** the turn ran on the standard path, whatever you asked for.

The GPU deployment (`docker-compose.gpu.yml`, and the `--gpu` installer profile) ships with these variables **already set**, so levels 1–3 are available out of the box there. The request default stays `0` unless the client asks for more.

---

## Configuration reference

| Variable | Default | Description |
|---|---|---|
| `GW_GLADH_CKPT` | *(unset)* | Capability pack that unlocks thinking levels **1** and **2**. Unset → those levels fall back to level 0. |
| `GW_GLADA_CKPT` | *(unset)* | Capability pack that unlocks thinking level **3 (MAX)**. Unset → level 3 falls back to the highest available level. |
| `GW_GLADH_DEVICE` | `auto` | Device for the level-1/2 pack. `auto` picks CUDA when visible, else CPU. |
| `GW_GLADA_DEVICE` | `auto` | Device for the level-3 pack. |
| `GW_FUSION_BANK` | `runs/glad_bert/fusion_bank_thinking_v3_gladh_v1.json` | Calibration artifact that keeps the higher levels on the *same* probability scale as level 0 — which is why thresholds do not have to be re-tuned per level. Missing/unreadable → the higher levels fall back to level 0. |

### Request field

| Field | Type | Default | Notes |
|---|---|---|---|
| `thinking_level` | `integer` | `0` | Clamped to `0…3`. Values above the maximum are clamped, not rejected. |
| `glad_thinking_level` | `integer` | — | Accepted alias, for clients that already namespace their extension fields. |

!!! warning "Hardware"
    Levels above 0 add VRAM on top of the always-on detector — modest next to an upstream generator model, but not zero. The extra capacity runs in its **own sub-process**, isolated from the always-on detector, and talks to the proxy over a local pipe. No additional network surface is opened.

---

## Choosing a level in practice

- **Per route, not per user.** The level is a property of *what the call does*, not of who made it. A support-chat endpoint at `0`, a contract-analysis endpoint at `3`.
- **Measure before you raise it globally.** Send a sample of your real traffic at level `1` and at level `2` and compare `axis_energy` against the level-0 baseline. [Policy Lens](../studio/policy-lens.md) does exactly this comparison over stored traffic.
- **Do not use it as a threshold substitute.** If a whole axis is too loose or too tight for your domain, move the threshold ([Detection Thresholds](../reference/thresholds.md)); the thinking level changes confidence, not policy.
