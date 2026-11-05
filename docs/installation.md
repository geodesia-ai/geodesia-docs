# Installation & Configuration

One guide, all the commands: from `install.sh` to a working, configured stack, to calling an LLM with an
API key and reading the safety verdicts.

??? abstract "Contents"
    1. [What you're installing](#1-what-youre-installing)
    2. [Prerequisites](#2-prerequisites)
    3. [Install with `install.sh`](#3-install-with-installsh)
    4. [What the installer created](#4-what-the-installer-created)
    5. [Configure the upstream LLM](#5-configure-the-upstream-llm)
    6. [Licensing & tiers](#6-licensing-tiers)
    7. [First start, sealed images and GPU memory](#7-first-start-sealed-images-and-gpu-memory)
    8. [Runtime gateway config](#8-runtime-gateway-config)
    9. [Create an Application + API key](#9-create-an-application-api-key)
    10. [Call the LLM with the key](#10-call-the-llm-with-the-key)
    11. [Interpret the response](#11-interpret-the-response)
    12. [Operations (logs, update, stop, health, metrics)](#12-operations)
    13. [Troubleshooting](#13-troubleshooting)
    14. [Full environment-variable reference](#14-full-environment-variable-reference)

---

## 1. What you're installing

Two containers, one command (`./install.sh both`):

| Container | Port | Role |
|---|---|---|
| **g1-proxy** (engine / gateway) | `8800` | OpenAI-compatible endpoint. Scores every prompt & answer on nine axes at [thinking levels](g1-proxy/thinking-levels.md) 0–3, can block, proxies to your LLM. Also serves [G1 Eval](g1-proxy/eval.md) (`/g1/eval`). |
| **g1-studio** (control plane + UI) | `8080` | Web UI + management API: Applications, API keys, policy, metrics, license. |

Or one container (`./install.sh light`):

| Container | Port | Role |
|---|---|---|
| **g1-proxy-light** | `8800` | Engine **and** its own UI on the same port. Everything g1-proxy serves — levels 0–3, `halluc_context`, closed-book, the reference check, G1 Eval, MuPAX, MCP — except the 9B judge, speech recognition and RAG. See [G1-Proxy Light](g1-proxy/light.md). |

```
  your client ──(API key)──▶  g1-proxy :8800  ──▶  your upstream LLM (OpenAI-compatible)
                                  ▲
                                  │ shares one DB (volume)
                              g1-studio :8080  ◀── you manage apps/keys here
```

Images are **self-contained** (detectors, readers and, for g1-proxy, RAG all baked in) and **sealed**: the model
weights are encrypted at rest inside the image and decrypted in memory at start-up. You only need to provide an
**upstream LLM** that speaks the OpenAI API (or Ollama's); token `logprobs` are recommended.

---

## 2. Prerequisites

```bash
# Docker + compose v2
docker --version
docker compose version

# GPU path (default): NVIDIA GPU + container toolkit
nvidia-smi
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi   # must print the GPU

# CPU path: nothing extra needed (use --cpu at install) — 1.0 releases are published as GPU images only

# An upstream LLM reachable and returning logprobs, e.g. ollama:
curl -s http://localhost:11434/api/tags        # ollama
# or an OpenAI-compatible server (vLLM/others):
curl -s http://localhost:8002/v1/models
```

The closed-book hallucination axis (`halluc_closedbook`) reads the answer's `logprobs`/`top_logprobs`; ollama
and vLLM both return them. Without them the other axes work and `halluc_closedbook` reports
`available: false` on chats (G1 Eval can still measure it from a second answer, see [G1 Eval](g1-proxy/eval.md)).

GPU memory (approximate peaks): **g1-proxy-light ~16 GB**, **g1-proxy ~24 GB** (32 GB recommended when the
upstream LLM shares the card). See [§7](#7-first-start-sealed-images-and-gpu-memory).

---

## 3. Install with `install.sh`

`install.sh` is the **single, self-contained installer** — one file, no cloud login (it embeds read-only
registry credentials so it can pull the images for you).

### Step 3.1 — Get the installer

`install.sh` is provided to you by Geodesia as a single file. Save it to an empty directory and make it
executable:

```bash
mkdir -p ~/geodesia && cd ~/geodesia
# put install.sh here (the file Geodesia gave you), then:
chmod +x install.sh
./install.sh --help          # shows every option
```

`--help` prints the full usage:

```text
USAGE
  ./install.sh [both|g1-proxy|g1-studio] [--cpu|--gpu] [LICENSE]
  ./install.sh light [--cpu|--gpu] [LICENSE]
  ./install.sh update
TARGETS   both (default) | g1-proxy | g1-studio | light | update
DEVICE    --gpu (default, needs NVIDIA + nvidia-container-toolkit) | --cpu (no GPU)
LICENSE   a signed GEO1.<base64> key, a path to license.json, or raw JSON (optional → FREE tier)
          or GLAD_LICENSE_TOKEN=glc_… (the installer fetches the signed key bound to it)
```

### Step 3.2 — Run it

```bash
# ── the common cases ────────────────────────────────────────────────
./install.sh both                      # engine + UI, GPU, FREE tier          (default)
./install.sh both --cpu                # engine + UI, CPU-only (only where a -cpu image is published)
./install.sh both 'GEO1.eyJwYXls...'   # with a signed license (paid tier)
./install.sh both --cpu 'GEO1.eyJ...'  # CPU-only + signed license
GLAD_LICENSE_TOKEN=glc_XXXXXXXXXXXXXXXXXXXXXXXX ./install.sh both   # license token → signed key fetched once

# ── only one component ──────────────────────────────────────────────
./install.sh g1-proxy                  # only the engine
./install.sh g1-studio                 # only the UI (point it at an external engine via GATEWAY_URL)

# ── single container: engine + UI on one port ───────────────────────
UPSTREAM_TYPE=ollama UPSTREAM_URL=http://localhost:11434 UPSTREAM_MODEL=ministral-3:3b \
GLAD_LICENSE_TOKEN=glc_XXXXXXXXXXXXXXXXXXXXXXXX ./install.sh light

# ── quick tier caps without signing a license ───────────────────────
CHATS_PER_DAY=2000 MAX_MODELS=5 ./install.sh both
FREE_UNLIMITED=1 ./install.sh both     # self-host, remove all caps

# ── point at your own upstream LLM at install time ──────────────────
UPSTREAM_TYPE=openai UPSTREAM_URL=http://localhost:8002 UPSTREAM_MODEL=ministral3 ./install.sh both
UPSTREAM_TYPE=ollama UPSTREAM_URL=http://localhost:11434 UPSTREAM_MODEL=llama3.1:8b ./install.sh both

# ── help ────────────────────────────────────────────────────────────
./install.sh --help
```

Common install-time env vars (all optional):

```bash
REG=...                # registry (default: europe-west1-docker.pkg.dev/glad-manifold-v2/glad)
TAG=latest             # image tag (moving 'latest' = newest protected build)
HTTP_PORT=8080         # UI port
GATEWAY_PORT=8800      # engine port
UPSTREAM_TYPE=ollama   # ollama | openai | vllm | sglang | trtllm | internal | azure-openai | bedrock | vertex
UPSTREAM_URL=http://localhost:11434
UPSTREAM_MODEL=        # no default: set it here or choose it later in the UI (Settings → Gateway)
GATEWAY_URL=http://localhost:8800   # where g1-studio reaches the engine
WORKDIR=$PWD/geodesia-g1            # where compose + config live
GLAD_LICENSE_TOKEN=glc_…            # license token (alternative to LICENSE=)
```

There is **no default upstream model**. With `UPSTREAM_MODEL` empty the engine starts without one and chat and
closed-book calibration wait until you choose it in the UI. On Ollama the installer lists the installed models,
and an explicit `UPSTREAM_MODEL` that Ollama does not have stops the install with that list (instead of an engine
that answers `404 model not found` on every call).

Example, everything at once:

```bash
UPSTREAM_TYPE=openai UPSTREAM_URL=http://localhost:8002 UPSTREAM_MODEL=ministral3 \
HTTP_PORT=8080 GATEWAY_PORT=8800 \
./install.sh both 'GEO1.eyJwYXlsb2FkIjp7...'
```

When it finishes it prints the URLs and a health hint. The first start takes a few minutes (see
[§7](#7-first-start-sealed-images-and-gpu-memory)). Verify:

```bash
curl -s http://localhost:8800/health          # {"ok": true, "tiers": {...}, ...}
curl -s http://localhost:8080/v1/glad/apps     # {"apps":[{"app_id":"default",...}]}   (both / g1-studio)
# open the UI:  http://<host>:8080   — with `light`: http://<host>:8800
```

---

## 4. What the installer created

Everything lives in `WORKDIR` (default `./geodesia-g1`):

```bash
cd geodesia-g1
ls -la                     # .env, docker-compose.yml, (license.json if you passed one)
cat .env                   # your resolved config (REG/TAG/DEVICE/UPSTREAM_*/...)
cat docker-compose.yml     # the generated stack (g1-proxy + g1-studio, or the single g1-proxy-light service)
docker compose ps          # running containers
docker volume ls | grep g1-data   # the shared DB/state volume
```

To change config later: edit `.env` (or `docker-compose.yml`) and re-apply:

```bash
docker compose up -d --force-recreate
```

`.env` holds the install-time values (`REG`, `TAG`, `DEVICE`, `UPSTREAM_*`, `GLAD_LICENSE_TOKEN`, …). Engine
variables that the installer does not write (for example `GW_EVAL_EXPORT_TOKEN`) go under `environment:` of the
engine service (`g1-proxy`, or `g1-proxy-light`) in `docker-compose.yml`:

```yaml
services:
  g1-proxy-light:
    environment:
      GW_PORT: "${GATEWAY_PORT}"
      # ...
      GW_EVAL_EXPORT_TOKEN: "<a long random string>"   # enables /g1/eval/export and /g1/eval/stats
```

A container reads its environment only when it is created: always follow an edit with
`docker compose up -d --force-recreate` (a plain `restart` keeps the old environment).

---

## 5. Configure the upstream LLM

The engine talks to your LLM via `GEODESIA_UPSTREAM_*`. Set them at install (env above) or edit `.env` and
recreate. **The container ENV is the source of truth** — a `POST .../g1-proxy/config` change is in-memory only
and is overwritten by the env on the next restart.

```bash
# ollama
UPSTREAM_TYPE=ollama  UPSTREAM_URL=http://localhost:11434  UPSTREAM_MODEL=llama3.1:8b

# vLLM / OpenAI-compatible (returns top_logprobs — required)
UPSTREAM_TYPE=openai  UPSTREAM_URL=http://localhost:8002   UPSTREAM_MODEL=ministral3
```

Test the upstream is reachable from the engine and returns logprobs:

```bash
curl -s http://localhost:8800/health | python3 -m json.tool
# look for  "upstream": "...",  "logprobs": true,  "logprobs_reason": ...,  "axes_gated": [],
#           "calibration": {"model": "...", "status": "done"}
```

If `logprobs` is `false`, `logprobs_reason` says why (`model_not_found`, `auth_error`, no logprobs returned, …)
and `halluc_closedbook` is listed in `axes_gated`: it is not measured on chats until the upstream returns
`top_logprobs`. The UI's **Test capabilities** button runs a fresh probe. Closed-book calibration (SLEDGE) runs
once per upstream model; see [Closed-Book Calibration](g1-proxy/closed-book-calibration.md).

---

## 6. Licensing & tiers

- **No license → FREE tier**: 20 chats/day, 1 model, 1 application.
- **Quick caps** (no signing), set at install: `CHATS_PER_DAY`, `MAX_MODELS`, `MAX_APPS` (`0` or `unlimited` =
  no cap), or `FREE_UNLIMITED=1`.
- **Signed license** (paid tier): pass the `GEO1.<base64>` key as the 2nd arg (or `LICENSE=`). Verified
  **offline** (Ed25519) — air-gap friendly, no license server.
- **License token** (`glc_…`): set `GLAD_LICENSE_TOKEN=glc_…` (or `GLC=`) and no `LICENSE`. The installer asks
  `https://license.geodesia.ai` **once, at install time**, for the signed key bound to the token, then proceeds as
  with a signed license; the running engine still verifies offline. A token the server rejects (expired, revoked,
  unknown) stops the install instead of silently installing the free tier. Without network access, pass the
  `GEO1.` key instead.

```bash
# lift the free caps (self-host)
FREE_UNLIMITED=1 ./install.sh both

# raise a single cap (e.g. allow up to 10 applications)
MAX_APPS=unlimited ./install.sh both

# apply a signed license
./install.sh both 'GEO1.eyJwYXlsb2FkIjp7...'
# or from a file
LICENSE=/path/license.json ./install.sh both
# or from a license token
GLAD_LICENSE_TOKEN=glc_XXXXXXXXXXXXXXXXXXXXXXXX ./install.sh light
```

The license token is a credential: keep it out of shell history and screenshots, and never use it as the
`Authorization` bearer of an API call (API calls use Application keys `g1k_…`, the gateway token, or — for the
Eval export — `GW_EVAL_EXPORT_TOKEN`).

Check the active tier:

```bash
curl -s http://localhost:8800/v1/glad/gateway/entitlements | python3 -m json.tool
```

> If you hit `application limit reached (free tier: 1 application)`, raise `MAX_APPS`/use a license, or for a
> dev box set `GLAD_FREE_MAX_APPLICATIONS=unlimited` on the g1-studio service and recreate.

---

## 7. First start, sealed images and GPU memory

- **Start-up takes a few minutes.** At the first start the engine decrypts the sealed weights, loads Geodesia-G,
  -H and -A and compiles the detector graph (4–5 minutes of warm-up, so that the first request is already fast).
  Watch the tiers:

    ```bash
    curl -s http://localhost:8800/health | python3 -c "import sys,json; print(json.load(sys.stdin)['tiers'])"
    # {'glad_g': {'state': 'ready'}, 'glad_h': {'state': 'ready'}, 'glad_a': {'state': 'ready'}}
    ```

    A request that asks for a thinking level whose tier is still `loading` or `starting` is served with the
    tiers that are ready, and says so in `geodesia.thinking.tiers_used`. Run benchmarks only once every tier is
    `ready`.
- **Sealed images.** Weights are stored encrypted in the image layers and decrypted in memory; nothing needs to be
  configured.
- **GPU memory.** g1-proxy-light needs about 16 GB at peak (the `halluc_context` reader and Geodesia-A run in
  int8); g1-proxy about 24 GB. If the upstream LLM runs on the same GPU, add its own memory. Disable the compiled
  graph with `GB_COMPILE=0` only if the engine logs a compilation failure (it falls back to eager mode by itself,
  never with a 5xx).

---

## 8. Runtime gateway config

You can change some settings live (no restart) — but **env wins on the next restart**, so make durable changes
in `.env`/compose.

```bash
# read current config
curl -s http://localhost:8800/v1/glad/gateway/config | python3 -m json.tool

# change the upstream model live (in-memory only)
curl -s -X POST http://localhost:8800/v1/glad/gateway/config \
  -H "Content-Type: application/json" -d '{"upstream_model":"ministral3"}'
```

---

## 9. Create an Application + API key

An **Application** is a tenant (its own upstream binding, policy, thresholds, budget). An **API key**
(`g1k_live_…`) is its runtime identity. Mint both on the control plane (`:8080`):

```bash
# 1) create the application  → copy the app_id
curl -s -X POST http://localhost:8080/v1/glad/apps \
  -H "Content-Type: application/json" \
  -d '{"name":"my-app","org_id":"default",
       "config":{"binding":{"upstream_type":"openai","base_url":"http://localhost:8002","model":"ministral3"}}}'
# → {"app_id":"my_app_1a2b3c","status":"active",...}

APP_ID=my_app_1a2b3c

# 2) create an API key  → copy api_key (shown ONCE)
curl -s -X POST http://localhost:8080/v1/glad/apps/$APP_ID/keys \
  -H "Content-Type: application/json" -d '{"role":"invoke"}'
# → {"key_id":"ak_...","api_key":"g1k_live_XXXX...","key_preview":"g1k_***XXXX","role":"invoke"}

API_KEY=g1k_live_XXXX...

# list / revoke keys
curl -s http://localhost:8080/v1/glad/apps/$APP_ID/keys
curl -s -X DELETE http://localhost:8080/v1/glad/apps/$APP_ID/keys/ak_...
```

`role`: `invoke` (call the LLM) or `admin` (also manage). Optional `"expires_at":"2026-12-31T00:00:00Z"`.
If `GEODESIA_ADMIN_TOKEN` is set on g1-studio, add `-H "X-Geodesia-Admin-Key: <token>"` to these writes.

---

## 10. Call the LLM with the key

Send OpenAI-style chat to the **engine** (`:8800`) with the key as a Bearer token.

**Non-streaming:**
```bash
curl -s http://localhost:8800/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"ministral3","stream":false,"max_tokens":30,
       "messages":[{"role":"user","content":"What is the capital of France?"}]}'
```

**Streaming (Server-Sent Events):**
```bash
curl -s -N http://localhost:8800/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"ministral3","stream":true,"max_tokens":40,
       "messages":[{"role":"user","content":"Name two prime numbers."}]}'
```

**Full scoring dump (generate + score, all metrics):**
```bash
curl -s http://localhost:8800/v1/glad/evaluate \
  -H "Authorization: Bearer $API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"ministral3","messages":[{"role":"user","content":"What is the capital of France?"}]}'
```

Key resolution order: explicit `application_id` in body / `X-Geodesia-App` header → the Bearer `g1k_…` key →
the `default` app. An unknown key falls back to `default` (HTTP 200, not 401).

---

## 11. Interpret the response

A normal OpenAI `chat.completion` **plus** exactly one extra key, `geodesia` (schema 1.1 — full reference in
[Response Format](reference/response-format.md)):

```json
{
  "choices": [{ "message": { "content": "The capital of France is **Paris**." },
                "finish_reason": "stop" }],
  "geodesia": {
    "schema_version": "1.1",
    "event": "final",
    "decision": "allowed",            // "allowed" | "flagged" | "blocked"  ← headline verdict
    "mode": "blocking",               // "blocking" (may withhold) | "passthrough" (report only)
    "reason": null,                   // { stage, axis, detail } when the decision is not "allowed"
    "axes": {
      "prompt_safety":     { "score": 0.0076, "threshold": 0.6377, "flagged": false, "available": true, "role": "enforce" },
      "jailbreak":         { "score": 0.3333, "threshold": 0.9864, "flagged": false, "available": true, "role": "enforce" },
      "rag_jailbreak":     { "score": null, "threshold": 0.5768, "flagged": false, "available": false,
                             "role": "advisory", "unavailable_reason": "no context supplied" },
      "halluc_context":    { "score": null, "threshold": 0.7551, "flagged": false, "available": false,
                             "role": "enforce", "unavailable_reason": "no context supplied" },
      "halluc_closedbook": { "score": 0.4687, "threshold": 0.8555, "flagged": false, "available": true,
                             "role": "advisory", "details": { "token_surprisal": [ ... ] } },
      "answer_safety":     { "score": 0.021, "threshold": 0.7953, "flagged": false, "available": true, "role": "enforce" }
    },
    "additional_axes": { ... }
  }
}
```

**The 9 axes** (each independent, own threshold):

| Axis | Region | Fires when |
|---|---|---|
| `prompt_safety` | input | the user's prompt is harmful |
| `jailbreak` | input | the prompt tries to override the rules |
| `rag_jailbreak` | context | injection via retrieved/context docs (`available` only with context) |
| `halluc_context` | answer | answer drifts from the provided context |
| `halluc_closedbook` | answer | answer is an unsupported fabrication (no context) |
| `answer_safety` | answer | the answer contains harmful content |
| `profanity` | input | the prompt is vulgar / abusive |
| `out_of_scope` | input | the prompt is off-topic for the declared scope (silent without one) |
| `prompt_complexity` | input | routing signal — picks Model A or Model B, never blocks |

**Per-axis fields:** `score` (the risk score you act on) vs `threshold`; `flagged` = the score is over the
threshold; `available: false` (with `score: null` and an `unavailable_reason`) means the axis had nothing to
read this turn — never read it as a zero; `role` is `enforce`, `advisory` or `classifier`. `halluc_closedbook`
adds `details.token_surprisal[]` (per-token `{index, text, surprisal, start, end}`), `details.uncertain_span`
and `details.method` as hallucination evidence.

**Decision:** `geodesia.decision` is `allowed`, `flagged` (a violation was detected but the answer was
delivered, e.g. in `passthrough` mode) or `blocked` (withheld). "Was this turn a violation?" is
`decision in ("flagged", "blocked")`.

**Detect a block** with `geodesia.decision=="blocked"` (the answer also has
`choices[0].finish_reason=="content_filter"`). `geodesia.reason.axis` says which axis caused it and
`geodesia.reason.stage` where (`input`, `output`, `tools`, `quota`). A blocked answer's content is a short
`[Geodesia blocked — <axis> (input)]` message.

**Streaming frames**, in order: (1) a first frame with `geodesia.event == "input_scan"` carries the prompt
scores (informational, no decision); (2) content frames carry `choices[0].delta.content` (concatenate them);
(3) the final frame (the one with `finish_reason`) carries `geodesia.event == "final"` with the full verdict —
take the decision only from this frame; (4) `data: [DONE]`. Minimal reader:

```python
import json, requests
r = requests.post("http://localhost:8800/v1/chat/completions",
    headers={"Authorization": f"Bearer {API_KEY}"},
    json={"model":"ministral3","stream":True,"max_tokens":40,
          "messages":[{"role":"user","content":"Name two prime numbers."}]}, stream=True)
answer, final = [], None
for line in r.iter_lines():
    if not line.startswith(b"data: "): continue
    body = line[6:]
    if body == b"[DONE]": break
    ch = json.loads(body)
    d = ch["choices"][0].get("delta", {})
    if d.get("content"): answer.append(d["content"])
    g = ch.get("geodesia")
    if g and g.get("event") == "final": final = g     # verdict only from the final event
print("".join(answer), "| decision:", final and final["decision"])
```

---

## 12. Operations

```bash
cd geodesia-g1

# health / version
curl -s http://localhost:8800/health
curl -s http://localhost:8800/version

# logs (follow)
docker compose logs -f
docker compose logs -f g1-proxy

# per-application usage
curl -s http://localhost:8080/v1/glad/apps/$APP_ID/metrics

# restart / recreate after a config change (a plain `restart` keeps the old environment)
docker compose up -d --force-recreate

# update to a newer image (asks per component)
./install.sh update

# stop / remove
docker compose down
docker compose down -v          # also wipe the DB/state volume (destroys apps, keys, logs)
```

**Reinstall from scratch** (for example to move from `both` to `light`, or after a broken update). `down -v`
destroys Applications, keys, calibrations and the Eval call log — export what you need first:

```bash
cd ~/geodesia/geodesia-g1
docker compose down -v
cd .. && rm -rf geodesia-g1
GLAD_LICENSE_TOKEN=glc_… UPSTREAM_TYPE=ollama UPSTREAM_URL=http://localhost:11434 \
UPSTREAM_MODEL=ministral-3:3b ./install.sh light
```

**Pin a version** instead of the moving `latest`: `TAG=1.0.10 ./install.sh light` pulls
`g1-proxy-light:1.0.10-gpu` (the light profile always adds `-gpu` or `-cpu` to `TAG`). Check what runs with
`curl -s http://localhost:8800/version`.

**G1 Eval call log.** Every `/g1/eval` call is recorded in a local SQLite database inside the data volume
(`/app/var/g1_eval.sqlite3`). Download it with the export API once `GW_EVAL_EXPORT_TOKEN` is set (see
[§4](#4-what-the-installer-created) and [G1 Eval](g1-proxy/eval.md)):

```bash
curl -s -H "Authorization: Bearer $EXPORT_TOKEN" \
  "http://127.0.0.1:8800/g1/eval/export?since=2026-11-01" > eval.ndjson
```

**Data that leaves the machine.** The reference check (`verify_references`) and live web search send the extracted references or queries
to outside services only when you enable them per request.

---

## 13. Troubleshooting

| Symptom | Fix |
|---|---|
| `application limit reached (free tier: 1)` | `MAX_APPS=unlimited ./install.sh both`, or set `GLAD_FREE_MAX_APPLICATIONS=unlimited` on g1-studio and recreate. |
| `health` shows `logprobs: false`, only 8 axes | Upstream doesn't return `top_logprobs`. Use ollama or vLLM with logprobs on. |
| Chat 404 / upstream error | `UPSTREAM_URL`/`UPSTREAM_MODEL` wrong or LLM down. `curl` the upstream directly; fix `.env`; `docker compose up -d --force-recreate`. |
| Model config changes revert after restart | Expected — the container ENV wins. Put durable values in `.env`/compose, not the runtime `POST config`. |
| Deploy health-poll fails / auto-rollback | The health endpoint is `/health` (not `/healthz`). |
| `--gpu` container won't start on a GPU-less host | The 1.0 releases are published as GPU images only; install on a host with an NVIDIA GPU and the container toolkit. |
| `license token rejected: …` / `license server unreachable` | The `glc_` token is expired, revoked or mistyped, or the host cannot reach `https://license.geodesia.ai`. Ask support for a new token, or pass the signed `GEO1.` key as the second argument. |
| `model not found` / UI says closed-book not available | The upstream model is not installed on Ollama or not set. `ollama pull <model>` or pick an installed one in Settings → Gateway. |
| Level 3 answers with `tiers_used` without `geodesia_a` | Geodesia-A is still starting: check `/health` → `tiers.glad_a.state`. |
| `/g1/eval/export` answers `403 export_disabled` | `GW_EVAL_EXPORT_TOKEN` is not set on the engine service: add it and recreate. `401` = the bearer does not match it. |
| A changed variable has no effect | The container was restarted, not recreated: `docker compose up -d --force-recreate`. |

---

## 14. Full environment-variable reference

**Install-time (in `.env` / `install.sh`):**

| Var | Default | Meaning |
|---|---|---|
| `REG` | `europe-west1-docker.pkg.dev/glad-manifold-v2/glad` | image registry |
| `TAG` | `latest` | image tag (g1-proxy: `TAG` or `TAG-cpu`; g1-proxy-light: `TAG-gpu` or `TAG-cpu`; g1-studio: `TAG`) |
| `HTTP_PORT` | `8080` | UI port (g1-studio) |
| `GATEWAY_PORT` | `8800` | engine port (g1-proxy) |
| `UPSTREAM_TYPE` | `ollama` | `ollama\|openai\|vllm\|sglang\|trtllm\|internal\|azure-openai\|bedrock\|vertex` |
| `UPSTREAM_URL` | `http://localhost:11434` | your LLM base URL |
| `UPSTREAM_MODEL` | (empty) | model name the LLM serves — no default; choose it here or in the UI |
| `GATEWAY_URL` | `http://localhost:8800` | where g1-studio reaches the engine |
| `WORKDIR` | `$PWD/geodesia-g1` | install directory |
| `LICENSE` | (unset) | `GEO1.<b64>` \| path \| raw JSON |
| `GLAD_LICENSE_TOKEN` (`GLC`) | (unset) | `glc_…` license token, exchanged once for the signed key |
| `LICENSE_API_URL` | `https://license.geodesia.ai/v1/glad/license-tokens/validate` | where the installer validates the token |
| `CHATS_PER_DAY`/`MAX_MODELS`/`MAX_APPS` | (unset) | free-tier caps (`0`/`unlimited` = no cap) |
| `FREE_UNLIMITED` | `0` | `1` = remove all free caps |

**Engine runtime (g1-proxy service `environment:`):**

| Var | Default | Meaning |
|---|---|---|
| `GW_DEVICE` | `cuda` (`cpu` with `--cpu`) | detector device |
| `GW_GLADH_CKPT` / `GW_GLADH_DEVICE` | (unset) / `auto` | Capability pack + device for [thinking levels](g1-proxy/thinking-levels.md) 1–2 |
| `GW_GLADA_CKPT` / `GW_GLADA_DEVICE` | (unset) / `auto` | Capability pack + device for thinking level 3 (MAX) |
| `GW_BLOCK_INPUT` | `1` | block flagged prompts (vs score-only) |
| `GW_INJECT_SYSTEM` | `1` | inject the safety system prompt |
| `GW_MAX_THINKING_LEVEL` | `3` | highest thinking level served |
| `GW_PROMPT_WINDOW` | `50` | user turns read by the input scan (`0` = last message only) |
| `GW_EVAL_LOG` / `GW_EVAL_CAPTURE_BODY` | `1` / `1` | G1 Eval call log; store the request bodies |
| `GW_EVAL_EXPORT_TOKEN` | (unset) | bearer for `/g1/eval/export` and `/g1/eval/stats` (unset = disabled) |
| `GW_ENFORCE_OUT_OF_SCOPE` | `0` | decide `out_of_scope` with the dedicated scope reader |
| `GLAD_FREE_MAX_CHATS_PER_DAY` / `_MAX_MODELS` / `_MAX_APPLICATIONS` | free-tier | runtime caps (`unlimited` to lift) |

**Quick reference — endpoints:**

| Action | Method + path | Port |
|---|---|---|
| Health / version | `GET /health`, `GET /version` | 8800 |
| Chat (with key) | `POST /v1/chat/completions` + `Authorization: Bearer g1k_…` | 8800 |
| Full scoring | `POST /v1/glad/evaluate` | 8800 |
| Evaluation only (no LLM call) | `POST /g1/eval` | 8800 |
| Eval call log | `GET /g1/eval/stats`, `GET /g1/eval/export` + `Authorization: Bearer <GW_EVAL_EXPORT_TOKEN>` | 8800 |
| Gateway config | `GET/POST /v1/glad/gateway/config` | 8800 |
| Entitlements | `GET /v1/glad/gateway/entitlements` | 8800 |
| Create/list app | `POST/GET /v1/glad/apps` | 8080 |
| Create/list/revoke key | `POST/GET/DELETE /v1/glad/apps/{id}/keys[/{key_id}]` | 8080 |
| App metrics | `GET /v1/glad/apps/{id}/metrics` | 8080 |

Key format `g1k_live_…` (shown once). Verdict = `geodesia.decision` (`allowed`/`flagged`/`blocked`), why =
`geodesia.reason`. Block tells: `decision=="blocked"` / `finish_reason=="content_filter"`.
