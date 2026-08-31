# G-1 Studio — Complete API Map

Every HTTP surface **G-1 Studio** exposes. Studio is the **control plane**: Applications, policies, keys, cost, compliance, governance, reporting and model management. Anything that happens *while a request is in flight* is on the other service — see the [G1-Proxy API map](../g1-proxy/api-reference.md).

!!! info "Where to send the request"
    In the packaged product a **single port (8080)** fronts both services, and Studio's paths are served **as-is**:

    ```
    http://localhost:8080/v1/glad/apps
    ```

    G1-Proxy sits behind the `/gw` prefix on the same port. In a split deployment Studio also listens directly on **`:8199`**.

!!! warning "Authentication"
    Two **independent** guards, each with its own key, both header-based.

    **Control-plane RBAC** — guards every **write** on Applications, organisations and API keys (marked **🔑** below). Roles ascend `viewer < app_editor < org_admin < platform_admin`. A caller authenticates with either:

    - `X-Geodesia-Admin-Key: <GEODESIA_ADMIN_TOKEN>` → `platform_admin`, or
    - `Authorization: Bearer g1k_…` → `org_admin` if the key's role is `admin`, otherwise `viewer` (read-only on the control plane).

    While **neither** `GEODESIA_ADMIN_TOKEN` is set **nor** any `admin`-role API key exists, an anonymous caller is treated as `platform_admin` — the control plane is open. That is deliberate, so a local single-tenant install works out of the box. **Configuring either one immediately closes it**: from that moment an unauthenticated caller is a `viewer` and every write returns `403`.

    **Licence-token administration** — a *separate* key, `GLAD_LICENSE_ADMIN_KEY`, on the endpoints marked **🔐**. Supply it as `X-Geodesia-Admin-Key`, `Authorization: Bearer …`, or the `admin_key` query parameter. Unlike RBAC it fails **closed**: with the variable unset the endpoints return `503`, and a wrong key returns `401`.

    Reads on Applications, cost and compliance are **not** guarded by either. Put Studio behind your own authentication if that matters to you.

---

## One call, to check you are pointed at the right service

```bash
curl -s http://localhost:8080/v1/glad/health | jq
# {"status": "ok", "version": "…"}
```

If that answers, everything below is reachable. If you were expecting chat or detection, you want the
other service — see the [G1-Proxy API map](../g1-proxy/api-reference.md).

## Applications & organisations

An **Application** is the unit of management: one upstream LLM with G1-Hummingbird in the middle, owning its own policy, thresholds, RAG collection, cost centre and compliance posture. Full guide: [Managing Applications](applications.md).

### Metadata

| Method | Path | | What it does |
|---|---|---|---|
| `GET` | `/v1/glad/apps/meta` | | `{axes, extra_axes, supports_axis, supported_laws, default_config}`. **Read this first** — it tells you which axes the *served* detector actually has, so you never write a policy for an axis that will never be scored. |
| `POST` | `/v1/glad/apps/upstream/models` | | Discover the models a candidate backend offers. Body: `{upstream_type, base_url, api_key, application_id}`. Send `api_key: "***"` together with `application_id` to use that app's stored key — the browser never has to hold the real value. |

### Organisations

| Method | Path | | What it does |
|---|---|---|---|
| `GET` | `/v1/glad/orgs` | | List organisations. |
| `POST` | `/v1/glad/orgs` | 🔑 | Create one. Body: `{name, org_id?, max_applications?}`. **201**. |
| `GET` | `/v1/glad/orgs/{org_id}/apps` | | Applications in this organisation. |
| `GET` | `/v1/glad/orgs/{org_id}/cost/summary` | | Rolled-up spend. Query: `period` (`YYYY-MM`). |
| `GET` | `/v1/glad/orgs/{org_id}/cost/forecast` | | Month-end projection. Query: `period`, `method` (default `run_rate`). |

### Application lifecycle

| Method | Path | | What it does |
|---|---|---|---|
| `GET` | `/v1/glad/apps` | | List Applications. Query: `org_id`, `status`. |
| `POST` | `/v1/glad/apps` | 🔑 | Create one. Body: `{name, org_id?, config?}`. **201**. |
| `GET` | `/v1/glad/apps/{app_id}` | | Full record, including the resolved config. Upstream API keys are masked as `***`. |
| `PUT` | `/v1/glad/apps/{app_id}` | 🔑 | Rename and/or patch the config. Body: `{name?, config?}`. |
| `DELETE` | `/v1/glad/apps/{app_id}` | 🔑 | Delete the Application. |
| `POST` | `/v1/glad/apps/{app_id}/pause` | 🔑 | Stop accepting traffic; keep the configuration. |
| `POST` | `/v1/glad/apps/{app_id}/resume` | 🔑 | Accept traffic again. |
| `POST` | `/v1/glad/apps/{app_id}/kill` | 🔑 | Per-Application kill switch. Distinct from the deployment-wide [kill switch](../compliance/kill-switch.md). |

### Application configuration

Each of these reads or replaces one section of the app config. `PUT` takes the section wrapped under its own key.

| Method | Path | | Body | Section |
|---|---|---|---|---|
| `GET` / `PUT` | `/v1/glad/apps/{app_id}/policy` | 🔑 | `{policy: {…}}` | Per-axis `thresholds` and `enforcement` (`block` \| `annotate` \| `off`), `block_input`, `inject_system`, `scope`, `rag_collection`, `feedback_learning`, `optional_detectors`, `streaming_brake`. |
| `GET` / `PUT` | `/v1/glad/apps/{app_id}/routing` | 🔑 | `{complex_routing: {…}}` | `{enabled, threshold, complex_binding}` — send easy prompts to the cheap model and hard ones to the capable one. See [Token & Cost Control](../g1-proxy/cost-control.md). |
| `GET` / `PUT` | `/v1/glad/apps/{app_id}/cost` | 🔑 | `{cost: {…}}` | Token rates, `budget_month`, `alert_pct`, `on_budget_exceeded`, `alert_recipients`. |

!!! danger "Policy validation is strict, deliberately"
    `PUT …/policy` rejects any axis name not present on the served checkpoint, and any threshold outside `[0,1]` or enforcement outside `block|annotate|off`. A policy carrying a threshold for an axis that will never be scored is a dead slider in the UI and a silent lie in an audit — so it is refused rather than stored.

### Observability & export

| Method | Path | | What it does |
|---|---|---|---|
| `GET` | `/v1/glad/apps/{app_id}/metrics` | | Traffic and detection counters. Query: `since`, `until` (ISO 8601). |
| `GET` | `/v1/glad/apps/{app_id}/messages` | | Recent real requests, each with per-axis detector probabilities and the live block/allow decision. Query: `session_id`, `limit` (≤ 2000, default 300). This is what [Policy Lens](policy-lens.md) replays. |
| `GET` | `/v1/glad/apps/{app_id}/export` | 🔑 | Download everything this Application ever saw. Query: `fmt` = `sqlite` (one file) \| `csv` \| `jsonl` (a zip, one file per table). Honours `Content-Disposition`. |
| `GET` | `/v1/glad/apps/{app_id}/export.sqlite` | 🔑 | The `fmt=sqlite` case as a fixed path. |

### Cost per Application

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/apps/{app_id}/cost/summary` | Spend for a period. Query: `period` (`YYYY-MM`). |
| `GET` | `/v1/glad/apps/{app_id}/cost/daily` | Daily series. Query: `since`, `until`. |
| `GET` | `/v1/glad/apps/{app_id}/cost/forecast` | Month-end projection. Query: `period`, `method` (default `run_rate`), `today`. |

Full treatment: [Cost & FinOps](cost.md).

### API keys

| Method | Path | | What it does |
|---|---|---|---|
| `GET` | `/v1/glad/apps/{app_id}/keys` | | List keys. Only prefixes and metadata — the secret is shown **once**, at creation. |
| `POST` | `/v1/glad/apps/{app_id}/keys` | 🔑 | Mint a key. Body: `{role}` (default `invoke`). **201**. **The plaintext key is in this response and nowhere else.** |
| `DELETE` | `/v1/glad/apps/{app_id}/keys/{key_id}` | 🔑 | Revoke immediately. |

---

## Compliance dashboard

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/health` | `{status, version}`. The liveness probe for the control plane. |
| `GET` | `/v1/glad/dashboard` | Aggregate compliance posture. Query: `deployer_id`. |
| `GET` | `/v1/glad/dashboard/charts` | Time series behind the dashboard charts. Query: `deployer_id`, `application_id`. |
| `GET` | `/v1/glad/scorecard` | Per-framework regulatory scorecard. Query: `deployer_id`. |
| `GET` | `/v1/glad/provider-identity` | Provider/deployer identity used in generated documents. |
| `GET` | `/v1/glad/retention/status` | Retention policy state and what is due for deletion. |

See [Compliance Dashboard](../compliance/dashboard.md).

---

## Kill switch

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/kill-switch/status` | Query: `deployer_id` (default `"default"`). |
| `POST` | `/v1/glad/kill-switch/activate` | Body: `{deployer_id?, reason?, activated_by?}`. |
| `POST` | `/v1/glad/kill-switch/deactivate` | Same body. |

See [Kill Switch](../compliance/kill-switch.md).

---

## Human oversight

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/oversight/pending` | The review queue as a bare array. Query: `review_level`, `limit` (default 50), `application_id`. |
| `GET` | `/v1/glad/oversight/summary` | Counters for a dashboard tile. |
| `POST` | `/v1/glad/oversight/review` | Create a review by hand. Body: `{call_id, review_trigger, review_level?, reviewer_id?}`. |
| `POST` | `/v1/glad/oversight/decide` | Close a review. Body: `{review_id, decision, reviewer_id?, reviewer_notes?, override_justification?}`. **404** on an unknown `review_id`. |

Reviews are keyed by **`review_id`**, not `call_id`. Full guide with request/response shapes: [Human Oversight](../compliance/oversight.md).

---

## FRIA (Fundamental Rights Impact Assessment)

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/fria` | List dossiers. Query: `deployer_id` (default `"default"`). |
| `POST` | `/v1/glad/fria` | Create a dossier. |
| `GET` | `/v1/glad/fria/auto-prefill` | Draft answers from what the deployment already knows. Query: `deployer_id`, `deployment_context`. |
| `GET` | `/v1/glad/fria/{fria_id}` | One dossier. |
| `PUT` | `/v1/glad/fria/{fria_id}` | Update it. |
| `POST` | `/v1/glad/fria/{fria_id}/approve` | Sign it off. Body: `{assessor_name}`. |
| `POST` | `/v1/glad/fria/{fria_id}/archive` | Archive it. |
| `GET` | `/v1/glad/fria/{fria_id}/evidence` | Declared evidence attached to the dossier. |
| `GET` | `/v1/glad/fria/{fria_id}/runtime-evidence` | **Measured** evidence from real traffic. Query: `window_days` (default 90). This is what turns the dossier from a questionnaire into a record. |
| `GET` | `/v1/glad/fria/{fria_id}/export` | Synchronous export. Query: `fmt` = `pdf` \| `docx` \| `json`. |
| `POST` | `/v1/glad/fria/{fria_id}/export/async` | The same export as a background job — see [Document jobs](#document-jobs). |

See [FRIA](../compliance/fria.md).

---

## Audit chain & watermark

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/chain/status` | Ledger snapshot plus the newest 20 entries. |
| `GET` | `/v1/glad/chain/verify` | Full HMAC re-verification. No parameters. |
| `POST` | `/v1/glad/watermark/verify` | Body: `{response_text\|text, call_id?, watermark_id?, session_id?}`. With `call_id` **and** `watermark_id` it verifies that specific record; with text alone it scans the watermark log. |

See [Audit Chain](../compliance/audit-chain.md) and [AI Watermark](../compliance/watermark.md).

---

## Reports & documents

| Method | Path | What it does |
|---|---|---|
| `POST` | `/v1/glad/report` | Generate a compliance report as structured data. |
| `POST` | `/v1/glad/report/pdf` | The same report rendered to PDF, returned inline. |
| `POST` | `/v1/glad/report/pdf/async` | As a background job. |
| `POST` | `/v1/glad/deployer-manual` | Deployer transparency manual as structured data. |
| `POST` | `/v1/glad/deployer-manual/pdf` | Rendered to PDF. |
| `POST` | `/v1/glad/deployer-manual/pdf/async` | As a background job. |
| `GET` | `/v1/glad/legal/frameworks` | The catalogue of supported legal frameworks. |
| `GET` | `/v1/glad/legal/{framework_id}` | One framework in full. |

`POST /v1/glad/report` body: `{company_name, company_country, model_description, applicable_laws?, session_ids?, call_ids?, from_date?, to_date?, include_json?, language?, fria_id?, deployer_id?, contact_email?}` — every field has a default.

`POST /v1/glad/deployer-manual` body: `{company_name, company_country, contact_email, safety_threshold, halluc_threshold, retention_months, review_days, applicable_laws?, language?}`.

See [Reports & Manuals](../compliance/reports.md).

### Document jobs

Rendering a full multi-framework PDF over a busy database can take minutes — long enough for a proxy or load balancer to cut the connection. The `/async` variants return a job instead.

| Method | Path | What it does |
|---|---|---|
| *(start)* | `POST …/pdf/async` or `…/export/async` | Returns `{job_id, token}`. |
| `GET` | `/v1/glad/doc-jobs/{job_id}/status?token=…` | `{status, error?}`. `status` reaches `done` or `error`; `error` carries the real reason (e.g. *no logged calls / no applicable law selected*), which is what tells the user what to fix. |
| `GET` | `/v1/glad/doc-jobs/{job_id}/download?token=…` | The finished file. |

The `token` is required on both follow-up calls and scopes access to that one job.

---

## Explainability

| Method | Path | What it does |
|---|---|---|
| `POST` | `/v1/glad/causal-explainability/analyze` | Synchronous attribution. Body: `{prompt, response, context?, method?, model_path?}`. |
| `POST` | `/v1/glad/causal-explainability/jobs` | Start it as a job. Returns `{job_id, status, cached, result?}` — a cache hit comes back `completed` with the result already attached. |
| `GET` | `/v1/glad/causal-explainability/jobs/{job_id}` | Poll one job. |
| `POST` | `/v1/glad/causal-explainability/stream` | The same analysis over **SSE**, emitting `start` / `progress` / `done` / `error` events. |
| `POST` | `/v1/glad/mupax/explain` | MuPAX token attribution. Body: `{prompt, response, language?, top_k?, min_token_len?, strip_system_prompt?}`. |
| `GET` | `/v1/glad/mupax/languages` | Languages MuPAX can tokenise. |

`strip_system_prompt` defaults to `true` and removes the constitutional system prompt before attribution — MuPAX should judge user text and generated response, not your instructions.

The Studio path needs a **local model**. The [proxy's `analyze`](../g1-proxy/api-reference.md#explainability) is the black-box path and is what an OpenAI-API deployment uses. See [Causal Explainability](../g1-proxy/causal-xai.md).

---

## Model management

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/models/available` | `{active, models_root, switchable, candidates[]}`. Each candidate carries `loadable` and `loadable_reason` — a delta bundle whose backbone is not cached locally reports `loadable: false` **before** you try to switch to it. |
| `POST` | `/v1/glad/models/switch` | Body: `{name}`. Atomically re-points the active symlink and **exits the process after ~1 s** so the container restart policy reloads the new bundle. Returns `{ok, restarting: true, active}` first. **409** when the deployment is single-checkpoint or the bundle is not loadable; **404** when the name is unknown. |
| `GET` | `/v1/glad/families` | Model-family catalogue with install targets and installed state. |
| `POST` | `/v1/glad/families/install` | Body: `{name, target: "bundle"\|"vllm"\|"sglang", overwrite?, fetch_backbone?, hf_token?}`. Returns `{accepted: true, …}` and installs in the background. **409** while another install is running. |
| `GET` | `/v1/glad/families/install/status` | Progress: `{running, family, target, done, total, message, error, path, …}`. |

!!! warning "`models/switch` restarts the service"
    It responds, then exits. Any in-flight request on the same worker dies with it. Treat it as a deploy operation, not a runtime toggle.

### Model recalibration

Mounted on **both** services under `/v1/glad/calibration/…`.

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/calibration/capabilities` | What this build can recalibrate. |
| `GET` | `/v1/glad/calibration/status` | Current run state. |
| `GET` | `/v1/glad/calibration/registry` | Per-model calibration artifacts on disk. |
| `POST` | `/v1/glad/calibration/probe` | Check a generator before committing to a run. Body: `{model, gen_url?, api_key?, application_id?}`. |
| `POST` | `/v1/glad/calibration/run` | Body: `{model, gen_url?, gen_model?, mode: "fast"\|"deep", force?, fraction?, api_key?, application_id?}`. |
| `POST` | `/v1/glad/calibration/stop` | Stop the running job. |

---

## Licence tokens

Customer installer tokens. Full guide: [Licensing & Entitlements](licensing.md).

| Method | Path | | What it does |
|---|---|---|---|
| `GET` | `/v1/glad/license-tokens/models` | | The models a token may be scoped to. |
| `POST` | `/v1/glad/license-tokens/validate` | | Body: `{token, model_id, platform}`. **No auth** — this is what an installer calls. |
| `GET` | `/v1/glad/license-tokens/validate` | | Same check via query string. Called with no `token`, it returns a usage hint instead of an error. |
| `GET` | `/v1/glad/license-tokens` | 🔐 | List issued tokens. |
| `POST` | `/v1/glad/license-tokens` | 🔐 | Issue one. Body: `{customer_name, notes?, allowed_models?, expires_at?\|expires_days?\|expires_months?, license_key?, license_id?, plan?, tier?, max_chats_per_month?, max_models?, max_applications?}`. |
| `POST` | `/v1/glad/license-tokens/{token_id}/revoke` | 🔐 | Revoke a token. |

---

## Detection preferences & database

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/threshold-prefs` | Deployer-level threshold profile. Query: `profile_id` (default `default`). |
| `POST` | `/v1/glad/threshold-prefs` | **Partial** update: `{prompt_safety_threshold?, answer_safety_threshold?, halluc_threshold?, combined_halluc_threshold?}`. **400** when the body carries no value at all. |
| `GET` | `/v1/glad/db/config` | Current database backend. |
| `POST` | `/v1/glad/db/test` | Body: `{url}`. Test a PostgreSQL connection without committing to it. |
| `POST` | `/v1/glad/db/deploy` | Body: `{url, use}`. Deploy the schema and optionally switch to it. |
| `POST` | `/v1/glad/db/use-sqlite` | Switch back to the bundled SQLite. |

`GEODESIA_DB_URL` in the environment always wins over the stored choice, and a switch takes effect after a restart.

---

## Chat history

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/chat-sessions` | Sessions persisted by Studio. |
| `GET` | `/v1/glad/chat-messages?session_id=…` | Turns of one session with the detection payload each was served with. **400** without `session_id`. |

---

## OpenAI-compatible surface (local model)

Studio also serves an OpenAI-shaped API against a **locally loaded** model. Most integrations should target [G1-Proxy](../g1-proxy/api-reference.md#inference) instead — the proxy is the path that fronts your own upstream.

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/models` | Models Studio can serve locally. |
| `POST` | `/v1/chat/completions` | Chat against the local model, scored. |
| `POST` | `/v1/base/chat/completions` | The **unmodified base** model, unscored. Useful as a side-by-side control when demonstrating what the detection layer changes. |
| `POST` | `/v1/completions` | Legacy text completion. |
| `POST` | `/v1/glad/report` *(OpenAI router)* | Report generation on the local-model path. |
| `GET` | `/v1/glad/memory-stats` | Host and GPU memory. |
| `POST` | `/v1/glad/cuda-empty-cache` | Release cached CUDA memory. |
| `GET` | `/v1/glad/files/{filename}` | Download a generated artifact. |

---

## Studio-local endpoints

These are mounted on the Studio app **outside** `/v1`, so the unified port on 8080 does not route to them — they are reachable only on Studio's own port (`:8199` by default). They need a locally loaded research model and are not part of a normal integration.

| Method | Path | What it does |
|---|---|---|
| `GET` | `/health` | Studio process liveness plus a public config snapshot. |
| `POST` | `/glad/evaluate` | Single-call evaluation against the local model. |
| `POST` | `/glad/export_audit` | Export an audit bundle for one session or a list of calls. |
| `POST` | `/glad/finetune` | Submit a fine-tuning job. **202**. |
| `GET` | `/glad/finetune/status/{job_id}` | Poll it. **404** on an unknown job. |

For production scoring use `POST /v1/glad/evaluate` on **G1-Proxy** instead — see [Evaluate](../product-api/evaluate.md).

---

## Miscellaneous

| Method | Path | What it does |
|---|---|---|
| `GET` | `/v1/glad/version` | Studio's component version. The proxy reports its own at `/gw/version`. |
| `GET` | `/v1/glad/documentation` | `docs/USER_GUIDE.md` as `text/markdown`. |
| `POST` | `/v1/glad/admin/reset-demo` | **Destructive.** Wipes chat history, calls, audit chain, FRIA records, kill-switch log, watermarks, retention events, reviews and notifications. Query: `keep_threshold_prefs` (default `true`). Returns the per-table delete counts. |

!!! danger "`admin/reset-demo` is not reversible"
    It exists so a prototype can be handed to a fresh audience. There is no confirmation step and no undo. Do not expose it on a deployment that holds real traffic.
