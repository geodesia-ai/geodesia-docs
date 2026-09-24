# Thinking Levels

**Thinking level** is a per-request dial that trades a little latency for a stricter, more careful verdict from **G1-Hummingbird**. It is a single integer on the request body — `thinking_level` — and it changes *how hard the detector thinks* about the turn, not *what* it reports: the response shape, the axis names and the thresholds are identical at every level.

Level `2` (**Extra High Thinking**) is the default since 2026-09-18: it is what you get if you never send the field at all. Send `0` (**Low Latency**) explicitly when you need the fastest verdict and accept a weaker one on borderline traffic.

!!! abstract "TL;DR"
    Leave the field out (level `2`, Extra High Thinking) for ordinary traffic. Send `0` (Low Latency) only when latency matters more than accuracy. Use `1` for a stricter opinion only on the calls that are genuinely borderline. Use `2` for high-stakes turns. Use `3` — **MAX** — when correctness matters more than latency and you want the strictest verdict the product can produce.

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
      }' | jq '{decision: .geodesia.decision, thinking: .geodesia.thinking, axes: .geodesia.axes}'
    ```

=== "Python"

    ```python
    import httpx

    r = httpx.post(
        "http://localhost:8080/gw/v1/chat/completions",
        json={
            "model": "my-model",
            "stream": False,
            "thinking_level": 3,          # 0 (Low Latency) … 3 (Max); omitted = 2 (default)
            "messages": [{"role": "user", "content": "Summarise the contract clause…"}],
        },
        timeout=120,
    )
    g = r.json()["geodesia"]
    thinking = g.get("thinking")           # absent → the turn ran on the level-0 path
    print(g["decision"], thinking and thinking["level"], thinking and thinking["tiers_used"])
    ```

=== "TypeScript"

    ```ts
    const res = await fetch("http://localhost:8080/gw/v1/chat/completions", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: "my-model",
        stream: false,
        thinking_level: 3,               // 0 (Low Latency) … 3 (Max); omitted = 2 (default)
        messages: [{ role: "user", content: "Summarise the contract clause…" }],
      }),
    })
    const g = (await res.json()).geodesia
    console.log(g.decision, g.thinking?.level, g.thinking?.tiers_used)
    ```

### What comes back

The usual OpenAI-shaped response, with the `geodesia` object carrying one extra section, `thinking`:

```json
{
  "geodesia": {
    "schema_version": "1.0",
    "event": "final",
    "decision": "allowed",
    "mode": "blocking",
    "reason": null,
    "axes": {
      "prompt_safety": { "score": 0.02, "threshold": 0.6377, "flagged": false, "available": true, "role": "enforce" },
      "halluc_context": { "score": 0.41, "threshold": 0.7551, "flagged": false, "available": true, "role": "enforce" }
    },
    "thinking": { "level": 3, "tiers_used": ["geodesia_a", "geodesia_g", "geodesia_h"] }
  }
}
```

| Field | When present | Meaning |
|---|---|---|
| `thinking` | only when a level ≥ 1 was requested *and* this deployment could serve it | **Absent** means the turn was served on the standard (level-0) path — either you asked for level 0, or the deployment cannot serve levels above 0. |
| `thinking.level` | with `thinking` | The level this turn actually ran at. |
| `thinking.tiers_used` | with `thinking` | The detector tiers that **actually** contributed: `geodesia_g` (the always-on detector), `geodesia_h` (levels 1–2 pack), `geodesia_a` (level-3 pack), `native`. It can list fewer tiers than the level implies — e.g. a level-3 turn with no `geodesia_a` ran at level-2 depth. |
| `thinking.escalated` | level 1 only | `true` when the turn was uncertain enough to do the extra work; `false` when the standard path was confident and the extra work was skipped. |

Nothing else about the payload changes. A fused axis still reports one `score`, one `threshold` and one `flagged` — the level affects how that number was reached, not how you read it.

!!! note "Fusion can only add a flag, never remove one"
    At level ≥ 1 the axis `score` shown is the fused score. If the fused score alone would *not* cross the threshold but the level-0 verdict did, the axis stays `flagged: true` and carries `details.flag_kept_from_level_0: true`. Paying for a higher level can never release a turn that level 0 would have caught. See [`details` fields](../reference/response-format.md#details-fields).

---

## The four levels

| Level | Name | Extra work | When to use it |
|---|---|---|---|
| `0` | **Low Latency** | none | Lowest latency: Geodesia-G only. Trades accuracy for speed on borderline traffic; ask for it explicitly. |
| `1` | **High Thinking** | only on turns the detector is *unsure* about | Broad quality lift for near-zero average cost. Confident calls behave exactly like level 0. |
| `2` | **Extra High Thinking** *(default)* | on every turn | High-stakes traffic where you would rather pay the latency on every request than miss a borderline call. |
| `3` | **Max Thinking** | on every turn, maximum depth | The strictest verdict the product produces. Legal, medical, financial, agentic tool-use — anywhere a miss is expensive. |

### Why level 1 is nearly free

Level 1 only does the extra work when at least one axis lands inside a *gray band* around its own calibrated threshold — a direct measure of "this call is genuinely uncertain". Requests the detector is already confident about (in either direction) are served on the level-0 path, so level 1's **average** latency sits far closer to level 0's than to level 2's. Whether escalation happened for a given turn is reported back as `thinking.escalated`.

Levels 2 and 3 escalate unconditionally, so the extra work is always paid there and `thinking.escalated` is not reported.

### Level 3 is additive

Level 3 does not *replace* level 2 — it adds depth on top of it. If the level-3 capability cannot answer for a given turn (pack absent, worker unavailable, an axis it has no opinion on), that turn is still served at the deepest stage that *did* answer, and `thinking.tiers_used` says which tiers that was. A level-3 request never degrades below what a level-2 request would have produced.

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
    If a requested level is not available on this deployment — pack missing, path unreadable, calibration artifact absent — the turn is served at the deepest level that *is* available, in the worst case level 0. A client asking for level 3 against a level-0 deployment gets a correct answer, not a 4xx. The way to tell from the client side is `geodesia.thinking`: if it is **absent** the turn ran on the standard path, whatever you asked for; if it is present, `thinking.tiers_used` lists the tiers that actually contributed.

The GPU deployment (`docker-compose.gpu.yml`, and the `--gpu` installer profile) ships with these variables **already set**, so levels 1–3 are available out of the box there. The request default is level `2` unless the client asks for another level.

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
| `thinking_level` | `integer` | `2` | Clamped to `0…3`. Values above the maximum are clamped, not rejected. The deployment default can be changed with `GW_DEFAULT_THINKING_LEVEL`. |

!!! warning "Hardware"
    Levels above 0 add VRAM on top of the always-on detector — modest next to an upstream generator model, but not zero. The extra capacity runs in its **own sub-process**, isolated from the always-on detector, and talks to the proxy over a local pipe. No additional network surface is opened.

---

## Choosing a level in practice

- **Per route, not per user.** The level is a property of *what the call does*, not of who made it. A support-chat endpoint at `0`, a contract-analysis endpoint at `3`.
- **Measure before you raise it globally.** Send a sample of your real traffic at level `1` and at level `2` and compare `geodesia.axes` against the level-0 baseline. [Policy Lens](../studio/policy-lens.md) does exactly this comparison over stored traffic.
- **Do not use it as a threshold substitute.** If a whole axis is too loose or too tight for your domain, move the threshold ([Detection Thresholds](../reference/thresholds.md)); the thinking level changes confidence, not policy.
