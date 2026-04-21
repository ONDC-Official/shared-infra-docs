# Gateway

Documentation and reference materials for the ONDC Gateway service.

---

## Contents

| File | Description |
| --- | --- |
| [Gateway_ErrorCodes.md](./Gateway_ErrorCodes.md) | Complete list of error codes returned by the Gateway, grouped by category (Auth, Digest, Context, Schema, Payload, Participant, Routing, Policy, Dispatch, Receiver, Infra) with descriptions and resolutions |
| [Gateway_Troubleshooting.md](./Gateway_Troubleshooting.md) | Troubleshooting guide covering common failure scenarios and step-by-step resolutions for NP integration |

---

## Gateway Overview

The ONDC Gateway is a shared infrastructure service that routes Beckn protocol messages between Network Participants (BAPs and BPPs) on the ONDC network. It validates authentication, schema compliance, participant registration, and applicable policies before routing each request.

---

## API Endpoints

All endpoints accept **POST** requests with `Content-Type: application/json`.

### Request Actions (BAP → Network)

These are initiated by Buyer Applications (BAPs) and routed to eligible Seller Applications (BPPs).

| Endpoint | Description |
| --- | --- |
| `POST /search` | Discover products or services. Broadcast to eligible BPPs (async). |
| `POST /select` | Select items from a provider's catalog. |
| `POST /init` | Initialise an order. |
| `POST /confirm` | Confirm an order. |
| `POST /status` | Check order status. |
| `POST /track` | Track a fulfillment. |
| `POST /cancel` | Cancel an order. |
| `POST /update` | Update an order. |
| `POST /support` | Request support for an order. |
| `POST /rating` | Submit a rating. |

### Response Actions (BPP → Network)

These are initiated by Seller Applications (BPPs) as callbacks in response to request actions.

| Endpoint | Description |
| --- | --- |
| `POST /on_search` | Respond to a search request with catalog data. |
| `POST /on_select` | Respond to a select request with pricing/availability. |
| `POST /on_init` | Respond to an init request with order draft. |
| `POST /on_confirm` | Respond to a confirm request with order confirmation. |
| `POST /on_status` | Respond to a status request with order status. |
| `POST /on_track` | Respond to a track request with tracking details. |
| `POST /on_cancel` | Respond to a cancel request with cancellation status. |
| `POST /on_update` | Respond to an update request with updated order. |
| `POST /on_support` | Respond to a support request with support details. |
| `POST /on_rating` | Respond to a rating request with acknowledgement. |

### IGM Actions (Issue & Grievance Management)

| Endpoint | Description |
| --- | --- |
| `POST /issue` | Raise an issue against an order. |
| `POST /issue_status` | Check the status of a raised issue. |
| `POST /on_issue` | Respond to an issue with resolution details. |
| `POST /on_issue_status` | Respond to an issue status request. |

### Health Check Endpoints

These endpoints are used by load balancers and Kubernetes probes. They do **not** require authentication.

| Endpoint | Method | Description |
| --- | --- | --- |
| `/health` | GET | Readiness check — returns JSON `{"status": "ready"}` (200) or `{"status": "unready", "reason": "..."}` (503). Checks L1 cache, sync dispatch, schema validator, and message broker health. |
| `/health` | HEAD | Same readiness check — returns status code only (200 or 503), no body. |
| `/live` | HEAD | Liveness probe — always returns 200. Confirms the process is running. |

---

## Authentication

All action endpoints (24 endpoints listed above) require HTTP Signature authentication using **ed25519**.

### Authorization Header Format

```
Authorization: Signature keyId="<subscriber_id>|<uk_id>|ed25519",
  algorithm="ed25519",
  created=<unix_timestamp>,
  expires=<unix_timestamp>,
  headers="(created) (expires) digest",
  signature="<base64_signature>"
```

### Digest Header

```
Digest: BLAKE-512=<base64_hash_of_request_body>
```

### Requirements

1. The `keyId` must reference a signing key registered and active in the ONDC Registry.
2. The signing key must cover the domain being requested.
3. `algorithm` must be `ed25519`.
4. `headers` must be `(created) (expires) digest` in that exact order.
5. The `Digest` header must contain a valid BLAKE-512 hash of the request body.
6. Server clocks must be synchronised via NTP — the gateway enforces a clock-skew tolerance.
7. The participant must have at least one configuration in `SUBSCRIBED` status.

---

## Request and Response Format

### Request

All requests follow the Beckn protocol structure:

```json
{
  "context": {
    "domain": "ONDC:RET10",
    "action": "search",
    "country": "IND",
    "city": "std:080",
    "bap_id": "buyer.example.com",
    "bap_uri": "https://buyer.example.com/ondc",
    "transaction_id": "txn-uuid",
    "message_id": "msg-uuid",
    "timestamp": "2026-04-21T10:00:00.000Z",
    "version": "2.0.0"
  },
  "message": {
    /* action-specific payload */
  }
}
```

### Success Response (ACK)

```json
{
  "context": { /* echoed from request */ },
  "message": { "ack": { "status": "ACK" } }
}
```

### Error Response (NACK)

```json
{
  "context": { /* echoed from request, or null if body was not read */ },
  "message": { "ack": { "status": "NACK" } },
  "error": {
    "type": "Gateway",
    "code": "10001",
    "message": "Authentication Failed: Signing key not found in registry (REF:07)"
  }
}
```

> **Note:** The `context` field is `null` only when rejection happens before the body is read (e.g. service overload).

---

## Validation Pipeline

Requests pass through the following validation stages in order. The first failure results in a NACK response — subsequent stages are skipped.

```
Request
  ① Concurrency Limiter ── reject if at capacity (503)
  ② Body Reader ────────── reject if body missing/unreadable/too large
  ③ Audit ──────────────── log request (no rejection)
  ④ Signature Validation ─ reject if auth fails (401)
  ⑤ Digest Validation ──── flag if digest fails (checked later)
  ⑥ Context Validation ─── reject if context fields invalid (200 NACK)
  ⑦ Participant Validation  reject if NP not registered/subscribed (200 NACK)
  ⑧ Schema Validation ──── reject if payload fails schema (200 NACK)
  ⑨ Handler ────────────── policy check → target resolution → dispatch
```

---

## Environments

For Gateway IP addresses to whitelist, see [Common — IP Addresses](../common/README.md).

---

## Error Handling

- See [Gateway_ErrorCodes.md](./Gateway_ErrorCodes.md) for the complete error code reference.
- See [Gateway_Troubleshooting.md](./Gateway_Troubleshooting.md) for troubleshooting common issues.

---

*© ONDC. This document is intended for Network Participants of the ONDC network.*
