# AI Watermark

Every AI-generated response is stamped with a **latent HMAC-SHA256 watermark** recorded in a log, so
later you can answer "did this deployment write this text?" — and detect whether it was edited since.
One endpoint does the checking, in three modes depending on what you still have: the text, the
identifiers, or both.

---

## Endpoints

### POST /v1/glad/watermark/verify

**What it does.** Answers "did this deployment generate this text?" One endpoint, **three modes**, chosen automatically by what you send:

| You send | Mode | What it proves |
|---|---|---|
| `response_text` (or `text`) **+** `call_id` **+** `watermark_id` | **live** | The watermark matches *this exact text*. Detects tampering. |
| `call_id` **+** `watermark_id`, no text | **lookup** | The watermark is registered for that call. Does **not** check text integrity — suitable for a third-party auditor who has the identifiers but not the content. |
| `response_text` (or `text`) alone | **search** | Scans the watermark log for an entry matching the text. This is the paste-a-snippet path. |

Sending none of those returns **400**.

---

=== "curl"

    ```bash
    # live — full integrity check
    curl -s -X POST http://localhost:8080/v1/glad/watermark/verify \
      -H "Content-Type: application/json" \
      -d '{
        "call_id":       "call_abc123",
        "watermark_id":  "a8b3c1d4e5f6…",
        "response_text": "The capital of France is Paris."
      }' | jq

    # search — just paste the text
    curl -s -X POST http://localhost:8080/v1/glad/watermark/verify \
      -H "Content-Type: application/json" \
      -d '{"text": "The capital of France is Paris."}' | jq '{valid, call_id, match_mode}'
    ```

=== "Python"

    ```python
    import httpx

    def verify(text: str, call_id: str | None = None, watermark_id: str | None = None):
        body = {"response_text": text}
        if call_id and watermark_id:
            body |= {"call_id": call_id, "watermark_id": watermark_id}
        r = httpx.post("http://localhost:8080/v1/glad/watermark/verify", json=body, timeout=60)
        r.raise_for_status()                      # 400 when the body identifies nothing
        return r.json()

    res = verify("The capital of France is Paris.", "call_abc123", "a8b3c1d4e5f6…")
    print(res["mode"], res["valid"])
    print(res["message"])
    ```

=== "TypeScript"

    ```ts
    async function verify(text: string, callId?: string, watermarkId?: string) {
      const body: Record<string, string> = { response_text: text }
      if (callId && watermarkId) { body.call_id = callId; body.watermark_id = watermarkId }
      const r = await fetch("http://localhost:8080/v1/glad/watermark/verify", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(body),
      })
      if (!r.ok) throw new Error(await r.text())
      return r.json()
    }

    const res = await verify("The capital of France is Paris.", "call_abc123", "a8b3c1d4e5f6…")
    console.log(res.mode, res.valid, res.message)
    ```

### What comes back

```json
{
  "verification_id": "VRF-3F9A1C7B2E5D8046A1B2",
  "valid": true,
  "call_id": "call_abc123",
  "watermark_id": "a8b3c1d4e5f6…",
  "method": "hmac_sha256_v1",
  "mode": "live",
  "timestamp": "2026-06-10T10:23:45Z",
  "verified_at": "2026-06-18T09:00:00+00:00",
  "applicable_laws": ["EU_AI_ACT", "CA_SB_942"],
  "lang": "en",
  "message": "Verification successful. The watermark_id matches the response text. …",
  "provider": "Geodesia S.R.L.",
  "product_version": "…"
}
```

| Field | Description |
|---|---|
| `valid` | Whether the check passed. |
| `mode` | Which of the three paths ran: `live`, `lookup` or the search path. |
| `verification_id` | A `VRF-…` id for this verification act, so the check itself can be cited. |
| `timestamp` | When the watermarked call was generated (`null` if the log entry is gone). |
| `verified_at` | When this verification ran. |
| `applicable_laws` | The frameworks the original call was recorded under. |
| `message` | A human-readable verdict, suitable for showing to a non-technical auditor. |
| `provider` / `product_version` | Who issued the watermark and with which build. |

### Request fields

| Field | Type | Description |
|---|---|---|
| `response_text` | `string` | The text to verify. |
| `text` | `string` | Alias for `response_text`. |
| `call_id` | `string` | The call the text came from. |
| `watermark_id` | `string` | The HMAC recorded for that call. |
| `session_id` | `string` | Improves the live check when the original call belonged to a session. |

!!! tip "Search mode survives copy-paste"
    A strict HMAC is computed over the exact bytes, so text copied out of a rendered chat — with end-of-turn markers and thinking blocks stripped — will not match. Search mode tries the exact HMAC first and then falls back to a normalised comparison against the stored response, which is what makes "paste the paragraph you received" actually work.

!!! warning "A failed verification is not proof of forgery"
    `valid: false` means *this deployment's log does not vouch for this text as given*. The text may have been edited, or the identifiers may be wrong, or it may have come from a different deployment. Read `message` before drawing a conclusion.

---

## What Is an AI Watermark?

An AI watermark is a signal that allows the origin of a text produced by a language model to be verified. Geodesia's
watermark is **latent**: nothing is added to the text and nothing is added to the API response. It is a
cryptographic record kept by the deployment, which can later confirm — through the verification endpoint — that a
given text was produced by this deployment and has not been edited since.

It supports the marking and detectability requirements of:

- **EU AI Act Article 50** — AI-generated content must be marked in a way that is detectable
- **California SB 942** — AI-generated content must carry detectable disclosure
- **Italy 132/2025** — AI content marking requirements

---

## How It Works

For **every answer the gateway returns** (G1-Proxy and G1-Proxy Light, streaming or not), after the response has
been sent, the gateway:

1. Computes `watermark_id = HMAC-SHA256(key, call_id | session_id | response_text)` (method `hmac_sha256_v1`) with a
   deployment-side key.
2. Writes it to the deployment's **watermark log** together with the `call_id`, a timestamp, the manifest label
   (`AI-GENERATED CONTENT`), the applicable laws and the answer text.
3. Links the call into the [audit chain](audit-chain.md).

Prompts that were withheld (blocked before any answer was generated) have no answer and are not watermarked.

To verify, send the text — and, if you have them, the `call_id` and `watermark_id` from the audit record — to
[`POST /v1/glad/watermark/verify`](#endpoints). The verifier recomputes the HMAC over the text you supplied and
compares it with the logged value.

!!! info "The watermark is not in the API response"
    The chat response carries **no** watermark field: the `geodesia` object ([schema 1.0](../reference/response-format.md))
    has no `watermark` key and the answer text is not altered. The record lives server-side. To find the
    `call_id` / `watermark_id` of an answer, open it in **Audit & Chain** in G-1 Studio, or simply paste the text
    into **Verify watermark** there (the *search* mode above). If your application must show an explicit
    "AI-generated" notice to end users, display it in your own UI.

---

## Watermark Configuration

Watermark logging on the gateway is always on for returned answers; there is no switch that adds it to the
response. The `watermark` section of [config.yaml](../configuration/index.md#watermark-section) applies to the
product backend.

---

## Regulatory Coverage

| Law | Requirement | Coverage |
|---|---|---|
| EU AI Act Art. 50(1) | Disclose that content is AI-generated | Deployer's responsibility in its UI; each call's manifest label is recorded in the watermark log |
| EU AI Act Art. 50(2) | Detectable machine-readable marker | Latent HMAC-SHA256 token |
| CA SB 942 §22757(a) | Watermark or tag AI-generated content | Latent watermark + logged manifest label |
| Italy 132/2025 Art. 4 | Mark AI-generated content | Latent token + logged manifest label |
| UK DUAA 2025 | Transparency of AI content | Logged manifest label; end-user notice in the deployer's UI |
