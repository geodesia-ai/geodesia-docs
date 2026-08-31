# Token & Cost Control

Two of the nine [detection axes](detection-axes.md) answer a commercial question rather than a safety
one. **`out_of_scope`** refuses off-topic traffic *before* the upstream call, so those tokens are never
billed. **`prompt_complexity`** routes easy prompts to a cheap model and hard ones to a capable one.
Both come from the pass you are already paying for — no extra latency, no extra model call.

---

## `out_of_scope` — refusing before you pay

An LLM charges you for every token it reads and every token it writes. A question your assistant was never meant to answer costs you the full price of a wrong answer — and then costs you again in support, in trust, and (for a regulated deployment) in an audit record you have to explain.

`out_of_scope` moves that decision **in front of** the upstream call. When the axis is part of the blocking set, the gateway returns a refusal directly and the request never leaves the building:

```
prompt ──▶ detector (one pass) ──▶ out_of_scope 0.998 ──▶ refusal
                                                          │
                                        upstream LLM ─────┘ never called
                                        input tokens: 0
                                        output tokens: 0
```

### Declaring the scope

The axis is **conditional by construction**: "off topic" has no meaning until somebody says what the topic is. The scope is read from the **system message(s) of the incoming request**, and in G-1 Studio the Application's `policy.scope` field supplies the default:

```bash
curl -s -X PUT http://localhost:8080/v1/glad/apps/travel_support/policy \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "scope": "You are a customer-support assistant for a travel booking website. You help travellers search flights, hotels and car rentals, change or cancel bookings, and explain fare rules, baggage allowances and refund policies. Stay strictly on travel topics."
    }
  }'
```

A client that sends its own system message overrides it for that conversation. Geodesia's own injected constitutional prompt is added *after* this read and can never be mistaken for the scope.

!!! warning "No scope, no axis"
    Without a declared scope the axis sits at 0.03–0.12 and never fires — correctly, because nothing is off-topic when there is no topic. With a scope of even 20 characters it reads ≈ 0.999 for an off-topic question and ≈ 0.0009 for an in-scope one. If `out_of_scope` looks dead in your deployment, check the scope before checking the model.

### Turning it into an actual block

`out_of_scope` is an [additional axis](detection-axes.md#primary-axes-vs-additional-axes): it ships
**annotate-only**, and the gateway-wide `GW_PROMPT_BLOCK_AXES` **will not** promote it — that switch is
global and silent, and an annotate-grade signal should not gain the power to refuse every customer's
traffic from one line in a deployment file.

Enable it where the decision belongs: on the **Application** that wants it, through its enforcement policy.

```bash
curl -X PATCH "$G1/v1/glad/apps/$APP_ID" \
  -H "Authorization: Bearer $ADMIN_KEY" -H "Content-Type: application/json" \
  -d '{"policy": {"enforcement": {"out_of_scope": "block"}}}'
```

(The same control is a dropdown in g1-studio → **Applications → Policy → Enforcement**.) Scoping it to one
Application is the point: an assistant with a tight declared scope wants the refusal, a general-purpose one
does not, and the two can share a gateway.

The refusal comes back in the shape the caller expects — an OpenAI `finish_reason: "content_filter"` response, an Ollama `done` message, or a stream frame for streaming clients — with the full verdict attached:

```json
{
  "id": "geodesia-block-1754131200000",
  "choices": [{"index": 0,
               "message": {"role": "assistant",
                           "content": "[Geodesia blocked — out of scope (input)]"},
               "finish_reason": "content_filter"}],
  "glad_decision": "blocked",
  "geodesia": {
    "glad_decision": "blocked",
    "flagged_axis": "out_of_scope",
    "axis_energy": {"out_of_scope": {"p_detector": 0.998, "threshold": 0.90, "flag": true}}
  }
}
```

The blocked call is still written to the audit ledger with its axis scores — you keep the compliance record without paying for the generation.

### What it is worth

The saving is exact and you can read it off your own traffic: every request the axis refuses is a request whose input **and** output tokens you did not buy. Because the gateway logs blocked calls with `prompt_blocked = 1`, the [Cost & FinOps](../studio/cost.md) view can show refused volume next to spend, and the Application metrics endpoint reports it directly:

```bash
curl -s http://localhost:8080/v1/glad/apps/travel_support/metrics | jq
```

```json
{ "total": 12840, "prompt_blocked": 1173, "answer_blocked": 12, "hallucinated": 41, "grounded": 9902 }
```

!!! tip "Tune it on your own traffic first"
    Do not guess the threshold. Run the axis in `annotate` mode for a few days, then open [Policy Lens](../studio/policy-lens.md) and drag the `out_of_scope` threshold: it re-decides every stored request and tells you exactly which real questions you would have refused. Promote the axis to `block` once that list contains nothing you wanted to answer.

---

## Complexity routing (Model A → Model B)

Most production traffic is easy. *"What time is it in Rome?"*, *"Where is my order?"*, *"Summarise this paragraph"* — these do not need the frontier model you are paying frontier prices for. A minority of prompts genuinely do.

`prompt_complexity` is a binary **complex / simple** classifier over the user's prompt. When routing is enabled, the gateway compares it against a threshold and picks the upstream binding accordingly:

| | Model A (primary binding) | Model B (`complex_binding`) |
|---|---|---|
| Answers | `p(prompt_complexity) ≤ threshold` | `p(prompt_complexity) > threshold` |
| Typical role | small / cheap / local | large / capable / expensive |
| Configuration | the Application's `binding` | a **partial** override — unset fields inherit Model A |
| Billing rate | `cost.input_per_mtok` / `cost.output_per_mtok` | `cost.complex_input_per_mtok` / `cost.complex_output_per_mtok` (falls back to Model A's rates) |

The common case is *same provider, same key, different model* — so setting `complex_binding.model` alone is enough. Model B can also be a completely different provider (a local Ollama for the cheap tier, a hosted API for the hard tier), in which case you set its `upstream_type`, `base_url` and key too.

### Where it runs in the pipeline

Routing happens **after** the input-block decision and **before** generation:

1. one detector pass → safety axes + `out_of_scope` + `prompt_complexity`;
2. if an input guardrail fired → refuse (a blocked prompt is never routed — you do not pay to route something you refused);
3. if `p(prompt_complexity) > threshold` → swap in Model B's binding;
4. forward, generate, validate the answer as usual.

The model that **actually answered** is echoed in the response payload and written to the cost ledger — not the configured default — so your FinOps numbers stay honest when routing is on.

### Configuring it

In **G-1 Studio → Applications**, the *"Route complex prompts to a different model"* toggle reveals the threshold slider and the Model B fields, with a **↻ discover** button that lists the models available on that upstream. If the served checkpoint has no `prompt_complexity` axis, the Studio shows an amber warning and the controls stay inert.

Over the API:

```bash
curl -s -X PUT http://localhost:8080/v1/glad/apps/support_bot/routing \
  -H "Content-Type: application/json" \
  -d '{
    "complex_routing": {
      "enabled": true,
      "threshold": 0.5,
      "complex_binding": { "model": "gpt-5" }
    }
  }'
```

```json
{
  "complex_routing": {
    "enabled": true,
    "threshold": 0.5,
    "complex_binding": {"upstream_type": null, "base_url": null, "model": "gpt-5",
                        "region": null, "api_key": "***"}
  },
  "config_version": 7
}
```

| Field | Default | Meaning |
|---|---|---|
| `enabled` | `false` | Master switch. Off ⇒ Model A only, and no routing logic runs at all. |
| `threshold` | `0.5` | `p(prompt_complexity)` **strictly greater** than this routes to Model B. `0.5` is the classifier's training boundary. |
| `complex_binding.model` | `""` | The only field usually set. Empty ⇒ no real routing (Model B would be Model A). |
| `complex_binding.upstream_type` / `base_url` / `region` / `api_key` | `null` | Optional overrides; unset fields inherit Model A. A masked or omitted `api_key` on update **keeps** the stored credential. |

Set the Model B token rates in the Application's cost block so the ledger prices each call at the rate of the model that answered:

```bash
curl -s -X PUT http://localhost:8080/v1/glad/apps/support_bot/cost \
  -H "Content-Type: application/json" \
  -d '{"cost": {"input_per_mtok": 0.15, "output_per_mtok": 0.60,
                "complex_input_per_mtok": 1.25, "complex_output_per_mtok": 10.00}}'
```

### Reading the routing decision

Whenever routing is enabled, the response's `geodesia` payload carries a `routing` block — which axis decided, what it scored, the threshold it was compared against, and which model actually answered:

```json
"geodesia": {
  "routing": {
    "enabled": true,
    "used_complex_model": true,
    "axis": "prompt_complexity",
    "score": 0.8134,
    "threshold": 0.5,
    "model": "gpt-5"
  }
}
```

`used_complex_model: false` with `axis_missing: true` means the served detector has no `prompt_complexity` axis and the request fell back to Model A. Because the block is present on every routed call, the trade-off stays auditable after the fact — you can always show *why* a given answer came from the cheap model.

### Failure behaviour

Routing **never fails a request**. If the axis is unavailable — an older checkpoint, `GB_EXTRA_AXES` not enabled — the gateway logs a distinct `[complex-routing] axis 'prompt_complexity' unavailable` marker and falls back to Model A. That is a silent cost regression rather than an outage (all "complex" traffic collapses onto the cheap model), which is exactly why it is logged loudly. Check `supports_axis.prompt_complexity` on `GET /v1/glad/apps/meta` before relying on it.

!!! warning "`prompt_complexity` is not a guardrail"
    Its enforcement mode is `off` and must stay that way. It is a routing boundary, not a risk score — promoting it to `block` would refuse every hard question your users ask.

---

## Choosing the threshold

The two axes want opposite kinds of tuning, because their errors cost different things.

| | `out_of_scope` | `prompt_complexity` |
|---|---|---|
| Cost of a **false positive** | You refuse a customer you should have served. Expensive, visible. | You pay Model B prices for an easy question. Cheap, invisible. |
| Cost of a **false negative** | You pay for an answer outside your remit. | A hard question gets a weak answer. Visible in quality. |
| Sensible default | `0.90` — refuse only when the model is very sure | `0.50` — the classifier's own boundary |
| How to tune | [Policy Lens](../studio/policy-lens.md) on real traffic, in `annotate` mode first | Move it and watch the Model-A / Model-B split in [Cost & FinOps](../studio/cost.md) |

Both are per-Application. A public support bot, an internal research assistant and a compliance reviewer can share one deployment and one detector while drawing these lines in completely different places.

---

## See also

- [Detection Axes](detection-axes.md) — what all nine axes score and how enforcement is grouped
- [Policy Lens](../studio/policy-lens.md) — set these thresholds against your own traffic, with the counterfactual computed before you apply it
- [Cost & FinOps](../studio/cost.md) — spend, forecast and budget enforcement per Application
- [Managing Applications](../studio/applications.md) — where `scope`, `complex_routing` and the cost rates live in the config
