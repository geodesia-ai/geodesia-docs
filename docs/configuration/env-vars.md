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
| `GW_PROMPT_BLOCK_AXES` | `prompt_safety,jailbreak` | Which prompt-region axes may **refuse** a request. Only **primary** axes are accepted: additional axes (`profanity`, `out_of_scope`) and the routing classifier (`prompt_complexity`) are ignored with a warning on stderr — promote off-topic refusal per Application instead. See [Detection Axes](../g1-proxy/detection-axes.md#primary-axes-vs-additional-axes). |
| `GW_ADDITIONAL_AXES` | `profanity,out_of_scope,prompt_complexity` | Which axes are **additional**: they travel in the payload's `additional_axes` object instead of `axis_energy`, carry `tier: "additional"`, and cannot be promoted to blocking by configuration. Set it to `""` to make every axis primary. |
| `GW_ADDITIONAL_INLINE` | `0` | `1` also emits the additional axes inside `axis_energy` — a migration bridge for a client written before `additional_axes` existed. |
| `GB_EXTRA_AXES` | *(per checkpoint)* | Axes the served head adds on top of the base six, as `name:region,…` (`region ∈ ans\|prompt\|context`). The 9-axis head ships `profanity:prompt,out_of_scope:prompt,prompt_complexity:prompt`. Read by the detector, the gateway, the policy schema and the feedback store from this single source. |
| `GW_SYSTEM_AS_CONTEXT` | `0` | `1` treats system messages as grounding context for `halluc_context`. Off by default: a system prompt is an instruction, not evidence — it feeds `out_of_scope` instead. Enable only if your deployment ships its knowledge base inside the system message. |

### Detection Thresholds (Gateway)

These override the thresholds stored in the database for the gateway instance.

| Variable | Default | Description |
|---|---|---|
| `GW_THR_PROMPT_SAFETY` | from DB | Prompt safety threshold [0–1] |
| `GW_THR_ANSWER_SAFETY` | from DB | Answer safety threshold [0–1] |
| `GW_THR_HALLUC` | from DB | Hallucination threshold [0–1] |
| `GW_THR_COMBINED_HALLUC` | from DB | Combined hallucination threshold [0–1] |
| `GW_THR_JAILBREAK` | from DB | Jailbreak detection threshold [0–1] |

!!! tip "Prefer per-Application thresholds"
    These gateway-wide overrides predate G-1 Studio. In a multi-Application deployment set thresholds in the Application's `policy` instead — they are versioned, audited, hot-reloaded, and tunable against real traffic with [Policy Lens](../studio/policy-lens.md).

### Causal Explainability (Gateway)

Black-box causal attribution over the companion detector. See [Causal Explainability](../g1-proxy/causal-xai.md).

| Variable | Default | Description |
|---|---|---|
| `GW_XAI_MAXLEN` | `512` | Maximum token length for XAI scoring passes. |
| `GW_XAI_SRC_CHARS` | `2400` | Maximum characters of each region (prompt / context / answer) submitted for attribution. |
| `GW_XAI_DCA_RHO` | `0.90` | Sufficiency coverage a subset must reproduce to be **certified**. Raising it to force certificates produces false explanations. |
| `GW_XAI_DCA_FLOOR` | `0.01` | Relevance noise floor in probability units — below it a unit is `irrelevant`. |
| `GW_XAI_DCA_MINBASE` | `0.5` | Minimum axis probability for a score to count as a *decision worth explaining*. This is the **attribution floor**, deliberately distinct from the decision threshold. |
| `GW_ANALYZE_MIN_MAXLEN` | `512` | Minimum maxlen when the gateway auto-halves on OOM. |

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

### Remote G1-Hummingbird scoring

| Variable | Default | Description |
|---|---|---|
| `GEODESIA_SCORING_MODE` | `"embedded"` | `embedded` runs the G1-Hummingbird monitor **in-process** (zero behaviour change). `remote` swaps it for a thin HTTP client that talks to a separate scoring pool — for horizontal scale-out of detection compute. (`src/glad_minimal/scoring/`) |
| `GEODESIA_SCORING_URL` | — | Base URL of the remote scoring server (`python -m glad_minimal.scoring.server`, e.g. `http://host:8810`). **Required** when `GEODESIA_SCORING_MODE=remote`. |

### Thinking Levels

Per-request depth dial (`thinking_level` `0`–`3`, `3` = MAX). **Off by default (level 0) → the extra capacity is never loaded, zero overhead.** See [Thinking Levels](../g1-proxy/thinking-levels.md).

| Variable | Default | Description |
|---|---|---|
| `GW_GLADH_CKPT` | — | Capability pack that unlocks thinking levels **1–2**. Unset → those levels silently fall back to level 0. |
| `GW_GLADH_DEVICE` | `auto` | Device for the level-1/2 pack. `auto` → CUDA when visible, else CPU. Runs in its own sub-process, with its own device selection. |
| `GW_GLADA_CKPT` | — | Capability pack that unlocks thinking level **3 (MAX)**. Unset → a level-3 request is served at the deepest level available. |
| `GW_GLADA_DEVICE` | `auto` | Device for the level-3 pack. |
| `GW_FUSION_BANK` | `runs/glad_bert/fusion_bank_thinking_v3_gladh_v1.json` | Calibration artifact that keeps every level on the same probability scale as level 0, so thresholds do not have to be re-tuned per level. Missing/unreadable → same fallback as an unset pack. |

### Live Web Search

On by default; the chat searches the live web, screens every page through the G1-Hummingbird firewall, and grounds the answer in the safe pages. A **Tavily** API key gives reliable, rate-limit-free results; without one, the key-less DuckDuckGo engine is used as a fallback. See [Live Web Search](../g1-proxy/web-search.md).

| Variable | Default | Description |
|---|---|---|
| `GW_WEBSEARCH_ENABLED` | `1` | Master switch. `1` = available; `0` → the per-request `web_search` flag is a no-op. |
| `GW_WEBSEARCH_API_KEY` | — | Tavily API key. **Takes precedence over a UI/file-set key.** `TAVILY_API_KEY` is an accepted alias. |
| `GW_WEBSEARCH_KEY_FILE` | `/app/var/websearch_tavily.key` | Path to the out-of-band key file the Studio UI writes (mode `0600`, never baked into the image). |
| `GW_WEBSEARCH_PROVIDER` | auto | Force `tavily` or `duckduckgo`. Default: `tavily` if a key is present, else `duckduckgo`. |
| `GW_WEBSEARCH_MAX_RESULTS` | `5` | Number of search results requested. |
| `GW_WEBSEARCH_MAX_READ` | `6` | Maximum number of safe pages used as grounding context. |
| `GW_WEBSEARCH_TIMEOUT` | `12` | Per-request HTTP timeout (seconds) for the search + page fetches. |
| `GW_WEBSEARCH_SCREEN_CHARS` | `4000` | Chars of each page screened by the firewall. |
| `GW_WEBSEARCH_GROUND_CHARS` | `2200` | Chars of each safe page kept as grounding context. |
| `GW_WEBSEARCH_RJ_THR` | calibrated | Override the `rag_jailbreak` firewall threshold for page screening. Unset → per-axis calibrated default. |

### Human Feedback Loop

Opt-in episodic exemplar bank built from approved chat feedback. Off by default → detection is byte-identical. See [Self-Evolving Security](../g1-proxy/self-evolving.md).

| Variable | Default | Description |
|---|---|---|
| `GW_FEEDBACK_BANK` | `off` | Master switch. `on` builds + consults the per-model exemplar bank from approved feedback; `off` → never built, zero overhead. |
| `GW_BANK_V2` | `off` | Use the **Contrastive Safety Memory** bank (dangerous/benign twins). Requires `GW_FEEDBACK_BANK=on`. |
| `GW_FEEDBACK_BANK_TAU` | `0.88` | Cosine-similarity floor below which an exemplar is ignored (exact-pattern recall). |
| `GW_FEEDBACK_BANK_GAIN` | `1.0` | How hard a perfect match pushes the probability (`1.0` = fully to 0/1 at `sim == 1`, `weight == 1`). |
| `GW_FEEDBACK_AUTOAPPROVE` | `off` | `on` ⇒ a flag whose plain-language problem resolves to an axis enters the engine with no curator. `other` / unresolvable flags still queue. |
| `GW_RETRAIN_CMD` | *(unset)* | Trainer command for a `mode: "weights"` re-train, with `{corpus}` / `{out}` placeholders. Unset ⇒ the corpus is exported and the job returns `prepared`. |
| `GW_RETRAIN_CWD` | *(unset)* | Working directory for the trainer subprocess. |
| `GW_RETRAIN_DIR` | `<db dir>/retrain` | Where re-train corpora, logs and outputs are written. |
| `GW_RETRAIN_AUTOPROMOTE` | `1` | On a successful `weights` job, point the live detector at the new checkpoint and restart. Set `0` for regulated deployments where a human signs off on every model change. |

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
