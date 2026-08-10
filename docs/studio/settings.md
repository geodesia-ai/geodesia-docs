# Settings

The Settings page configures the **platform-wide** proxy: upstream binding, Constitutional Intelligence,
the numeric solver, licence, database. Everything **per-Application** — model binding, detection policy,
thresholds, calibration, RAG, cost, governance — lives under **Applications → pick an app → Edit**, not
here. Every control on the page is one field on one API object.

---

## Do it from the API

**What it does.** Everything on this page is one object on the gateway. `GET` it to read the live
configuration, `POST` a partial patch to change it — applied immediately and persisted to
`GW_CONFIG_FILE`, no restart.

=== "curl"

    ```bash
    # read (the upstream key comes back masked as "***", never in clear)
    curl -s http://localhost:8080/gw/v1/glad/gateway/config | jq

    # patch — send only what changes
    curl -s -X POST http://localhost:8080/gw/v1/glad/gateway/config \
      -H "Content-Type: application/json" \
      -d '{"upstream_type":"ollama","upstream_base_url":"http://localhost:11434",
           "upstream_model":"llama3.1:8b","inject_system_prompt":false}' | jq '.ok'
    ```

=== "Python"

    ```python
    import httpx

    c = httpx.Client(base_url="http://localhost:8080/gw", timeout=60)

    cfg = c.get("/v1/glad/gateway/config").json()
    print(cfg["upstream_type"], cfg["upstream_model"])

    r = c.post("/v1/glad/gateway/config", json={
        "upstream_model": "llama3.1:8b",
        "inject_system_prompt": False,
        # leave "***" (or omit the key) to keep the stored upstream credential
        "upstream_api_key": "***",
    }).json()
    print(r["ok"], r["config"]["upstream_model"])
    ```

=== "TypeScript"

    ```ts
    const BASE = "http://localhost:8080/gw"

    const cfg = await fetch(`${BASE}/v1/glad/gateway/config`).then(r => r.json())

    const r = await fetch(`${BASE}/v1/glad/gateway/config`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ upstream_model: "llama3.1:8b", inject_system_prompt: false }),
    }).then(r => r.json())
    console.log(r.ok)
    ```

**What comes back** — `{"ok": true, "config": { … }}` with the full configuration after the merge, minus
the upstream key.

!!! info "Save applies live — with one exception"
    The detector and the numeric solver hot-reload on the next request. The **bind host/port** does not:
    the gateway reads its listening port from `GW_PORT` at process start, so a new bind is persisted but
    only takes effect after you recreate the container — see [Changing the bind port](#changing-the-bind-port).

!!! danger "Three keys the endpoint refuses"
    `system_prompt`, `internal_vllm_cmd` and `internal_vllm_url` are silently dropped, and
    `upstream_type: "internal"` is rejected with a `400`. They feed a subprocess launch and the
    constitutional prompt; they are settable by env/CLI only, so a remote caller cannot reach them.

    When `GW_API_TOKEN` is set, this endpoint requires `Authorization: Bearer <token>`.

---

## The sections

### Gateway URL

Where **this console** reaches the running gateway. For the normal, co-located deployment leave it as
`http://localhost:8800` — the Studio backend reverse-proxies the console to the gateway over the
same-origin `/gw` path, so the exact port here does not need to be reachable from your browser. Only set a
different value when the gateway runs on **another host**.

!!! tip "‘Save failed — Failed to fetch’?"
    That happened in older builds when you typed a port the gateway wasn't actually listening on — the
    console tried to POST directly to that dead port. Current builds route a `localhost`/loopback gateway
    through the same-origin `/gw` proxy, so Save works regardless of the port shown here. If you still see
    it with a **remote** Gateway URL, the host/port is genuinely unreachable from the browser.

### Exposed API — Bind host & port

The interface and TCP port where **your downstream clients** connect to the gateway's OpenAI-compatible
API (`http://<host>:<port>/v1`). `0.0.0.0` = all interfaces, `127.0.0.1` = local-only.

!!! warning "Bind changes need a container recreate"
    Saving a new bind host/port **persists** it, but the gateway is already bound and re-reads its port
    from `GW_PORT` at launch, so the change applies only after you recreate the container — see below.

### Constitutional Intelligence

Deployment-wide default: when on, the gateway prepends Geodesia's safety/truthfulness system prompt to
every request. Turn it off only if you supply your own system prompt. **Applies live on Save.**

### Numeric Solver (FinQA — optional)

Off / PoT / strong / API modes for numeric-reasoning verification. **Applies live** (the model loads lazily
on the first numeric request; a strong solver may pull a 7B model on first use).

### Thinking Levels

A per-request depth dial (`thinking_level` `0`–`3`, `3` = MAX). Which levels this deployment can serve is set
by `GW_GLADH_CKPT` / `GW_GLADA_CKPT` (env/CLI only, not from this page) — see
[Thinking Levels](../g1-proxy/thinking-levels.md).

### Plan & License

Paste a vendor-signed licence key to unlock higher limits, and read your current **tier, company, daily
chat limit / used / remaining, models and expiry**. Applying or removing a licence is immediate. Details:
[Licensing & Entitlements](licensing.md).

### Database

SQLite by default; optionally point Studio at an existing PostgreSQL. `GEODESIA_DB_URL` (env) always
overrides the UI choice, and a database switch takes effect after a restart.

---

## Changing the bind port

Because the gateway binds the port given at launch (`GW_PORT`), change it in your install's `.env` and
recreate the container:

```bash
cd ~/geodesia-g1                       # your install directory (has docker-compose.yml + .env)
sed -i 's/^GATEWAY_PORT=.*/GATEWAY_PORT=8801/' .env
docker compose up -d --force-recreate g1-proxy
```

The `GATEWAY_PORT` in `.env` drives the gateway's `--port` (and, with `network_mode: host`, the port
clients connect to). After the recreate, update the **Gateway URL** field only if the console is on a
different host.

---

## Restarting to apply other changes

| You changed… | What to do |
|---|---|
| Numeric solver · Constitutional Intelligence · thresholds · upstream | **Just Save** — hot-reloaded on the next request. |
| **Bind host / port** | Save, then **recreate the container** with the new `GATEWAY_PORT` (above). |
| **Database** (SQLite ↔ PostgreSQL) | Save, then `docker compose up -d --force-recreate g1-proxy g1-studio`. |
| **Thinking level availability** (`GW_GLADH_CKPT`) | Env/CLI only — recreate the container with the new value. |

!!! note "Full restart / clean reinstall"
    To restart the whole stack: `docker compose restart` (or `up -d --force-recreate`). For a clean
    reinstall from scratch see the [Installer](../installer.md) — e.g. `./install.sh both --gpu 'GEO1.<licence>'`.
