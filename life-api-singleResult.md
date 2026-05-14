# Life API — POST `/singleresult` (Fetch Single Life Quote)

**Full URL:** `POST /api/minterprise/v2/products/life/singleresult`
**Service:** `transactional-flows`
**Controller:** `SachetController.java:130` → `getSingleResult()`

---

## Purpose

Fetches or re-calculates a detailed premium quote for one specific life plan variant selected by the user. Typically called after the user picks a product from the listing page and wants the full premium breakdown (riders, taxes, benefit summary). The result is persisted in MongoDB for downstream use (comparison, proposal, issuance).

---

## Sample cURL

```bash
curl --location 'http://localhost:9098/api/minterprise/v2/products/life/singleresult' \
--header 'Content-Type: application/json' \
--header 'x-tenant: axisbank' \
--header 'x-broker: axisbank' \
--header 'x-partner-id: 647f458828effa0001044980' \
--header 'Authorization: <token>' \
--data '{
    "data": {
        "requestId": "AHX5828XJB7",
        "vertical": "LIFE",
        "keys": ["P126"],
        "subKey": 2,
        "riderMeta": [
            {
                "subKey": 2,
                "productCode": "P126",
                "optionCode": 2,
                "riderInfoList": [
                    {
                        "riderCode": "R75",
                        "riderName": "Critical Illness Benefit",
                        "riderDesc": "Critical Illness Benefit rider provides coverage against upto 60 major critical illnesses",
                        "riderShortDesc": "CI rider short description",
                        "riderCategory": "Critical Illness",
                        "riderApiCode": "",
                        "isCoverAmountEditable": true,
                        "isCoverAmountIncludedInBasePlan": false,
                        "riderPolicyTerm": 15,
                        "riderPremiumPaymentTerm": 15,
                        "inBuilt": false,
                        "isSelected": true,
                        "productCode": "P126",
                        "optionCode": "2",
                        "riderSumAssured": 1000000,
                        "riderPremium": 1964,
                        "riderPremiumWithoutTax": 0,
                        "currency": "INR",
                        "extraInfo": {
                            "optRiderCode": "R096C01"
                        }
                    }
                ],
                "premiumPaymentTerm": 25,
                "policyTerm": 30,
                "sumAssured": 10000000,
                "basePlanPremium": 139956
            }
        ],
        "offerMeta": []
    }
}'
```

---

## Request

### Headers

| Header | Example Value | Description |
|---|---|---|
| `x-tenant` | `axisbank` | Tenant identifier |
| `x-broker` | `axisbank` | Broker/channel code |
| `x-partner-id` | `647f458828effa0001044980` | Partner/RM MongoDB ID |
| `Authorization` | `<token>` | Auth token |

### Body

Wrapped in `PayloadWrapper` → `data`:

```json
{
  "data": {
    "requestId": "AHX5828XJB7",
    "vertical": "LIFE",
    "keys": ["P126"],
    "subKey": 2,
    "riderMeta": [ ... ],
    "offerMeta": []
  }
}
```

#### Top-level fields

| Field | Type | Required | Description |
|---|---|---|---|
| `requestId` | String | Yes | Quote session reference ID (same as `referenceId` in other APIs) |
| `vertical` | String | Yes | Always `"LIFE"` |
| `keys` | Array\<String\> | Yes | Product code(s) to fetch quote for (e.g., `["P126"]`) |
| `subKey` | Integer | Yes | Option code / variant index for the selected product |
| `riderMeta` | Array | No | Selected rider configuration — see below |
| `offerMeta` | Array | No | Offer metadata (usually empty) |

#### `riderMeta[]` fields

| Field | Type | Description |
|---|---|---|
| `subKey` | Integer | Option code (must match top-level `subKey`) |
| `productCode` | String | Insurer product code (e.g., `P126`) |
| `optionCode` | Integer | Plan variant/option code |
| `premiumPaymentTerm` | Integer | PPT in years for this plan |
| `policyTerm` | Integer | Policy term in years |
| `sumAssured` | Long | Sum assured in INR |
| `basePlanPremium` | Long | Base plan premium amount |
| `riderInfoList` | Array | List of selected riders |

#### `riderMeta[].riderInfoList[]` fields

| Field | Type | Description |
|---|---|---|
| `riderCode` | String | Internal rider code (e.g., `R75`) |
| `riderName` | String | Display name of rider |
| `riderCategory` | String | Category label (e.g., `Critical Illness`) |
| `riderApiCode` | String | Insurer's API code for the rider |
| `isCoverAmountEditable` | Boolean | Can the rider SA be changed by user |
| `isCoverAmountIncludedInBasePlan` | Boolean | Whether rider SA is part of base sum assured |
| `riderPolicyTerm` | Integer | Rider coverage term |
| `riderPremiumPaymentTerm` | Integer | Rider PPT |
| `inBuilt` | Boolean | `true` = free/built-in rider |
| `isSelected` | Boolean | `true` = user has selected this rider |
| `riderSumAssured` | Long | Rider cover amount in INR |
| `riderPremium` | Long | Rider premium amount |
| `riderPremiumWithoutTax` | Long | Rider premium excluding GST |
| `currency` | String | Always `"INR"` |
| `extraInfo` | Map | Insurer-specific extra params (e.g., `optRiderCode`) |

---

## Response

### Success (HTTP 200)

```json
{
  "data": {
    "referenceId": "AHXFK8QMWHY",
    "quotes": [
      {
        "quoteId": "QT_HDFC_001",
        "productCode": "HDFC_CLICK_2_PROTECT_LIFE",
        "productName": "Click 2 Protect Life",
        "insurerCode": "HDFC",
        "premium": 12500.00,
        "premiumWithTax": 14750.00,
        "sumAssured": 5000000,
        "policyTerm": 30,
        "premiumPaymentTerm": 20,
        "paymentFrequency": 12,
        "riders": [],
        "features": []
      }
    ]
  },
  "meta": { "status": "SUCCESS", "error": false }
}
```

### Failure (HTTP 400)

```json
{
  "data": {
    "errors": [{ "field": "referenceId", "message": "quote not found", "errorCode": "NO_DATA_FOUND" }]
  },
  "meta": { "status": "FAILURE", "error": true }
}
```

---

## Flowchart

```
Client
  │
  │  POST /api/minterprise/v2/products/life/singleresult
  │  Headers: x-tenant, x-broker, x-partner-id, x-provider
  │  Body: { "data": { referenceId, selectedPlans, providers } }
  ▼
SachetController.getSingleResult()
  │
  ├─ ISachetProductAggregatorFactory.getPremiumAggregator("life")
  │    └─ Returns SachetLifeAggregator
  │
  └─ SachetLifeAggregator.getSingleResult(payload, tenant, broker, partnerId, provider)
       │
       ├─ Parse PayloadWrapper.data → QuotationRequest
       ├─ Validate: referenceId not blank → else 400 MISSING_FIELD
       ├─ Validate: selectedPlans not empty → else 400 MISSING_FIELD
       │
       ├─ quotationService.getRequestFromDB("life", referenceId)
       │    └─ MongoDB: grid_based_quotation_request
       │         Find by referenceId → QuotationRequest (original user inputs)
       │
       ├─ Merge selected plan overrides into request
       │    (productCode, optionCode, policyTerm, premiumPaymentTerm, etc.)
       │
       ├─ Enrich request with IH integration metadata
       │    (integrationProvider, internalProductCode, fieldMappings)
       │
       └─ ihService.sendRequestToIH(provider, internalProductCode, broker,
                                    "LIFE", TMRequestType.PREMIUM, scope)
              │
              ├─ Calls Integration Hub → Insurer quote endpoint
              │
              └─ On success:
                   ├─ Map IH response → QuotationResponse
                   └─ Persist in MongoDB: grid_based_quotation_result
                        (referenceId + quoteId as identifiers)
  │
  ├─[QuotationResponse] → ResponseEntity 200 + PayloadWrapper.success(response)
  ├─[ErrorResponseData] → ResponseEntity 400 + PayloadWrapper.failure(error)
  └─[empty]             → ResponseEntity 200 empty
```

---

## Functions & Where Things Happen

| Function | File | What it does |
|---|---|---|
| `getSingleResult()` (controller) | `SachetController.java:130` | Entry point, delegates to aggregator |
| `getSingleResult()` (aggregator) | `SachetLifeAggregator.java` | Validates, loads DB request, calls IH |
| `getRequestFromDB()` | `IQuotationService` / impl | Loads original `QuotationRequest` from MongoDB |
| `sendRequestToIH()` | `IHService` | Dispatches enriched request to Integration Hub |
| `fetchResultFromDB()` | `IQuotationService` / impl | (Used in downstream flows) Loads existing result |

---

## Database / External Calls

| System | Collection / Endpoint | Operation | When |
|---|---|---|---|
| **MongoDB** | `grid_based_quotation_request` | Read — load original quote request | Always |
| **MongoDB** | `grid_based_quotation_result` | Write — persist fresh quote result | On IH success |
| **Integration Hub** | Insurer quote endpoint (per provider) | POST — compute premium | Always |

---

## Error Codes

| Scenario | HTTP | Error Code |
|---|---|---|
| `referenceId` blank | 400 | `MISSING_FIELD` |
| `selectedPlans` empty | 400 | `MISSING_FIELD` |
| Quote not found in DB | 400 | `NO_DATA_FOUND` |
| IH / insurer error | 400 | `SERVER_ERROR` |
