## About This Document

This document lists all error codes returned by the ONDC Registry Lookup API (`POST /v3.0/lookup`). Each entry describes the error code, the HTTP status returned, the category of the error, the ONDC-defined message, a description of the root cause, and the recommended resolution.

Errors are grouped into five categories:
- **Auth** – issues with the `Authorization: Signature` header or signing key
- **Business** – valid requests that yield no matching participants
- **Schema** – malformed or insufficient request bodies
- **Method** – wrong HTTP method or missing `Content-Type`
- **Server** – internal or upstream infrastructure failures

---

## 4. Error Codes


### Auth Errors

| Code | HTTP Status | Category | ONDC Message | Description | Resolution |
| --- | --- | --- | --- | --- | --- |
| 15000 | 401 | Auth | Authentication failed | Generic authentication failure — Authorization header missing, unparseable, or key not found. | Add valid Authorization: Signature header; verify signing key pair is registered in the registry |
| 15005 | 401 | Auth | Invalid key ID in Authorization header | The keyId in the Authorization header is malformed or references an unknown subscriber/key combination. | Fix keyId format to subscriber_id\|uk_id\|algorithm (pipe-separated); verify uk_id is registered |
| 15010 | 401 | Auth | The auth header is not valid | Authorization header is present but structurally malformed — missing required Signature fields. | Ensure all Signature fields are present: keyId, algorithm, created, expires, headers, signature |
| 15015 | 401 | Auth | Missing Authorization header | Authorization header is completely absent from the request. | Add Authorization: Signature ... header to every lookup request |
| 15020 | 401 | Auth | The request has expired | The created or expires timestamps in the Authorization header triggered replay protection. | Regenerate Authorization header with current Unix timestamp; sync system clock via NTP |
| 15025 | 401 | Auth | There is mismatch in the signature algorithm | The algorithm field in Authorization does not match the expected value (must be ed25519). | Set algorithm=ed25519 in the Authorization Signature header |
| 15030 | 401 | Auth | Invalid headers are present in header parameters | The headers parameter in Authorization contains non-standard or absent header names. | Use only "(created) (expires) digest" in the headers param |
| 15035 | 401 | Auth | Signature verification failed | Authorization header is structurally valid but the Ed25519 signature does not verify. | Recompute signature over exact (created), (expires), digest values sent; use POST /signature/generate |

### Business Errors

| Code | HTTP Status | Category | ONDC Message | Description | Resolution |
| --- | --- | --- | --- | --- | --- |
| 15040 | 404 | Business | Subscriber not found | The subscriber_id (or subscriber derived from keyId) is not found in the registry or cache. | Verify subscriber is SUBSCRIBED via GET /admin/participants/:id; cache refreshes periodically |
| 15045 | 200 | Business | No matching network participant found | Query is valid but no participant matches the filter combination. HTTP 200 with empty result — not a failure. | Broaden query filters (different domain, city *, omit optional fields); verify participants are registered |

### Schema Errors

| Code | HTTP Status | Category | ONDC Message | Description | Resolution |
| --- | --- | --- | --- | --- | --- |
| 15050 | 400 | Schema | Please provide valid schema. ERROR : JSON Request Invalid format | Request body is not valid JSON, exceeds 64 KB, or is missing required fields. | Fix JSON syntax; ensure domain+country fields present; verify body is ≤ 64 KB |
| 15055 | 400 | Schema | At least two of participant_id, subscriber_id, country, ukId, city, domain, type must be provided | Fewer than two filter fields provided — insufficient parameters for a valid lookup (lookup-invalid). | Provide ≥ 2 of: participant_id, subscriber_id, country, ukId, city, domain, type |

### Method Errors

| Code | HTTP Status | Category | ONDC Message | Description | Resolution |
| --- | --- | --- | --- | --- | --- |
| 15060 | 405 | Method | Method not allowed | HTTP method is not POST. The lookup endpoint only accepts POST. | Use POST /v3.0/lookup (not GET, PUT, DELETE, etc.) |
| 15065 | 200 | Method | Please provide valid schema. ERROR : Content-Type header must be application/json | Content-Type header is absent or not application/json. Response is HTTP 200 with error body (ONDC convention). | Set Content-Type: application/json header on the request |

### Server Errors

| Code | HTTP Status | Category | ONDC Message | Description | Resolution |
| --- | --- | --- | --- | --- | --- |
| 15070 | 500 | Server | Internal server error / Upstream dependency error / Service not ready | Three scenarios share this code: (1) unexpected internal error, (2) upstream Redis/DB unreachable, (3) cache not yet primed at service startup. ONDC spec has no dedicated 5xx codes beyond UNKNOWN_ERROR. | Retry with exponential backoff; wait ~30 s on service start for cache priming; check Redis/DB health if persistent |