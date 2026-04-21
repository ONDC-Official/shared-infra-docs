# Gateway — Troubleshooting Guide

> **Last updated:** 21 April 2026

## About This Document

This document provides troubleshooting guidance for Network Participants (BAPs, BPPs) integrating with the ONDC Gateway. It covers common failure scenarios, likely root causes, and step-by-step resolution approaches.

For the complete list of error codes, see [Gateway_ErrorCodes.md](./Gateway_ErrorCodes.md).

---

## 1. Authentication Failures (10001, HTTP 401)

### Symptom

Every request returns HTTP 401 with `error.code` = `10001`.

### Likely Causes and Resolution

| Cause | How to Diagnose | Resolution |
|-------|-----------------|------------|
| **Missing Authorization header** | Message contains "Authorization header not found" | Add `Authorization: Signature ...` header to every POST request. |
| **Malformed Signature format** | Message contains "Authorization header is malformed" | Verify the header follows `Signature keyId="subscriber_id|uk_id|ed25519", algorithm="ed25519", created=<unix_ts>, expires=<unix_ts>, headers="(created) (expires) digest", signature="<base64>"`. |
| **Invalid keyId** | Message contains "Invalid keyId format" | Use pipe-separated format: `subscriber_id|uk_id|ed25519`. All three fields are required. |
| **Wrong algorithm** | Message contains "Unsupported signing algorithm" | Set `algorithm="ed25519"`. No other algorithms are accepted. |
| **Clock drift** | Message contains "Signature timestamps have expired" | Run `ntpq -p` or equivalent to verify NTP sync. The gateway allows a configurable clock-skew tolerance (typically 60 seconds). |
| **Key not found** | Message contains "Signing key not found in registry" | Verify via [Registry Lookup API](../registry/Lookup_API.md) that the `uk_id` is registered for the target domain. Check key status is `ACTIVE` and within its validity period. |
| **Key expired** | Message contains "Signing key validity period has expired" | Register a new signing key in the registry and update the `keyId` in requests. |
| **Signature mismatch** | Message contains "Signature verification failed" | Recompute the signature. Common issues: (1) signing the wrong body, (2) using a different private key than the registered public key, (3) incorrect `(created)`/`(expires)` values in the signing input. |
| **Signer not subscribed** | Message contains "Signer has no SUBSCRIBED configuration" | The participant exists but has no config in `SUBSCRIBED` status. Contact the registry provider to verify and restore subscription status. |

### Debugging Checklist

1. Verify server clock: `date -u` should match UTC within a few seconds.
2. Verify key registration: call the Registry Lookup API with your `subscriber_id` and confirm the signing key is present, `ACTIVE`, and within its validity period.
3. Regenerate the signature after any key rotation.
4. Compare the exact body bytes used for signature generation with the body being sent — they must be identical (no whitespace changes, no re-serialisation).

---

## 2. Digest Failures (10201, HTTP 401)

### Symptom

Request returns HTTP 401 with `error.code` = `10201`.

### Likely Causes and Resolution

| Cause | Resolution |
|-------|------------|
| Missing `Digest` header | Add `Digest: BLAKE-512=<base64>` header to the request. |
| Wrong algorithm prefix | Use `BLAKE-512=` (not `SHA-256=` or others). |
| Body hash mismatch | Ensure the BLAKE-512 hash is computed over the exact request body bytes being sent (after JSON serialisation, before any HTTP encoding). |

---

## 3. Context Validation Failures (10401, HTTP 200)

### Symptom

Request returns NACK with `error.code` = `10401`.

### Common Issues

| Issue | Example Message | Resolution |
|-------|-----------------|------------|
| Invalid `city` format | `'/city' does not match pattern '^(std:\d{1,5}|\*)$'` | Use `std:NNN` format (e.g. `std:011`) or `*` for all cities. |
| Missing `version` | `context.version or context.core_version is required` | Always include `context.version` (or `context.core_version` for older specs). This field is required for downstream schema validation. |
| Invalid `domain` | `'/domain' does not match pattern ...` | Use a valid ONDC domain code (e.g. `ONDC:RET10`, `ONDC:FIS12`). |
| Invalid `action` | `'/action' is not one of [search, on_search, ...]` | Use one of the 24 supported action names. |

---

## 4. Schema Validation Failures (10601, HTTP 200)

### Symptom

Request returns NACK with `error.code` = `10601`.

### Resolution

1. The `error.message` lists the specific field paths and constraints that failed (e.g. `REQUIRED_PROVIDER_ID: $.message.order.provider.id must be present`).
2. Validate the full request payload against the published ONDC schema for the domain, action, and version declared in the context.
3. Ensure the version in `context.version` matches the schema version the NP is using.

---

## 5. Participant Validation Failures (11001–11005, HTTP 200)

### Symptom

Request returns NACK with `error.code` in the range 11001–11005.

### Resolution by Code

| Code | Cause | Steps |
|------|-------|-------|
| **11001** | Sender NP ID not found | Verify `bap_id` (forward flow) or `bpp_id` (callback flow) is registered for the target `domain` via the Registry Lookup API. |
| **11002** | Receiver NP ID not found | Verify `bpp_id` (forward flow) or `bap_id` (callback flow) is registered for the target `domain`. |
| **11003** | Receiver URI mismatch | The `bpp_uri` (or `bap_uri`) in the request must exactly match the URI registered in the registry for the receiver NP identifier. |
| **11004** | Sender not subscribed | The sender's config status is not `SUBSCRIBED` for the specified domain (e.g. `INACTIVE`, `SUSPENDED`). Contact the registry provider to restore subscription status. |
| **11005** | Receiver not subscribed | The receiver's config status is not `SUBSCRIBED`. Contact the receiver or the gateway operator. |

---

## 6. Routing Failures (11201, HTTP 200)

### Symptom

Async `search` request returns NACK with `error.code` = `11201`.

### Likely Causes

- No NPs are registered for the requested domain + city + country combination.
- NPs are registered but all were excluded by an applicable policy.

### Resolution

1. Verify network coverage for the requested combination.
2. Try broadening the request (e.g. use `city: "*"` if applicable).
3. Contact the gateway operator to review applicable policies.

---

## 7. Policy Blocks (11401, HTTP 200)

### Symptom

Request returns NACK with `error.code` = `11401`.

### Resolution

- **Network policy block**: A gateway-level policy restricts the requested domain/action. Contact the gateway operator.
- **Sender policy block**: The sender's own participant policy forbids delivery to all candidate receivers. Review the participant's policy configuration.
- **Bilateral policy block**: A policy specific to the sender–receiver pair is in force. Contact the gateway operator or the relevant NP.

---

## 8. Timeout and Dispatch Failures (11601, HTTP 504)

### Symptom

Request returns HTTP 504 with `error.code` = `11601`.

### Resolution

1. Retry with exponential backoff (start at 1 second, double per attempt, cap at 60 seconds).
2. Apply ~10% random jitter to avoid synchronised retries.
3. The downstream operation may or may not have completed — design retries to be idempotent where feasible.
4. If persistent, report to the gateway operator.

---

## 9. Receiver Failures (11801–11802, HTTP 200)

### Symptom

Request returns NACK with `error.code` = `11801` (unreachable) or `11802` (timeout).

### Resolution

| Code | Cause | Steps |
|------|-------|-------|
| **11801** | Receiver endpoint unreachable | The receiver is unavailable (connection refused, DNS failure, TLS error) or has been temporarily blocked after repeated failures. Allow 30–60 seconds before retrying the same receiver. |
| **11802** | Receiver did not respond in time | The receiver may be slow or overloaded. Retry after a delay or contact the receiver operator. |

---

## 10. Infrastructure Failures (12001–12002, HTTP 503)

### Symptom

Request returns HTTP 503 with `error.code` = `12001` or `12002`.

### Resolution

| Code | Cause | Steps |
|------|-------|-------|
| **12001** | Gateway temporarily unavailable | An internal service is down. Retry with exponential backoff. |
| **12002** | Gateway at capacity | The per-pod concurrency limit has been reached. Retry with a longer initial delay (e.g. 2 seconds). If sustained, reduce request rate. |

---

## 11. General Debugging Tips

1. **Include the full `error.message` in support requests.** The trailing `(REF:nn)` tag enables ONDC engineering to pinpoint the exact failure path.
2. **Check NTP sync regularly.** Clock drift is the most common cause of persistent 10001 failures.
3. **Validate against the Registry Lookup API** before escalating participant-related errors (11001–11005). Confirm the NP identifier is registered, the domain matches, and the config status is `SUBSCRIBED`.
4. **Log `transaction_id`, `message_id`, and `error.message`** from every NACK response. These are required when raising a support request.
5. **Implement exponential backoff with jitter** for all retryable errors (11601, 11801–11802, 12001–12002).

---

## 12. Support

| Issue Category | Contact |
|----------------|---------|
| Gateway errors (10001–12002) | Gateway operator |
| Registry/subscription issues | ONDC registry support |
| Network coverage (routing) | Refer to the ONDC coverage map or contact the domain operator |
| Policy blocks | Gateway operator or network administrator |

### Information to Include

1. The `error.code` (e.g. `10001`).
2. The full `error.message` verbatim, including any `(REF:nn)` suffix.
3. The `transaction_id` and `message_id` from the request context.
4. The approximate timestamp (UTC).
5. The HTTP status code returned.

---

*© ONDC. This document is intended for Network Participants of the ONDC network.*
