# Control Plane API

The **control plane** is how you create and manage Organizations, Applications, policy, cost configuration, and API keys in G-1 Studio. It is the management counterpart to the [chat data plane](../gateway/chat-api.md): the control plane decides *what* an Application is, the data plane *runs* requests through it.

Every control-plane route is mounted under **`/v1/glad`** — right next to the compliance API — on the same single-origin server as the gateway and the web UI. The routes operate on the shared Studio database (Organizations, Applications, API keys, the usage ledger).

!!! note "Open locally, gateable in production"
    On a local or air-gapped install the control plane is **open** by default — there is nothing to authenticate against, so you can create Applications and keys immediately. As soon as you configure a platform admin token *or* a single admin API key, every **write** (create / update / delete) requires authentication. Reads (`GET`) are never gated. See [Authentication & RBAC](#authentication-rbac) below.

---

## Authentication & RBAC

The control plane has four roles, in ascending privilege:

```
viewer  <  app_editor  <  org_admin  <  platform_admin
```

A caller's role is resolved from the request headers:

| Header | Role granted | Notes |
|---|---|---|
| `X-Geodesia-Admin-Key: <token>` | `platform_admin` | Must equal the `GEODESIA_ADMIN_TOKEN` environment variable. Full access to every Organization and Application. |
| `Authorization: Bearer <admin app key>` | `org_admin` | An Application API key created with `role: "admin"`. Scoped to the key's Application's Organization. |
| `Authorization: Bearer <invoke key>` | `viewer` | An Application API key created with `role: "invoke"` (the default). Read-only on the control plane; its real job is data-plane routing. |
| *(none, and nothing configured)* | `platform_admin` | **Open mode.** When neither `GEODESIA_ADMIN_TOKEN` is set **nor** any admin key exists, an anonymous caller is treated as `platform_admin`. |
| *(none, but auth is configured)* | `viewer` | Once a token or an admin key exists, anonymous callers fall back to `viewer` — they can read but cannot write. |

!!! warning "How open mode closes"
    The control plane is open **only** when *no* `GEODESIA_ADMIN_TOKEN` is set **and** *no* active admin API key exists anywhere in the database. Create one admin key — or set the env token — and writes immediately start requiring authentication. There is no half-open state.

### Role required per write

Reads are always permitted. Writes require at least the role shown:

| Operation | Minimum role |
|---|---|
| Create / update an Application (`POST /apps`, `PUT /apps/{id}`) | `app_editor` |
| Pause / resume / kill an Application | `app_editor` |
| Update policy / cost (`PUT /apps/{id}/policy`, `PUT /apps/{id}/cost`) | `app_editor` |
| Delete an Application (`DELETE /apps/{id}`) | `org_admin` |
| Create an Organization (`POST /orgs`) | `org_admin` |
| Create / revoke an API key | `org_admin` |

An `org_admin` resolved from an app key may only act on **its own** Organization — a cross-org write returns `403 org scope mismatch`. A `platform_admin` has no such restriction.

When a write is denied, the API returns:

```json
{ "detail": "requires role >= org_admin (you are viewer)" }
```

---

## Route reference

Every route, grouped by resource. The **Role** column is the minimum role for that route (reads are open; the gate only applies once auth is configured).

### Meta

| Method | Path | Role | Purpose |
|---|---|---|---|
| `GET` | `/v1/glad/apps/meta` | — | The 6 detection axes, the supported-law catalog, and a complete default config — used to render the New Application form. |
| `POST` | `/v1/glad/apps/upstream/models` | — | Discover the models available on an upstream. Live for `ollama` / vLLM / OpenAI-compatible servers; a curated catalog for Bedrock / Vertex. |

### Organizations

| Method | Path | Role | Purpose |
|---|---|---|---|
| `POST` | `/v1/glad/orgs` | `org_admin` | Create (or upsert) an Organization. |
| `GET` | `/v1/glad/orgs` | — | List all Organizations. |
| `GET` | `/v1/glad/orgs/{org_id}/apps` | — | List the Applications in an Organization. |
| `GET` | `/v1/glad/orgs/{org_id}/cost/summary` | — | Cost breakdown by Application for the Organization. |
| `GET` | `/v1/glad/orgs/{org_id}/cost/forecast` | — | Projected month-end spend across all the Organization's Applications. |

### Applications

| Method | Path | Role | Purpose |
|---|---|---|---|
| `POST` | `/v1/glad/apps` | `app_editor` | Create an Application. Subject to the entitlement cap and the per-org cap (below). |
| `GET` | `/v1/glad/apps` | — | List Applications (optional `org_id` / `status` filters). |
| `GET` | `/v1/glad/apps/{app_id}` | — | Fetch one Application (full config). |
| `PUT` | `/v1/glad/apps/{app_id}` | `app_editor` | Update name and/or config. Bumps `config_version`. |
| `DELETE` | `/v1/glad/apps/{app_id}` | `org_admin` | Delete an Application and its keys. The `default` Application cannot be deleted. |
| `POST` | `/v1/glad/apps/{app_id}/pause` | `app_editor` | Set status to `paused`. |
| `POST` | `/v1/glad/apps/{app_id}/resume` | `app_editor` | Set status back to `active`. |
| `POST` | `/v1/glad/apps/{app_id}/kill` | `app_editor` | Set status to `killed` (kill-switch). |

### Policy & cost configuration

| Method | Path | Role | Purpose |
|---|---|---|---|
| `GET` | `/v1/glad/apps/{app_id}/policy` | — | The Application's policy block (thresholds, enforcement, etc.). |
| `PUT` | `/v1/glad/apps/{app_id}/policy` | `app_editor` | Merge fields into the policy block. |
| `GET` | `/v1/glad/apps/{app_id}/cost` | — | The Application's cost configuration (rates, budget). |
| `PUT` | `/v1/glad/apps/{app_id}/cost` | `app_editor` | Merge fields into the cost configuration. |

### Metrics & cost

| Method | Path | Role | Purpose |
|---|---|---|---|
| `GET` | `/v1/glad/apps/{app_id}/metrics` | — | Call counts from the `calls` table: total, prompt-blocked, answer-blocked, hallucinated, grounded. |
| `GET` | `/v1/glad/apps/{app_id}/cost/summary` | — | Month-to-date spend, token totals, blocked count, average cost per call. |
| `GET` | `/v1/glad/apps/{app_id}/cost/daily` | — | Per-day cost series (for charts). |
| `GET` | `/v1/glad/apps/{app_id}/cost/forecast` | — | Projected month-end spend (`run_rate` / `moving_avg` / `linreg` / `linreg_dow`). |

### API keys

| Method | Path | Role | Purpose |
|---|---|---|---|
| `GET` | `/v1/glad/apps/{app_id}/keys` | — | List the Application's keys (preview + metadata only — never the secret). |
| `POST` | `/v1/glad/apps/{app_id}/keys` | `org_admin` | Mint a new key. The plaintext is returned **exactly once**. |
| `DELETE` | `/v1/glad/apps/{app_id}/keys/{key_id}` | `org_admin` | Revoke (deactivate) a key. |

---

## Worked examples

The examples below assume the server is on `http://localhost:8080` and the control plane is open (local install). If you have configured a platform token, add `-H "X-Geodesia-Admin-Key: $GEODESIA_ADMIN_TOKEN"` to each write.

### 1. Create an Organization

```bash
curl -s http://localhost:8080/v1/glad/orgs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme Legal",
    "country": "IT",
    "license_type": "evaluation",
    "max_applications": 5
  }'
```

```json
{
  "org_id": "acme_legal",
  "name": "Acme Legal",
  "country": "IT",
  "license_type": "evaluation",
  "max_applications": 5,
  "created_at": "2026-06-18T09:00:00.000000Z",
  "updated_at": "2026-06-18T09:00:00.000000Z"
}
```

`org_id` is slugged from `name` if you omit it. Returns `201 Created`. Calling it again with the same `org_id` upserts the name/country.

### 2. Create an Application

```bash
curl -s http://localhost:8080/v1/glad/apps \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Contract Reviewer",
    "org_id": "acme_legal",
    "config": {
      "binding": {"upstream_type": "ollama", "base_url": "http://localhost:11434", "model": "llama3.1:8b"},
      "calibration_profile": "llama3.1:8b"
    }
  }'
```

```json
{
  "app_id": "contract_reviewer_4f2a9c",
  "config_version": 1,
  "status": "active",
  "app": {
    "app_id": "contract_reviewer_4f2a9c",
    "org_id": "acme_legal",
    "name": "Contract Reviewer",
    "status": "active",
    "config_version": 1,
    "config": { "schema_version": 1, "binding": { "...": "..." }, "policy": { "...": "..." } }
  }
}
```

If you omit `config`, the Application is created with the complete default config (all 6 axes, serving-calibrated thresholds). Returns `201 Created`. An invalid config returns `400` with the validation message.

!!! danger "The free-tier Application cap"
    Application creation is subject to a global, license-driven entitlement cap. On the **free tier** the limit is **1 Application** (the seeded `default` Application already counts toward it). When you exceed it, `POST /apps` returns `403`:

    ```json
    {
      "detail": "application limit reached (free tier: 1 application(s)). Add a license token to create more."
    }
    ```

    A signed license token raises (or removes) the cap. A separate **per-Organization** cap (`max_applications` on the Org) also applies — exceeding it returns `403 max_applications reached for org (license limit)`. The `default` Organization is exempt from the per-org cap. See [Licensing](licensing.md).

### 3. Set the Application's policy

Policy updates are a **merge** — you send only the fields you want to change.

```bash
curl -s -X PUT http://localhost:8080/v1/glad/apps/contract_reviewer_4f2a9c/policy \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "thresholds": {"prompt_safety": 0.75, "jailbreak": 0.55},
      "enforcement": {"answer_safety": "block"},
      "block_input": true
    }
  }'
```

```json
{
  "policy": {
    "thresholds": {"prompt_safety": 0.75, "jailbreak": 0.55, "rag_jailbreak": 0.05, "halluc_context": 0.32, "halluc_closedbook": 0.58, "answer_safety": 0.90},
    "enforcement": {"prompt_safety": "block", "jailbreak": "block", "rag_jailbreak": "block", "halluc_context": "annotate", "halluc_closedbook": "annotate", "answer_safety": "block"},
    "block_input": true
  },
  "config_version": 2
}
```

The cost configuration works the same way via `PUT /apps/{id}/cost` (merge `currency`, `input_per_mtok`, `output_per_mtok`, `budget_month`, `on_budget_exceeded`, …). The full policy/cost field shape is documented in [Managing Applications](applications.md).

### 4. Mint and use an API key

```bash
curl -s http://localhost:8080/v1/glad/apps/contract_reviewer_4f2a9c/keys \
  -H "Content-Type: application/json" \
  -d '{"role": "invoke"}'
```

```json
{
  "key_id": "ak_9b1f2c3d",
  "app_id": "contract_reviewer_4f2a9c",
  "key_preview": "g1k_***x8Qv",
  "role": "invoke",
  "api_key": "g1k_live_8sQ3...x8Qv"
}
```

!!! warning "Shown once"
    The plaintext `api_key` is returned **only on creation** — only its SHA-256 hash is stored. Copy it now; you cannot retrieve it later. `GET /keys` returns the preview and metadata, never the secret.

The key is the Application's runtime identity on the data plane. Pass it as a Bearer token on a chat request to route that request through this Application:

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer g1k_live_8sQ3...x8Qv" \
  -H "Content-Type: application/json" \
  -d '{"model":"llama3.1:8b","stream":false,"messages":[{"role":"user","content":"Summarise clause 4."}]}'
```

A `role: "admin"` key additionally acts as an `org_admin` on the control plane (see [RBAC](#authentication-rbac)).

### 5. Read metrics and forecast cost

```bash
curl -s http://localhost:8080/v1/glad/apps/contract_reviewer_4f2a9c/metrics
```

```json
{
  "application_id": "contract_reviewer_4f2a9c",
  "total": 1284,
  "prompt_blocked": 17,
  "answer_blocked": 5,
  "hallucinated": 31,
  "grounded": 1102
}
```

```bash
curl -s "http://localhost:8080/v1/glad/apps/contract_reviewer_4f2a9c/cost/forecast?method=linreg"
```

```json
{
  "period": "2026-06",
  "currency": "EUR",
  "spent_mtd": 42.18,
  "days_elapsed": 18,
  "days_in_month": 30,
  "projected_month": 71.94,
  "budget": 100.0,
  "projected_pct": 0.7194,
  "over_budget": false,
  "method": "linreg",
  "ci80": [61.40, 82.48]
}
```

The forecast is deterministic and numpy-only. `run_rate` (the default) extrapolates the month-to-date average; `moving_avg` uses the trailing 7 days; `linreg` / `linreg_dow` fit a trend (the latter applies a day-of-week factor) and return an 80% confidence band in `ci80`. Pass `budget` via the Application's cost config — the route reads `budget_month` from it. The summary endpoint (`/cost/summary`) returns the same period's token totals and `avg_cost_per_call`.

---

## Data-plane routing header

The control plane defines Applications; the data plane selects one **per request** (`_resolve_app`). A chat or RAG request picks its Application via either an **explicit** identifier or an **Application API key**, in this order:

1. **Explicit id / header** — the body field **`application_id`** (alias **`app_id`**), or the **`X-Geodesia-App: <app_id>`** request header. An explicit id **always wins**, including an explicit `default`.
2. **Application API key** — an `invoke` (or `admin`) key sent as **`Authorization: Bearer g1k_live_…`**, used **only as a fallback** when no explicit id/header is present. The gateway verifies the key and routes the request to the Application that key belongs to.

The gateway resolves that identifier to the Application's config — its upstream binding, `block_input`, and per-axis thresholds are merged into the request. An **unknown** identifier, or the literal `default`, falls back to the global / `default` configuration, so existing single-upstream integrations keep working unchanged.

!!! note "Only `g1k_`-prefixed bearers are looked up"
    When resolving from a Bearer token, the gateway considers **only** tokens that begin with `g1k_`. Any other Bearer — including the gateway's own `GW_API_TOKEN` — is ignored for app resolution, so it does not accidentally route a request. A **revoked, expired, or unknown** `g1k_` key does not error: it simply falls through to the `default` Application.

**Route via the header (or `application_id` in the body):**

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "X-Geodesia-App: contract_reviewer_4f2a9c" \
  -H "Content-Type: application/json" \
  -d '{"model":"llama3.1:8b","stream":false,"messages":[{"role":"user","content":"Is this NDA mutual?"}]}'
```

**Route via an Application API key (no header needed):**

```bash
curl -s http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer g1k_live_8sQ3...x8Qv" \
  -H "Content-Type: application/json" \
  -d '{"model":"llama3.1:8b","stream":false,"messages":[{"role":"user","content":"Is this NDA mutual?"}]}'
```

The key belongs to exactly one Application, so it resolves the Application implicitly — a raw API client that authenticates with just `Authorization: Bearer g1k_live_…` is still routed to, scoped to, and billed against its own Application. If you send **both** an explicit id/header **and** a `g1k_` key, the explicit id/header wins. For the full chat request/response contract, see the [Chat API](../gateway/chat-api.md).

---

## See also

- [Managing Applications](applications.md) — the Application config shape (binding, policy, cost, governance) and lifecycle.
- [Licensing](licensing.md) — the entitlement cap, license tokens, and how they raise the Application limit.
- [Cost & FinOps](cost.md) — the usage ledger, daily roll-up, budgets, and forecasting model.
- [Chat API](../gateway/chat-api.md) — the data-plane endpoint that consumes the routing header and the per-Application policy.
