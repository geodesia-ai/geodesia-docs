# Audit Chain

The Audit Chain is a **tamper-evident, append-only log** that records every significant event in the Geodesia G-1 system. Each log entry is cryptographically linked to the previous one via HMAC-SHA256, making any retroactive modification detectable.

---

## Design

The audit chain is inspired by blockchain-style ledger design, but implemented as a simple hash-linked list stored in the SQLite database:

```
Entry N-1                Entry N                  Entry N+1
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│ event_id     │         │ event_id     │         │ event_id     │
│ timestamp    │         │ timestamp    │         │ timestamp    │
│ event_type   │  HMAC   │ event_type   │  HMAC   │ event_type   │
│ payload      │ ──────► │ payload      │ ──────► │ payload      │
│ prev_hash    │         │ prev_hash    │         │ prev_hash    │
│ entry_hash ──┼─────────┼──► (copied)  │         │ entry_hash   │
└──────────────┘         └──────────────┘         └──────────────┘
```

Each `entry_hash` is computed over: `prev_hash + timestamp + event_type + payload`. If any past entry is modified, the hash chain breaks, and `GET /v1/glad/chain/verify` reports the tampered entry.

---

## What Gets Logged

Every entry in the chain has an `event_type`. The following events are automatically recorded:

| Event Type | When Recorded |
|---|---|
| `inference_call` | Every call through the evaluate endpoint or gateway |
| `prompt_blocked` | When a prompt is blocked by the safety or jailbreak detector |
| `answer_blocked` | When an answer is withheld by the safety or hallucination detector |
| `kill_switch_activated` | When the kill switch is engaged |
| `kill_switch_deactivated` | When the kill switch is disengaged |
| `oversight_review` | When a human reviewer records a decision |
| `oversight_escalation` | When a pending review is escalated |
| `fria_created` | When a FRIA dossier is created |
| `fria_approved` | When a FRIA dossier is approved |
| `model_switched` | When the active model checkpoint is changed |
| `threshold_changed` | When detection thresholds are updated |
| `watermark_verified` | When a watermark is verified |
| `config_changed` | When the gateway configuration is modified |

---

## Endpoints

### GET /v1/glad/chain/status

Returns the current audit chain status.

```bash
curl http://localhost:8199/v1/glad/chain/status
```

#### Response

```json
{
  "total_entries": 14821,
  "latest_entry_id": "chain_abc123",
  "latest_timestamp": "2026-06-10T10:23:45Z",
  "integrity": "ok",
  "first_entry_at": "2026-04-22T09:00:00Z",
  "deployer_id": "acme-corp"
}
```

| Field | Description |
|---|---|
| `total_entries` | Total number of entries in the chain |
| `latest_entry_id` | ID of the most recent entry |
| `integrity` | `"ok"` if the chain is intact; `"tampered"` if any entry fails verification |
| `first_entry_at` | When the chain was first written to |

---

### GET /v1/glad/chain/verify

Performs full cryptographic verification of the audit chain — recomputes every HMAC from the genesis entry and checks that each `entry_hash` and `prev_hash` match.

```bash
curl http://localhost:8199/v1/glad/chain/verify
```

!!! warning "Performance"
    Full chain verification reads and recomputes the HMAC of every entry. For very large deployments (>100K entries), this may take several seconds. Use `depth` to limit the verification window for routine checks.

#### Query Parameters

| Parameter | Default | Description |
|---|---|---|
| `deployer_id` | — | Limit verification to one deployer's entries. |
| `depth` | `full` | Number of entries to verify from the tail, or `"full"` for the entire chain. |

#### Response

```json
{
  "verified": true,
  "entries_checked": 14821,
  "first_tampered_entry": null,
  "duration_ms": 412,
  "chain_hash": "a8b3c1..."
}
```

| Field | Description |
|---|---|
| `verified` | `true` if the chain is intact |
| `entries_checked` | Number of entries that were verified |
| `first_tampered_entry` | If `verified` is `false`, the ID and timestamp of the first tampered entry |
| `chain_hash` | A single hash representing the entire chain state. Share this with auditors for spot-checking without giving access to the full log. |

---

### GET /v1/glad/chain/entries

Retrieve entries from the audit chain. Supports filtering and pagination.

```bash
curl "http://localhost:8199/v1/glad/chain/entries?event_type=prompt_blocked&limit=50"
```

#### Query Parameters

| Parameter | Description |
|---|---|
| `event_type` | Filter to a specific event type |
| `since` | ISO 8601 timestamp — entries after this time |
| `until` | ISO 8601 timestamp — entries before this time |
| `call_id` | Filter to entries for a specific call |
| `session_id` | Filter to entries for a session |
| `deployer_id` | Filter to a specific deployer |
| `limit` | Maximum results (default 100, max 1000) |
| `offset` | Pagination offset |

#### Response Entry Structure

```json
{
  "entry_id": "chain_abc123",
  "timestamp": "2026-06-10T10:23:45Z",
  "event_type": "inference_call",
  "deployer_id": "acme-corp",
  "call_id": "call_xyz",
  "session_id": "sess_xyz",
  "payload": {
    "prompt_blocked": false,
    "answer_blocked": false,
    "trigger_axis": null,
    "scores_summary": {
      "prompt_safety": 0.12,
      "answer_safety": 0.08,
      "halluc_context": 0.23
    }
  },
  "entry_hash": "a8b3c1...",
  "prev_hash": "7f2d4e..."
}
```

---

## Regulatory Mapping

| Requirement | Law | How the Audit Chain Covers It |
|---|---|---|
| Logging and record-keeping | EU AI Act Art. 12 | Every inference call logged with scores, timestamps, and call ID |
| Tamper-proof logs | EU AI Act Art. 12(3) | HMAC-SHA256 hash chain; any modification breaks verification |
| Post-market monitoring | EU AI Act Art. 72 | Call log forms the basis for monitoring trends and anomalies |
| Audit trail | SOC 2 CC7.2 | Append-only log with cryptographic verification |
| Accountability | ISO 42001 §6.1 | Complete event trail for AI system decisions |
| Data subject rights | GDPR Art. 22 | Inference calls include session and call ID for subject access requests |

---

## Exporting the Audit Chain

For regulatory submission, export the full chain or a time-bounded slice:

```bash
curl "http://localhost:8199/v1/glad/chain/entries?since=2026-01-01T00:00:00Z&limit=1000" \
  > audit_chain_q1_2026.json
```

For a complete audit bundle including FRIA, chain snapshot, and call statistics, use [POST /v1/glad/report](reports.md).
