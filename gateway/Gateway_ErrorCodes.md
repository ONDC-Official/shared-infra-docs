# Gateway — Error Code Reference

> **Last updated:** 21 April 2026

## About This Document

This document lists all error codes returned by the ONDC Gateway when a request is rejected (NACK response). Each entry describes the ONDC error code, the HTTP status returned, the error category, the message format, the root cause, and the recommended resolution.

Errors are grouped into eleven categories:
- **Auth** — issues with the `Authorization: Signature` header, signing key, or signer subscription status
- **Digest** — body integrity check failures (`Digest` header)
- **Context** — invalid or missing ONDC `context` fields
- **Schema** — full request payload does not conform to the ONDC schema
- **Payload** — request body is missing, unreadable, or exceeds the size limit
- **Participant** — referenced NP identifier is not registered, not subscribed, or has a URI mismatch
- **Routing** — no eligible participants to receive the request
- **Policy** — request blocked by an applicable network or participant policy
- **Dispatch** — gateway timeout or client disconnect
- **Receiver** — receiver endpoint is unreachable or did not respond
- **Infra** — gateway is temporarily unable to process the request

---

## Response Format

Every gateway response is either an **ACK** (success) or a **NACK** (failure with error details).

### ACK (HTTP 200)

```json
{
  "context": { /* echoed from request */ },
  "message": { "ack": { "status": "ACK" } }
}
```

### NACK

```json
{
  "context": { /* echoed from request, or null if body was not read */ },
  "message": { "ack": { "status": "NACK" } },
  "error": {
    "type": "Gateway",
    "code": "<ONDC code>",
    "message": "<description> (REF:nn)"
  }
}
```

| Field | Description |
|-------|-------------|
| `error.type` | Always `"Gateway"` for errors raised by the gateway. |
| `error.code` | Numeric error code identifying the failure category. |
| `error.message` | Human-readable description. May include a trailing `(REF:nn)` diagnostic suffix — see below. |

> **About `(REF:nn)`:** The `error.message` may include a trailing diagnostic tag such as `(REF:07)`. This tag is **for ONDC internal use only** and helps ONDC engineering identify the specific failure path. NPs **should not parse, depend on, or branch on REF values** in their code. When raising a support request, include the full `error.message` verbatim so the REF tag is preserved.

---

## Error Codes

### Auth Errors (10001)

| Code | HTTP | Category | Message | Description | Resolution |
|------|------|----------|---------|-------------|------------|
| 10001 | 401 | Auth | Authentication Failed: Authorization header not found | `Authorization` header is missing from the request. | Add `Authorization: Signature ...` header to every request. |
| 10001 | 401 | Auth | Authentication Failed: Authorization header is malformed | The `Signature` scheme or format is invalid. | Ensure the header follows the required format: `Signature keyId="...", algorithm="ed25519", created=..., expires=..., headers="(created) (expires) digest", signature="..."` |
| 10001 | 401 | Auth | Authentication Failed: Invalid keyId format in Authorization header | `keyId` is not in the required format. | Use `subscriber_id|uk_id|ed25519` (pipe-separated). |
| 10001 | 401 | Auth | Authentication Failed: Unsupported signing algorithm, must be ed25519 | Algorithm is not `ed25519`. | Set `algorithm="ed25519"` in the Authorization header. |
| 10001 | 401 | Auth | Authentication Failed: Required signing parameters missing or incomplete | `(created)`, `(expires)`, or `digest` params are absent from the `headers` field. | Include all three: `headers="(created) (expires) digest"`. |
| 10001 | 401 | Auth | Authentication Failed: Signing parameters are out of required order | Header params are present but not in the correct sequence. | Use exact order: `(created) (expires) digest`. |
| 10001 | 401 | Auth | Authentication Failed: Signing key not found in registry | The NP is not registered, the key ID is unknown, the key does not cover the requested domain, or the key has been revoked. | Verify the signing key is registered and active in the ONDC registry. Confirm the key covers the target domain. |
| 10001 | 401 | Auth | Authentication Failed: Signature timestamps have expired | The `(created)` / `(expires)` timestamps are outside the permitted clock-skew window. | Synchronise server clock via NTP. Regenerate the signature with current timestamps. |
| 10001 | 401 | Auth | Authentication Failed: Signing key is not yet valid | The key's `valid_from` timestamp is in the future. | Wait until the key becomes active, or use a different registered key. |
| 10001 | 401 | Auth | Authentication Failed: Signing key validity period has expired | The key's `valid_till` timestamp is in the past. | Register a new signing key and update the `keyId` accordingly. |
| 10001 | 401 | Auth | Authentication Failed: Signature verification failed | The ed25519 signature does not verify against the registered public key. | Recompute the signature over the exact `(created)`, `(expires)`, and `digest` values sent. Verify the correct private key is being used. |
| 10001 | 401 | Auth | Authentication Failed: Signer has no SUBSCRIBED configuration | The participant exists and the signature is valid, but no config entry is in `SUBSCRIBED` status. | Confirm the participant's subscription status is `SUBSCRIBED` in the ONDC registry. |

### Digest Errors (10201)

| Code | HTTP | Category | Message | Description | Resolution |
|------|------|----------|---------|-------------|------------|
| 10201 | 401 | Digest | Digest validation failed: Digest header is missing | `Digest` HTTP header is absent from the request. | Add `Digest: BLAKE-512=<base64-hash>` header. |
| 10201 | 401 | Digest | Digest validation failed: Unsupported digest algorithm, must be BLAKE-512 | The `Digest` header does not use the `BLAKE-512=` prefix. | Use `BLAKE-512=` as the algorithm prefix. |
| 10201 | 401 | Digest | Digest validation failed: Request body digest does not match | The BLAKE-512 hash of the body does not match the value in the `Digest` header. | Recompute the BLAKE-512 hash of the exact request body being sent. |

### Context Errors (10401)

| Code | HTTP | Category | Message | Description | Resolution |
|------|------|----------|---------|-------------|------------|
| 10401 | 200 | Context | context validation failed: *{detail}* | An ONDC context field fails the inline schema check — field patterns, format rules, or missing required fields. The detail text identifies the specific field and constraint. | Inspect the `error.message` for the exact field path. Validate the `context` block against the ONDC specification for the relevant domain and action. |

### Schema Errors (10601)

| Code | HTTP | Category | Message | Description | Resolution |
|------|------|----------|---------|-------------|------------|
| 10601 | 200 | Schema | Schema validation failed: *{detail}* | The full request payload does not conform to the ONDC schema for the specified domain, action, and version. | Inspect the `error.message` for specific validation errors and field paths. Validate against the published ONDC schema for the relevant domain, action, and version. |

### Payload Errors (10801)

| Code | HTTP | Category | Message | Description | Resolution |
|------|------|----------|---------|-------------|------------|
| 10801 | 200 | Payload | Request body is required | No body present in the request. | Always include a request body for POST requests. |
| 10801 | 200 | Payload | Unable to read request body | Body I/O read failure. | Retry once. If persistent, investigate the client's connection stability. |
| 10801 | 413 | Payload | Request payload exceeds the configured size limit | Body size exceeds the gateway's configured maximum. | Reduce the payload size (e.g. limit items in `on_search` responses). |

### Participant Errors (11001–11005)

| Code | HTTP | Category | Message | Description | Resolution |
|------|------|----------|---------|-------------|------------|
| 11001 | 200 | Participant | Invalid {field}. Valid {field} must be passed for domain {domain} | Sender NP ID (`bap_id` for forward flow, `bpp_id` for callback) not found in registry for the specified domain. | Verify the NP identifier is registered for the target domain. |
| 11002 | 200 | Participant | Invalid {field}. Valid {field} must be passed for domain {domain} | Receiver NP ID (`bpp_id` for forward flow, `bap_id` for callback) not found in registry. | Verify the receiver NP identifier is registered for the target domain. |
| 11003 | 200 | Participant | Receiver URI does not match the registered URI | Receiver URI (`bpp_uri` or `bap_uri`) differs from the value registered in the registry. | Use the URI registered for the receiver NP identifier. |
| 11004 | 200 | Participant | {field} config status is {status} for domain {domain}. Only SUBSCRIBED participants are allowed | Sender participant exists but is not in `SUBSCRIBED` status (e.g. `INACTIVE`, `SUSPENDED`). | Contact the registry provider to restore `SUBSCRIBED` status. |
| 11005 | 200 | Participant | {field} config status is {status} for domain {domain}. Only SUBSCRIBED participants are allowed | Receiver participant exists but is not in `SUBSCRIBED` status. | Contact the receiver or the gateway operator. |

### Routing Errors (11201)

| Code | HTTP | Category | Message | Description | Resolution |
|------|------|----------|---------|-------------|------------|
| 11201 | 200 | Routing | No eligible participants found for routing | Zero broadcast targets for the requested domain/city/country, or all targets were excluded by a participant policy. | Verify network coverage for the requested combination. If coverage exists, contact the gateway operator to review applicable policies. |

### Policy Errors (11401)

| Code | HTTP | Category | Message | Description | Resolution |
|------|------|----------|---------|-------------|------------|
| 11401 | 200 | Policy | Request blocked by gateway network policy | A network-level policy blocks the requested domain/action combination. | Contact the gateway operator. |
| 11401 | 200 | Policy | Request blocked by sender participant policy | The sender's own participant policy forbids all targets for this request. | Contact the gateway operator or review the participant's policy configuration. |
| 11401 | 200 | Policy | Request blocked by bilateral participant policy | A bilateral `INTER-NP` policy blocks the specific sender–receiver pair. | Contact the gateway operator or the relevant Network Participant. |

### Dispatch Errors (11601–11602)

| Code | HTTP | Category | Message | Description | Resolution |
|------|------|----------|---------|-------------|------------|
| 11601 | 504 | Dispatch | Gateway request timed out | The gateway timed out while processing the request. | Retry with exponential backoff. Design retries to be idempotent. |
| 11602 | 499 | Dispatch | Client closed connection before response | The client (NP) disconnected before the gateway sent a response. Logged for audit; the response is never delivered. | Informational — no action required. |

### Receiver Errors (11801–11802)

| Code | HTTP | Category | Message | Description | Resolution |
|------|------|----------|---------|-------------|------------|
| 11801 | 200 | Receiver | Receiver endpoint unreachable | Transport failure reaching the receiver: connection refused, DNS failure, TLS error, or the receiver has been temporarily blocked after repeated failures. | Retry after a delay. If the same receiver fails repeatedly, allow 30–60 seconds for recovery before retrying. |
| 11802 | 200 | Receiver | Receiver did not respond in time | The receiver was reached but did not return an HTTP response within the deadline. | Retry after a delay or contact the receiver. |

### Infra Errors (12001–12002)

| Code | HTTP | Category | Message | Description | Resolution |
|------|------|----------|---------|-------------|------------|
| 12001 | 503 | Infra | Gateway is temporarily unable to process this request | An internal gateway service is unavailable (e.g. internal dispatch, schema validator, registry cache, or message queue). | Retry with exponential backoff. |
| 12002 | 503 | Infra | Gateway overloaded, please retry with exponential backoff | The gateway's per-pod concurrency limit has been reached. | Retry with exponential backoff using a longer initial delay. If persistent, reduce request rate. |

---

## HTTP Status Summary

| HTTP | Meaning | ONDC Codes |
|------|---------|------------|
| **200** | ACK (success) or NACK (business/validation error) | 10401, 10601, 10801 (body missing/unreadable), 11001–11005, 11201, 11401, 11801–11802 |
| **401** | Unauthorized | 10001 (auth), 10201 (digest) |
| **413** | Payload Too Large | 10801 (body exceeds size limit) |
| **499** | Client Closed | 11602 |
| **503** | Service Unavailable | 12001, 12002 (retry with backoff) |
| **504** | Gateway Timeout | 11601 (retry) |

---

## Retryable vs Non-Retryable

### Retryable Errors

| Category | Codes | Recommended Action |
|----------|-------|--------------------|
| Dispatch | 11601 | Retry with exponential backoff |
| Receiver | 11801, 11802 | Retry after delay; allow 30–60 s recovery for repeated failures |
| Infra | 12001, 12002 | Retry with exponential backoff |

### Non-Retryable Errors

| Category | Codes | Recommended Action |
|----------|-------|--------------------|
| Auth | 10001 | Correct credentials or signing key |
| Digest | 10201 | Correct the `Digest` header computation |
| Context | 10401 | Correct context fields |
| Schema | 10601 | Correct request payload structure |
| Payload | 10801 | Correct request body |
| Participant | 11001–11005 | Verify NP identifiers and subscription state |
| Routing | 11201 | Verify network coverage |
| Policy | 11401 | Contact gateway operator |

---

*© ONDC. This document is intended for Network Participants of the ONDC network.*
