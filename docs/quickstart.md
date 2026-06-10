# Quick Start

This guide gets you from zero to a running Geodesia G-1 gateway in under 10 minutes, connected to an existing LLM backend.

## Prerequisites

- Docker and Docker Compose, **or** Python 3.11+ with a virtual environment
- An accessible LLM endpoint: vLLM, Ollama, OpenAI, or any OpenAI-compatible server

---

## Option A — Docker Compose (recommended)

The `docker-compose.gateway.yml` file in the repository defines two services:

| Service | Purpose |
|---|---|
| `generator` | Official `vllm/vllm-openai` container serving your model |
| `glad-gateway` | Geodesia G-1 gateway — the validation layer |

### 1. Fixed checkpoint (pre-calibrated)

Use this profile if you have a pre-calibrated Geodesia checkpoint (`.pt` file):

```bash
# In the project root
docker compose -f deploy/docker-compose.gateway.yml --profile fixed up
```

The gateway starts on **port 8800** and proxies requests to the `generator` container on port 8000.

### 2. Customer profile (first-boot calibration)

Use this when deploying to a new customer or a new model. The gateway automatically calibrates the closed-book detector against the upstream model on first boot:

```bash
docker compose -f deploy/docker-compose.gateway.yml --profile customer up
```

During first boot you will see calibration progress in the logs. Once complete, the gateway writes a calibrated checkpoint and reloads without restarting.

!!! note "Closed-book calibration"
    Calibration is only required once per model. It adapts the closed-book fabrication detector to the specific vocabulary distribution of your LLM. Subsequent restarts skip it and use the cached checkpoint.

---

## Option B — Direct Python

```bash
# 1. Create and activate a virtual environment
python3.11 -m venv .venv && source .venv/bin/activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Start the gateway (Ollama example)
GW_V5_CKPT=runs/glad_bert/glad_bert_v8_unified.pt \
GW_BLOCK_INPUT=1 \
  python -m glad_minimal.gateway.geodesia_gateway \
    --host 0.0.0.0 --port 8800 \
    --upstream-type ollama \
    --upstream-url http://localhost:11434 \
    --upstream-model llama3.2

# vLLM example
GW_V5_CKPT=runs/glad_bert/glad_bert_v8_unified.pt \
GW_BLOCK_INPUT=1 \
  python -m glad_minimal.gateway.geodesia_gateway \
    --host 0.0.0.0 --port 8800 \
    --upstream-type vllm \
    --upstream-url http://localhost:8000 \
    --upstream-model my-model
```

---

## Sending Your First Request

The gateway exposes an OpenAI-compatible `/v1/chat/completions` endpoint. Any client that works with OpenAI will work here unchanged — just change the `base_url`:

=== "Python (openai SDK)"

    ```python
    from openai import OpenAI

    client = OpenAI(
        base_url="http://localhost:8800/v1",
        api_key="not-required",   # leave any non-empty string
    )

    response = client.chat.completions.create(
        model="my-model",
        messages=[{"role": "user", "content": "What is the capital of France?"}],
    )

    print(response.choices[0].message.content)
    # "The capital of France is Paris."

    # Access Geodesia detection results
    geodesia = response.model_extra.get("geodesia") or {}
    print("Brake:", geodesia.get("brake"))
    print("Axes:", geodesia.get("axis_energy"))
    ```

=== "curl"

    ```bash
    curl -s http://localhost:8800/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{
        "model": "my-model",
        "stream": false,
        "messages": [
          {"role": "user", "content": "What is the capital of France?"}
        ]
      }' | python3 -m json.tool
    ```

=== "JavaScript (fetch)"

    ```javascript
    const response = await fetch("http://localhost:8800/v1/chat/completions", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: "my-model",
        stream: false,
        messages: [{ role: "user", content: "What is the capital of France?" }],
      }),
    });

    const data = await response.json();
    console.log(data.choices[0].message.content);
    console.log("Detection:", data.geodesia);
    ```

---

## Check the Gateway Health

```bash
curl http://localhost:8800/health
```

```json
{
  "ok": true,
  "upstream_type": "vllm",
  "upstream": "http://localhost:8000",
  "internal_vllm": "stopped",
  "logprobs": true,
  "axes": 5
}
```

| Field | Meaning |
|---|---|
| `logprobs` | Whether the upstream returns per-token log-probabilities. Required for the closed-book fabrication axis. |
| `axes` | `5` when logprobs are available (all five detection axes active); `4` when logprobs are unavailable (closed-book axis disabled automatically). |

---

## Start the Product Backend (compliance features)

The compliance dashboard, FRIA, audit chain, and other regulatory features are served by a separate backend process. You can run it without a GPU model — the compliance features do not need one.

```bash
MODEL_HOST_PATH=""  \
GLAD_DEVICE=cpu     \
  python -m uvicorn main:app --host 127.0.0.1 --port 8199
```

The product backend is then available at `http://localhost:8199/v1/glad/`.

---

## Next Steps

| Goal | Guide |
|---|---|
| Connect a different LLM backend | [Upstream Backends](gateway/backends.md) |
| Configure detection thresholds | [Detection Thresholds](reference/thresholds.md) |
| Enable blocking mode | [Enforcement Modes](gateway/enforcement-modes.md) |
| Upload documents for RAG | [Knowledge Base](rag/index.md) |
| Set up EU AI Act compliance | [FRIA](compliance/fria.md) |
| Understand the full config.yaml | [Configuration Reference](configuration/index.md) |
