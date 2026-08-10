# Studio UI — Component Reference

Every screen and control in the G-1 Studio UI, and the exact API call behind it. Use this page when you want to **do from code what you just did by clicking** — or when you are debugging and need to know which endpoint a panel is actually hitting.

!!! info "How to read the paths"
    The Studio UI is a single-page app served from the same origin as the API, so every path below is same-origin and relative.

    | Prefix | Reaches | Documented in |
    |---|---|---|
    | `/v1/…` | **G-1 Studio** (control plane) | [Studio API map](api-reference.md) |
    | `/gw/…` | **G1-Proxy** (data plane), with `/gw` stripped before it arrives | [G1-Proxy API map](../g1-proxy/api-reference.md) |

    So the **Knowledge Base** page calling `/gw/v1/glad/rag/collections` is calling the proxy's `/v1/glad/rag/collections`.

!!! tip "The header every per-app control sends"
    The Application picker in the top bar writes the active app id to browser storage, and the UI attaches it as **`X-Geodesia-App`** on every proxy call that is scoped per Application — RAG, feedback, audio, web search, chat. If you reproduce those calls from code and omit the header, you get the `default` Application's data, not the one you were looking at.

---

## Reproducing any control from code

Every row below is a real HTTP call. Take the path, prefix the host, and send it — the UI has no private
API.

```bash
# what the Application picker reads
curl -s http://localhost:8080/v1/glad/apps | jq '[.apps[].app_id]'

# what a per-Application panel sends (note the header)
curl -s http://localhost:8080/gw/v1/glad/rag/collections \
  -H "X-Geodesia-App: support_bot" | jq
```

## Chrome — always on screen

| Component | What it is | Calls |
|---|---|---|
| **Sidebar** | Navigation. Entries come from the router, not from the API. | — |
| **App Switcher** (top bar) | Picks the active Application. Populated from the application store. Selecting one re-scopes every page below. | `GET /v1/glad/apps` |
| **App Scope Banner** | The strip reminding you which Application you are looking at. Reads the store; makes no call of its own. | — |
| **Version readout** | Both component versions side by side — they are released independently and can differ. Available to the UI through a shared composable; not every build surfaces it in the chrome. | `GET /v1/glad/version` (Studio) · `GET /gw/version` (Proxy) |
| **Update / licence banner** | Polls for an available update and for a licence expiring within 14 days. | `GET /gw/v1/glad/gateway/notifications` |
| **Backend-ready gate** | Blocks the UI with a "starting up" state until both services answer. | `GET /v1/glad/health` · `GET /gw/v1/glad/gateway/calibration` |
| **Calibration banner** | The strip that appears while a closed-book recalibration is running. | `GET /gw/v1/glad/gateway/calibration` · `GET /v1/glad/calibration/status` |

---

## Dashboard

Route `/` — the landing page.

| Control | What it shows | Calls |
|---|---|---|
| Health tiles | Overall compliance posture. | `GET /v1/glad/dashboard` |
| Regulatory scorecard | Per-framework coverage. | `GET /v1/glad/scorecard` |
| Traffic & detection charts | Passed / blocked / hallucinated / unsafe over time, scoped to the active Application. | `GET /v1/glad/dashboard/charts?application_id=…` |

---

## Applications

Route `/applications`. The control-plane editor. See [Managing Applications](applications.md).

| Control | What it does | Calls |
|---|---|---|
| Application list | | `GET /v1/glad/apps` |
| **New Application** | | `POST /v1/glad/apps` 🔑 |
| Name / config save | | `PUT /v1/glad/apps/{app_id}` 🔑 |
| Delete | | `DELETE /v1/glad/apps/{app_id}` 🔑 |
| **Pause** / **Resume** | Stop or resume traffic without losing the configuration. | `POST /v1/glad/apps/{app_id}/pause` · `/resume` 🔑 |
| **Kill** | Per-Application kill switch. | `POST /v1/glad/apps/{app_id}/kill` 🔑 |
| Axis threshold sliders + enforcement selectors | The nine axes, each with a threshold and `block` / `annotate` / `off`. The set of sliders is built from what the served detector reports it has. | `GET /v1/glad/apps/meta` → `PUT /v1/glad/apps/{app_id}/policy` 🔑 |
| **Upstream** panel (type, base URL, model, key) | | field edits → `PUT /v1/glad/apps/{app_id}` 🔑 |
| **Discover models** button | Lists what the backend offers. Sends `api_key: "***"` plus `application_id` so the stored key is used and the browser never holds it. | `POST /v1/glad/apps/upstream/models` |
| **Test connection** | Connects, lists models, probes log-probability support, returns a sample reply and its latency. | `POST /gw/upstream/test` |
| **Test capabilities** | Runs the detector on built-in safe/unsafe pairs and reports which axes separate. Touches **no** upstream. | `POST /gw/test-capabilities` 🔒 |
| **Complex routing** panel | Enable, threshold, and the Model-B binding. | `PUT /v1/glad/apps/{app_id}/routing` 🔑 |
| **Cost** panel | Token rates, monthly budget, alert bands, over-budget policy. | `PUT /v1/glad/apps/{app_id}/cost` 🔑 |
| Cost preview | | `GET …/cost/summary` · `…/cost/daily` · `…/cost/forecast` |
| **API keys** table | List, mint, revoke. The plaintext key is shown **once**, in the create response. | `GET`/`POST /v1/glad/apps/{app_id}/keys` · `DELETE …/keys/{key_id}` 🔑 |
| **Export data** | Downloads everything this Application saw. Honours the server's `Content-Disposition` filename. | `GET /v1/glad/apps/{app_id}/export?fmt=sqlite\|csv\|jsonl` 🔑 |
| **Model calibration** card | Embedded per-app closed-book recalibration for the selected model. | `GET /v1/glad/calibration/capabilities` · `/status` · `POST /v1/glad/calibration/run` |

---

## Chat

Route `/chat`. The interactive console — every detection surface in one place.

| Control | What it does | Calls |
|---|---|---|
| Message composer → **Send** | Sends the turn through the full pipeline. Attaches the active Application, the enforcement mode, threshold overrides, the thinking level, RAG selection and the web-search flag as body fields. | `POST /gw/v1/chat/completions` (proxy path) or `POST /v1/chat/completions` (local-model path) |
| **Thinking level** dropdown | Sets `thinking_level` `0`–`3` on the request body. Level 3 is MAX. | body field on the chat call — [Thinking Levels](../g1-proxy/thinking-levels.md) |
| **Mode** toggle (block / passthrough) | Sets `mode` on the body, overriding the Application's enforcement for this turn only. | body field — [Enforcement Modes](../g1-proxy/enforcement-modes.md) |
| **Web search** toggle | Sets `web_search: true`. | body field — [Live Web Search](../g1-proxy/web-search.md) |
| **Knowledge base** picker | Sets `rag: {collection_id, …}`. | body field — [Knowledge Base](../rag/index.md) |
| **Microphone** button | Streams PCM to the voice guard and brakes mid-utterance. | `WS /gw/v1/glad/audio/stream`, config from `GET /gw/v1/glad/audio/status` |
| **Score overlay** / **Energy panel** | Per-axis probability, threshold and flag. Read straight from `geodesia.axis_energy` in the chat response — **no second call**. | — |
| **Token heatmap** / **Prompt XAI** / **Response XAI** cards | The dual-surface attribution that runs automatically at verdict time: which *prompt* tokens caused a block, which *answer* tokens caused a flag. | `POST /gw/v1/glad/causal-explainability/analyze` with `method: "dca_dual"` |
| **RAG sources** panel | Retrieved passages and claim verification. From `geodesia.rag` in the response. | — |
| **Web research** panel | Pages searched, screened and used. From the response. | — |
| **Flag this answer** | Files a correction into the review queue. The form's axes and verdicts come from the schema endpoint rather than being hard-coded. | `GET /gw/v1/glad/feedback/schema` → `POST /gw/v1/glad/feedback` |
| Bank badge | How many approved corrections are live in the exemplar bank. | `GET /gw/v1/glad/feedback/bank/status` |

---

## Knowledge Base

Route `/knowledge`. See [Knowledge Base](../rag/index.md). Everything here is on the **proxy**, scoped by `X-Geodesia-App`.

| Control | What it does | Calls |
|---|---|---|
| Collection list | | `GET /gw/v1/glad/rag/collections` |
| **New collection** | | `POST /gw/v1/glad/rag/collections` |
| Delete collection | | `DELETE /gw/v1/glad/rag/collections/{id}` |
| **Upload document** | Multipart POST returns **202** immediately; the UI then polls the progress endpoint until `stage` is `done` or `error`. It does **not** hold the connection open — a large PDF would outlive any proxy timeout. | `POST …/collections/{id}/documents` → poll `GET /gw/v1/glad/rag/ingest/progress` |
| Delete document | | `DELETE …/collections/{id}/documents/{doc_id}` |
| **Test query** box | Retrieve passages without sending a chat turn. | `POST …/collections/{id}/query` |
| Status strip | Whether the RAG stack is available at all. | `GET /gw/v1/glad/rag/status` |

---

## Cost & FinOps

Route `/cost`. See [Cost & FinOps](cost.md).

| Control | What it shows | Calls |
|---|---|---|
| Organisation total | | `GET /v1/glad/orgs/{org_id}/cost/summary` |
| Per-Application spend | | `GET /v1/glad/apps/{app_id}/cost/summary` |
| Daily series chart | | `GET /v1/glad/apps/{app_id}/cost/daily` |
| Budget & projection chart | Month-end projection against the budget band. | `GET /v1/glad/apps/{app_id}/cost/forecast?method=run_rate` |

---

## FRIA

Route `/fria`. See [FRIA](../compliance/fria.md).

| Control | What it does | Calls |
|---|---|---|
| Dossier list | | `GET /v1/glad/fria?deployer_id=…` |
| **New FRIA** | | `POST /v1/glad/fria` |
| **Auto-prefill** | Drafts the answers the deployment can already prove. | `GET /v1/glad/fria/auto-prefill` |
| Save | | `PUT /v1/glad/fria/{id}` |
| **Approve** | | `POST /v1/glad/fria/{id}/approve` |
| **Archive** | | `POST /v1/glad/fria/{id}/archive` |
| **Runtime evidence** panel | The measured half of the dossier: what the last 90 days of real traffic actually shows. | `GET /v1/glad/fria/{id}/runtime-evidence?window_days=90` |
| **Export** (PDF / DOCX / JSON) | Runs as a background job with a progress state, because a full render can outlive a connection timeout. | `POST /v1/glad/fria/{id}/export/async` → `GET /v1/glad/doc-jobs/{job_id}/status?token=…` → `…/download?token=…` |

---

## Kill Switch

Route `/kill-switch`. See [Kill Switch](../compliance/kill-switch.md).

| Control | What it does | Calls |
|---|---|---|
| State indicator | | `GET /v1/glad/kill-switch/status?deployer_id=…` |
| **Activate** (reason required) | | `POST /v1/glad/kill-switch/activate` |
| **Deactivate** | | `POST /v1/glad/kill-switch/deactivate` |

---

## Oversight

Route `/oversight`. See [Human Oversight](../compliance/oversight.md).

| Control | What it does | Calls |
|---|---|---|
| Review queue | Pending reviews, scoped to the active Application. | `GET /v1/glad/oversight/pending?application_id=…` |
| Counters | | `GET /v1/glad/oversight/summary` |
| **Approve / Reject / Escalate / Modify** | Sends `review_id` plus the decision. `escalated` opens a fresh review one level up. | `POST /v1/glad/oversight/decide` |
| **Verify chain** button | Proves the decision trail has not been altered. | `GET /v1/glad/chain/verify` |

---

## Feedback

Route `/feedback`. The curation queue. See [Human Feedback Loop](../g1-proxy/feedback.md). All on the **proxy**.

| Control | What it does | Calls |
|---|---|---|
| Queue table | Filterable by status, axis, region, Application. | `GET /gw/v1/glad/feedback?status=…&axis=…` |
| Counters | | `GET /gw/v1/glad/feedback/stats` |
| **Approve / Reject** with axis, verdict, note, weight | | `POST /gw/v1/glad/feedback/{id}/review` |
| Delete | | `DELETE /gw/v1/glad/feedback/{id}` |
| **Export corpus** | | `GET /gw/v1/glad/feedback/export?status=approved` |
| Bank status | | `GET /gw/v1/glad/feedback/bank/status` |
| **Retrain** — *memory* | Refreshes the exemplar bank immediately. | `POST /gw/v1/glad/feedback/retrain` `{mode:"memory"}` |
| **Retrain** — *weights* | Exports a corpus and launches the trainer; the button then polls the job. | `POST …/retrain` `{mode:"weights"}` → `GET …/retrain/status?job_id=…` |
| **Auto-judge** card | The idle-CPU reviewer: on/off, model, queue depths, disagreements, promotions. | `GET /gw/v1/glad/feedback/auto/status` · `/auto/config` · `PUT /auto/config` · `GET /auto/items` |
| **Preview judge prompt** | Read-only. You configure the description of your organisation; the audit protocol around it is not a setting. | `GET /gw/v1/glad/feedback/auto/prompt-preview?axis=…` |

---

## Audit & Chain

Route `/audit`. See [Audit Chain](../compliance/audit-chain.md) and [AI Watermark](../compliance/watermark.md).

| Control | What it does | Calls |
|---|---|---|
| Ledger snapshot + recent entries | | `GET /v1/glad/chain/status` |
| Retention status | | `GET /v1/glad/retention/status` |
| **Verify watermark** | Paste text, or give `call_id` + `watermark_id` for a full live verification. | `POST /v1/glad/watermark/verify` |

---

## Reports

Route `/reports`. See [Reports & Manuals](../compliance/reports.md).

| Control | What it does | Calls |
|---|---|---|
| **Generate report** (data) | | `POST /v1/glad/report` |
| **Download report PDF** | Background job with progress. | `POST /v1/glad/report/pdf/async` → doc-job poll → download |
| **Deployer manual** (data) | | `POST /v1/glad/deployer-manual` |
| **Download manual PDF** | Background job with progress. | `POST /v1/glad/deployer-manual/pdf/async` → doc-job poll → download |

If a render fails, the failure reason surfaces from the job's `error` field — it is the message that tells the user what to fix (*no logged calls*, *no applicable law selected*, …), not a generic failure.

---

## Legal Frameworks

Route `/legal`.

| Control | Calls |
|---|---|
| Framework list | `GET /v1/glad/legal/frameworks` |
| Framework detail | `GET /v1/glad/legal/{framework_id}` |

---

## Causal Explainability

Route `/causal-explainability`. See [Causal Explainability](../g1-proxy/causal-xai.md).

| Control | What it does | Calls |
|---|---|---|
| Session / message picker | Loads a past turn to analyse. | `GET /v1/glad/chat-sessions` · `GET /v1/glad/chat-messages?session_id=…` |
| **Analyse** — proxy path | Black-box attribution over the detector. No local generator, no GPU. This is the path an OpenAI-API deployment uses. | `POST /gw/v1/glad/causal-explainability/analyze` |
| **Analyse** — local path | Needs a locally loaded model. Runs as a job with a progress bar. | `POST /v1/glad/causal-explainability/jobs` → `GET …/jobs/{job_id}` |
| **Cancel** | Aborts the in-flight request, not just the poll loop. | client-side abort |
| Dual-surface cards | Prompt and answer attributed separately. | `analyze` with `method: "dca_dual"` |

---

## MuPAX

Route `/mupax`.

| Control | Calls |
|---|---|
| **Explain** | `POST /v1/glad/mupax/explain` |
| Language selector | `GET /v1/glad/mupax/languages` |

---

## Model Calibration

Route `/calibration`.

| Control | Calls |
|---|---|
| Capability check | `GET /v1/glad/calibration/capabilities` |
| Artifact registry | `GET /v1/glad/calibration/registry` |
| **Probe generator** | `POST /v1/glad/calibration/probe` |
| **Run** (fast / deep) | `POST /v1/glad/calibration/run` |
| Live status | `GET /v1/glad/calibration/status` |
| **Stop** | `POST /v1/glad/calibration/stop` |

---

## Agent Flow

Route `/agent-flow`. See [Agent Flow](../agent-flow/index.md).

| Control | Calls |
|---|---|
| Demo picker | `GET /v1/agent-flow/demos` |
| Graph / trace | `GET /v1/agent-flow/trace/{demo_type}` |
| **Run** | `POST /v1/agent-flow/run/{demo_type}` |
| **Compute stability** | `POST /v1/agent-flow/compute_pss/{demo_type}` |

---

## Settings

Route `/settings`. See [Settings](settings.md).

| Card | What it configures | Calls |
|---|---|---|
| **Licence** | Paste a vendor-signed key; read tier, company, daily chat limit, models, expiry. Applying or removing is immediate. | `GET`/`POST`/`DELETE /gw/v1/glad/gateway/license` · `GET /gw/v1/glad/gateway/entitlements` |
| **Gateway** | Upstream binding, enforcement defaults, constitutional prompt switch, streaming cadence, numeric solver. Saves apply live. | `GET`/`POST /gw/v1/glad/gateway/config` |
| **PII guard** status | Whether redaction is *effective* — an enabled switch on an image without the library protects nothing, and this card says so instead of showing a green shield. | `GET /gw/v1/glad/pii/status` |
| **MCP** | Which MCP servers and interceptors are listening, and the per-surface actions. Read-only here; configured by env. | `GET /gw/v1/glad/mcp/status` |
| **Voice input (ASR)** | Whisper model picker for the live-voice guard, plus a **Test microphone** button. | `GET /gw/v1/glad/audio/status` · `POST /gw/v1/glad/audio/utterance` |
| **Web search** | Store the provider API key out-of-band. The key is never returned in clear — only a masked hint. `env_locked` means an environment variable wins. | `GET`/`POST /gw/v1/glad/websearch/config` |
| **Database** | SQLite by default; point Studio at an existing PostgreSQL, test it, deploy the schema. `GEODESIA_DB_URL` always overrides; a switch takes effect after a restart. | `GET /v1/glad/db/config` · `POST /v1/glad/db/test` · `/db/deploy` · `/db/use-sqlite` |
| **Detection thresholds** (deployer profile) | The deployer-wide default profile, distinct from per-Application policy. | `GET`/`POST /v1/glad/threshold-prefs` |

!!! note "Gateway settings talk to the proxy directly when it is remote"
    When the configured gateway is co-located (`localhost`, `127.0.0.1`, `::1`) or the URL is unparseable, the UI goes through the same-origin `/gw` proxy. Only a genuinely remote gateway is addressed by its own URL. If you are reproducing these calls, target the proxy directly.

---

## Documentation & API Docs

| Route | What it is | Calls |
|---|---|---|
| `/documentation` | The bundled user guide, rendered in-product so it works air-gapped. | `GET /v1/glad/documentation` (Studio) or `GET /gw/v1/glad/documentation` (proxy-only deployments) |
| `/api-docs` | An in-product, browsable endpoint catalogue. Static data compiled into the UI — it makes no API call to build the list. | — |

---

## Customer Tokens

Route `/_g1-license-vault` — unlisted, not in the sidebar. Requires `GLAD_LICENSE_ADMIN_KEY` supplied as `X-Geodesia-Admin-Key`; without the variable configured server-side the endpoints return `503`.

| Control | Calls |
|---|---|
| Model scope list | `GET /v1/glad/license-tokens/models` |
| **Issue token** | `POST /v1/glad/license-tokens` 🔐 |

Listing and revoking are API-only — `GET /v1/glad/license-tokens` and `POST /v1/glad/license-tokens/{token_id}/revoke`, both 🔐. This page only issues.
