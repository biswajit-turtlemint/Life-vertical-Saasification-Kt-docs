# Life API — GET `/policies/issue/status` (Policy Issuance Status)

**Full URL:** `GET /api/minterprise/v2/products/life/policies/issue/status?referenceId=AHXFK8QMWHY`
**Service:** `transactional-flows`
**Controller:** `SachetController.java:460` → `getPolicyIssueStatus()`

---

## Purpose

Fetches the current issuance status of a life policy from MongoDB. Supports three lookup modes:
- By `referenceId` — returns a single policy record
- By `policyNumber` — returns a single policy record
- By `userId` — returns all policies for that user (list)

No Integration Hub call is made — this is a pure MongoDB read. Used by the UI to show policy status (ACTIVE / PENDING / EXPIRED / LAPSED) after issuance.

---

## Sample cURL

```bash
curl --location 'http://localhost:9098/api/minterprise/v2/products/life/policies/issue/status?referenceId=AHXFK8QMWHY' \
--header 'Authorization: <token>'
```

---

## Request

### Headers

| Header | Description |
|---|---|
| `Authorization` | Auth token |

### Query Parameters

| Param | Type | Required | Example | Description |
|---|---|---|---|---|
| `referenceId` | String | One of | `AHXFK8QMWHY` | Quote/application reference ID |
| `userId` | String | One of | — | User who purchased the policy — returns all their policies |
| `policyNumber` | String | One of | — | Issued policy number |
| `isPolicyExpired` | Boolean | No (default `true`) | `false` | If `false`, EXPIRED status policies are excluded from results |

At least one of `referenceId`, `userId`, or `policyNumber` must be provided. If none are provided → `400 MISSING_FIELD`.

**Lookup priority:** `referenceId` → `policyNumber` → `userId`

---

## Response

### By referenceId or policyNumber — single record (HTTP 200)

```json
{
  "data": {
    "referenceId": "AHXFK8QMWHY",
    "policyNumber": "POL123456",
    "userId": "usr_001",
    "productCode": "life",
    "status": "ACTIVE",
    "createdAt": "2026-01-15T10:30:00",
    "updatedAt": "2026-01-16T09:00:00",
    "insurerCode": "HDFC",
    "productName": "Click 2 Protect Life",
    "premiumAmount": 14750.0,
    "policyTerm": 30,
    "sumAssured": 5000000
  },
  "meta": { "status": "SUCCESS", "error": false }
}
```

### By userId — list (HTTP 200)

```json
{
  "data": [
    { /* IssuanceResult */ },
    { /* IssuanceResult */ }
  ],
  "meta": { "status": "SUCCESS", "error": false }
}
```

### Not Found (HTTP 400)

```json
{
  "data": {
    "errors": [{ "field": "referenceId", "message": "Policy not found for referenceId", "errorCode": "MISSING_FIELD" }]
  },
  "meta": { "status": "FAILURE", "error": true }
}
```

---

## Flowchart

```
Client
  │
  │  GET /api/minterprise/v2/products/life/policies/issue/status
  │      ?referenceId=AHXFK8QMWHY
  │      &isPolicyExpired=true
  ▼
SachetController.getPolicyIssueStatus()
  │
  └─ issuanceService.fetchPoliciesWithErrorResp(referenceId, userId, policyNumber, "life", isPolicyExpired)
       │                   [IssuanceServiceV2Impl.java:65]
       │
       ├─ Determine fieldName for error messages:
       │    referenceId present → "referenceId"
       │    userId present      → "userId"
       │    else                → "policyNumber"
       │
       └─ getPolicyIssueStatus(referenceId, userId, policyNumber, "life", isPolicyExpired)
              │                [IssuanceServiceV2Impl.java:92]
              │
              ├─[referenceId OR policyNumber provided]
              │    properties = { "referenceId": value }  or  { "policyNumber": value }
              │
              │    Try:
              │      issuanceResultService.findOneByPropertiesV2(
              │          properties,
              │          sortField: "createdAt",
              │          collectionClass: Class.forName(
              │              Constants.getBeanClassName("life", IssuanceResult.class)))
              │      ← Queries product-specific collection: "life_issuance_result"
              │
              │    Catch (class not found):
              │      issuanceResultService.findOneByProperties(properties, "createdAt")
              │      ← Fallback: queries generic issuance collection
              │
              │    switchIfEmpty → Mono.just(new IssuanceResult())  ← empty result
              │
              │    Filter:
              │      if isPolicyExpired=false AND result.status == "EXPIRED"
              │        → return empty IssuanceResult (will map to 400 POLICY_NOT_FOUND)
              │
              ├─[userId provided]
              │    getPolicyIssueStatusByUserId(userId, "life", isPolicyExpired)
              │      properties = { "userId": userId, "productCode": "life" }
              │      issuanceResultService.findAllByProperties(properties)
              │      → Flux<IssuanceResult> → collectList()
              │
              │      if isPolicyExpired=false:
              │        filter: keep only records where status != "EXPIRED"
              │
              │      returns List<IssuanceResult>
              │
              └─[none provided]
                   → 400 MISSING_FIELD "referenceId/userId/policyNumber is required"

       Back in fetchPoliciesWithErrorResp():
         ├─[ErrorResponseData]     → pass through as error
         ├─[List] empty            → 400 POLICY_NOT_FOUND
         ├─[IssuanceResult] blank  → 400 POLICY_NOT_FOUND
         └─[IssuanceResult] found  → return result
  │
  ├─[ErrorResponseData] → ResponseEntity 400 + PayloadWrapper.failure(error)
  └─[success]           → ResponseEntity 200 + PayloadWrapper.success(result)
```

---

## Collection Resolution

`Constants.getBeanClassName("life", IssuanceResult.class)` resolves to the fully qualified class name of the product-specific `IssuanceResult` subclass (e.g., `com.sachetProduct.beans.life.LifeIssuanceResult`), which maps to the `life_issuance_result` MongoDB collection.

If the class cannot be found (e.g., new product without a specific class), it falls back to the generic `IssuanceResultRepository.findOneByProperties()` which queries the base collection.

---

## Functions & Where Things Happen

| Function | File | Line | What it does |
|---|---|---|---|
| `getPolicyIssueStatus()` (controller) | `SachetController.java` | 460 | Entry point, delegates to service |
| `fetchPoliciesWithErrorResp()` | `IssuanceServiceV2Impl.java` | 65 | Wraps result with proper error if not found |
| `getPolicyIssueStatus()` (service) | `IssuanceServiceV2Impl.java` | 92 | Core lookup — routes by query param |
| `getPolicyIssueStatusByUserId()` | `IssuanceServiceV2Impl.java` | 128 | userId-specific list lookup with optional expired filter |
| `findOneByPropertiesV2()` | `IssuanceResultRepository` | — | Queries product-specific collection by class |
| `findOneByProperties()` | `IssuanceResultRepository` | — | Fallback: queries generic collection |
| `findAllByProperties()` | `IssuanceResultRepository` | — | Returns all policies for a userId |
| `Constants.getBeanClassName()` | `Constants.java` | — | Resolves product-specific collection class name |

---

## Database / External Calls

| System | Collection | Operation | When |
|---|---|---|---|
| **MongoDB** | `life_issuance_result` (via `getBeanClassName`) | `findOne` sorted by `createdAt` | By referenceId / policyNumber |
| **MongoDB** | Generic issuance collection (fallback) | `findOne` | When class resolution fails |
| **MongoDB** | `life_issuance_result` | `findAll` filtered by userId + productCode | By userId |
| **Integration Hub** | Not called | — | Never |

---

## Policy Status Values

| Status | Meaning |
|---|---|
| `ACTIVE` | Policy is in force |
| `PENDING` | Issuance initiated, awaiting insurer confirmation |
| `EXPIRED` | Policy has lapsed/expired |
| `LAPSED` | Premium unpaid, policy lapsed |

---

## Error Codes

| Scenario | HTTP | Error Code |
|---|---|---|
| None of `referenceId`, `userId`, `policyNumber` provided | 400 | `MISSING_FIELD` |
| Policy not found for given params | 400 | `MISSING_FIELD` (via `POLICY_NOT_FOUND` message) |
| Policy found but status=EXPIRED and `isPolicyExpired=false` | 400 | `MISSING_FIELD` (treated as not found) |

---

## Related API

`POST /products/life/policies/issue` — Triggers policy issuance. Before issuing, it calls `getPolicyIssueStatus()` internally to guard against duplicate issuance (returns `400 DUPLICATE_REQUEST` if a policy already exists for the `referenceId`).
