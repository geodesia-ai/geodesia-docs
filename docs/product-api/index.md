# Product API — Introduction

Two services, two jobs. Almost every integration question comes down to picking the right one.

| | **G1-Proxy** | **G-1 Studio** |
|---|---|---|
| Role | Data plane — everything that happens while a request is in flight | Control plane — governance, configuration, reporting |
| Needs a GPU | Yes (the detector) | No |
| Sits in the request path | Yes | No |
| Typical direct port | `8800` | `8199` |
| Behind the unified port `8080` | under `/gw/…` | as-is |
| Complete endpoint list | [G1-Proxy API map](../g1-proxy/api-reference.md) | [Studio API map](../studio/api-reference.md) |

If you are wiring an application to Geodesia, you are almost always talking to **G1-Proxy**.

---

## Base URLs

The packaged installer serves both behind **one** port:

```
http://localhost:8080/gw/…     → G1-Proxy
http://localhost:8080/…        → G-1 Studio
```

In a split deployment each service also listens on its own port, and the proxy has **no `/gw` prefix** there:

```
http://localhost:8800/v1/chat/completions      # G1-Proxy, direct
http://localhost:8199/v1/glad/apps             # G-1 Studio, direct
```

Every example in these docs uses the unified `:8080` form. Drop the `/gw` and change the port if you are addressing the proxy directly.

---

## The three calls most integrations need

| You want | Call | Docs |
|---|---|---|
| A guarded chat completion | `POST /gw/v1/chat/completions` | [Chat API](../g1-proxy/chat-api.md) |
| Generate + score in one shot, for batch work | `POST /gw/v1/glad/evaluate` | [Evaluate](evaluate.md) |
| Token-level attribution for a decision | `POST /gw/v1/glad/causal-explainability/analyze` | [Explainability](explainability.md) |

---

## Authentication

| Surface | Guard | Default |
|---|---|---|
| G1-Proxy mutating endpoints (config, calibration, capabilities, attribution) | `Authorization: Bearer <GW_API_TOKEN>` or `X-Gateway-Token` | **Open** when `GW_API_TOKEN` is unset |
| G1-Proxy chat / evaluate | none of its own; an Application key (`Authorization: Bearer g1k_…`) *routes* the request rather than gating it | Open |
| Studio control-plane writes (Applications, orgs, keys) | `X-Geodesia-Admin-Key: <GEODESIA_ADMIN_TOKEN>` or an `admin`-role `g1k_…` bearer | **Open** until an admin token or admin key exists |
| Studio licence-token administration | `X-Geodesia-Admin-Key: <GLAD_LICENSE_ADMIN_KEY>` | **Closed** — `503` when the variable is unset |

The open-by-default posture makes a local single-tenant install work with zero configuration. **Set `GW_API_TOKEN` and `GEODESIA_ADMIN_TOKEN` before either service is reachable from a network** — without them, upstream re-pointing and Application configuration are unauthenticated.

---

## Scoping a call to an Application

Every per-Application surface reads the same header:

```
X-Geodesia-App: support_bot
```

On the chat and evaluate endpoints you can send `application_id` on the body instead, and an Application API key in `Authorization` resolves the Application on its own. Omit all three and you get the `default` Application. See [How an Application is resolved](../g1-proxy/chat-api.md#how-an-application-is-resolved).

---

## Compliance without a GPU

G-1 Studio is designed to run **without** a model. Dashboard, FRIA, oversight, kill switch, audit chain, reports and the whole control plane work normally on a CPU-only box; only the Studio-local research endpoints (`/glad/evaluate`, `/glad/finetune`) need a checkpoint loaded in-process, and they are not part of a normal integration.
