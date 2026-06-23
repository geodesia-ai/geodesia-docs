# Environment Variables

All Geodesia G-1 services can be configured via environment variables. Variables always take precedence over `config.yaml` values.

---

## Product Backend

Set these variables when starting the product backend (`python -m uvicorn main:app ...`).

### Core

| Variable | Default | Description |
|---|---|---|
| `MODEL_HOST_PATH` | — | Absolute path to the model checkpoint directory. If unset, the evaluate endpoint is disabled and only compliance APIs are available. |
| `GLAD_DEVICE` | `"cuda:0"` | CUDA device to load the model on. Values: `"cuda:0"`, `"cuda:1"`, `"cpu"`. |
| `GLAD_CONFIG` | `"config.yaml"` | Path to the YAML configuration file. |
| `GLAD_DB_PATH` | `"var/glad.sqlite3"` | Path to the SQLite database file. |
| `GLAD_LOG_LEVEL` | `"info"` | Logging verbosity: `"debug"`, `"info"`, `"warning"`, `"error"`. |

### Security

| Variable | Description |
|---|---|
| `GLAD_LICENSE_TOKEN` | License token (`glc_…`). Required for authenticated deployments. |
| `GLAD_AUDIT_HMAC_KEY` | 64-character hex key for signing the audit chain. If unset, a random key is generated (chain cannot be verified across restarts). |
| `GLAD_SMTP_PASSWORD` | SMTP password for email notifications. |
| `GLAD_WEBHOOK_SECRET` | Webhook signing secret for notification webhooks. |

### Explainability

| Variable | Default | Description |
|---|---|---|
| `GLAD_PSS_N_SAMPLES` | `5` | Default PSS resample count |
| `GLAD_PSS_TEMPERATURE` | `0.7` | Default PSS temperature |
| `GLAD_PSS_MATCH_MODE` | `"ngram"` | Default PSS alignment algorithm |

### Fine-Tuning

| Variable | Default | Description |
|---|---|---|
| `GLAD_FINETUNE_OUTPUT_DIR` | `"runs/"` | Directory where fine-tuned checkpoints are written |
| `GLAD_MAX_FINETUNE_JOBS` | `1` | Maximum concurrent fine-tuning jobs |

---

## Gateway

Set these variables when starting the gateway (`python -m uvicorn geodesia_gateway:app ...`).

### Core

| Variable | Default | Description |
|---|---|---|
| `GLAD_API_URL` | `"http://localhost:8199"` | URL of the product backend |
| `GW_PORT` | `8800` | Port the gateway listens on |
| `GW_HOST` | `"0.0.0.0"` | Bind address for the gateway |
| `GW_DB_PATH` | same as product `GLAD_DB_PATH` | Path to the SQLite database. By default, the gateway and product backend share one database. Override to use a separate file. |
| `GW_LOG_LEVEL` | `"info"` | Logging level |

### Upstream

| Variable | Default | Description |
|---|---|---|
| `GW_UPSTREAM_URL` | — | URL of the upstream LLM backend (vLLM, Ollama, OpenAI, etc.) |
| `GW_UPSTREAM_TYPE` | `"vllm"` | Upstream type: `"vllm"`, `"openai"`, `"ollama"`, `"sglang"`, `"trtllm"`, `"internal"` |
| `GW_UPSTREAM_MODEL` | — | Model name or path passed to the upstream (e.g., `"mistralai/Mistral-7B-v0.3"`) |
| `GW_API_KEY` | — | API key forwarded to OpenAI-compatible upstreams |
| `GW_MAXLEN` | `2048` | Maximum input token length sent to upstream. Must match what the upstream was compiled/configured for. |

### Enforcement

| Variable | Default | Description |
|---|---|---|
| `GW_BLOCK_INPUT` | `0` | Set to `1` to enable input blocking (prompt safety + jailbreak enforcement). `0` = annotate only. |
| `GW_BLOCK_OUTPUT` | `0` | Set to `1` to enable output blocking (answer safety + hallucination enforcement). |
| `GW_DEFAULT_MODE` | `"block"` | Default enforcement mode: `"block"` or `"passthrough"` |

### Detection Thresholds (Gateway)

These override the thresholds stored in the database for the gateway instance.

| Variable | Default | Description |
|---|---|---|
| `GW_THR_PROMPT_SAFETY` | from DB | Prompt safety threshold [0–1] |
| `GW_THR_ANSWER_SAFETY` | from DB | Answer safety threshold [0–1] |
| `GW_THR_HALLUC` | from DB | Hallucination threshold [0–1] |
| `GW_THR_COMBINED_HALLUC` | from DB | Combined hallucination threshold [0–1] |
| `GW_THR_JAILBREAK` | from DB | Jailbreak detection threshold [0–1] |

### RAG

| Variable | Default | Description |
|---|---|---|
| `GW_RAG_ENABLED` | `0` | Set to `1` to enable the RAG module |
| `GW_RAG_DB_PATH` | `"var/rag.lancedb"` | Path to the LanceDB vector store |
| `GW_RAG_EMBEDDING_MODEL` | `"BAAI/bge-m3"` | Embedding model for document ingestion |
| `GW_RAG_RERANKER_MODEL` | `"BAAI/bge-reranker-v2-m3"` | Reranking model for retrieval |
| `GW_RAG_TOP_K` | `5` | Number of documents to retrieve |

### Numeric Solver

| Variable | Default | Description |
|---|---|---|
| `GW_NUMERIC_SOLVER` | `"disabled"` | Enable the numeric faithfulness solver. Values: `"disabled"`, `"judge"`. When `"judge"`, uses an external LLM to verify arithmetic claims. |
| `GW_NUMERIC_SOLVER_URL` | — | URL of the numeric solver LLM endpoint |
| `GW_NUMERIC_SOLVER_MODEL` | — | Model name for the numeric solver |

---

## G-1 Studio (Applications, cost, scale-out)

These variables configure the **G-1 Studio** control plane (Applications, organizations, keys), the FinOps cost engine, and the multi-replica scale-out path. They are **additive and backward-compatible**: an existing single-upstream deployment surfaces as the `default` Application with zero behaviour change, so all of these are optional.

### Storage backend

| Variable | Default | Description |
|---|---|---|
| `GEODESIA_DB_URL` | (the resolved `GLAD_DB_PATH`) | Selects where all Studio state lives. `sqlite:////abs/path/glad.sqlite3` (note the **four** slashes for an absolute path; `sqlite:///rel/path` is relative to CWD) — the same artifact then runs from a laptop file to a cloud VM (e.g. SQLite on an EBS volume). `postgresql+psycopg://user:pw@host/db` points the control-plane + cost stores at Postgres for multi-replica scale-out. The `settings.database_path` and the gateway `_db()` both honor this. (`src/glad_minimal/storage.py`, `db_backend.py`) |

!!! note "Split storage on multi-replica"
    The control-plane and cost stores are backend-agnostic (portable DDL + `ON CONFLICT` upserts), so they can run on Postgres while the legacy compliance DB stays on SQLite (single-writer, on a shared FS). See the Studio deployment notes for the recommended split.

### RBAC (control-plane access)

| Variable | Default | Description |
|---|---|---|
| `GEODESIA_ADMIN_TOKEN` | — | Platform-admin token that gates **control-plane writes** (Applications / orgs / keys / policy / cost). Sent by the caller as the header `X-Geodesia-Admin-Key`. When **unset and no admin API key exists**, the control plane is **open** (anonymous caller is treated as `platform_admin`) — preserving the local/single-tenant experience. As soon as an admin token or an admin API key is configured, writes require authentication. (`src/glad_minimal/apps/rbac.py`) |

An admin can alternatively authenticate with a control-plane API key (`Authorization: Bearer g1k_…` with the `admin` role). Role hierarchy: `viewer < app_editor < org_admin < platform_admin`.

### Budget alerts (FinOps)

| Variable | Default | Description |
|---|---|---|
| `GEODESIA_ALERT_WEBHOOK` | — | Outbound webhook URL for **budget alert/block events**. When set, each newly-crossed budget event is POSTed here as JSON; the per-Application `alert_recipients` list rides **inside** the payload so your own relay can fan it out to email / Slack / PagerDuty. When **unset**, events are still recorded in the in-app `notification_log` (no network egress). There is **no SMTP in the product** and the webhook URL is server-side only — never per-Application, never in the DB. (`src/glad_minimal/cost/alerts.py`) |

See [Cost & Budget](../studio/cost.md) for the FinOps engine, budget bands, and the projection chart.

### Remote GLAD-Hummingbird scoring

| Variable | Default | Description |
|---|---|---|
| `GEODESIA_SCORING_MODE` | `"embedded"` | `embedded` runs the GLAD-Hummingbird monitor **in-process** (zero behaviour change). `remote` swaps it for a thin HTTP client that talks to a separate scoring pool — for horizontal scale-out of detection compute. (`src/glad_minimal/scoring/`) |
| `GEODESIA_SCORING_URL` | — | Base URL of the remote scoring server (`python -m glad_minimal.scoring.server`, e.g. `http://host:8810`). **Required** when `GEODESIA_SCORING_MODE=remote`. |

### Deep Scan (GLAD-Tapestry)

Opt-in 8B guardian second opinion, blended into the safety and hallucination axes. **Off by default → never loaded, zero overhead.** See [Deep Scan](../gateway/deep-scan.md).

| Variable | Default | Description |
|---|---|---|
| `GW_DEEP_SCAN` | `off` | Platform availability switch. `on` makes GLAD-Tapestry loadable (still lazy-loaded on first use); also requires `deep_scan: true` per request. |
| `GW_DEEP_SCAN_DIR` | — | Path to a **trained GLAD-Tapestry export** directory (preferred — geometry-head continuous scorer). |
| `GW_DEEP_SCAN_MODEL` | `ibm-granite/granite-guardian-4.1-8b` | Guardian model id for the zero-shot fallback (when `GW_DEEP_SCAN_DIR` is unset). |
| `GW_DEEP_SCAN_QUANT` | `4bit` | `4bit` (bnb nf4, ~5 GB VRAM, needs CUDA) or empty for bf16 (~16 GB). |
| `GW_DEEP_SCAN_DEVICE` | `auto` | `auto` → CUDA → MPS → CPU. Pin to e.g. `cuda:0`. |
| `GW_DEEP_SCAN_LORA` | — | Optional QLoRA symbiont adapter directory for the zero-shot guardian. |

### Cloud upstreams & secrets

| Variable | Default | Description |
|---|---|---|
| `GEODESIA_SECRET_PROVIDER` | `"file"` | Selects how `secret://app/name` upstream credential references are resolved: `file` (local JSON secrets file, air-gap friendly) or `aws_secrets` (AWS Secrets Manager, with local-file fallback). (`src/glad_minimal/secrets_provider.py`) |
| `GEODESIA_SECRET__<APP>__<NAME>` | — | Per-secret **env override**, checked first regardless of provider. `<APP>` / `<NAME>` are the `secret://<app>/<name>` reference segments, upper-cased with non-alphanumerics replaced by `_` (e.g. `secret://app_invoices_eu/upstream` → `GEODESIA_SECRET__APP_INVOICES_EU__UPSTREAM`). |
| `AWS_REGION` | — | Region for AWS Bedrock upstreams **and** AWS Secrets Manager. |
| `GOOGLE_APPLICATION_CREDENTIALS` | — | Path to the service-account JSON for Google Vertex (Application Default Credentials). |

!!! danger "Credentials are never stored in plaintext"
    An Application's `binding.api_key_ref` holds a `secret://app/<name>` **reference**, never the secret. The provider resolves it at request time (env override → file → AWS Secrets Manager). Bedrock and Vertex use the host's IAM role / ADC and need no key at all. See [Cloud Upstreams](../studio/cloud-upstreams.md).

### Free-tier entitlement overrides

The free tier (no signed license) defaults to **1 Application / 1 model / 20 chats per day**. These dev/test overrides relax those caps locally; **unset** leaves the product defaults unchanged. Each accepts an **integer**, or `0` / `unlimited` / `none` for **unlimited**. (`src/glad_minimal/entitlements.py`)

| Variable | Default (free tier) | Description |
|---|---|---|
| `GLAD_FREE_MAX_APPLICATIONS` | `1` | Override the free-tier maximum number of Applications. |
| `GLAD_FREE_MAX_MODELS` | `1` | Override the free-tier maximum number of distinct upstream models. |
| `GLAD_FREE_MAX_CHATS_PER_DAY` | `20` | Override the free-tier daily chat cap (UTC-day bucketed). |

A **signed license** supersedes the free tier entirely and declares its own `max_applications` / `max_models` / `max_chats_per_day` (any of which may be `null` for unlimited); these env overrides apply only when no valid license is installed. See [Licensing](../studio/licensing.md).

!!! info "License token"
    `GLAD_LICENSE_TOKEN` (documented under [Security](#security)) carries the entitlement that sets `max_applications` and unlocks additional Applications beyond the free-tier cap.

---

## Docker Compose Variables

When using the provided Docker Compose setup, all variables above can be placed in a `.env` file alongside `docker-compose.yml`:

```bash
# .env
GLAD_LICENSE_TOKEN=glc_your_token_here
MODEL_HOST_PATH=/models/geodesia-g1
GLAD_DEVICE=cuda:0
GW_UPSTREAM_URL=http://vllm:8000
GW_UPSTREAM_TYPE=vllm
GW_UPSTREAM_MODEL=mistralai/Mistral-7B-v0.3
GW_BLOCK_INPUT=1
GW_BLOCK_OUTPUT=1
```

---

## Precedence Order

When the same setting is available in multiple places, this is the resolution order (highest to lowest precedence):

1. **Per-request override** — `threshold_overrides` in the API request body
2. **Environment variable** — `GW_THR_*` or `GLAD_*`
3. **Database threshold prefs** — set via `POST /v1/glad/threshold-prefs`
4. **config.yaml** value
5. **Compiled default**
