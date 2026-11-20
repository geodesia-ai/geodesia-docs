# The `install.sh` Installer

`install.sh` is the single, self-contained, one-command installer for the full Geodesia G-1 stack
(**g1-proxy** engine + **g1-studio** UI) or for the single-container **g1-proxy-light**. It embeds read-only
registry credentials, pulls the self-contained images, writes the config, and starts everything. This page is
the complete reference.

!!! tip "In a hurry?"
    ```bash
    chmod +x install.sh
    UPSTREAM_MODEL=ministral-3:3b ./install.sh both     # engine + UI, GPU, free tier
    # open http://<host>:8080  ·  engine at http://<host>:8800

    GLAD_LICENSE_TOKEN=glc_… UPSTREAM_MODEL=ministral-3:3b ./install.sh light   # one container
    # open http://<host>:8800  (UI and engine on the same port)
    ```

---

## Synopsis

```text
./install.sh [both|g1-proxy|g1-studio] [--cpu|--gpu] [--deep-scan-cpu] [LICENSE]
./install.sh light [--cpu|--gpu] [LICENSE]
./install.sh update
./install.sh -h | --help
```

- **First positional** = target (default `both`).
- **`--cpu` / `--gpu`** = image variant (default `--gpu`). Can appear anywhere in the arguments.
- **Second positional** (or `LICENSE=`) = a signed license key (optional; omit → free tier). Alternatively
  `GLAD_LICENSE_TOKEN=glc_…`, see [Licensing](#licensing-second-argument).

---

## Requirements

| Path | Needs |
|---|---|
| Any | Docker + Docker Compose v2 (the installer offers to install the Compose plugin if it is missing); an OpenAI-compatible or Ollama upstream LLM, ideally returning `logprobs` |
| `--gpu` (default) | NVIDIA GPU + `nvidia-container-toolkit`; about 16 GB of GPU memory for `light`, 24 GB for `g1-proxy` (plus the upstream LLM's own memory if it shares the card) |
| `--cpu` | nothing extra — but see the note under [Device](#device-cpu-gpu): 1.0 releases are published as GPU images only |

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
| `light` | **g1-proxy-light alone** — engine *and* its own UI in **one** container, on the gateway port | evaluation, a single machine |

!!! info "The `light` profile"
    One container, no separate UI: everything g1-proxy serves — nine axes at **thinking levels 0–3**
    (Geodesia-G, -H and -A), the `halluc_context` reader, closed-book with SLEDGE, the reference check,
    [G1 Eval](g1-proxy/eval.md), MuPAX and MCP — **except** the 9B judge, speech recognition and RAG.
    It carries no 8B deep-scan model, so `DEEP_SCAN` does not apply to it. Its images are tagged
    `TAG-gpu` (and `TAG-cpu` where one is published), and `install.sh update` re-pulls the variant the
    install originally chose. See [G1-Proxy Light](g1-proxy/light.md).
| `update` | — | checks the registry for newer images and updates per component (see [Updating](#updating)) |

---

## Device (`--cpu` / `--gpu`)

- `--gpu` **(default)** — the companion detector runs on CUDA. Requires an NVIDIA GPU and the
  container toolkit. Pulls the g1-proxy image tagged `TAG`.
- `--cpu` — no GPU needed; G1-Hummingbird runs on CPU. Pulls the g1-proxy image
  tagged `TAG-cpu`. (g1-studio is architecture-agnostic and always uses `TAG`.)

!!! warning "CPU images"
    Releases from 1.0.2 onward are published as **GPU images only** (`TAG-gpu`; g1-proxy also as plain
    `TAG`). A `--cpu` install needs a `-cpu` image for the chosen `TAG`; without one the pull fails. Ask
    support if you need a CPU build.

The flag can appear anywhere; these are equivalent:
```bash
./install.sh both --cpu LICENSE
./install.sh --cpu both LICENSE
```

---

## Licensing (second argument)

The license defines your tier (chats/day, models, apps, expiry). It is a **vendor-signed** key, verified
**offline** by the engine (Ed25519 — air-gap friendly). You can pass it four ways:

```bash
./install.sh both 'GEO1.eyJwYXlsb2FkIjp7...'   # the GEO1.<base64> token (recommended)
LICENSE=/path/license.json ./install.sh both    # a path to a license.json
LICENSE='{"payload":{...},"sig":"..."}' ./install.sh both   # raw JSON
GLAD_LICENSE_TOKEN=glc_XXXXXXXXXXXXXXXXXXXXXXXX ./install.sh light   # license token
```

**License token (`glc_…`).** With `GLAD_LICENSE_TOKEN` (or `GLC=`) set and no `LICENSE`, the installer asks
the license server (`LICENSE_API_URL`, default `https://license.geodesia.ai/v1/glad/license-tokens/validate`)
for the signed key bound to the token, **once, at install time**, then continues exactly as with a signed key.
The running containers never call the license server. If the server rejects the token (expired, revoked,
unknown) or cannot be reached, the install **stops** — it never falls back to the free tier for a licensed
customer. On an air-gapped host, pass the `GEO1.` key instead. The token is a credential: keep it out of shell
history, and never send it as the `Authorization` bearer of API calls.

- **No license → FREE tier**: 20 chats/day, 1 model, 1 application.
- **Quick caps without signing** (free tier only): environment variables, `0`/`unlimited` = no cap.

  ```bash
  CHATS_PER_DAY=2000 MAX_MODELS=5 MAX_APPS=3 ./install.sh both
  FREE_UNLIMITED=1 ./install.sh both        # remove all caps (self-host)
  ```

- **Block a customer**: issue a signed license with `--max-chats-per-day 0`, or an expired one.
- **`GLC=` / `GLAD_LICENSE_TOKEN=`** — the `glc_` token is also written to `.env` and passed to the
  containers for identification.

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
| `UPSTREAM_MODEL` | (empty) | the model your LLM serves — **no default**: choose it here or later in the UI |
| `GATEWAY_URL` | `http://localhost:8800` | where g1-studio reaches the engine (for `g1-studio`-only installs) |
| `WORKDIR` | `$PWD/geodesia-g1` | where the install files live |
| `LICENSE` | (unset) | signed key / path / raw JSON |
| `GLAD_LICENSE_TOKEN` (`GLC`) | (unset) | `glc_` license token, exchanged once for the signed key |
| `LICENSE_API_URL` | `https://license.geodesia.ai/v1/glad/license-tokens/validate` | license-token endpoint |
| `DEEP_SCAN` | `on` | 8B deep-scan judge (g1-proxy only; `--deep-scan-cpu` moves it to the CPU) |
| `CHATS_PER_DAY` `MAX_MODELS` `MAX_APPS` | (unset) | free-tier caps (`0`/`unlimited` = no cap) |
| `FREE_UNLIMITED` | `0` | `1` = lift all free caps |
| `SA_JSON_B64` | (embedded) | override the built-in registry service-account key |
| `REGISTRY_TOKEN` / `GOOGLE_APPLICATION_CREDENTIALS` | (unset) | alternative registry auth |

Point at your own upstream at install time:
```bash
UPSTREAM_TYPE=openai UPSTREAM_URL=http://localhost:8002 UPSTREAM_MODEL=ministral3 ./install.sh both
UPSTREAM_TYPE=ollama UPSTREAM_URL=http://localhost:11434 UPSTREAM_MODEL=llama3.1:8b ./install.sh both
```

**No default model.** With `UPSTREAM_MODEL` empty the engine starts without a model; chat and closed-book
calibration wait until you choose one in the UI (Settings → Gateway). With `UPSTREAM_TYPE=ollama` the installer
asks Ollama which models are installed (`/api/tags`): an explicit `UPSTREAM_MODEL` that is not installed stops
the install with the list; an unreachable Ollama only prints a warning.

!!! warning "Closed-book needs the answer's `logprobs`"
    The closed-book hallucination axis reads `top_logprobs`; ollama and vLLM both provide them. If they are
    missing, `/health` shows `logprobs: false` with the reason in `logprobs_reason`, and `halluc_closedbook`
    is listed in `axes_gated` (the other axes are unaffected).

---

## What the installer does

1. Resolves the license into `./license.json` (if provided — from a `glc_` token it first asks the license
   server for the signed key) and derives the tier env.
2. Authenticates to the registry — order: embedded read-only SA → `REGISTRY_TOKEN` →
   `GOOGLE_APPLICATION_CREDENTIALS` → `gcloud` → assume the images are public/local.
3. Pulls the images (`g1-proxy:TAG` or `TAG-cpu`, and/or `g1-studio:TAG`).
4. Writes, into `WORKDIR` (default `./geodesia-g1`):
   - **`.env`** (chmod 600) — your resolved `REG/TAG/DEVICE/UPSTREAM_*/...`
   - **`docker-compose.yml`** — the generated 1- or 2-service stack (host networking; `gpus: all` unless
     `--cpu`; the engine gets `GW_BLOCK_INPUT=1`, `GW_INJECT_SYSTEM=1`, the upstream binding,
     and the license/caps env). With `light` it has a single `g1-proxy-light` service.
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

The first start takes a few minutes: the sealed weights are decrypted, the three tiers load and the
detector graph is compiled. Wait until every tier is `ready`:

```bash
curl -s http://localhost:8800/health          # {"ok": true, "logprobs": true, "tiers": {"glad_g": {"state": "ready"}, ...}, ...}
curl -s http://localhost:8080/v1/glad/apps     # {"apps":[{"app_id":"default",...}]}   (both / g1-studio)
# open the UI:  http://<host>:8080   — light: http://<host>:8800
```

Next: [create an Application + API key](developer-quickstart.md) and make your first call.

---

## Updating

```bash
cd geodesia-g1        # the WORKDIR from the original install
./install.sh update
```

`update` reloads the persisted `.env` (so it re-pulls the exact same variant and profile), checks the
registry for a newer image **per component** (g1-proxy, g1-studio, g1-proxy-light), and asks before applying
each. Non-interactive stdin → safe default: does **not** update (the newer image is still pulled locally for
next time). `update` follows the tag in `.env` (normally the moving `latest`); to move to a specific release,
re-run the installer with `TAG=<version>` (for `light`: `TAG=1.0.10` pulls `g1-proxy-light:1.0.10-gpu`).

---

## Operate

```bash
cd geodesia-g1
docker compose logs -f              # follow logs (both services)
docker compose logs -f g1-proxy     # engine only
docker compose up -d --force-recreate   # re-apply after editing .env / compose (restart alone keeps the old env)
docker compose down                 # stop
docker compose down -v              # stop AND wipe the DB/state volume (destroys apps, keys, logs)
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `docker login ... failed` | The embedded key is read-only and time-limited; pass your own with `SA_JSON_B64=`/`REGISTRY_TOKEN=`, or ensure the images are already present. |
| `--gpu` container won't start | No GPU/toolkit on the host. CPU images are not published for 1.0 releases: install on a GPU host. |
| `license token rejected` / `license server unreachable` | Expired or mistyped `glc_` token, or no access to `license.geodesia.ai`. Ask support, or pass the `GEO1.` key as the second argument. |
| `UPSTREAM_MODEL='…' is not installed on the Ollama` | `ollama pull <model>` first, or choose one of the installed models the installer lists. |
| `/health` shows `logprobs: false` | Read `logprobs_reason`: model missing, authentication, or an upstream that returns no `top_logprobs` (use ollama or vLLM). |
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
HTTP_PORT=8080 GATEWAY_PORT=8800 \
./install.sh both 'GEO1.eyJwYXlsb2FkIjp7...'

curl -s http://localhost:8800/health
# → open http://localhost:8080
```
