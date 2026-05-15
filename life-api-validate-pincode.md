# Life API — POST `/lookups/validatePincode` (Pincode Serviceability Check)

**Full URL:** `POST /api/minterprise/v1/products/life/lookups/validatePincode`
**Service:** `application`
**Controller:** `LookupController.java:114` → `validatePincodes()`

---

## Purpose

Checks whether one or more pincodes are serviceable for life insurance. Returns city/state metadata for each pincode. When an insurer code is provided, the check is performed against that insurer's LOCATION master (per-pincode, sequential). Without an insurer, a generic TM pincode master lookup is performed (first pincode only).

---

## Sample cURL

```bash
curl --location 'https://pro.axisbank.saas-sanity.turtle-feature.com/api/minterprise/v1/products/life/lookups/validatePincode' \
--header 'authorization: <token>' \
--header 'content-type: application/json' \
--header 'x-broker: axisbank' \
--data '{"pincodes":["411045"],"insurer":"TATAAIALI"}'
```

---

## Request

### Headers

| Header | Example Value | Description |
|---|---|---|
| `x-broker` | `axisbank` | Broker code |
| `Authorization` | `<token>` | Auth token |

### Body (raw JSON — **no** `PayloadWrapper` envelope)

```json
{
  "pincodes": ["411045"],
  "insurer": "TATAAIALI"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `pincodes` | `List<String>` | Yes | One or more 6-digit pincodes |
| `insurer` | `String` | No | Insurer code (e.g. `TATAAIALI`, `HDFC`, `MAX`). If absent, generic TM pincode master is used |

---

## Response

### Success (HTTP 200)

```json
{
  "data": [
    {
      "pincode": "400001",
      "city": "MUMBAI",
      "state": "MAHARASHTRA"
    },
    {
      "pincode": "560001",
      "city": "BENGALURU",
      "state": "KARNATAKA"
    }
  ],
  "meta": { "status": "SUCCESS", "error": false }
}
```

When a pincode is not found in the master, the entry still appears but with `null` city/state:

```json
{ "pincode": "999999", "city": null, "state": null }
```

### Empty result cases

- Empty `pincodes` list → `[]`
- Missing master-service config (`masterServiceV2Host` or `integrationHubApiKey` blank) → `[]`
- Network/upstream error for a specific pincode → that pincode returns `[]` (error swallowed, logged)

---

## Flowchart

```
Client
  │
  │  POST /api/minterprise/v1/products/life/lookups/validatePincode
  │  Header: x-broker
  │  Body: { pincodes: [...], insurer: "HDFC" }
  ▼
LookupController.validatePincodes()
  │
  └─ PincodeUtils.validatePincodes(productCode="life", broker, request)
       │
       ├─[productCode == "life" or "health"]
       │       │
       │       └─ newPincodesValdiator(request, broker, productCode)
       │              │
       │              ├─ Guard: pincodes empty → return []
       │              ├─ Guard: masterServiceV2Host or apiKey blank → return []
       │              │
       │              ├─ resolveMasterServiceBroker(broker)
       │              │    └─ Always returns "turtlemint" (hardcoded)
       │              │
       │              ├─[insurer present]
       │              │       │
       │              │       └─ fetchPincodesByInsurer(pincodes, masterBroker, insurer, productCode)
       │              │              │
       │              │              For EACH pincode (Flux.concatMap — sequential):
       │              │                │
       │              │                ├─ Build filter request body:
       │              │                │    { filterGroup: { combinator: "and",
       │              │                │      rules: [{ field:"pincode", operator:"=", value: pincode }] } }
       │              │                │
       │              │                ├─ POST masterServiceV2Host
       │              │                │       /api/v2/masters/turtlemint/insurer/{INSURER_UPPER}
       │              │                │       /LOCATION/LIFE/filter-with-query
       │              │                │    Headers: x-api-key, Content-Type: application/json
       │              │                │
       │              │                ├─ extractMasterServiceDataAsList(response)
       │              │                │    → response.data (List)
       │              │                │
       │              │                └─ mapLifePincodeMasterRows(rows, pincode)
       │              │                     → { city, pincode, state } per row
       │              │                       (keys tried: city/cityCode/city_code,
       │              │                                    state/stateCode/state_code)
       │              │              │
       │              │              └─ flattenLifePincodeMasterRows(rowsByPincode)
       │              │                   → flat List<Map>
       │              │
       │              └─[no insurer]
       │                      │
       │                      └─ fetchGenericPincodes(pincodes[0], masterBroker, productCode)
       │                             │
       │                             ├─ GET masterServiceV2Host
       │                             │       /api/v2/masters/turtlemint/LOCATION/LIFE
       │                             │       /filter?pincode={pincode}
       │                             │    Headers: x-api-key
       │                             │
       │                             ├─ extractFirstMasterServiceDataRow(response)
       │                             │    → first item of response.data
       │                             │
       │                             └─ mapLifePincodeMasterRow(row, pincode)
       │                                  → [{ city, pincode, state }] or []
       │
       └─[other productCode] → legacy pincodeValidateURL (POST, old path — not life/health)
  │
  └─ PayloadWrapper.generateResponse(result, null)
       │
       ▼
  HTTP 200
```

---

## Functions & Where Things Happen

| Function | File | Line | What it does |
|---|---|---|---|
| `validatePincodes()` (controller) | `LookupController.java` | 114 | Entry point: reads `x-broker` header, reads raw JSON body as a map (no `PayloadWrapper`), extracts `pincodes[]` and `insurer` fields, delegates to `PincodeUtils.validatePincodes()`, wraps result in `PayloadWrapper.generateResponse()` |
| `validatePincodes()` (util) | `PincodeUtils.java` | 55 | Routes based on `productCode`: for `"life"` or `"health"` calls `newPincodesValdiator()`; for other products calls legacy `pincodeValidateURL` (POST to old path) |
| `newPincodesValdiator()` | `PincodeUtils.java` | 65 | Guards: `pincodes` null/empty → return `[]`; `masterServiceV2Host` or `ihApiKey` blank → return `[]` (logs warning). Calls `resolveMasterServiceBroker()` (always returns `"turtlemint"`). Routes: if `insurer` present → `fetchPincodesByInsurer()`; else → `fetchGenericPincodes()` |
| `resolveMasterServiceBroker()` | `PincodeUtils.java` | 135 | Always returns the hardcoded string `"turtlemint"` regardless of the `x-broker` header. This is intentional — Master Service v2 uses turtlemint as the canonical broker for LOCATION master lookups |
| `fetchPincodesByInsurer()` | `PincodeUtils.java` | 86 | Uses `Flux.concatMap` to query Master Service v2 **sequentially** for each pincode (one call per pincode). For each: calls `buildPincodeFilterRequest()` to create the filter body, POSTs to `/api/v2/masters/turtlemint/insurer/{INSURER_UPPER}/LOCATION/LIFE/filter-with-query`. Calls `extractMasterServiceDataAsList()` (reads `response.data`). Maps each row via `mapLifePincodeMasterRows()`. Collects and flattens all results |
| `fetchGenericPincodes()` | `PincodeUtils.java` | 114 | Only queries the **first pincode** in the list. GETs `/api/v2/masters/turtlemint/LOCATION/LIFE/filter?pincode={pincode}`. Calls `extractFirstMasterServiceDataRow()` (takes first item of `response.data`). Maps via `mapLifePincodeMasterRow()` |
| `buildPincodeFilterRequest()` | `PincodeUtils.java` | 163 | Constructs the filter body: `{ filterGroup: { combinator: "and", rules: [{ field: "pincode", operator: "=", value: "<pincode>" }] } }`. This is Master Service v2's filter-with-query DSL format |
| `extractMasterServiceDataAsList()` | `PincodeUtils.java` | — | Reads `response.data` as a `List`. Returns empty list if response is null or `data` key is missing |
| `mapLifePincodeMasterRows()` | `PincodeUtils.java` | — | Iterates master rows and calls `mapLifePincodeMasterRow()` on each. Collects into a flat list |
| `mapLifePincodeMasterRow()` | `PincodeUtils.java` | 219 | Normalizes one master row to `{ city, pincode, state }`. Tries multiple key names: city from `city` / `cityCode` / `city_code`; state from `state` / `stateCode` / `state_code`. Returns empty list if row is blank |

---

## Database / External Calls

| System | Endpoint | Method | When |
|---|---|---|---|
| **Master Service v2** | `/api/v2/masters/turtlemint/insurer/{INSURER}/LOCATION/LIFE/filter-with-query` | POST | Insurer code present — one call per pincode |
| **Master Service v2** | `/api/v2/masters/turtlemint/LOCATION/LIFE/filter?pincode={pincode}` | GET | No insurer — first pincode only |
| **MongoDB** | Not used | — | — |

> **Important:** `broker` from the `x-broker` header is **ignored** when calling master service. It always uses `"turtlemint"` (`resolveMasterServiceBroker()` hardcodes this).

> **Important:** The no-insurer path only queries the **first pincode** in the list. Multiple pincodes without an insurer → only the first is checked.

---

## Error Codes

| Scenario | HTTP | Behaviour |
|---|---|---|
| Empty pincodes list | 200 | Returns `[]` |
| Master service config missing | 200 | Returns `[]` (logs warning) |
| Master service error for a pincode | 200 | That pincode's result is `[]` (error swallowed) |
