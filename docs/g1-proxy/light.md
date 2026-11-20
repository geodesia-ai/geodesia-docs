# G1-Proxy Light — one container, and what it leaves out

**G1-Proxy Light** is the same engine as G1-Proxy, packaged as **one container that also serves its own
configuration UI**. It exists for places where an install must be a single click and a single image: a
cloud catalog listing, an evaluation on one machine, an appliance.

The important part of this page is the second half — **what it does not have, and why**. A profile that
quietly omits a capability is worse than one that does not exist, because the first thing a missing
capability does is look like a bug.

## The rule that keeps the two in step

Engine code is never edited in the packaging repos. There is one canonical codebase, and everything the
light profile does differently lives behind **one switch** — `GW_LIGHT=1`, implemented in
`gateway/light_surfaces.py`. The two products cannot drift apart, because they are the same source.

## What it keeps

Since 0.4.4 the light profile carries **every detection capability G1-Proxy serves**, thinking level 3 included.
It leaves out only three components, listed in the next table.

| | |
|---|---|
| detection axes | **all nine**, same weights, same thresholds, plus the dedicated `prompt_safety` reading for framed harm |
| thinking levels | **0 · 1 · 2 · 3** (Geodesia-G, Geodesia-H, Geodesia-A), default **2** |
| `halluc_context` | the same grounding reader as G1-Proxy, including the long-document read |
| `halluc_closedbook` + SLEDGE | yes — including **recalibration on the model you actually serve** |
| reference check (`verify_references`) | yes — open sources plus the local reference-pertinence classifier |
| [G1 Eval](eval.md) (`/g1/eval`) | yes — the endpoint, the OpenRouter-compatible façades, the call log and the **Eval** page in the UI |
| MCP guard (`glad.*` tools) | yes |
| Causal explainability (MuPAX / DCA, clause and word level for `halluc_context`) | yes |
| chat transcript export (JSONL) | yes — the route is there; see the note on the Oversight screen below |
| configuration UI | **served by the proxy itself**, on the same port |
| Applications | **one** — the single Application is seeded at boot |

!!! warning "Level 3 needs a 24 GB GPU"
    Geodesia-A adds about 9 GB of GPU memory when level 3 is actually used. On a 16 GB GPU levels 0–2 run, and
    the Geodesia-A tier does not start: it is reported as failed for lack of memory, retried after an hour
    (`GW_TIER_OOM_RETRY_S`), and a level-3 request is served at level 2. The response says so in
    `geodesia.thinking.tiers_used` — the request never fails, and never pretends. To leave level 3 out on
    purpose, set `GW_GLADA_CKPT=` (empty) in the container environment.

## What it does not have

Every row below is a deliberate omission, not an oversight. The reason matters, because it tells you
whether you can turn it on.

| missing | why | can I enable it? |
|---|---|---|
| **Knowledge Base (RAG)** | no embedder, no reranker, no vector store in the image; the `/v1/glad/rag/*` routes are not served | no — use G1-Proxy |
| **Audio input** | the speech-recognition model is not shipped | no — use G1-Proxy |
| **Idle judge / feedback queue** | no `llama.cpp` binary and no judge model (GGUF) in the image | **yes** — mount a binary and a model and set `GW_IDLE_JUDGE=1`; the capability endpoint then reports it as available |
| **Dashboard, Kill Switch, Audit, Legal, FRIA, Reports, Human Oversight** | these read the G-1 Studio control-plane backend, which is a different container | no — install G-1 Studio |
| **Compliance report generation** | `reportlab` / `matplotlib` are not installed | no |
| **Multiple Applications** | the light profile is single-Application by design | no |
| **Bundled documentation site** | stripped from the UI bundle to keep the image small | — (you are reading it) |

!!! note "The Oversight screen, and the export button on it"
    `Download all chats (JSONL)` lives on the **Human Oversight** screen, which the light profile hides
    by default because the rest of that screen is a Studio surface. The **export route itself is served**
    (`GET /v1/glad/chat-export`, scoped to the Application), so the capability is there either way: call
    it directly, or unhide the screen with `GW_UI_HIDDEN_VIEWS`.

## Do not trust this page — ask the box

The list above is what the image ships with. What **your** deployment can do is one request away, and it
is read from the artifacts on disk, never declared by hand:

```bash
curl -s http://localhost:8800/v1/glad/capabilities | jq '{tiers, tiers_live, max_thinking_level, features, hidden: .ui.hidden_views}'
```

```json
{
  "tiers": { "glad_g": true, "glad_h": true, "glad_a": true },
  "max_thinking_level": 3,
  "features": { "mcp": true, "mupax": true, "xai": true, "closedbook": true, "recalibrate": true,
                "rag": false, "audio": false, "idle_judge": false, "multi_application": false },
  "hidden": ["dashboard", "kill-switch", "knowledge", "reports", "fria", "audit", "legal",
             "feedback", "oversight", "license-tokens", "documentation", "agent-flow"]
}
```

`tiers` says what the image carries; `tiers_live` says what is running right now (`ready`, `starting`,
`failed`, `stopped` or `not_installed`) — a Geodesia-A that did not fit in GPU memory shows up there.

The `tiers` keys of this admin endpoint (`glad_g`, `glad_h`, `glad_a`) are the internal identifiers of
Geodesia-G, Geodesia-H and Geodesia-A. In a chat response the same tiers appear in `geodesia.thinking.tiers_used`
as `geodesia_g`, `geodesia_h`, `geodesia_a` (see [Thinking Levels](thinking-levels.md)).

Two ceilings are reported separately, and conflating them is a support ticket waiting to happen: what the
**image** can do (are the Geodesia-H artifacts present?) and what the **licence** allows. The effective level
is the lower of the two.

`GW_UI_HIDDEN_VIEWS="a,b,c"` overrides the hidden list without rebuilding — an empty string hides nothing.

## Which one should I install?

| you want | install |
|---|---|
| one machine, one click, no second container | **G1-Proxy Light** |
| compliance surfaces (audit trail, FRIA, reports, oversight queue) | G1-Proxy **+** G-1 Studio |
| thinking level 3 (Geodesia-A) | either — the light needs a 24 GB GPU for it |
| score texts you already have, without an LLM ([G1 Eval](eval.md)) | either |
| a knowledge base (RAG) or audio input | G1-Proxy |
| many Applications with separate policies and keys | G1-Proxy **+** G-1 Studio |

The installer takes the choice as a flag — see [Installer](../installer.md), profile `light`.

## Versions

The light profile is **one deployable**, so it reports **one version**: engine and UI live in the same
image and cannot be updated separately. The Studio split reports three (control plane, UI bundle, proxy)
because those are three images, and they have been out of step with each other before — which is exactly
why they are shown apart.
