# Product API — Introduction

The **Product Backend** exposes a set of REST endpoints for AI evaluation, compliance, audit, and regulatory management. It runs separately from the gateway (typically on port `8199`) and does not need to be in the real-time request path.

The product backend is designed to be deployed **without a GPU** for compliance-only use cases. When `MODEL_HOST_PATH` is not set, the evaluate endpoint is unavailable, but all compliance, audit, FRIA, oversight, dashboard, and reports endpoints work normally.

## Base URL

```
http://localhost:8199
```

In production behind nginx, all `/v1/*` paths are typically proxied to the product backend.

## Route Groups

| Prefix | Description |
|---|---|
| `/glad/evaluate` | Direct AI evaluation (generate + score in one call) |
| `/glad/export_audit` | Export audit bundle for regulatory submission |
| `/glad/finetune` | Fine-tuning job management |
| `/v1/` | OpenAI-compatible chat (with full evaluate pipeline) |
| `/v1/glad/` | Compliance, audit, FRIA, oversight, kill switch, models, etc. |
| `/v1/agent-flow/` | Multi-agent pipeline debugger |
| `/health` | Health check |

## OpenAI-Compatible Endpoint

The product backend also exposes an OpenAI-compatible `/v1/chat/completions` endpoint. Unlike the gateway, this endpoint runs the full evaluate pipeline inline — it generates a response with the loaded model and scores it in the same HTTP call. It accepts all the same extensions as the gateway endpoint, plus additional explain and credit-tier parameters.

See [Evaluate Endpoint](evaluate.md) for a full reference.
