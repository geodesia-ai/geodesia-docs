# Cloud Upstreams & Scale-Out

Geodesia G-1 Studio runs unchanged from a developer laptop to a multi-replica cloud deployment. The same artifact talks to **managed cloud LLMs** (AWS Bedrock, Google Vertex, Azure OpenAI), resolves provider credentials through a **pluggable secret provider**, stores its control-plane and cost state on **SQLite or Postgres**, offloads GLAD-BERT detection to a **remote scoring pool**, and ingests RAG documents through an **exactly-once claim queue** that is safe across replicas.

This page documents each of those scale-out surfaces. For a full step-by-step walkthrough of taking a PC deployment to AWS, see the section ["PC → AWS deployment guide"](#pc-aws-deployment-guide) at the bottom.

---

## Cloud upstream adapters

An Application's binding selects which LLM it talks to. In addition to the local OpenAI-compatible backends (`vllm`, `sglang`, `trtllm`, `openai`, `ollama`, `internal` — see [Upstream Backends](../gateway/backends.md)), `binding.upstream_type` can be one of three **managed cloud providers**:

| `upstream_type` | Provider | SDK | Auth | Logprobs |
|---|---|---|---|---|
| `bedrock` | AWS Bedrock | `boto3` (lazy) | IAM role / instance profile | ❌ |
| `vertex` | Google Vertex / Gemini | `google-genai` (lazy) | Application Default Credentials | ❌ |
| `azure-openai` | Azure OpenAI | OpenAI-compatible path | api-key header | ✅ |

The gateway resolves the streaming adapter for a binding via `get_adapter(upstream_type)`. Bedrock and Vertex return a dedicated async streaming callable; the OpenAI-compatible providers (including Azure) return `None` and are handled inline by the gateway's existing HTTP path. Every adapter yields `(delta_text, token_records)` tuples and fills a `usage_out` dict with real `prompt_tokens` / `completion_tokens` when the provider reports them — so the cost ledger records actual provider usage rather than an estimate (the gateway falls back to `~chars/4` only when usage is absent).

### Bedrock

The Bedrock adapter calls boto3's `converse_stream` and bridges its synchronous event stream to async via a worker thread and a bounded queue. OpenAI chat messages are translated into Bedrock's Converse format:

- **System messages are split out** into a separate `system` block list (Bedrock keeps system prompts separate from the conversation).
- **Consecutive same-role turns are merged** — Bedrock rejects non-alternating user/assistant roles.
- **The conversation must start with a user turn** — if it does not, a placeholder user turn is inserted.

Inference parameters map across as `max_tokens → maxTokens`, `temperature → temperature`, `top_p → topP`. **Real usage tokens** are read from the stream's `metadata` event (`usage.inputTokens` / `usage.outputTokens`).

Authentication uses the standard AWS credential chain — an IAM task role on ECS/EKS, an instance profile on EC2, or environment credentials. **There is no API key.** The region comes from the app's `binding.region` or, failing that, the ambient `AWS_REGION`.

!!! note "boto3 is an optional runtime dependency"
    `boto3` is imported lazily — it is only needed on the box that actually talks to Bedrock. Install it there with `pip install boto3`; deployments that never use a Bedrock binding do not need it.

### Vertex

The Vertex adapter uses the `google-genai` client with **Application Default Credentials** — set `GOOGLE_APPLICATION_CREDENTIALS` to a service-account JSON file, or rely on workload identity. When `GOOGLE_CLOUD_PROJECT` is set the client runs against the Vertex backend (`location` from `binding.region` or `GOOGLE_CLOUD_LOCATION`, default `us-central1`); otherwise it falls back to the Gemini API with `GOOGLE_API_KEY`.

OpenAI messages are translated to Gemini `contents` (the `assistant` role becomes `model`; system messages become a `system_instruction`). Inference params map as `max_tokens → max_output_tokens`, `temperature`, `top_p`. **Usage tokens** come from the final chunk's `usage_metadata` (`prompt_token_count` / `candidates_token_count`).

!!! note "google-genai is imported lazily"
    Like boto3, `google-genai` is a runtime-optional dependency loaded only when a Vertex binding is used.

### Azure OpenAI

Azure OpenAI is **not a separate adapter** — `get_adapter("azure-openai")` returns `None` and the gateway serves it through its existing OpenAI-compatible HTTP path, passing the Azure API key as an `api-key` header. Because it is an OpenAI-compatible endpoint, it **does** expose per-token logprobs, so all detection axes remain available.

!!! warning "Bedrock and Vertex do not expose per-token logprobs"
    The `halluc_closedbook` axis is computed from per-token log-probabilities. Bedrock and Vertex do not return them (their adapters always yield empty `token_records`), so for those upstreams the closed-book axis is **automatically advisory / off** — no configuration change is required. The gateway degrades to its 4/5-axis mode exactly as it does for Ollama. The other axes (prompt safety, jailbreak, RAG/context-injection, grounded hallucination, answer safety) work fully. See [Detection Axes](../gateway/detection-axes.md) for what each axis needs.

---

## Secrets

Provider credentials are **never stored in plaintext in the database**. An Application binding holds `binding.api_key_ref`, which is a reference of the form `secret://<app>/<name>` (for example `secret://app_invoices_eu/upstream`). The secrets provider resolves that reference to the actual key at request time, trying these sources in order:

1. **Environment override** — `GEODESIA_SECRET__<APP>__<NAME>`, where app and name are upper-cased with non-alphanumeric characters replaced by `_`. For the example above this is `GEODESIA_SECRET__APP_INVOICES_EU__UPSTREAM`.
2. **Local secrets file** — `var/secrets.json` (override the path with `GEODESIA_SECRETS_FILE`). Accepts either a flat `{"secret://…": "value"}` map or a nested `{app: {name: value}}` map. Intended for air-gapped deployments where the file is OS-protected.
3. **AWS Secrets Manager** — used when `GEODESIA_SECRET_PROVIDER=aws_secrets`; the secret id is `geodesia/<app>/<name>`. If the manager lookup fails, the resolver falls back to the local file.

The provider is selected by `GEODESIA_SECRET_PROVIDER` (`file`, the default, or `aws_secrets`). Resolved values are cached in-process. A literal (non-`secret://`) value passes through unchanged, which is convenient for local development.

!!! tip "Bedrock and Vertex need no key reference"
    Because Bedrock uses an IAM role and Vertex uses ADC, bindings for those providers leave `api_key_ref` empty — there is no key to store or resolve. Only the OpenAI-compatible / Azure path needs a `secret://` reference.

---

## Pluggable storage

A single environment variable, `GEODESIA_DB_URL`, decides where all G-1 Studio state lives, so the same artifact runs from a laptop SQLite file to a shared cloud database.

| `GEODESIA_DB_URL` | Backend | Notes |
|---|---|---|
| _(unset)_ | SQLite at the default path | Laptop / single node |
| `sqlite:////abs/path/glad.sqlite3` | SQLite (absolute) | **Four** slashes for an absolute path — e.g. on an EBS volume |
| `sqlite:///relative/glad.sqlite3` | SQLite (relative to CWD) | Three slashes |
| `postgresql+psycopg://user:pw@host/db` | Postgres | Multi-replica scale-out |

Both engines are exposed through one `StorageBackend` DAO surface whose `connect()` yields a connection with a uniform `execute(sql, params)` returning **dict rows**. The control-plane and cost stores (`AppStore`, `CostStore`) and the RAG `IngestionJobs` queue all run on this backend, using portable SQL only:

- Placeholders are written as `?` or `:name`; the Postgres backend translates them to `%s` / `%(name)s` automatically.
- Upserts use `INSERT … ON CONFLICT(keys) DO UPDATE SET …` (works on both engines) — never the SQLite-only `INSERT OR REPLACE`.
- Only `TEXT` / `INTEGER` / `REAL` types and `CREATE TABLE IF NOT EXISTS` are used.

The SQLite backend opens connections in WAL mode; the Postgres backend strips the SQLAlchemy-style `+psycopg` suffix and connects via psycopg with `dict_row`.

!!! note "psycopg is a lazy, cloud-only dependency"
    The Postgres backend imports `psycopg` only inside `connect()`. To use a Postgres URL you must `pip install 'psycopg[binary]'` **and** have a live Postgres server — the dialect translation is complete, but it needs a real server to validate against. Deployments on SQLite never import psycopg.

!!! warning "Split storage on multi-replica"
    The control-plane and cost stores run on the pluggable backend, so they can move to Postgres. The **legacy compliance DAO still uses raw SQLite directly.** On a multi-replica deployment this means the control-plane and cost data live on shared Postgres while the **compliance database remains a single-writer SQLite file on a shared filesystem** (e.g. an EBS volume mounted by one writer). Plan accordingly: the compliance store is not yet horizontally scalable. The `resolve_sqlite_path()` helper deliberately raises `NotImplementedError` on a Postgres URL for the SQLite-only legacy paths, so a misconfigured compliance engine fails loudly rather than silently dropping writes.

---

## Remote GLAD-BERT scoring pool

By default the GLAD-BERT detector runs **embedded** inside the gateway as an in-process singleton monitor. For horizontal scale you can offload detection to a separate GPU pool so that engine replicas can be CPU-only and the GPU scales independently.

| `GEODESIA_SCORING_MODE` | Behaviour | Extra config |
|---|---|---|
| `embedded` _(default)_ | In-process monitor — **byte-identical**, zero behaviour change | — |
| `remote` | Thin HTTP client against a Scoring Service pool | `GEODESIA_SCORING_URL` (required) |

In `remote` mode the gateway swaps the embedded monitor for a `RemoteScoringClient` that mirrors the monitor's `analyze` and `score_axis_batch` signatures, so the gateway's hot path is unchanged. The client POSTs to `/score` and `/score_batch` on the Scoring Service.

Start the Scoring Service on a GPU host:

```bash
python -m glad_minimal.scoring.server --host 0.0.0.0 --port 8810
```

Then point each engine replica at it:

```bash
export GEODESIA_SCORING_MODE=remote
export GEODESIA_SCORING_URL=http://scoring-host:8810
```

The service is a micro FastAPI wrapping the same `GladBertStreamingMonitor`; it honours the same checkpoint/device env vars as the gateway (`GW_V5_CKPT`, `GW_V5_MODEL`, `GW_DEVICE`, `GW_MAXLEN`, `GW_CADENCE`) and exposes a `/healthz` endpoint.

!!! note "Remote mode requires a URL"
    Setting `GEODESIA_SCORING_MODE=remote` without `GEODESIA_SCORING_URL` raises at startup — there is no silent fallback to embedded.

---

## RAG ingestion claim-queue

Per-Application RAG ingestion is multi-replica safe. Each uploaded document creates a queued row in the `ingestion_jobs` table (carrying `application_id`, `collection_id`, `doc_id`, status, progress and the worker that claimed it). A worker on **any** replica claims a job with an **atomic guarded UPDATE**:

```sql
UPDATE ingestion_jobs SET status='claimed', claimed_by=?, claimed_at=?, updated_at=?
WHERE job_id=? AND status='queued'
```

The claim succeeds only if exactly one row was updated (`rowcount == 1`); a replica that loses the race sees zero rows and moves on. This guarantees **exactly-once** document processing across replicas. On SQLite the single-writer WAL serializes the claim; on Postgres the same guarded UPDATE (or `SELECT … FOR UPDATE SKIP LOCKED`) is race-free. The queue runs on the pluggable `StorageBackend`, so it behaves identically on SQLite (single box) and Postgres (shared RDS across replicas). Progress is written back to the row for the SSE ingestion UI.

---

## PC → AWS deployment guide

A complete, step-by-step guide for taking a Geodesia G-1 Studio deployment from a developer PC to a running AWS deployment — an **EC2 g5** instance for the GLAD-BERT detector plus **AWS Bedrock** as the LLM upstream — lives in the repository as `DEPLOY_PC_TO_AWS_BEDROCK.md`. It walks through the three env vars that change between PC and cloud (DB location, secret provider, AWS region) and the single Application binding switch (`upstream_type: bedrock`).
