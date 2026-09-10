# The `glad.*` Tools

The [Guard Server](guard-server.md) exposes **seven tools**. They are the whole agentic surface: an
agent calls them to vet what it is about to **trust** and about to **do**, and gets back a per-axis
probability, the threshold it was compared against, and a flag — not an opinion.

!!! warning "They report. They cannot enforce."
    Every tool here returns a verdict; none of them stops anything. The agent chooses whether to call
    them and whether to obey. If you need enforcement that does not depend on the model's goodwill, use
    the [Interceptor](interceptor.md), which sits in the path, or the
    [hooks](#making-the-checks-automatic) below.

---

## The seven tools

| Tool | What it is for | Call it |
|---|---|---|
| `glad.scan_resource` | Untrusted content — a web page, a file, a RAG passage, a tool result | **before** it enters the model's context |
| `glad.scan_toolset` | An MCP server's tool descriptions — poisoning, rug-pulls | **when connecting** to a server, and again if it changes |
| `glad.verify_tool_call` | The arguments of an outbound call | **before** it executes |
| `glad.verify_answer` | An answer against the sources it claims to rest on | **before** replying |
| `glad.redact_pii` | Personal data | **before** anything leaves the boundary |
| `glad.analyze` | Free text on all nine axes | when none of the above fits |
| `glad.explain` | Why a decision came out that way, token by token | after a flag, to show or audit it |

### The order that matters

The four verbs map onto the two moments where an agent can be compromised:

```
       ┌── reads ──────────────┐            ┌── acts ─────────────────┐
       │ scan_resource         │            │ verify_tool_call        │
       │ scan_toolset          │            │ redact_pii              │
       └───────────────────────┘            └─────────────────────────┘
                    ↓                                    ↑
              enters context ───→ the model reasons ─────┘
                                          ↓
                                    verify_answer  →  reply
```

The pattern worth naming is **read-untrusted-then-send-outward**: a page or tool result carries an
instruction, the model absorbs it, and the next outbound call carries data it was never meant to send.
`scan_resource` on the way in and `verify_tool_call` on the way out close it from both ends. Either
alone leaves the other half open.

---

## Reading a verdict

Every result carries the same three things per axis, and they must be quoted together:

```json
{
  "axes": {
    "rag_jailbreak": { "p": 0.94, "threshold": 0.62, "flagged": true,  "available": true },
    "halluc_context": { "p": null, "threshold": 0.71, "flagged": false, "available": false }
  }
}
```

* **`p`** — the probability on that axis.
* **`threshold`** — what it was compared against. A bare probability is not a verdict: the same 0.55
  is a flag on one axis and background noise on another.
* **`available`** — whether the axis was **measured at all**.

!!! danger "`available: false` is not `clean`"
    An axis reports `available: false` when it had nothing to read — no grounding context, no logprobs.
    That is *not measured*, which is a different statement from *nothing found*. Rendering it as 0%, or
    folding it into an average as if it were a pass, turns silence into reassurance. The
    [`grounding`](../g1-proxy/grounding.md) block follows the same rule: not measurable is `null`,
    never `0`.

---

## Making the checks automatic

A tool the model must *choose* to call is a tool the model can skip — under pressure, exactly when it
matters. If your host supports **hooks**, wire the checks there instead: hooks fire on their own, the
model does not get a vote.

For hosts without hooks, the [Interceptor](interceptor.md) achieves the same by construction: it brokers
the downstream MCP server, so every message is scanned whether or not anyone asked.

---

## Related

* **[Agent Skill (SKILL.md)](../agent-setup/SKILL.md)** — the same seven tools written *for an agent to
  consume*: when to call each one, worked examples, and a bootstrap block that installs itself. This
  page is the human-facing reference; that one is what you hand to the agent.
* [Guard Server](guard-server.md) — transports, ports, connecting a host
* [Detection Axes](../g1-proxy/detection-axes.md) — what each of the nine axes measures
* [Causal Explainability](../g1-proxy/causal-xai.md) — what `glad.explain` returns
* [Policy](policy.md) — per application, per axis, per tool
