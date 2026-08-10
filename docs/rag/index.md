# Knowledge Base (RAG)

Upload your documents, and the LLM answers from them instead of from memory. Retrieved passages become
the grounding context the faithfulness axis scores against, and the answer's claims are verified
citation by citation — so "the model made it up" becomes a measurable event rather than a suspicion.

**Formats:** PDF, Word, PowerPoint, Markdown, HTML, Excel, CSV, plain text.

---

## API Reference

Every RAG route is mounted at `/v1/glad/rag/` on **G1-Proxy** — reached as `/gw/v1/glad/rag/…` through the unified port on 8080, or directly on `:8800` in a split deployment.

!!! warning "Collections belong to Applications"
    Send **`X-Geodesia-App: <app_id>`** on every call. Operations on a collection owned by a different Application return **404**, not 403 — the API does not leak the existence of another tenant's data. Omit the header (or send `default`) and you get the unscoped, single-tenant view.

---

### The upload flow, end to end

The one thing worth reading before the endpoint list: **uploading is asynchronous**. `POST …/documents` returns **`202 Accepted`** as soon as the bytes are in, and the parse → chunk → embed → index pipeline runs in the background. You poll `/ingest/progress` until it reports `done`.

That is deliberate. Embedding runs at roughly a second per chunk on CPU; a synchronous upload would hold the connection open for minutes and get cut by any proxy with an origin timeout — which surfaces to the user as an unexplained network failure rather than a slow upload.

=== "curl"

    ```bash
    APP="support_bot"
    BASE="http://localhost:8080/gw/v1/glad/rag"

    # 1. create a collection
    CID=$(curl -s -X POST "$BASE/collections" \
      -H "Content-Type: application/json" -H "X-Geodesia-App: $APP" \
      -d '{"name": "company-policies"}' | jq -r .collection_id)

    # 2. upload — returns 202 immediately
    curl -s -X POST "$BASE/collections/$CID/documents" \
      -H "X-Geodesia-App: $APP" \
      -F "file=@/path/to/policy.pdf" -F "name=Refund policy 2026"

    # 3. poll until the background ingest finishes
    until [ "$(curl -s "$BASE/ingest/progress" -H "X-Geodesia-App: $APP" | jq -r .stage)" = "done" ]; do
      curl -s "$BASE/ingest/progress" -H "X-Geodesia-App: $APP" | jq -c '{stage, detail, pct}'
      sleep 2
    done
    ```

=== "Python"

    ```python
    import time, httpx

    c = httpx.Client(base_url="http://localhost:8080/gw/v1/glad/rag",
                     headers={"X-Geodesia-App": "support_bot"}, timeout=120)

    coll = c.post("/collections", json={"name": "company-policies"}).json()
    cid = coll["collection_id"]

    with open("policy.pdf", "rb") as fh:
        r = c.post(f"/collections/{cid}/documents",
                   files={"file": ("policy.pdf", fh, "application/pdf")},
                   data={"name": "Refund policy 2026"})
    assert r.status_code == 202          # accepted, not finished

    while True:
        p = c.get("/ingest/progress").json()
        print(p["stage"], p["detail"], f"{p['pct']}%")
        if p["stage"] == "error":
            raise RuntimeError(p["error"])
        if p["stage"] == "done":
            break
        time.sleep(2)

    hits = c.post(f"/collections/{cid}/query",
                  json={"query": "What is the refund window?", "top_k": 5}).json()
    print(hits["n_sources"], "sources")
    print(hits["context"][:400])
    ```

=== "TypeScript"

    ```ts
    const BASE = "http://localhost:8080/gw/v1/glad/rag"
    const APP = { "X-Geodesia-App": "support_bot" }

    const coll = await fetch(`${BASE}/collections`, {
      method: "POST",
      headers: { ...APP, "Content-Type": "application/json" },
      body: JSON.stringify({ name: "company-policies" }),
    }).then(r => r.json())

    // Multipart: let the browser set the boundary — do NOT force a JSON content-type.
    const form = new FormData()
    form.append("file", file)                    // a File from an <input type="file">
    form.append("name", "Refund policy 2026")

    const up = await fetch(`${BASE}/collections/${coll.collection_id}/documents`, {
      method: "POST", headers: APP, body: form,
    })
    if (up.status !== 202 && !up.ok) throw new Error(await up.text())

    for (;;) {
      const p = await fetch(`${BASE}/ingest/progress`, { headers: APP }).then(r => r.json())
      if (p.stage === "error") throw new Error(p.error)
      if (p.stage === "done") break
      await new Promise(r => setTimeout(r, 2000))
    }

    const hits = await fetch(`${BASE}/collections/${coll.collection_id}/query`, {
      method: "POST",
      headers: { ...APP, "Content-Type": "application/json" },
      body: JSON.stringify({ query: "What is the refund window?", top_k: 5 }),
    }).then(r => r.json())
    console.log(hits.n_sources, hits.context)
    ```

**What comes back from the query**

```json
{
  "context": "Our return policy allows refunds within 30 days of purchase…",
  "sources": [
    {
      "text": "Our return policy allows refunds within 30 days of purchase…",
      "score": 0.94,
      "document_id": "doc_b5c2e1a3",
      "title": "Refund policy 2026",
      "page": 3
    }
  ],
  "n_sources": 4
}
```

`context` is the concatenated passage text, ready to hand straight to `context` on a [chat request](../g1-proxy/chat-api.md). `sources` is the same material itemised, for citations and for showing the user where the answer came from.

---

### Endpoints

#### `GET /v1/glad/rag/status`

Whether the stack is up, which parser is active, and what it accepts.

```json
{
  "ok": true,
  "parser": "docling",
  "supported": [".csv", ".docx", ".html", ".md", ".pdf", ".pptx", ".txt", ".xlsx"],
  "n_collections": 3
}
```

`parser` is `docling` when the full multi-format parser is available and `fallback` when it is not — worth checking, because the fallback preserves reading order and tables far less well. A failure returns `{"ok": false, "error": "…"}` with HTTP 200, so branch on `ok`, not on the status code.

#### `POST /v1/glad/rag/collections`

Body: `{"name": "company-policies"}`. `name` is the only field — it defaults to `"Untitled"`. Returns the created collection.

#### `GET /v1/glad/rag/collections`

Returns `{"collections": [ … ]}` — an object, not a bare array.

#### `GET /v1/glad/rag/collections/{collection_id}`

One collection with its documents. **404** if it does not exist *or* belongs to another Application.

#### `DELETE /v1/glad/rag/collections/{collection_id}`

Deletes the collection, its documents and its embeddings. Returns `{"ok": true}`; **404** if unknown.

#### `POST /v1/glad/rag/collections/{collection_id}/documents`

**Multipart.** Returns **202** and ingests in the background.

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | file | ✅ | The document. Supported extensions are whatever `/status` reports. |
| `name` | string | — | Display title. Falls back to the filename. |

Response: `{"status": "accepted", "file": "policy.pdf"}`.

| Status | Meaning |
|---|---|
| `202` | Accepted; poll `/ingest/progress`. |
| `400` | Empty file. |
| `404` | Unknown collection, or one owned by another Application. |
| `413` | Over the upload cap — 50 MB by default, `GW_RAG_MAX_UPLOAD_BYTES`. |
| `415` | Unsupported file type. |

!!! warning "One ingest at a time"
    Progress is tracked as a single global state, so `/ingest/progress` describes **the most recent upload**, not a specific one. Upload documents sequentially — wait for `done` before starting the next — or you will not be able to tell whose progress you are reading.

#### `GET /v1/glad/rag/ingest/progress`

```json
{ "active": true, "file": "policy.pdf", "stage": "embedding", "detail": "31/47", "pct": 66, "error": null }
```

`stage` runs `parsing` → `chunking` → `embedding` → `indexing` → `done`, or lands on `error` with `error` set. `idle` means nothing has been uploaded yet.

#### `DELETE /v1/glad/rag/collections/{collection_id}/documents/{doc_id}`

Removes the document and cascade-deletes its chunks. `{"ok": true}`, or **404** for an unknown collection or document.

#### `POST /v1/glad/rag/collections/{collection_id}/query`

Retrieve without sending a chat turn — useful for testing a collection and for building your own pipeline.

| Field | Type | Default | Description |
|---|---|---|---|
| `query` | `string` | ✅ | The question. |
| `top_k` | `integer` | *(server default)* | Passages to return after reranking. |
| `rerank` | `boolean` | *(server default)* | Cross-encoder rerank after retrieval. Better relevance, one extra forward pass. |

Omitting `top_k` or `rerank` uses the server's configured defaults rather than a fixed number — send them explicitly if you need determinism.

---

## Using RAG in Chat Requests

To use a knowledge base in a chat request, add the `rag` field:

```json
{
  "model": "my-model",
  "stream": false,
  "messages": [{"role": "user", "content": "What is our refund window?"}],
  "rag": {
    "collection_id": "c_a3f7b2d1",
    "top_k": 5,
    "rerank": true,
    "verify": true,
    "verify_deep": true
  }
}
```

### RAG Chat Request Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `collection_id` | `string` | ✅ | ID of the collection to retrieve from. |
| `top_k` | `integer` | `5` | Maximum chunks to retrieve and inject into the prompt. |
| `rerank` | `boolean` | `true` | Apply the cross-encoder reranker. Slightly slower but significantly more accurate for ambiguous queries. |
| `verify` | `boolean` | `true` | Run claim-level grounding verification after the answer is generated. |
| `verify_deep` | `boolean` | `true` | When `true`, verification uses the hallucination detection model for each claim (more accurate). When `false`, falls back to lexical overlap (faster, less accurate). |

### RAG in the Response

When RAG is active, the `geodesia.rag` field in the response contains retrieval and verification details:

```json
"geodesia": {
  "rag": {
    "collection_id": "c_a3f7b2d1",
    "n_sources": 3,
    "sources": [
      {
        "text": "Our return policy allows refunds within 30 days...",
        "score": 0.94,
        "document_title": "policy.pdf",
        "page": 3
      }
    ],
    "verification": {
      "n_total": 2,
      "n_grounded": 2,
      "ungrounded": false,
      "claims": [
        {
          "claim": "refunds within 30 days",
          "grounded": true,
          "citation": "Our return policy allows refunds within 30 days..."
        }
      ]
    }
  },
  "brake": false
}
```

| Field | Description |
|---|---|
| `n_sources` | Number of chunks retrieved |
| `sources` | List of retrieved chunks with text, relevance score, and document metadata |
| `verification.n_total` | Total claims extracted from the answer |
| `verification.n_grounded` | Claims supported by the retrieved chunks |
| `verification.ungrounded` | `false` when all claims are grounded — triggers hallucination suppression |
| `verification.claims` | Per-claim grounding status and the matching citation |

---

## How It Works

```
1. You upload a document → Docling parses it → chunked into ~480 tokens with 64-token overlap
2. Each chunk is embedded with BGE-M3 (multilingual) → stored in LanceDB
3. On a RAG-enabled chat request:
   a. Retrieve top-K chunks most relevant to the user's question (dense retrieval + reranking)
   b. Inject the retrieved context into the upstream LLM's prompt
   c. The LLM answers using the context
   d. Geodesia verifies each claim in the answer against the retrieved chunks
   e. If all claims are verified with citations → halluc_context flag suppressed
   f. If any claim is ungrounded → halluc_context flags normally
```

---

## Collections

Documents are organised into **collections**. A collection is a named group of documents that shares an embedding index. You can have multiple collections for different topics or customers.

---

## Configuration

RAG-specific environment variables:

| Variable | Default | Description |
|---|---|---|
| `GW_RAG_DIR` | `runs/rag_store` | Directory where the LanceDB embedding store is saved. Must be writable. |
| `GW_RAG_DEVICE` | `cuda:0` | Device for the embedding model. Use `cpu` on machines where the GPU is fully occupied by the LLM. |
| `GW_RAG_EMBED_MODEL` | `BAAI/bge-m3` | Hugging Face model ID for the text embedding model. BGE-M3 is multilingual and recommended. |
| `GW_RAG_RERANK` | `1` | Set to `0` to disable the reranker globally. |
| `GW_RAG_RERANK_MODEL` | `BAAI/bge-reranker-v2-m3` | Hugging Face model ID for the cross-encoder reranker. |
| `GW_RAG_TOPK` | `5` | Default number of chunks to retrieve (overridable per-request). |
| `GW_RAG_OVERFETCH` | `20` | Number of candidates retrieved by the dense retriever before reranking. Higher = more recall at the cost of reranker speed. |
| `GW_RAG_CTX_MAXCHARS` | `6000` | Maximum characters of retrieved context injected into the prompt. Long contexts are truncated. |
| `GW_RAG_MAX_CLAIMS` | `12` | Maximum number of claims extracted from the answer for claim-level verification. |

!!! tip "GPU allocation"
    If your GPU is fully occupied by the LLM, set `GW_RAG_DEVICE=cpu`. BGE-M3 on CPU is slower for large uploads (~10–30 seconds per document) but runs fine. After the initial indexing, retrieval from CPU is typically fast enough for real-time use.
