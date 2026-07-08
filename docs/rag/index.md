# Knowledge Base (RAG)

Geodesia G-1 includes a built-in **Retrieval-Augmented Generation (RAG)** system that lets you upload documents and have the LLM answer questions grounded in those documents. Every retrieved chunk is passed to the faithfulness detection axis, and the claims in the answer are verified citation-by-citation against the source material.

**Supported document formats:** PDF, Word (.docx), PowerPoint (.pptx), Markdown (.md), HTML, Excel (.xlsx), CSV, plain text (.txt)

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

## API Reference

All RAG endpoints are mounted at `/v1/glad/rag/` on the gateway.

### GET /v1/glad/rag/status

Returns whether the RAG service is loaded and ready.

```bash
curl http://localhost:8800/v1/glad/rag/status
```

```json
{"ok": true, "embed_model": "BAAI/bge-m3", "device": "cuda:0", "store_dir": "runs/rag_store"}
```

---

### POST /v1/glad/rag/collections

Create a new document collection.

**Request:**
```json
{"name": "company-policies", "description": "Internal HR and legal policies"}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | ✅ | Human-readable name for the collection. |
| `description` | `string` | — | Optional description. |

**Response:**
```json
{"collection_id": "c_a3f7b2d1", "name": "company-policies", "created_at": "2026-06-10T12:00:00Z"}
```

---

### GET /v1/glad/rag/collections

List all collections.

```bash
curl http://localhost:8800/v1/glad/rag/collections
```

**Response:**
```json
[
  {
    "collection_id": "c_a3f7b2d1",
    "name": "company-policies",
    "description": "Internal HR and legal policies",
    "document_count": 3,
    "chunk_count": 142,
    "created_at": "2026-06-10T12:00:00Z"
  }
]
```

---

### DELETE /v1/glad/rag/collections/{collection_id}

Delete a collection and all its documents and embeddings.

```bash
curl -X DELETE http://localhost:8800/v1/glad/rag/collections/c_a3f7b2d1
```

---

### POST /v1/glad/rag/collections/{collection_id}/documents

Upload a document to a collection. The document is automatically parsed, chunked, and embedded.

```bash
curl -X POST \
  http://localhost:8800/v1/glad/rag/collections/c_a3f7b2d1/documents \
  -F "file=@/path/to/policy.pdf"
```

**Multipart fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | file | ✅ | The document to upload. Supported: PDF, DOCX, PPTX, MD, HTML, XLSX, CSV, TXT. Maximum 100 MB. |
| `title` | string | — | Optional document title. If omitted, the filename is used. |

**Response:**
```json
{
  "document_id": "doc_b5c2e1a3",
  "title": "policy.pdf",
  "chunk_count": 47,
  "status": "indexed"
}
```

**Parsing notes:**
- PDF and DOCX files are parsed with **Docling** (IBM's multi-format parser), which preserves reading order, headings, and tables better than simple text extraction.
- Large files may take several seconds to process. Embeddings are computed synchronously; the response is returned when indexing is complete.

---

### DELETE /v1/glad/rag/collections/{collection_id}/documents/{document_id}

Delete a single document and cascade-remove its embedded chunks.

```bash
curl -X DELETE \
  http://localhost:8800/v1/glad/rag/collections/c_a3f7b2d1/documents/doc_b5c2e1a3
```

---

### POST /v1/glad/rag/collections/{collection_id}/query

Query a collection directly (without a full chat request). Returns the most relevant chunks for a given question.

**Request:**
```json
{
  "query": "What is the refund window?",
  "top_k": 5,
  "rerank": true
}
```

| Field | Type | Default | Description |
|---|---|---|---|
| `query` | `string` | ✅ | The question or search query. |
| `top_k` | `integer` | `5` | Number of chunks to return (after reranking). |
| `rerank` | `boolean` | `true` | Whether to apply the cross-encoder reranker (BGE-reranker-v2-m3) after initial retrieval. Reranking improves relevance at the cost of an additional model forward pass. |

**Response:**
```json
{
  "chunks": [
    {
      "text": "Our return policy allows refunds within 30 days of purchase...",
      "score": 0.94,
      "document_id": "doc_b5c2e1a3",
      "document_title": "policy.pdf",
      "page": 3,
      "heading": "Return Policy"
    }
  ]
}
```

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
