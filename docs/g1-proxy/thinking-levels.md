# Thinking Levels — GLAD-G + GLAD-H fusion

**Thinking level** is a per-request dial that trades latency for a second, independent opinion from **GLAD-H** (a Qwen3Guard-Stream backbone running the same Symbiont geometry as [GLAD-Hummingbird](detection-axes.md)) blended into GLAD-G's verdict. Level 0 — the request default — is the GLAD-G-only detector you already know, **byte-identical** to a deployment with thinking levels turned off entirely.

!!! abstract "When to use it"
    Use level 1 for a small, mostly-free quality lift on borderline requests. Use level 2 for the highest-stakes turns where the extra GLAD-H forward pass is worth paying on every request. For everything else, level 0 is the fast default.

---

## The three levels

| Level | Tiers | GLAD-H invoked | Strategy |
|---|---|---|---|
| `0` *(default)* | GLAD-G only | never | Today's detector, unchanged. |
| `1` | GLAD-G + GLAD-H | only when an axis is near GLAD-G's own threshold | **Cascade gray-zone** — GLAD-H is a second opinion consulted only on borderline calls. |
| `2` | GLAD-G + GLAD-H | always | **Max-percentile-OR** — the stronger of the two tiers wins on every axis. |

The pair currently shipped is **GLAD-G-v3 + GLAD-H-v1** — chosen after evaluating 11 fusion strategies on two model-generation red-team benchmarks (garak, ExploitGym); see the [benchmark report](https://github.com/geodesia-ai) for the full comparison. Cascade (level 1) matches or beats Max-OR on most axes at roughly half the latency cost; Max-OR (level 2) is the strategy to reach for when only quality matters.

### Why level 1 is sometimes free

GLAD-H's cost is *per turn*, not per axis: one extra forward pass answers every axis at once. Level 1 only pays that cost when **at least one** axis's GLAD-G percentile falls inside a gray band around GLAD-G's own calibrated threshold — a proxy for "this call is genuinely uncertain." Requests GLAD-G is already confident about (in either direction) never invoke GLAD-H at all, so level 1's *average* latency sits well below level 2's.

---

## Enabling thinking levels

Thinking levels require **two** things — a platform-level model, and a per-request level:

1. **Platform switch** — set `GW_GLADH_CKPT` to a GLAD-H checkpoint path when starting the gateway. This is what makes levels 1/2 *available*; GLAD-H is still lazily loaded only on the first request that actually needs it. Unset (or a missing/unreadable path) → any `thinking_level` in a request silently falls back to level 0. It never turns into a hard error.
2. **Per-request** — set `thinking_level: 1` or `thinking_level: 2` on the request. In G-1 Studio this is the **thinking level** dropdown in the chat panel (*Normal* / *High* / *Extra High Thinking*).

```bash
# start the gateway with GLAD-H available
GW_GLADH_CKPT=/app/runs/glad_bert/glad_bert_gladh_v1_psjbasft_live.pt \
GW_FUSION_BANK=/app/runs/glad_bert/fusion_bank_thinking_v3_gladh_v1.json \
  python -m glad_minimal.gateway.geodesia_gateway --host 0.0.0.0 --port 8800 ...
```

```bash
# per request — ask for the Cascade gray-zone fusion (level 1)
curl -s http://localhost:8800/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-model",
    "stream": false,
    "thinking_level": 1,
    "messages": [{"role": "user", "content": "..."}]
  }'
```

The GPU deployment (`docker-compose.gpu.yml`) ships with `GW_GLADH_CKPT` and `GW_FUSION_BANK` **already set** — level 1/2 are available out of the box on that profile; the request default stays level 0 unless the client asks for more.

---

## Reading the result

When `thinking_level` is 1 or 2, the response's `geodesia` object carries a small block alongside `axis_energy`:

```json
{
  "geodesia": {
    "thinking_level": 1,
    "thinking_tiers_used": ["glad_g", "glad_h"],
    "thinking_escalated": true
  }
}
```

- `thinking_level` echoes what was requested (clamped to `0`/`1`/`2`).
- `thinking_tiers_used` lists which tiers actually contributed to this response — `["glad_g"]` alone means GLAD-H either wasn't invoked (level 1, no escalation) or wasn't available (missing checkpoint/fusion bank).
- `thinking_escalated` (level 1 only) — `true` when at least one axis's percentile fell inside the gray band and GLAD-H was actually consulted for this turn; `false` when GLAD-G alone was confident enough.

Per-axis, a fused axis reports the **fused** `p_detector`/`threshold`/`flag` — there is no separate raw-GLAD-H block like `deep_scan`'s, because the fusion is a single recalibrated score per axis, not two side-by-side opinions.

---

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `GW_GLADH_CKPT` | *(unset)* | Path to the GLAD-H checkpoint. Unset → thinking levels 1/2 fall back to level 0. |
| `GW_GLADH_DEVICE` | `auto` | `auto` picks CUDA when visible, else CPU. GLAD-H runs in its own sub-process (see below) with its own device selection. |
| `GW_FUSION_BANK` | `runs/glad_bert/fusion_bank_thinking_v3_gladh_v1.json` | The calibrated fusion artifact (percentile maps + thresholds for both strategies) built by `scripts/glad_bert/build_thinking_fusion_bank.py`. Missing/unreadable → same fallback as an unset checkpoint. |

!!! warning "Hardware"
    GLAD-H (Qwen3Guard-Stream-0.6B + LoRA) adds a modest amount of VRAM on top of GLAD-Hummingbird — small relative to an upstream generator model, but not zero. It runs in a **separate sub-process** from the main gateway: GLAD-G and GLAD-H read conflicting environment-variable-gated architecture settings at import time, and sharing one Python process would silently corrupt one of the two builds. The sub-process talks to the gateway over a local, unauthenticated stdio pipe — no additional network exposure.
