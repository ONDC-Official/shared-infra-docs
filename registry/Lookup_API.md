# Lookup API Specification — Reference

_Last updated: 2026-04-16_

## About This Document

This document is the reference specification for the ONDC Registry Lookup API. It covers two versions of the lookup endpoint:

- **`/lookup` (V1/Legacy)** — public, unauthenticated endpoint using the V2 filter set for backward compatibility.
- **`/v2.0/lookup` (V2)** — authenticated endpoint requiring an ONDC `Authorization: Signature` header and a `Digest` header.

The document details the available endpoints, required request headers and body fields, filtering rules, pagination, and the structure of both success and error responses. A health-check endpoint (`GET /health`) is also described.

---

## 1. Endpoints

| Service | Method | Path | Auth | Description | Notes |
| --- | --- | --- | --- | --- | --- |
| LookupSvc | POST | /lookup | None (public) | Legacy Lookup — no auth, V2 filter set | Backward compat. No Authorization header required. Uses V2 request/response format. Fields participant_id, include, select, filter are unknown in this endpoint and rejected (error 15050). |
| LookupSvc | POST | /v2.0/lookup | ONDC Signature | V2 Lookup — authenticated | subscriber_id and/or ≥ 2 of: country, ukId, city, domain, type. One flat object per Config in response. Fields participant_id, include, select, filter are rejected. |
| LookupSvc | GET | /health | None | Health Check | Returns {status: ok\|unavailable, cache: ok\|unavailable}. HTTP 503 when cache not ready. |

## 2. Request Fields

### Required Auth Headers (/v2.0/lookup)

| Field Name | Required | Type | Description | Example | Validation |
| --- | --- | --- | --- | --- | --- |
| Authorization | Yes | string | ONDC Signature header. Format: Signature keyId="subscriber_id\|uk_id\|algorithm",algorithm="ed25519",created=<unix>,expires=<unix>,headers="(created) (expires) digest",signature="<base64>" | Signature keyId="seller.example.com\|key-001\|ed25519",... | All fields required: keyId (pipe format), algorithm=ed25519, created, expires, headers, signature |
| Digest | Yes | string | BLAKE-512 hash of the raw request body bytes, base64-encoded. | BLAKE-512=base64digestvalue== | Format: BLAKE-512=<base64>; must match exact request body bytes |
| Content-Type | Yes | string | Must be application/json. | application/json | Fixed value: application/json |

### V2 / V1 Body Fields (/v2.0/lookup and /lookup)

| Field Name | Required | Type | Description | Example | Validation |
| --- | --- | --- | --- | --- | --- |
| subscriber_id | No* | string | Participant subscriber domain name. At least subscriber_id OR two other filters required. | b2b.seller.example.com | minLength:1, maxLength:255 |
| country | No* | string | ISO 3166-1 alpha-3 country code filter. | IND | pattern: ^[A-Z]{3}$ |
| city | No* | string | STD city code filter. Specific code only — wildcards rejected. | std:080 | pattern: ^std:\d{2,5}$; wildcard not allowed |
| domain | No* | string | ONDC business domain filter. | ONDC:RET10 | pattern: ^[A-Za-z0-9]+:[A-Za-z0-9]+$ |
| type | No* | string | Network participant type filter. | BAP \| BPP | enum: BAP, BPP |
| ukId | No* | string | Unique key ID filter — matches a registered key ukId. | key-001 | minLength:1 |
| max_results | No | integer | Max participants to return. 0 = server-default cap applies. | 100 | min: 0; negative values return 416 error |
| offset | No | integer | Pagination offset — reserved. Setting offset > 0 disables the server-side response cache. | 0 | min: 0; default 0 |

> * At least TWO filter fields required for /v2.0/lookup. Fields participant_id, include, select, filter are unknown in V2 and rejected (error 15050). /lookup (legacy) uses the same body but requires NO Authorization header.

## 3. Response Fields

### V2 / V1 flat response — /v2.0/lookup and /lookup

| Field Path | Description | Example |
| --- | --- | --- |
| [*].subscriber_id | Subscriber domain name. | b2b.seller.example.com |
| [*].status | Subscription status. | SUBSCRIBED |
| [*].ukId | Unique key ID for this config entry. | key-001 |
| [*].subscriber_url | Callback / subscriber URL for this config. | https://api.seller.example.com/ondc |
| [*].country | Country code (ISO 3166-1 alpha-3). | IND |
| [*].domain | ONDC domain code for this config. | ONDC:RET10 |
| [*].valid_from | Key validity start (ISO 8601). | 2026-01-01T00:00:00.000Z |
| [*].valid_until | Key validity end (ISO 8601). | 2027-01-01T00:00:00.000Z |
| [*].type | Subscriber type: BAP \| BPP. | BPP |
| [*].signing_public_key | Ed25519 signing public key (base64 SPKI). | MCowBQYDK2VwAyEA... |
| [*].encr_public_key | X25519 encryption public key (base64 SPKI). | MCowBQYDK2VuAyEA... |
| [*].created | Record creation timestamp (ISO 8601). | 2026-01-01T00:00:00.000Z |
| [*].updated | Record last-updated timestamp (ISO 8601). | 2026-04-01T12:30:00.000Z |
| [*].br_id | Internal config UUID. | aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaa0 |
| [*].city | City STD code for this config entry. | std:080 |

### Error response fields — all endpoints

| Field Path | Description | Example |
| --- | --- | --- |
| error.code | ONDC error code — plain numeric string. See Error-codes sheet. | 15055 |
| error.message | Human-readable error description. | At least two of participant_id, subscriber_id, ... must be provided |
| message.ack.status | Always 'NACK' for error responses. | NACK |
| request_id | Echoed from X-Request-ID header, or auto-generated UUID if absent. | a1b2c3d4-e5f6-7890-abcd-ef1234567890 |

