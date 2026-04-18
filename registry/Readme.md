# Registry

Documentation and reference materials for the ONDC Registry service.

---

## Contents

| File | Description |
| --- | --- |
| [Lookup_API.md](./Lookup_API.md) | Reference specification for the ONDC Registry Lookup API — endpoints, request/response fields, authentication, and error structure for `/lookup` (V1) and `/v2.0/lookup` (V2) |
| [Lookup_API_ErrorCodes.md](./Lookup_API_ErrorCodes.md) | Complete list of error codes returned by the Lookup API, grouped by category (Auth, Business, Schema, Method, Server) with descriptions and resolutions |
| [Lookup_API_Troubleshooting.md](./Lookup_API_Troubleshooting.md) | Troubleshooting guide for the Lookup API covering common failure scenarios and step-by-step resolutions for both V1 and V2 |
| [ONDC Registry.postman_collection.json](./ONDC%20Registry.postman_collection.json) | Postman collection for testing ONDC Registry API endpoints |
| [ONDC Registry Environment.postman_environment.json](./ONDC%20Registry%20Environment.postman_environment.json) | Postman environment file with pre-configured variables for Registry API testing |

---

## Using the Postman Collection

### Import

1. Open **Postman** and click **Import** (top-left).
2. Import the collection: select [`ONDC Registry.postman_collection.json`](./ONDC%20Registry.postman_collection.json).
3. Import the environment: select [`ONDC Registry Environment.postman_environment.json`](./ONDC%20Registry%20Environment.postman_environment.json).

### Set the Active Environment

4. In the top-right environment dropdown, select **ONDC Registry Environment**.

### Configure Variables

5. Open the environment and fill in any required variables such as `subscriber_id`, and `ondc_registry_base_url` for the target environment (pre-prod or prod).

### Send Requests

6. Open a request from the collection (e.g. **Lookup v2**), review the pre-filled headers and body, and click **Send**.


