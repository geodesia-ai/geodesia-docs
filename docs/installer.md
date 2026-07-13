# The `install.sh` Installer

`install.sh` is the single, self-contained, one-command installer for the full Geodesia G-1 stack
(**g1-proxy** engine + **g1-studio** UI). It embeds read-only registry credentials, pulls the
self-contained images, writes the config, and starts everything. This page is the complete reference.

!!! tip "In a hurry?"
    ```bash
    chmod +x install.sh
    ./install.sh both            # engine + UI, GPU, free tier
    # open http://<host>:8080  ·  engine at http://<host>:8800
    ```

---

## Synopsis

```text
./install.sh [both|g1-proxy|g1-studio] [--cpu|--gpu] [LICENSE]
./install.sh update
./install.sh -h | --help
```

- **First positional** = target (default `both`).
- **`--cpu` / `--gpu`** = image variant (default `--gpu`). Can appear anywhere in the arguments.
- **Second positional** (or `LICENSE=`) = a signed license token (optional; omit → free tier).

---

## Requirements

| Path | Needs |
|---|---|
| Any | Docker + Docker Compose v2; an OpenAI-compatible upstream LLM returning `logprobs` |
| `--gpu` (default) | NVIDIA GPU + `nvidia-container-toolkit` |
| `--cpu` | nothing extra — GLAD-BERT **and** deep scan run on CPU |

```bash
docker --version && docker compose version
nvidia-smi                      # only for --gpu
curl -s http://localhost:11434/api/tags   # your upstream (example: ollama)
```

---

## Targets (first argument)

| Target | Installs | Notes |
|---|---|---|
| `both` *(default)* | g1-proxy **+** g1-studio | the normal install |
| `g1-proxy` | engine only | headless / API-only |
| `g1-studio` | UI only | point it at an external engine with `GATEWAY_URL=` |
| `update` | — | checks the registry for newer images and updates per component (see [Updating](#updating)) |

---

## Device (`--cpu` / `--gpu`)

- `--gpu` **(default)** — the companion detector and deep scan run on CUDA. Requires an NVIDIA GPU and the
  container toolkit. Pulls the g1-proxy image tagged `TAG`.
- `--cpu` — no GPU needed; GLAD-BERT **and** GLAD-Tapestry deep scan run on CPU. Pulls the g1-proxy image
  tagged `TAG-cpu`. (g1-studio is architecture-agnostic and always uses `TAG`.)

The flag can appear anywhere; these are equivalent:
```bash
./install.sh both --cpu LICENSE
./install.sh --cpu both LICENSE
```

!!! note "Deep scan on CPU"
    Deep scan defaults **on**. On `--cpu` it loads the deep-scan model in bf16 (~16 GB RAM) on first use and is
    noticeably slower than GPU. Set `DEEP_SCAN=off` before installing to disable it.

---

## Licensing (second argument)

The license defines your tier (chats/day, models, apps, expiry). It is a **vendor-signed** token, verified
**offline** (Ed25519 — no license server, air-gap friendly). You can pass it three ways:

```bash
./install.sh both 'GEO1.eyJwYXlsb2FkIjp7...'   # the GEO1.<base64> token (recommended)
LICENSE=/path/license.json ./install.sh both    # a path to a license.json
LICENSE='{"payload":{...},"sig":"..."}' ./install.sh both   # raw JSON
```

- **No license → FREE tier**: 20 chats/day, 1 model, 1 application.
- **Quick caps without signing** (free tier only): environment variables, `0`/`unlimited` = no cap.

  ```bash
  CHATS_PER_DAY=2000 MAX_MODELS=5 MAX_APPS=3 ./install.sh both
  FREE_UNLIMITED=1 ./install.sh both        # remove all caps (self-host)
  ```

- **Block a customer**: issue a signed license with `--max-chats-per-day 0`, or an expired one.
- **`GLC=` / `GLAD_LICENSE_TOKEN=`** — an optional `glc_` identifier passed through for identification.

The signed license is delivered to the containers as the `GLAD_LICENSE` env (base64 of the JSON), **not** a
read-only bind-mount (a `:ro` mount makes the UI's "apply license" fail with EBUSY). The UI can re-apply a
license into the writable `/app/var` volume, which then takes precedence.

Check the resulting tier after install:
```bash
curl -s http://localhost:8800/v1/glad/gateway/entitlements | python3 -m json.tool
```

---

## Environment variables

All optional, with sensible defaults. Set them inline before the command.

| Var | Default | Meaning |
|---|---|---|
| `REG` | `europe-west1-docker.pkg.dev/glad-manifold-v2/glad` | image registry |
| `TAG` | `latest` | image tag (moving `latest` = newest protected build) |
| `HTTP_PORT` | `8080` | g1-studio UI port |
| `GATEWAY_PORT` | `8800` | g1-proxy engine port |
| `UPSTREAM_TYPE` | `ollama` | `ollama` · `openai` · `vllm` · `sglang` · `trtllm` · `internal` · `azure-openai` · `bedrock` · `vertex` |
| `UPSTREAM_URL` | `http://localhost:11434` | your LLM base URL |
| `UPSTREAM_MODEL` | `llama3.1:8b` | the model your LLM serves |
| `DEEP_SCAN` | `on` | `on`/`off` — geometric MoE second-opinion judge |
| `GATEWAY_URL` | `http://localhost:8800` | where g1-studio reaches the engine (for `g1-studio`-only installs) |
| `WORKDIR` | `$PWD/geodesia-g1` | where the install files live |
| `LICENSE` | (unset) | signed token / path / raw JSON |
| `CHATS_PER_DAY` `MAX_MODELS` `MAX_APPS` | (unset) | free-tier caps (`0`/`unlimited` = no cap) |
| `FREE_UNLIMITED` | `0` | `1` = lift all free caps |
| `SA_JSON_B64` | (embedded) | override the built-in registry service-account key |
| `REGISTRY_TOKEN` / `GOOGLE_APPLICATION_CREDENTIALS` | (unset) | alternative registry auth |

Point at your own upstream at install time:
```bash
UPSTREAM_TYPE=openai UPSTREAM_URL=http://localhost:8002 UPSTREAM_MODEL=ministral3 ./install.sh both
UPSTREAM_TYPE=ollama UPSTREAM_URL=http://localhost:11434 UPSTREAM_MODEL=llama3.1:8b ./install.sh both
```

!!! warning "The upstream must return `logprobs`"
    The closed-book hallucination axis needs `top_logprobs`. ollama and vLLM both provide them. If missing,
    the engine runs with 4 axes instead of 5 (`/health` shows `logprobs: false`).

---

## What the installer does

1. Resolves the license into `./license.json` (if provided) and derives the tier env.
2. Authenticates to the registry — order: embedded read-only SA → `REGISTRY_TOKEN` →
   `GOOGLE_APPLICATION_CREDENTIALS` → `gcloud` → assume the images are public/local.
3. Pulls the images (`g1-proxy:TAG` or `TAG-cpu`, and/or `g1-studio:TAG`).
4. Writes, into `WORKDIR` (default `./geodesia-g1`):
   - **`.env`** (chmod 600) — your resolved `REG/TAG/DEVICE/UPSTREAM_*/DEEP_SCAN/...`
   - **`docker-compose.yml`** — the generated 1- or 2-service stack (host networking; `gpus: all` unless
     `--cpu`; the engine gets `GW_BLOCK_INPUT=1`, `GW_INJECT_SYSTEM=1`, `GW_DEEP_SCAN`, the upstream binding,
     and the license/caps env)
   - **`license.json`** (chmod 600) — if you passed a license
5. Starts everything: `docker compose up -d --force-recreate` (so re-running the installer always applies the
   new image/license/config).
6. Prints the URLs, the active tier, and the logs/stop commands.

Inspect it afterwards:
```bash
cd geodesia-g1
cat .env
cat docker-compose.yml
docker compose ps
```

---

## Verify the install

```bash
curl -s http://localhost:8800/health          # {"ok": true, "logprobs": true, "axes": 5, ...}
curl -s http://localhost:8080/v1/glad/apps     # {"apps":[{"app_id":"default",...}]}
# open the UI:  http://<host>:8080
```

Next: [create an Application + API key](developer-quickstart.md) and make your first call.

---

## Updating

```bash
cd geodesia-g1        # the WORKDIR from the original install
./install.sh update
```

`update` reloads the persisted `.env` (so it re-pulls the exact same variant, e.g. `-cpu`), checks the
registry for a newer image **per component**, and asks before applying each. Non-interactive stdin → safe
default: does **not** update (the newer image is still pulled locally for next time).

---

## Operate

```bash
cd geodesia-g1
docker compose logs -f              # follow logs (both services)
docker compose logs -f g1-proxy     # engine only
docker compose up -d --force-recreate   # re-apply after editing .env / compose
docker compose down                 # stop
docker compose down -v              # stop AND wipe the DB/state volume (destroys apps, keys, logs)
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `docker login ... failed` | The embedded key is read-only and time-limited; pass your own with `SA_JSON_B64=`/`REGISTRY_TOKEN=`, or ensure the images are already present. |
| `--gpu` container won't start | No GPU/toolkit on the host → install with `--cpu`. |
| `/health` shows `logprobs: false` (4 axes) | Upstream doesn't return `top_logprobs`; use ollama or vLLM with logprobs on. |
| Chat 404 / upstream error | `UPSTREAM_URL`/`UPSTREAM_MODEL` wrong or the LLM is down. `curl` it directly, fix `.env`, `docker compose up -d --force-recreate`. |
| `application limit reached (free tier: 1)` | Raise `MAX_APPS`, use a license, or `FREE_UNLIMITED=1`. |
| Config reverts after restart | The container ENV is the source of truth — put durable values in `.env`, not the runtime `POST config`. |
| Re-running the installer didn't apply changes | It uses `--force-recreate`; make sure you edited the `.env` in the **`WORKDIR`** and re-ran from there. |

---

## Full example

```bash
mkdir -p ~/geodesia && cd ~/geodesia
chmod +x install.sh

UPSTREAM_TYPE=openai UPSTREAM_URL=http://localhost:8002 UPSTREAM_MODEL=ministral3 \
DEEP_SCAN=on HTTP_PORT=8080 GATEWAY_PORT=8800 \
./install.sh both 'GEO1.eyJwYXlsb2FkIjp7...'

curl -s http://localhost:8800/health
# → open http://localhost:8080
```
