# Settings

The **Settings** page in G-1 Studio configures the **platform-wide** gateway (the Geodesia G1-Proxy
service): upstream binding, Constitutional Intelligence, the numeric solver, your licence and
the database. Everything that is **per-Application** — model binding, detection policy, thresholds,
closed-book calibration, RAG, cost and governance — lives under **Applications → pick an app → Edit**, not
here.

!!! info "What Save does — and what it doesn't"
    Clicking **Save configuration** writes the config to the gateway (`POST /v1/glad/gateway/config`) and
    **hot-reloads** the detector and numeric solver on the next request — no restart
    needed. The **one exception is the bind host/port**: the gateway picks its listening port at process
    start from `GW_PORT` (the container launch), so a new bind takes effect only after you **recreate the
    container** (see [Changing the bind port](#changing-the-bind-port)). The Save status line tells you
    which case you are in.

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

### Thinking Levels (GLAD-H second opinion)

`thinking_level` 1/2 blend in GLAD-H, a second detector, per request. Platform availability is set via
`GW_GLADH_CKPT` (env/CLI only, not from this page) — see [Thinking Levels](../g1-proxy/thinking-levels.md).

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
