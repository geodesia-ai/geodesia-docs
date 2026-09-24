# Live Web Search

Grounding an answer in the live internet means pulling untrusted text into the prompt — which is exactly
how indirect prompt injection works. G1-Proxy searches, then **screens every fetched page through the
detector** and grounds the answer only in the ones that pass. One boolean on the chat body turns it on.

---

## Use it

**What it does.** Set `web_search: true` on a `POST /v1/chat/completions` request. The proxy searches, screens
each fetched page through the detector, keeps only the safe ones as grounding context, and streams the search
progress to you as [research events](#streaming-research-events) before the answer. On by default
(`GW_WEBSEARCH_ENABLED=1`).

!!! warning "A web-search request is always answered as a stream"
    With `web_search: true` the response is **always** an SSE stream (`text/event-stream`) — research events
    first, then the answer, then the `final` event — **whatever `stream` says**. Read it with a streaming client.
    The pages used as sources arrive as `page_read` research events (`url`, `title`); they are **not** repeated
    under `geodesia.rag` in the final event.

=== "curl"

    ```bash
    curl -N -s http://localhost:8080/gw/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{
        "model": "my-model",
        "stream": true,
        "web_search": true,
        "messages": [{"role":"user","content":"What were the headline announcements at the latest Apple event?"}]
      }'
    ```

=== "Python"

    ```python
    from openai import OpenAI

    client = OpenAI(base_url="http://localhost:8080/gw/v1", api_key="not-needed-locally")

    stream = client.chat.completions.create(
        model="my-model",
        stream=True,                                   # web search always streams
        messages=[{"role": "user", "content": "What were the headline announcements at the latest Apple event?"}],
        extra_body={"web_search": True},
    )

    sources, verdict = [], None
    for chunk in stream:
        g = (chunk.model_extra or {}).get("geodesia")
        if g and g["event"] == "research":
            ev = g["research"]
            if ev["type"] == "page_read":
                sources.append(ev["url"])
            elif ev["type"] == "page_blocked":
                print(f"blocked {ev['url']} ({ev['axis']}: {ev['reason']})")
        elif g and g["event"] == "final":
            verdict = g
        if chunk.choices and chunk.choices[0].delta.content:
            print(chunk.choices[0].delta.content, end="", flush=True)

    print("\nsources:", sources)
    print("decision:", verdict["decision"])
    ```

=== "TypeScript"

    ```ts
    import OpenAI from "openai"

    const client = new OpenAI({ baseURL: "http://localhost:8080/gw/v1", apiKey: "not-needed-locally" })

    const stream = await client.chat.completions.create({
      model: "my-model",
      stream: true,                                  // web search always streams
      messages: [{ role: "user", content: "What were the headline announcements at the latest Apple event?" }],
      // @ts-expect-error — Geodesia extension field
      web_search: true,
    })

    const sources: string[] = []
    let verdict: any = null
    for await (const chunk of stream) {
      const g = (chunk as any).geodesia
      if (g?.event === "research" && g.research.type === "page_read") sources.push(g.research.url)
      if (g?.event === "final") verdict = g
      process.stdout.write(chunk.choices[0]?.delta?.content ?? "")
    }
    console.log("\nsources:", sources, "decision:", verdict?.decision)
    ```

**What comes back** — an SSE stream: `research` events narrating the search (which pages were found, read or
blocked, and why), the answer tokens, and the `final` event with the turn's verdict. The answer is grounded in
the safe pages, which are passed as context, so `halluc_context` measures faithfulness to them.

!!! info "A search that finds nothing usable does not become a refusal"
    If the engine is rate-limited, every page fails to fetch, or every page is blocked by the firewall,
    the proxy tells the model to answer from its own knowledge and append a one-line note that live
    results were unavailable — rather than emitting a flat *"I cannot search the web"*.

---

## Settings API

**What it does.** Two routes on **G1-Proxy** that read and set the search provider's API key out-of-band. The key is written to a file outside the image with mode `0600` and is **never returned in clear** — only a masked hint.

=== "curl"

    ```bash
    # read
    curl -s http://localhost:8080/gw/v1/glad/websearch/config | jq

    # set (or replace)
    curl -s -X POST http://localhost:8080/gw/v1/glad/websearch/config \
      -H "Content-Type: application/json" \
      -d '{"api_key": "tvly-abc…wxyz"}' | jq

    # remove
    curl -s -X POST http://localhost:8080/gw/v1/glad/websearch/config \
      -H "Content-Type: application/json" -d '{"api_key": ""}' | jq
    ```

=== "Python"

    ```python
    import httpx

    c = httpx.Client(base_url="http://localhost:8080/gw", timeout=30)

    cfg = c.get("/v1/glad/websearch/config").json()
    print(cfg["provider"], cfg["key_source"], cfg["key_hint"])

    if cfg["env_locked"]:
        raise SystemExit("key is pinned by the environment — change it there, not here")

    r = c.post("/v1/glad/websearch/config", json={"api_key": "tvly-abc…wxyz"})
    if r.status_code == 409:
        print(r.json()["error"])
    else:
        print("stored:", r.json()["key_hint"])
    ```

=== "TypeScript"

    ```ts
    const BASE = "http://localhost:8080/gw"

    const cfg = await fetch(`${BASE}/v1/glad/websearch/config`).then(r => r.json())
    if (cfg.env_locked) throw new Error("key is pinned by the environment")

    const res = await fetch(`${BASE}/v1/glad/websearch/config`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ api_key: "tvly-abc…wxyz" }),
    })
    if (res.status === 409) console.warn((await res.json()).error)
    ```

**What comes back**

```json
{
  "enabled": true,
  "provider": "tavily",
  "has_key": true,
  "key_hint": "tvly-ab…wxyz",
  "key_source": "file",
  "env_locked": false
}
```

| Field | Description |
|---|---|
| `enabled` | Master switch (`GW_WEBSEARCH_ENABLED`). `false` → the per-request `web_search` flag is a no-op. |
| `provider` | The engine that will actually be used: `tavily` when a key is present, else `duckduckgo`. |
| `has_key` | Whether any key is configured, from any source. |
| `key_hint` | A masked fragment, enough to tell two keys apart. Never the key. |
| `key_source` | `env` · `file` · `none`. |
| `env_locked` | `true` when the key comes from the environment. |

### Routes

| Method · Path | Body | Returns |
|---|---|---|
| `GET /v1/glad/websearch/config` | — | The object above. |
| `POST /v1/glad/websearch/config` | `{"api_key": "tvly-…"}` sets or replaces; `{"api_key": ""}` removes | `{ok: true, has_key, key_hint, provider}` |

!!! warning "409 when the environment owns the key"
    If `GW_WEBSEARCH_API_KEY` (or `TAVILY_API_KEY`) is set at deploy time, it always wins, and `POST` refuses with **409** and `{"ok": false, "error": "…"}` rather than silently doing nothing. Change it where it is set. A filesystem failure returns **500** with the same `{ok, error}` shape — branch on `ok`, not only on the status code.

---

## How it works

![Diagram](../assets/diagrams/gateway-web-search.svg){: .diagram }

1. **Search** — the gateway queries a search provider (see below).
2. **Fetch** — each result page is downloaded and its readable text extracted (scripts/nav/boilerplate stripped).
3. **Screen** — each page is scored on the firewall axes. A page is **blocked** if it trips `rag_jailbreak`, `answer_safety`, `prompt_safety`, or `jailbreak`.
4. **Ground** — the surviving safe pages become the grounding context (verified again by the normal `halluc_context` pass), and the model answers from them with `Source 1`, `Source 2`, … citations.
5. **Stream events** — in the chat UI you see, in real time, which pages were *read* vs *blocked* and **why**.

---

## Search providers

| Provider | Key needed | When used | Notes |
|---|---|---|---|
| **Tavily** *(recommended)* | yes | when a Tavily API key is configured | Reliable, rate-limit-free, returns clean extracted page content + an optional synthesised answer that nails factual lookups (prices, dates, names). |
| **DuckDuckGo** *(fallback)* | no | when no key is set, or Tavily errors | Free and key-less, but the public HTML endpoint is throttled and can intermittently return nothing. Fine for demos; not for production load. |

The provider is chosen automatically: **if a Tavily key is present, Tavily is used; otherwise DuckDuckGo.** If a Tavily call fails for any reason, the gateway transparently falls back to DuckDuckGo rather than failing the whole feature.

---

## Setting it up with a Tavily API key

### 1. Get a Tavily key

Create a free account at **[tavily.com](https://tavily.com)** and copy your API key — it looks like `tvly-xxxxxxxxxxxxxxxxxxxx`. The free tier is enough for evaluation; paid tiers raise the monthly search quota.

### 2. Provide the key to the gateway

There are **three** ways to configure the key. They are checked in this precedence order:

=== "A. From the UI (recommended)"

    In **G-1 Studio → Settings → Web search**, paste the key into the **Tavily API key** field and click **Save**.

    - The key is stored **out-of-band on the server** (a `0600` file, never in the image, env, git, or the API response).
    - After saving, the panel shows a masked hint (e.g. `tvly-ab…wxyz`) and a **Premium key set** badge. You can **Remove** it at any time.
    - This is the right path for a running container or a customer install — no restart needed.

    !!! note
        If the key was set via the environment at deploy time, the UI field is **locked** and shows *"configured via the server environment and cannot be changed here"* — the env var always wins (see option B).

=== "B. Environment variable"

    Set the key when starting the gateway. This is best for automated / IaC deployments and **takes precedence over a UI-set key**:

    ```bash
    GW_WEBSEARCH_API_KEY=tvly-xxxxxxxxxxxxxxxxxxxx \
      python -m glad_minimal.gateway.geodesia_gateway --host 0.0.0.0 --port 8800 ...
    ```

    `TAVILY_API_KEY` is accepted as an alias. In Docker Compose, add it to your `.env` file:

    ```dotenv
    GW_WEBSEARCH_API_KEY=tvly-xxxxxxxxxxxxxxxxxxxx
    ```

=== "C. Key file"

    Drop the key into a file the gateway reads at request time. The default path lives under the container's writable `var/` dir; override it with `GW_WEBSEARCH_KEY_FILE`:

    ```bash
    mkdir -p /app/var
    printf '%s' 'tvly-xxxxxxxxxxxxxxxxxxxx' > /app/var/websearch_tavily.key
    chmod 600 /app/var/websearch_tavily.key
    ```

    This is what the UI writes under the hood, and it survives restarts as long as the `var/` volume is persisted.

### 3. (Optional) tune the provider

Force a provider or adjust limits with the [environment variables](#environment-variables) below. Leaving everything unset gives the sensible defaults (Tavily when a key exists, 5 results, screen 6 pages).

### 4. Confirm it is live

```bash
curl -s http://localhost:8080/gw/v1/glad/websearch/config
# {"enabled": true, "provider": "tavily", "has_key": true, "key_hint": "tvly-ab…wxyz", "key_source": "file", "env_locked": false}
```

Then send a request with `web_search: true` — see [Use it](#use-it).

---

## Streaming research events

The gateway emits research events before the answer so the UI can narrate the search.
Each one is an SSE chunk with an empty delta whose `geodesia` object has `event: "research"` and carries one
progress event in `geodesia.research`:

```text
data: {"id":"chatcmpl-geodesia-ws-…","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":null}],
       "geodesia":{"schema_version":"1.0","event":"research",
                   "research":{"type":"search_started","query":"…","provider":"tavily"}}}
```

`research.type` is one of:

| `type` | Meaning |
|---|---|
| `search_started` | The query was sent; carries the `query` and the `provider` actually used. |
| `page_found` | A result `url` / `title` was returned by the search. |
| `page_read` | The page passed the firewall and is grounding the answer. Carries the page's per-axis screening scores (`axes`), an `excerpt`, and `snippet_only: true` when the page could not be fetched and the search snippet was used instead. |
| `page_blocked` | The page tripped a firewall axis. Carries `axes`, `axis` (the deciding axis: the one furthest over its threshold) and a human-readable `reason`. |
| `page_skipped` | The page could not be fetched (anti-bot 403, timeout, non-HTML); carries a `reason`. |
| `search_done` | Summary counts: `found`, `read`, `blocked`. |
| `search_error` | The search itself failed; carries an `error` message. The answer is still generated. |

The per-page `axes` inside a research event are the page-screening scores, one entry per firewall axis
(`rag_jailbreak`, `answer_safety`, `prompt_safety`, `jailbreak`) as `{"score": …, "threshold": …, "flagged": …}`.
They describe the page, not the turn: they are not the turn's `geodesia.axes`.

```json
"research": {
  "type": "page_blocked", "url": "https://…", "title": "…",
  "axes": {
    "rag_jailbreak": { "score": 0.9984, "threshold": 0.5768, "flagged": true },
    "answer_safety": { "score": 0.0421, "threshold": 0.7953, "flagged": false },
    "prompt_safety": { "score": 0.1033, "threshold": 0.6377, "flagged": false },
    "jailbreak":     { "score": 0.2210, "threshold": 0.9864, "flagged": false }
  },
  "axis": "rag_jailbreak",
  "reason": "Prompt-injection / manipulation hidden in the page"
}
```

Research events are informational. The turn's verdict arrives, as always, only in the last chunk, the one with
`event: "final"` (see [Streaming](../reference/response-format.md#streaming)):

```python
for chunk in stream:                      # parsed SSE chunks
    g = chunk.get("geodesia")
    if not g:
        continue
    if g["event"] == "research":
        ev = g["research"]
        print(ev["type"], ev.get("url", ""))
    elif g["event"] == "final":
        print("decision:", g["decision"])
```

If the search returns nothing usable (engine rate-limited, all pages unfetchable, or all blocked), the gateway instructs the model to answer from its own knowledge and append a one-line note that live results weren't available — rather than emitting a flat *"I cannot search the web"* refusal.

---

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `GW_WEBSEARCH_ENABLED` | `1` | Master switch. `1` = available; `0` = the `web_search` flag is a no-op. |
| `GW_WEBSEARCH_API_KEY` | — | Tavily API key. **Wins over a UI/file-set key.** `TAVILY_API_KEY` is an accepted alias. |
| `GW_WEBSEARCH_KEY_FILE` | `/app/var/websearch_tavily.key` | Path to the out-of-band key file (what the UI writes). |
| `GW_WEBSEARCH_PROVIDER` | auto | Force `tavily` or `duckduckgo`. Default: `tavily` if a key is present, else `duckduckgo`. |
| `GW_WEBSEARCH_MAX_RESULTS` | `5` | Number of search results requested. |
| `GW_WEBSEARCH_MAX_READ` | `6` | Maximum number of safe pages used as grounding context. |
| `GW_WEBSEARCH_TIMEOUT` | `12` | Per-request HTTP timeout (seconds) for search + page fetches. |
| `GW_WEBSEARCH_SCREEN_CHARS` | `4000` | Chars of each page text screened by the firewall (head, where injections sit). |
| `GW_WEBSEARCH_GROUND_CHARS` | `2200` | Chars of each safe page kept as grounding context. |
| `GW_WEBSEARCH_RJ_THR` | calibrated | Override the `rag_jailbreak` firewall threshold for page screening. Leave unset to use the model's per-axis calibrated threshold. |

!!! tip "Self-contained — no new dependencies"
    Web search uses only `httpx` + `bs4`, already shipped with the gateway. Tavily is reached over HTTPS; DuckDuckGo needs no key. The Tavily key is the **only** secret involved and is never persisted into the image.
