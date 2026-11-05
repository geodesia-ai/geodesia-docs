# Thinking Levels

**Thinking level** is a per-request dial that trades a little latency for a stricter, more careful verdict from **G1-Hummingbird**. It is a single integer on the request body — `thinking_level` — and it changes *how many detector tiers* judge the turn, not *what* is reported: the response shape and the axis names are identical at every level. What changes is who decides `prompt_safety` and `jailbreak` (see [What changes at levels 1–3](#what-changes-at-levels-1-3)).

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
    "schema_version": "1.1",
    "event": "final",
    "decision": "allowed",
    "mode": "blocking",
    "reason": null,
    "axes": {
      "prompt_safety": { "score": 0.02, "threshold": 0.5, "flagged": false, "available": true, "role": "enforce" },
      "halluc_context": { "score": 0.41, "threshold": 0.8164, "flagged": false, "available": true, "role": "enforce" }
    },
    "thinking": {
      "level": 3,
      "tiers_used": ["geodesia_a", "geodesia_g", "geodesia_h"],
      "tier_status": { "geodesia_h": { "status": "ok" }, "geodesia_a": { "status": "ok" } },
      "complete": true
    }
  }
}
```

| Field | When present | Meaning |
|---|---|---|
| `thinking` | only when a level ≥ 1 was requested *and* this deployment could serve it | **Absent** means the turn was served on the standard (level-0) path — either you asked for level 0, or the deployment cannot serve levels above 0. |
| `thinking.level` | with `thinking` | The level this turn actually ran at. |
| `thinking.tiers_used` | with `thinking` | The detector tiers that **actually** contributed: `geodesia_g` (the always-on detector), `geodesia_h` (levels 1–2 pack), `geodesia_a` (level-3 pack), `native`. It can list fewer tiers than the level implies — e.g. a level-3 turn with no `geodesia_a` ran at level-2 depth. |
| `thinking.escalated` | level 1 only | `true` when Geodesia-H was consulted; `false` when a gray-band calibration is installed and Geodesia-G was confident enough to skip it. |
| `thinking.tier_status` | with `thinking` | For each higher tier the level asked for, why it did or did not contribute: `ok`, `pending`, `starting`, `not_needed`, `not_woken`, `bank_missing`, `not_installed`, `failed_to_start`, `runtime_error`, `not_consulted`, `no_axes`, `no_contribution`. |
| `thinking.user_thresholds` | a threshold was chosen | The shift your chosen threshold applied to the fused decision (see below). |
| `thinking.complete` | with `thinking` | `false` on a partial streaming frame, while a requested tier has not answered yet. |

Nothing else about the payload changes. A fused axis still reports one `score`, one `threshold` and one `flagged` — the level affects how that number was reached, not how you read it.

---

## What changes at levels 1–3 { #what-changes-at-levels-1-3 }

**`prompt_safety` and `jailbreak` — one fused decision.** At levels 1–3 the prompt is judged by the tiers of the level together, and their verdict **replaces** the level-0 one for these two axes:

| Level | Who decides `prompt_safety` / `jailbreak` |
|---|---|
| `0` | Geodesia-G: the prompt is flagged when either axis crosses its own threshold. |
| `1` | Geodesia-H, with its own calibrated thresholds (same rule). |
| `2` | Geodesia-G and Geodesia-H fused: for each tier, the margin of the riskier of the two axes over that tier's threshold; the turn is flagged when the **mean of the margins** crosses the level-2 threshold. |
| `3` | Geodesia-G, Geodesia-H and Geodesia-A fused the same way, with the level-3 threshold. |

The axis that decided carries the fused score, rescaled so that the decision point is **`threshold: 0.5`**; the other one is reported just below it. Because two or three detectors vote, a higher level can also *clear* a prompt that Geodesia-G alone would have flagged: that is the point of asking more tiers, and it is why the thresholds of each level are calibrated on their own.

**The other detector axes — any tier can add a flag.** `answer_safety`, `rag_jailbreak`, `profanity` and `out_of_scope` are judged by every tier of the level against that tier's own threshold, and the axis is flagged when **at least one** tier crosses. The shown score is the highest margin, rescaled to `threshold: 0.5`. If the fused reading would not flag but the level-0 verdict did, the axis stays `flagged: true` with `details.flag_kept_from_level_0: true`: for these axes a higher level never removes a flag. (Geodesia-A does not take part in `rag_jailbreak`; `GB_LIVELLI_ESCLUDI` controls such exclusions.)

**Grounding is the same at every level.** `halluc_context` is read by the grounding model and `halluc_closedbook` by SLEDGE, identically at every level; the tiers do not fuse them.

**Your threshold still applies.** A threshold chosen on the Application (or with `threshold_overrides`) for a fused axis is translated into a shift of the decision, relative to Geodesia-G's default, and applied to every tier. The shift is reported in `thinking.user_thresholds`.

**Long inputs.** Every tier reads long prompts and documents in [sliding windows](detection-axes.md#long-inputs-sliding-windows), the same way Geodesia-G does.

---

## The four levels

| Level | Name | Extra work | When to use it |
|---|---|---|---|
| `0` | **Low Latency** | none | Lowest latency: Geodesia-G only. Trades accuracy for speed on borderline traffic; ask for it explicitly. |
| `1` | **High Thinking** | Geodesia-H | A second detector's opinion, with its own thresholds, at the cost of one tier. |
| `2` | **Extra High Thinking** *(default)* | Geodesia-G + Geodesia-H, fused | High-stakes traffic where you would rather pay the latency on every request than miss a borderline call. |
| `3` | **Max Thinking** | Geodesia-G + Geodesia-H + Geodesia-A, fused | The strictest verdict the product produces. Legal, medical, financial, agentic tool-use — anywhere a miss is expensive. |

### Level 1 and the gray band

A deployment can ship a **gray-band** calibration for level 1. With it, Geodesia-H is consulted only when at least one axis of Geodesia-G lands close to its threshold, and confident turns are served on the level-0 path — level 1's average latency then sits close to level 0's. Without it (the default in release 1.0.10) level 1 always consults Geodesia-H. `thinking.escalated` tells you which happened on each turn.

Levels 2 and 3 always consult their tiers, so `thinking.escalated` is not reported there.

### Level 3 is additive

Level 3 does not *replace* level 2 — it adds Geodesia-A on top of it. If Geodesia-A cannot answer for a given turn (pack absent, worker unavailable, an axis it has no opinion on), that turn is decided at level 2 with Geodesia-G and Geodesia-H, and `thinking.tiers_used` / `thinking.tier_status` say so. In streaming, the level-2 verdict is sent as a `progress` event before Geodesia-A answers.

---

## Enabling levels above 0

Two things have to line up — a deployment-level capability, and a per-request level.

1. **Deployment** — the extended-thinking capability packs must be present on the machine serving the proxy. Point `GW_GLADH_CKPT` at the pack that unlocks levels **1–2**, and `GW_GLADA_CKPT` at the pack that unlocks level **3**. Both are lazily loaded: a pack that is configured but never requested costs nothing.
2. **Per request** — send `thinking_level` on the body (or pick the level in the Studio chat panel's **Thinking level** dropdown).

```bash
# proxy started with levels 1-3 available
GW_GLADH_CKPT=/app/runs/glad_bert/<geodesia-h pack>.pt \
GW_GLADA_CKPT=/app/runs/glad_bert/<geodesia-a pack>.pt \
GB_CALIB_DIR=/app/runs/glad_bert \
  python -m glad_minimal.gateway.geodesia_gateway --host 0.0.0.0 --port 8800
```

!!! info "Graceful degradation, never a hard error"
    If a requested level is not available on this deployment — pack missing, path unreadable, calibration artifact absent — the turn is served at the deepest level that *is* available, in the worst case level 0. A client asking for level 3 against a level-0 deployment gets a correct answer, not a 4xx. The way to tell from the client side is `geodesia.thinking`: if it is **absent** the turn ran on the standard path, whatever you asked for; if it is present, `thinking.tiers_used` lists the tiers that actually contributed.

The GPU deployment (`docker-compose.gpu.yml`, the `--gpu` installer profile) and [G1-Proxy Light](light.md) ship with these variables **already set**, so levels 1–3 are available out of the box. The request default is level `2` unless the client asks for another level.

---

## Configuration reference

| Variable | Default | Description |
|---|---|---|
| `GW_GLADH_CKPT` | *(unset)* | Capability pack that unlocks thinking levels **1** and **2**. Unset → those levels fall back to level 0. |
| `GW_GLADA_CKPT` | *(unset)* | Capability pack that unlocks thinking level **3 (MAX)**. Unset → level 3 falls back to the highest available level. |
| `GW_GLADH_DEVICE` | `auto` | Device for the level-1/2 pack. `auto` picks CUDA when visible, else CPU. |
| `GW_GLADA_DEVICE` | `auto` | Device for the level-3 pack. |
| `GB_CALIB_DIR` | `runs/glad_bert` | Directory holding each tier's thresholds (`axis_calib.json`, `axis_calib_glad_h.json`, `axis_calib_glad_a.json`), the level-2 and level-3 fusion files (`fusione_L2.json`, `fusione_L3.json`) and, optionally, the level-1 gray band (`cascata_L1.json`). Missing files → the higher levels are served by Geodesia-G and say so (`tier_status: bank_missing`). |
| `GB_TIER_START_TIMEOUT` | `180` (`900` in the shipped images) | Seconds a higher tier may take to start before it is declared `failed_to_start`. Geodesia-A is a large model: a short timeout used to make level 3 fall back to level 2. |
| `GB_LIVELLI_ESCLUDI` | `glad_a:rag_jailbreak` | Tier/axis pairs left out of the per-tier maximum (`tier:axis,tier:axis`; `""` includes all). |

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
