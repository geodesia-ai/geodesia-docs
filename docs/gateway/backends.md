# Upstream Backends

Geodesia G-1 is model-agnostic. It can sit in front of any LLM backend that speaks the OpenAI or Ollama API. The upstream backend is configured once (via the UI's **Settings → Service Connection** page or via the API) and the gateway automatically adapts its detection strategy to what the backend can provide.

---

## Supported Backend Types

| Type key | Description | Logprobs | Closed-book axis |
|---|---|---|---|
| `vllm` | vLLM serving engine | ✅ Full support | ✅ 5 axes |
| `sglang` | SGLang serving framework | ✅ Full support | ✅ 5 axes |
| `trtllm` / `tensorrt-llm` | NVIDIA TensorRT-LLM | ✅ Full support | ✅ 5 axes |
| `openai` | OpenAI API or any OpenAI-compatible endpoint | ✅ When `logprobs=true` | ✅ 5 axes |
| `ollama` | Ollama (local models) | ❌ Not natively | ⚠️ 4 axes (5 with sidecar) |
| `internal` | vLLM managed and lifecycle-controlled by the gateway itself | ✅ Full support | ✅ 5 axes |

!!! info "4 vs 5 axes"
    The **closed-book fabrication** axis requires per-token log-probability values from the upstream. Log-probabilities are a measure of how "certain" the model is about each word it generates. When they are unavailable, this axis is automatically disabled and the gateway operates with 4 axes — no configuration change is needed. The `/health` endpoint tells you which mode is active.

---

## Configuring the Backend

### Via the UI

Navigate to **Settings → Service Connection**. You will find:

1. **Service type** — a dropdown to select `vLLM`, `SGLang`, `TRT-LLM`, `OpenAI`, `Ollama`, or `Internal`.
2. **URL** — the base URL of the upstream (`http://host:port`).
3. **API key** — required for OpenAI and hosted services; leave empty for local deployments.
4. **Test connection** — sends a probe request to the upstream and reports reachability, latency, available models, logprob support, and a sample reply.
5. **Model** — populated automatically from the upstream's model list after a successful test.
6. **Save** — persists the configuration to `GW_CONFIG_FILE` so it survives restarts.

The **"Exposed API (OpenAI-compatible)"** section shows the gateway's own base URL (`http://host:port/v1`). Point your downstream application here.

### Via the API

```bash
# Update gateway configuration
curl -s -X POST http://localhost:8800/v1/glad/gateway/config \
  -H "Content-Type: application/json" \
  -d '{
    "upstream_type": "vllm",
    "upstream_base_url": "http://localhost:8000",
    "upstream_model": "my-model",
    "upstream_api_key": ""
  }'
```

Any field in `GatewayConfig` can be updated. Changes take effect immediately on the next request.

---

## Backend-Specific Notes

### vLLM

vLLM is the recommended backend for production deployments on NVIDIA GPUs. It returns per-token log-probabilities natively, enabling all 5 detection axes.

```bash
# Start vLLM (example)
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3-8B-Instruct \
  --port 8000

# Configure gateway
curl -X POST http://localhost:8800/v1/glad/gateway/config \
  -d '{"upstream_type":"vllm","upstream_base_url":"http://localhost:8000","upstream_model":"meta-llama/Llama-3-8B-Instruct"}'
```

### SGLang

SGLang is fully OpenAI-compatible and supports log-probabilities. Use type `sglang`:

```bash
curl -X POST http://localhost:8800/v1/glad/gateway/config \
  -d '{"upstream_type":"sglang","upstream_base_url":"http://localhost:30000","upstream_model":"meta-llama/Llama-3-8B-Instruct"}'
```

### TensorRT-LLM

TensorRT-LLM deployments typically front-end with an OpenAI-compatible server. Use type `trtllm`:

```bash
curl -X POST http://localhost:8800/v1/glad/gateway/config \
  -d '{"upstream_type":"trtllm","upstream_base_url":"http://localhost:8000","upstream_model":"llama-3-8b"}'
```

### OpenAI API

To use the actual OpenAI API (or any hosted OpenAI-compatible service such as Azure OpenAI, Together AI, Groq, Mistral AI):

```bash
curl -X POST http://localhost:8800/v1/glad/gateway/config \
  -d '{
    "upstream_type": "openai",
    "upstream_base_url": "https://api.openai.com",
    "upstream_api_key": "sk-...",
    "upstream_model": "gpt-4o"
  }'
```

!!! warning "Log-probability access"
    Log-probabilities are available from the OpenAI API when you set `logprobs: true` in the request. The gateway does this automatically. Some models or pricing tiers may not support them — the gateway detects this on the first request and falls back to 4-axis mode.

## Cloud upstreams (Bedrock / Vertex / Azure)

Beyond the local OpenAI-compatible servers above, an Application can bind directly to a managed cloud LLM:

| Upstream type | Provider | Auth |
|---|---|---|
| `bedrock` | AWS Bedrock | IAM role / credentials |
| `vertex` | Google Vertex AI | Application Default Credentials (ADC) |
| `azure-openai` | Azure OpenAI | API key + endpoint |

The gateway ships native adapters for each, so no OpenAI-compatible shim is required. One caveat: Bedrock and Vertex do not return per-token log-probabilities, so the `halluc_closedbook` axis is automatically disabled for Applications bound to those upstreams (the remaining axes run normally).

See **[Cloud Upstreams](../studio/cloud-upstreams.md)** for the full per-provider configuration — adapter setup, IAM/ADC authentication, region binding, and the logprobs caveat in detail.

### Ollama

Ollama is a popular tool for running open-source models locally. **Ollama ≥ 0.12 exposes per-token log-probabilities** on its OpenAI-compatible `/v1` endpoint, which the gateway probes automatically — so the closed-book axis (full 5-axis mode) works out of the box. On **older Ollama (< 0.12)** the gateway runs in 4-axis mode, and you can recover the closed-book axis with a log-prob sidecar (see [Framework Compatibility → Ollama](compatibility.md#ollama)).

```bash
curl -X POST http://localhost:8800/v1/glad/gateway/config \
  -d '{
    "upstream_type": "ollama",
    "upstream_base_url": "http://localhost:11434",
    "upstream_model": "llama3.2"
  }'
```

#### Enabling the 5th axis with a logprob sidecar

To enable closed-book fabrication detection with Ollama, you need a **logprob sidecar**: a second instance of `llama.cpp` or another OpenAI-compatible server serving the same model with log-probability support. The gateway deterministically re-derives the answer through the sidecar to recover the needed signals.

```bash
# Configure sidecar (same model, different server, WITH logprobs)
curl -X POST http://localhost:8800/v1/glad/gateway/config \
  -d '{
    "upstream_type": "ollama",
    "upstream_base_url": "http://localhost:11434",
    "upstream_model": "llama3.2",
    "ollama_logprob_sidecar_url": "http://localhost:8080",
    "ollama_logprob_sidecar_model": "llama3.2"
  }'
```

| Field | Description |
|---|---|
| `ollama_logprob_sidecar_url` | Base URL of the OpenAI-compatible server that serves the same model with log-probabilities (e.g., llama.cpp server). |
| `ollama_logprob_sidecar_model` | Model name as known to the sidecar server. Defaults to `upstream_model` if empty. |

### Internal (self-managed vLLM)

When `upstream_type` is set to `internal`, the gateway launches and manages its own vLLM subprocess. It starts vLLM when selected and **frees GPU memory** when you switch to an external backend. This is useful for single-GPU deployments where you want the gateway to own the entire lifecycle.

```bash
curl -X POST http://localhost:8800/v1/glad/gateway/config \
  -d '{
    "upstream_type": "internal",
    "internal_vllm_cmd": "python -m vllm.entrypoints.openai.api_server --model my-model --port 8000",
    "internal_vllm_url": "http://localhost:8000",
    "upstream_model": "my-model"
  }'
```

| Field | Description |
|---|---|
| `internal_vllm_cmd` | Full shell command to start vLLM. The gateway runs it as a subprocess. |
| `internal_vllm_url` | Base URL where the internal vLLM is accessible after startup. |

---

## Testing a Connection

The `/upstream/test` endpoint performs a full connection probe and returns a diagnostic summary. It is called automatically when you click **Test connection** in the UI.

```bash
curl -s -X POST http://localhost:8800/upstream/test \
  -H "Content-Type: application/json" \
  -d '{
    "url": "http://localhost:8000",
    "type": "vllm",
    "api_key": "",
    "model": "my-model"
  }'
```

**Response:**

```json
{
  "reachable": true,
  "models": ["my-model"],
  "has_logprobs": true,
  "sample_reply": "OK",
  "latency_ms": 117,
  "error": null,
  "type": "vllm",
  "model": "my-model",
  "closed_book_available": true,
  "axes": 5
}
```

| Field | Description |
|---|---|
| `reachable` | `true` if the server responded with an HTTP status below 500 |
| `models` | List of model IDs available on the upstream |
| `has_logprobs` | `true` if the upstream returns per-token log-probabilities |
| `sample_reply` | The model's reply to "Reply with the single word OK" (up to 120 characters) |
| `latency_ms` | Round-trip latency in milliseconds |
| `error` | Error message if the connection failed, otherwise `null` |
| `closed_book_available` | Same as `has_logprobs` — whether the 5th detection axis is available |
| `axes` | `5` with logprobs, `4` without |

---

## Closed-Book Fabrication — no per-model setup

The closed-book fabrication detector is **cross-model**: it ships ready to use and runs on any upstream out of the box, with no per-model calibration or training step. When you switch to a different base model, nothing to do — point the gateway at the new upstream and the closed-book axis works immediately (provided the upstream returns per-token log-probabilities; otherwise the gateway simply runs in 4-axis mode).
