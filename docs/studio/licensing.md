# Licensing & Entitlements

Geodesia G-1 ships with everything unlocked enough to evaluate end-to-end, and gates the rest behind a **vendor-signed license**. There is no license server to call and no phone-home: a license is a small signed file that the companion verifies **offline** against an Ed25519 public key baked into the build. This page describes the tiers, what is metered, the license file format, and how you apply, clear, and inspect a license.

!!! note "Enforced locally, verified offline"
    The entitlement module runs inside the gateway and is **AES-protected in distribution**, so the limits cannot be trivially patched out. Verification uses a baked-in Ed25519 **public** key — no network, no license server. This is **air-gap friendly**: an installation that never touches the internet still enforces (and can be upgraded by dropping in a new license file).

---

## Tiers

There are two tiers. With no license present you are on **FREE**; dropping in a valid vendor-signed license moves you to **LICENSED** with whatever limits the license declares.

| | **FREE** (no license) | **LICENSED** (vendor-signed file) |
|---|---|---|
| Chats per day | **20** | as declared (`max_chats_per_day`, may be unlimited) |
| Upstream models | **1** | as declared (`max_models`, may be unlimited) |
| Applications | **1** | as declared (`max_applications`, may be unlimited) |
| Allowed models | any (`["*"]`) | as declared (`allowed_models`, `["*"]` = any) |
| Expiry | none | optional (`expires`, ISO timestamp) |
| Verification | — | Ed25519 signature, fully offline |

!!! info "Silent fall-back to FREE"
    An **absent, invalid, or expired** license does not error at startup — the companion silently falls back to the FREE tier. An expired license is treated exactly like no license (free limits), while still reporting its `license_id` for diagnostics.

In a license, a `null` value for `max_chats_per_day`, `max_models`, or `max_applications` means **unlimited** for that dimension.

---

## What is metered

Three things are tracked locally and checked on every chat:

1. **Daily chat count** — bucketed by **UTC calendar day**. The counter is persisted to a small JSON file (default `var/usage.json`, override with `GLAD_USAGE_FILE`), so it **survives restarts** and resets automatically at the next UTC day. When the day rolls over the daily counter resets to zero.
2. **Distinct models used** — every distinct upstream model name you route to is recorded. Once the number of distinct models reaches `max_models`, a request for a *new* model is refused (a model already counted still works). Models outside `allowed_models` (when it is not `["*"]`) are refused regardless of the count.
3. **Number of Applications** — the count of Applications in the control plane is checked against `max_applications`. This **includes the seeded `default` Application**, so on the free tier (`max_applications = 1`) the `default` app already consumes the single slot.

### Reading the active plan and usage

Query the gateway for the current entitlement plus today's usage. This is **read-only** — it never increments the chat counter.

```bash
curl -s http://localhost:8800/gw/v1/glad/gateway/entitlements | python3 -m json.tool
```

**Response shape** (from `entitlements.status()`):

```json
{
  "tier": "free",
  "customer": "",
  "source": "free",
  "license_id": null,
  "expires": null,
  "max_chats_per_day": 20,
  "chats_used_today": 3,
  "chats_remaining_today": 17,
  "max_models": 1,
  "models_used": 1,
  "max_applications": 1,
  "models_today": ["granite4.1:3b"],
  "allowed_models": ["*"]
}
```

| Field | Meaning |
|---|---|
| `tier` | `free`, or the tier name declared in the license (e.g. `pro`). |
| `customer` | Customer string from the license (empty on FREE). |
| `source` | `free`, `license`, or `license_expired`. Shows why these limits are in force. |
| `license_id` | The installed license's ID (`null` on FREE). |
| `expires` | License expiry (ISO timestamp) or `null`. |
| `max_chats_per_day` | Daily chat cap. `null` = unlimited. |
| `chats_used_today` | Chats counted so far in the current UTC day. |
| `chats_remaining_today` | Remaining for the day, or `null` when unlimited. |
| `max_models` | Distinct-model cap. `null` = unlimited. |
| `models_used` | Distinct models seen so far (lifetime, against the cap). |
| `max_applications` | Application cap. `null` = unlimited. |
| `models_today` | Distinct model names routed to today. |
| `allowed_models` | Model allow-list, `["*"]` = any. |

!!! tip "What happens when a limit is hit"
    A blocked chat comes back with a plain-language reason, for example: *"daily limit reached (20/20 chats today on the free tier). Add a license token for more."* or *"model limit reached (free: 1 model(s)). Upgrade your license to use more models."* The model allow/cap check runs **before** a chat is counted, so a refused model never consumes your daily allowance.

---

## License file format

A license is a JSON document with a `payload` and an Ed25519 `sig` over the canonical (sorted-key, compact) JSON of that payload:

```json
{
  "payload": {
    "license_id": "lic_abc",
    "customer": "Acme Spa",
    "tier": "pro",
    "max_chats_per_day": 5000,
    "max_models": 10,
    "max_applications": 25,
    "allowed_models": ["*"],
    "expires": "2027-06-18T00:00:00Z"
  },
  "sig": "<base64-encoded Ed25519 signature>"
}
```

| Payload field | Meaning |
|---|---|
| `tier` | Display name of the plan (defaults to `licensed` if absent). |
| `customer` | Customer / organization name. |
| `max_chats_per_day` | Daily chat cap. `null` = unlimited. |
| `max_models` | Distinct-model cap. `null` = unlimited. |
| `max_applications` | Application cap. `null` = unlimited. |
| `allowed_models` | List of permitted model names, or `["*"]` for any. |
| `expires` | Optional ISO timestamp. Past this instant the license is treated as expired (→ FREE). |
| `license_id` | Stable identifier for this license. |

The companion stores the installed license at `license.json` next to the install root (override the path with `GLAD_LICENSE_FILE`). On every refresh it re-reads the file, re-verifies the signature against the baked-in public key, and checks `expires`. **Any** failure — missing file, malformed JSON, bad signature, past expiry — falls back to FREE.

---

## Applying and clearing a license

You can apply a license either from the UI or directly against the gateway API. Either way the license string may be the **raw JSON document** shown above, or its **base64 encoding** (the convenient paste-able "license key").

### From the UI

Open **Settings → Plan & License**, paste the license key, and apply. The page then reflects the new tier and limits.

### From the API

Apply a license:

```bash
curl -s -X POST http://localhost:8800/gw/v1/glad/gateway/license \
  -H "Content-Type: application/json" \
  -d '{"license": "<raw-json-or-base64>"}'
```

The request body accepts the license under any of the keys `license`, `token`, or `key`. On success the endpoint returns the new `status()` plan; an invalid or expired signature returns **HTTP 400** with the reason in `detail`.

!!! note "The signature is the authorization"
    The license endpoints require no admin token — the **Ed25519 signature is the authorization**. An unsigned or tampered license is rejected outright, so there is nothing to gain by POSTing one. Only a license signed with the vendor's private key validates.

Clear the installed license (return to FREE):

```bash
curl -s -X DELETE http://localhost:8800/gw/v1/glad/gateway/license
```

This removes `license.json` and returns the (now free-tier) plan.

---

## Environment overrides for evaluation and self-hosting

For local testing, server deployments, and multi-Application self-hosting, you can lift the FREE-tier limits with environment variables. Each takes an **integer**, or one of `0` / `unlimited` / `none` to mean **unlimited**. Unset leaves the product default in place.

| Env var | Lifts | Default (unset) |
|---|---|---|
| `GLAD_FREE_MAX_CHATS_PER_DAY` | FREE daily chat cap | `20` |
| `GLAD_FREE_MAX_MODELS` | FREE distinct-model cap | `1` |
| `GLAD_FREE_MAX_APPLICATIONS` | FREE Application cap | `1` |

```bash
# Self-hosting: unlimited chats, several models, many Applications on the free tier
GLAD_FREE_MAX_CHATS_PER_DAY=unlimited \
GLAD_FREE_MAX_MODELS=0 \
GLAD_FREE_MAX_APPLICATIONS=50 \
  <start the gateway>
```

!!! warning "Overrides only lift the FREE tier"
    These variables affect the **FREE tier only**. If a valid signed license is installed it takes precedence and these variables are ignored — the license's declared limits win. Use the overrides for evaluation or for self-hosted deployments where you want more than one Application without issuing a license.

---

## How the Application cap manifests

The Application cap is a **global** entitlement check, enforced on Application creation across **all** Organizations (it is not per-org — it counts every Application, including the seeded `default`).

On the free tier (`max_applications = 1`), the `default` Application already fills the single slot, so:

- The control plane refuses creating a second Application with **HTTP 403**:

  ```text
  application limit reached (free tier: 1 application(s)). Add a license token to create more.
  ```

- In the UI, the **New Application** action is **disabled** once the `default` Application exists. You can still fully configure, key, and operate that one Application.

Lifting the cap is a licensing change (a license with a higher or unlimited `max_applications`), or — for self-hosting — `GLAD_FREE_MAX_APPLICATIONS`. See [Applications](applications.md) and the [Control-Plane API](control-plane-api.md) for the create routes and RBAC, and the [G-1 Studio overview](index.md) for how Applications and Organizations fit together.

---

## How licenses are issued

Licenses are issued by the **vendor** and signed with an Ed25519 **private** key that **never ships** with the product — only the matching public key is baked into the build. The vendor signs a license with `distrib/common/sign_license.py`, which emits both the `license.json` document and a base64 paste-able key. Because verification is purely cryptographic and offline, a customer can be upgraded simply by being handed a new signed license file — no server, no reinstall, no network.

!!! info "See also"
    - [Applications](applications.md) — the per-app config, keys, and the `max_applications` cap.
    - [Control-Plane API](control-plane-api.md) — the `/v1/glad/apps/*` surface and RBAC.
    - [G-1 Studio overview](index.md) — Organizations, the control plane, and the data plane.
