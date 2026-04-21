# shared-infra-docs

Documentation and reference materials for ONDC shared infrastructure services — Registry and Gateway.

---

## Repository Structure

### [`common/`](./common/)

| File | Description |
| --- | --- |
| [README.md](./common/README.md) | IP addresses to whitelist for ONDC Registry and Gateway across Pre-Production and Production environments |

---

### [`gateway/`](./gateway/)

| File | Description |
| --- | --- |
| [README.md](./gateway/README.md) | Gateway overview — API endpoints (24 ONDC actions + health checks), authentication, request/response format, and validation pipeline |
| [Gateway_ErrorCodes.md](./gateway/Gateway_ErrorCodes.md) | Complete list of error codes returned by the Gateway, grouped by category (Auth, Digest, Context, Schema, Payload, Participant, Routing, Policy, Dispatch, Receiver, Infra) with descriptions and resolutions |
| [Gateway_Troubleshooting.md](./gateway/Gateway_Troubleshooting.md) | Troubleshooting guide covering common failure scenarios and step-by-step resolutions for NP integration |

---

### [`registry/`](./registry/)

| File | Description |
| --- | --- |
| [README.md](./registry/README.md) | Registry documentation (in progress) |
| [Lookup_API.md](./registry/Lookup_API.md) | Reference specification for the ONDC Registry Lookup API — endpoints, request/response fields, authentication, and error structure for `/lookup` (V1) and `/v2.0/lookup` (V2) |
| [Lookup_API_ErrorCodes.md](./registry/Lookup_API_ErrorCodes.md) | Complete list of error codes returned by the Lookup API, grouped by category (Auth, Business, Schema, Method, Server) with descriptions and resolutions |
| [Lookup_API_Troubleshooting.md](./registry/Lookup_API_Troubleshooting.md) | Troubleshooting guide for the Lookup API covering common failure scenarios and step-by-step resolutions for both V1 and V2 |
| [ONDC Registry.postman_collection.json](./registry/ONDC%20Registry.postman_collection.json) | Postman collection for testing ONDC Registry API endpoints |
| [ONDC Registry Environment.postman_environment.json](./registry/ONDC%20Registry%20Environment.postman_environment.json) | Postman environment file with pre-configured variables for Registry API testing |