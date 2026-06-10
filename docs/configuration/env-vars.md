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
