# Gateway Configuration

Everything the proxy does at runtime lives in one `GatewayConfig` object: which upstream to call, what
to block, how often to re-read a stream, whether to redact PII. You can set any field three ways, in
decreasing priority: the API (applies live, persisted), a CLI flag, or an environment variable.

---

## Reading and Writing the Config at Runtime

### Get the current configuration

```bash
curl -s http://localhost:8800/v1/glad/gateway/config | python3 -m json.tool
```

**Response:** All `GatewayConfig` fields with `upstream_api_key` masked as `***`.

### Update any field

```bash
# Enable blocking mode for inputs
curl -s -X POST http://localhost:8800/v1/glad/gateway/config \
  -H "Content-Type: application/json" \
  -d '{"block_input": true}'

# Change thresholds
curl -s -X POST http://localhost:8800/v1/glad/gateway/config \
  -H "Content-Type: application/json" \
  -d '{"thresholds": {"prompt_safety": 0.80, "jailbreak": 0.65}}'

# Switch upstream model
curl -s -X POST http://localhost:8800/v1/glad/gateway/config \
  -H "Content-Type: application/json" \
  -d '{"upstream_model": "gemma2:9b"}'
```

Every `POST` triggers:

1. Immediate in-memory update of the specified fields
2. Detection model reload on the next request (since thresholds and checkpoint path may have changed)
3. Upstream logprob capability re-probe (since the upstream may have changed)
4. Config written to `GW_CONFIG_FILE` for persistence

### The config file

The config is written to `GW_CONFIG_FILE` (default `runs/gateway_config.json`) as a plain JSON file. You can edit it directly when the gateway is stopped, then restart:

```json
{
  "upstream_type": "vllm",
  "upstream_base_url": "http://localhost:8000",
  "upstream_model": "my-model",
  "upstream_api_key": "",
  "block_input": true,
  "block_output": true,
  "thresholds": {
    "halluc_context": 0.35,
    "halluc_closedbook": 0.50,
    "prompt_safety": 0.90,
    "answer_safety": 0.57,
    "jailbreak": 0.57
  }
}
```

Only the fields listed in the gateway source as "persistable" are saved: upstream settings, inbound host/port, CI injection, blocking flags, thresholds, numeric solver, and sidecar config. Internal state (loaded model, probed capabilities) is not persisted.

---

## Complete Configuration Reference

### Inbound (your application → gateway)

| Field | Env var | Default | Description |
|---|---|---|---|
| `inbound_host` | — | `0.0.0.0` | IP address the gateway binds to. Use `127.0.0.1` to restrict to localhost. |
| `inbound_port` | — | `8800` | TCP port the gateway listens on. |

### Upstream (gateway → your LLM)

| Field | Env var | Default | Description |
|---|---|---|---|
| `upstream_type` | — | `ollama` | Type of the upstream backend. One of `vllm`, `sglang`, `trtllm`, `openai`, `ollama`, `internal`. See [Backends](backends.md). |
| `upstream_base_url` | — | `http://localhost:11434` | Base URL of the upstream LLM (no trailing slash). For OpenAI use `https://api.openai.com`. |
| `upstream_api_key` | — | `""` | Bearer token for the upstream. Required for OpenAI and hosted services. Stored in the config file; masked (`***`) in `GET /v1/glad/gateway/config` responses. |
| `upstream_model` | — | `llama3.1:8b` | Model name to request from the upstream. Must match an ID in the upstream's `/v1/models` list. |

### Ollama logprob sidecar (optional, enables 5th axis with Ollama)

| Field | Env var | Default | Description |
|---|---|---|---|
| `ollama_logprob_sidecar_url` | `OLLAMA_LOGPROB_SIDECAR_URL` | `""` | Base URL of an OpenAI-compatible server that runs the **same model** as the Ollama upstream but with log-probability support (e.g., `llama.cpp --server`). When set, the gateway teacher-forces the answer through this sidecar to recover log-probabilities for the closed-book fabrication axis. |
| `ollama_logprob_sidecar_model` | `OLLAMA_LOGPROB_SIDECAR_MODEL` | `""` | Model name as known to the sidecar. Defaults to `upstream_model` when empty. |

### Internal vLLM lifecycle (only when `upstream_type = "internal"`)

| Field | Env var | Default | Description |
|---|---|---|---|
| `internal_vllm_cmd` | `GW_VLLM_CMD` | `""` | Full shell command to launch the self-managed vLLM subprocess. Example: `python -m vllm.entrypoints.openai.api_server --model my-model --port 8000`. |
| `internal_vllm_url` | `GW_VLLM_URL` | `http://localhost:8000` | URL where the internal vLLM listens once started. |

### Validation behaviour

| Field | Env var | Default | Description |
|---|---|---|---|
| `validate_input` | — | `true` | Whether to score the prompt before forwarding it. Disable only for debugging. |
| `validate_output` | — | `true` | Whether to score the model's answer before returning it. Disable only for debugging. |
| `block_input` | `GW_BLOCK_INPUT` | `false` | If `true`, prompts that exceed the prompt-safety or jailbreak threshold are **refused** before the upstream sees them. If `false`, unsafe prompts are annotated but still forwarded (passthrough). |
| `block_output` | — | `true` | If `true`, answers that exceed a threshold are **withheld** and replaced with a block notice. If `false`, answers are annotated but returned. |
| `cadence_tokens` | — | `32` | How often (in tokens) to score the in-progress generation during streaming. Lower values increase responsiveness at the cost of more scoring calls. |

### Constitutional Intelligence prompt

| Field | Env var | Default | Description |
|---|---|---|---|
| `inject_system_prompt` | `GW_INJECT_SYSTEM` | `true` | When `true`, prepends the Constitutional Intelligence (CI) system prompt to every request before forwarding to the upstream LLM. The CI prompt enforces Geodesia's built-in safety policy at the instruction level. Disable only if your own system prompt already covers safety, or for benchmarking. |
| `system_prompt` | `GW_CI_PROMPT` | (built-in) | The CI system prompt text. Override with `GW_CI_PROMPT` (inline text) or `GW_CI_PROMPT_FILE` (path to a Markdown file). The built-in prompt is the **G-1 Compact Constitutional Intelligence** spec. |

### Detection thresholds

| Field | Type | Default | Description |
|---|---|---|---|
| `thresholds` | `dict[str, float]` | See below | Per-axis detection thresholds in **probability space** (0.0–1.0). A score above the threshold for an axis causes that axis to flag. |

**Serving-calibrated defaults for the nine-axis head:**

| Axis key | Default | Enforcement |
|---|---|---|
| `prompt_safety` | `0.9215` | `block` |
| `jailbreak` | `0.9997` | `block` |
| `rag_jailbreak` | `0.2501` | `block` |
| `halluc_context` | `0.6475` | `annotate` |
| `halluc_closedbook` | `0.58` | `annotate` |
| `answer_safety` | `0.7295` | `annotate` |
| `profanity` | `0.90` | `annotate` |
| `out_of_scope` | `0.90` | `annotate` |
| `prompt_complexity` | `0.50` | `off` (routing boundary, not a safety threshold) |

!!! warning "These numbers do not transfer across checkpoints"
    They are the operating point of **one** calibration on **one** checkpoint — `prompt_safety` and
    `jailbreak` share a 2 % false-positive budget measured on a multilingual benign pool; the rest sit on
    their dev split. A new checkpoint means re-running the calibration. An Application created before a
    change keeps the thresholds stored in its own policy.

!!! note "Threshold direction"
    A **higher** threshold is **more permissive** (fewer flags); a **lower** one is **stricter**. Where each
    threshold is resolved from, and how to move one without guessing:
    [Threshold Tuning Guide](../reference/thresholds.md).

### Detection model

| Field | Env var | Default | Description |
|---|---|---|---|
| `v5_ckpt` | `GW_V5_CKPT` | bundled | Path to the Geodesia detection engine checkpoint. The active checkpoint determines detection quality; the bundled default is used unless you are issued a custom one. |
| `v5_model_id` | `GW_V5_MODEL` | bundled | Identifier of the backbone encoder the checkpoint was built for. Set automatically from the checkpoint; normally never changed. |
| `v5_maxlen` | `GW_MAXLEN` | `2048` | Maximum token length for the detection model. The faithfulness axis (`halluc_context`) benefits from the full 2048 when long documents are in the context. Reduce to `512` to decrease latency on short prompts at the cost of recall on long contexts. |

### Numeric solver (optional advanced feature)

The text-based detection model cannot do arithmetic. If your use case involves LLMs answering questions from numerical tables or financial documents, the opt-in numeric solver re-derives the answer mathematically and blends the result into the `halluc_context` score.

| Field | Env var | Default | Description |
|---|---|---|---|
| `numeric_solver` | `GW_NUMERIC_SOLVER` | `none` | Numeric verification mode. Options: `none` (disabled), `pot` (lightweight program-of-thought), `strong` (Qwen2.5-Coder-7B judge, AUROC 0.76 on FinQA — loads a 7B model), `api` (delegates to an external API). |
| `numeric_solver_model` | `GW_NUMERIC_MODEL` | `Qwen/Qwen2.5-Coder-7B-Instruct` | Model used when `numeric_solver` is `strong`. Any HuggingFace model ID that can perform code reasoning. |
| `numeric_solver_quant` | `GW_NUMERIC_QUANT` | `""` | Quantization for the numeric solver model. Set to `4bit` to reduce VRAM usage (e.g., to 5–6 GB). Leave empty for bf16 full precision. |
