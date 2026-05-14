# Life API — POST `/query` (List Available Life Products)

**Full URL:** `POST /api/minterprise/v2/products/life/query`
**Service:** `transactional-flows`
**Controller:** `SachetController.java:152` → `getLifeProductsQuery()`

---

## Purpose

Returns the catalogue of Life insurance plans available for the given broker/tenant/partner context. Used by the UI to build the product-selection and filter UI before a user starts quoting. Each entry carries meta-attributes like payout types, income terms, deferment periods, joint-life flag, and return-of-premium flag.

---

## Sample cURL

```bash
curl --location 'http://localhost:9098/api/minterprise/v2/products/life/query' \
--header 'Content-Type: application/json' \
--header 'x-tenant: axisbank' \
--header 'x-broker: axisbank' \
--header 'x-partner-id: 647f458828effa0001044980' \
--header 'Authorization: <token>' \
--data-raw '{
  "data": {
    "premiumRequest": {
      "pospUserName": "647f458828effa0001044980",
      "personalDetails": {
        "customerName": "ABID AHMAD SHAH",
        "userMobile": "7400400747",
        "userEmail": "SHAHABID37@GMAIL.COM",
        "customerId": "8080515100"
      },
      "proposerDetails": {},
      "riskInsured": {
        "insuredMembers": [
          {
            "insuredFullName": "ABID AHMAD SHAH",
            "dateOfBirth": "03/05/1985",
            "entryAge": 40,
            "gender": "M"
          }
        ]
      },
      "planDetails": {
        "policyType": "TERM",
        "planType": "Term",
        "categories": ["term"],
        "coverAmount": 10000000,
        "paymentFrequency": 12,
        "profileType": "term",
        "businessModel": "B2B",
        "investmentRisk": "high",
        "investmentTermCode": "medium",
        "benifitCalculationRate": 8,
        "tmOccupation": "salaried",
        "tmQualification": "Graduate_Post_Graduate",
        "isSmoking": false,
        "maritalStatus": "SINGLE",
        "incomeBracketCode": "7 Lakhs",
        "minIncome": 500000,
        "maxIncome": 699999,
        "tmPincode": 192125,
        "policyTerm": 35,
        "premiumPaymentTerm": 35,
        "maturityAge": 75
      },
      "vertical": "LIFE"
    }
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
| `x-provider` | *(optional)* | Filter to a specific insurer |
| `Authorization` | `<token>` | Auth token |

### Body

Wrapped in `PayloadWrapper` → `data.premiumRequest`:

```json
{
  "data": {
    "premiumRequest": {
      "pospUserName": "647f458828effa0001044980",
      "personalDetails": { ... },
      "proposerDetails": {},
      "riskInsured": { "insuredMembers": [ ... ] },
      "planDetails": { ... },
      "vertical": "LIFE"
    }
  }
}
```

#### `personalDetails`

| Field | Type | Description |
|---|---|---|
| `customerName` | String | Full name of the customer |
| `userMobile` | String | Customer mobile number |
| `userEmail` | String | Customer email |
| `customerId` | String | Internal customer ID |

#### `riskInsured.insuredMembers[]`

| Field | Type | Description |
|---|---|---|
| `insuredFullName` | String | Full name |
| `dateOfBirth` | String | Format: `dd/MM/yyyy` |
| `entryAge` | Integer | Age at entry |
| `gender` | String | `M` or `F` |

#### `planDetails`

| Field | Type | Description |
|---|---|---|
| `policyType` | String | e.g., `TERM`, `ULIP` |
| `planType` | String | e.g., `Term` |
| `categories` | Array\<String\> | e.g., `["term"]` |
| `coverAmount` | Long | Sum assured in INR |
| `paymentFrequency` | Integer | `12`=Annual, `1`=Monthly |
| `profileType` | String | e.g., `term` |
| `businessModel` | String | `B2B` or `B2C` |
| `tmOccupation` | String | e.g., `salaried`, `self_employed` |
| `tmQualification` | String | e.g., `Graduate_Post_Graduate` |
| `isSmoking` | Boolean | Smoker flag |
| `maritalStatus` | String | e.g., `SINGLE`, `MARRIED` |
| `incomeBracketCode` | String | Income bracket label |
| `minIncome` / `maxIncome` | Long | Annual income range in INR |
| `tmPincode` | Integer | Customer's residence pincode |
| `policyTerm` | Integer | Policy term in years |
| `premiumPaymentTerm` | Integer | PPT in years |
| `maturityAge` | Integer | Age at maturity |

Context fields injected from headers (no need to send in body):

| Field | Source |
|---|---|
| `productCode` | Path param (always `"life"`) |
| `tenant` | `x-tenant` header |
| `broker` | `x-broker` header |
| `partnerId` | `x-partner-id` header |
| `provider` | `x-provider` header |

If `data` is `null` or un-parseable → `400 INVALID_REQUEST` returned immediately before any service call.

---

## Response

### Success (HTTP 200)

```json
{
  "data": [
    {
      "productCode": "HDFC_CLICK_2_PROTECT_LIFE",
      "productName": "Click 2 Protect Life",
      "insurerCode": "HDFC",
      "insurerName": "HDFC Life",
      "planType": "TERM",
      "incomeTerms": [5, 10, 20],
      "payoutTypes": ["LUMP_SUM", "INCOME"],
      "defermentPeriods": [1, 5],
      "jointLife": false,
      "returnOfPremium": true
    }
  ],
  "meta": { "status": "SUCCESS", "error": false }
}
```

### Response Fields (`LifeProductQueryResponse.java`)

| Field | Type | Description |
|---|---|---|
| `productCode` | String | Insurer-specific product identifier |
| `productName` | String | Display name of the plan |
| `insurerCode` | String | Insurer short code |
| `insurerName` | String | Insurer full name |
| `planType` | String | e.g., `TERM`, `ULIP`, `ENDOWMENT` |
| `incomeTerms` | Set\<Integer\> | Available income payout term years |
| `payoutTypes` | Set\<String\> | Available payout options (normalized uppercase) |
| `defermentPeriods` | Set\<Integer\> | Available deferment period years |
| `jointLife` | Boolean | `true` if plan supports joint life |
| `returnOfPremium` | Boolean | `true` if plan supports ROP feature |

### Failure (HTTP 400)

```json
{
  "data": {
    "errors": [{ "field": "data", "message": "INVALID_REQUEST", "errorCode": "INVALID_REQUEST" }]
  },
  "meta": { "status": "FAILURE", "error": true }
}
```

---

## Flowchart

```
Client
  │
  │  POST /api/minterprise/v2/products/life/query
  │  Headers: x-tenant, x-broker, x-partner-id, x-provider
  │  Body: { "data": { ... } }
  ▼
SachetController.getLifeProductsQuery()
  │
  ├─[null data]─► 400 INVALID_REQUEST
  │
  ├─ Parse body → QuotationRequest
  ├─ Inject: productCode="life", tenant, broker, partnerId, provider
  │
  └─ Mono.fromCallable (boundedElastic scheduler)
       │
       ▼
  LifeProductQueryService.getLifeProducts(QuotationRequest)
       │
       ├─ Guard: productCode != "life" → return []
       │
       ├─ buildConfiguredProviders(provider)
       │     If x-provider header present → [{ "provider": "<value>" }]
       │     Else → []
       │
       ├─ LifeRequestValidationService.resolveValidatedProviders(request, providers)
       │     └─► Integration Hub API call
       │           Fetches configured life product rows for broker
       │           Filtered by provider if specified
       │           Returns LifeValidationResult {
       │             rowsByProvider: Map<String, List<Map<String, Object>>>
       │           }
       │
       └─ toLifeProductQueryResponse(validationResult)
             │
             For each provider → each row:
               ├─ Key = insurerCode|productCode
               ├─ De-dup: same key → merge fields into existing entry
               ├─ Merge incomeTerms  ← payoutBenefitTerms + payoutTerm
               ├─ Merge payoutTypes  ← normalized to uppercase
               ├─ Merge defermentPeriods ← defermentPeriod + calculatedDefermentPeriod
               └─ planFeatureList / planFeatureDetailsList scan:
                    "jointlife"       → setJointLife(true)
                    "returnofpremium" → setReturnOfPremium(true)
               (feature code normalization: lowercase, strip _, -, spaces)
             │
             └─► List<LifeProductQueryResponse>
  │
  └─ PayloadWrapper.success(response)
       │
       ▼
  HTTP 200
```

---

## Functions & Where Things Happen

| Function | File | Line | What it does |
|---|---|---|---|
| `getLifeProductsQuery()` | `SachetController.java` | 152 | Entry point, validates body, injects context, delegates |
| `getLifeProducts()` | `LifeProductQueryService.java` | 26 | Orchestrates validation + response building |
| `buildConfiguredProviders()` | `LifeProductQueryService.java` | 37 | Wraps `x-provider` header into provider list |
| `resolveValidatedProviders()` | `LifeRequestValidationService.java` | — | Calls IH to get configured provider-product rows |
| `toLifeProductQueryResponse()` | `LifeProductQueryService.java` | 48 | De-dups and merges rows into response objects |
| `mergeResponseFields()` | `LifeProductQueryService.java` | 97 | Appends income terms, payout types, deferment periods |
| `mergePlanFeatures()` | `LifeProductQueryService.java` | 116 | Sets `jointLife` / `returnOfPremium` flags |
| `applyFeatureCode()` | `LifeProductQueryService.java` | 141 | Normalizes feature code and flips boolean flags |

---

## Database / External Calls

| System | What | When |
|---|---|---|
| **Integration Hub** | Fetch configured providers & product master rows for broker | Always |
| **MongoDB** | Not used | — |

---

## Error Codes

| Scenario | HTTP | Error Code |
|---|---|---|
| Body `data` is null / invalid JSON | 400 | `INVALID_REQUEST` |
| IH returns no data | 200 | Empty `[]` array |
