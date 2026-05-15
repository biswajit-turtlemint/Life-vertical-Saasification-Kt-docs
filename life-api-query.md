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
       │     │
       │     ├─ normalizeLifeRequestForValidation()
       │     │     Extracts planDetails, riskInsured.insuredMembers[0], proposerDetails
       │     │     Derives entryAge from DOB if not set; derives maturityAge/policyTerm/PPT
       │     │     Derives investmentTermCode (short/medium/long) from policyTerm
       │     │
       │     ├─ [IH Call 1] fetchEnabledProductsFromIntegrationHub(broker)
       │     │     LifeProductCatalogueService.fetchProductDetails(broker)
       │     │     ├─ Redis cache check (key: "LifeProductCatalogue_<broker>", TTL: 4h)
       │     │     │     Cache hit → return cached product list
       │     │     └─ Cache miss → IH Product Management API
       │     │           POST /api/product-management/v1/products/details/filters
       │     │           → List<ProductDetailsResponse> (all products for broker)
       │     │           → normalizeProductDetailRows(): flattens PDP fields per variant
       │     │              (planType, optionCode, payoutTypes, planFeatureList, riders, etc.)
       │     │           → cached in Redis for 4h
       │     │
       │     ├─ Filter enabled product rows:
       │     │     - matchesPlanType (if planType specified)
       │     │     - matchesPolicyType (if planType absent)
       │     │     - matchesCategories (categoryTags must contain all requested categories)
       │     │     - matchesPaymentFrequency (paymentFrequencyModes)
       │     │     - matchesCurrency
       │     │     - matchesSelectedPlans (if selectedPlans in body)
       │     │     - matchesInsurer (if x-provider header set)
       │     │
       │     ├─ applyLifePostFiltersWithDefaults()
       │     │     - applyPayoutDefaults (turtlemint broker only — sets default payoutType)
       │     │     - filterByPayoutSettings (payoutType, payoutTerm, payoutFrequency)
       │     │     - filterByPlanFeatures (jointLife, returnOfPremium flags)
       │     │
       │     ├─ createLifeValidatorRows()
       │     │     Transforms product master rows into validator row format
       │     │     Injects entryAge, paymentFrequency, maturityAge, policyTerm, PPT from request
       │     │
       │     └─ [IH Call 2] applyNearestMatch()
       │           POST /api/product-management/v1/life-validator
       │           Sends: productCodes[], scope { entryAge, policyTerm, PPT, ... }
       │           Returns: per-product nearest valid policyTerm + premiumPaymentTerm + score
       │           Products with no match are dropped (score=0 means exact match, kept as-is)
       │           Not cached — called on every request
       │
       │     Groups surviving rows by insurerCode → LifeValidationResult { rowsByProvider }
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
| `getLifeProductsQuery()` | `SachetController.java` | 156 | Entry point: deserializes body into `QuotationRequest`, injects productCode/tenant/broker/partnerId/provider from headers, runs on `boundedElastic` scheduler (blocking-safe), wraps result in `PayloadWrapper.success()` |
| `getLifeProducts()` | `LifeProductQueryService.java` | 26 | Guard: returns `[]` if productCode is not `"life"`. Builds configured providers list, calls `resolveValidatedProviders`, then calls `toLifeProductQueryResponse` to transform rows |
| `buildConfiguredProviders()` | `LifeProductQueryService.java` | 37 | Wraps `x-provider` header value into `[{"provider": "<value>"}]`. Returns empty list if header absent — meaning: include all configured providers |
| `resolveValidatedProviders()` | `LifeRequestValidationService.java` | 49 | Full product validation pipeline: normalize request → fetch enabled products (IH + Redis) → filter by plan/category/frequency → apply payout defaults → nearest-match (IH) → group by provider. Returns `LifeValidationResult { rowsByProvider }` |
| `resolveLifePremiumRequest()` | `LifeRequestValidationService.java` | 182 | Extracts `planDetails` map from the full `premiumRequest`. This is the canonical source for planType, categories, coverAmount, PPT, etc. |
| `normalizeLifeRequestForValidation()` | `LifeRequestValidationService.java` | 192 | Merges planDetails + riskInsured.insuredMembers[0] (age, gender, DOB) + proposerDetails into a flat map. Calls `applyLifeDefaultDerivations()` to fill in missing age/maturityAge/policyTerm/PPT/investmentTermCode |
| `applyLifeDefaultDerivations()` | `LifeRequestValidationService.java` | 235 | Derives entryAge from DOB if not present. Derives maturityAge from age+PT or age-based defaults per category (term→65/75, retirement→100, whole-life→95). Derives PT from maturityAge-entryAge. Derives PPT from PT. Sets investmentTermCode (short/medium/long) based on PT |
| `fetchEnabledProductsFromIntegrationHub()` | `LifeRequestValidationService.java` | 622 | Calls `LifeProductCatalogueService.fetchProductDetails(broker)`. Checks Redis cache first (TTL: 4h). On cache miss → calls IH Product Management API `POST /api/product-management/v1/products/details/filters`. Normalizes response via `normalizeProductDetailRows()` |
| `normalizeProductDetailRows()` | `LifeRequestValidationService.java` | 635 | Flattens PDP field definitions into a plain map per product variant. Handles two shapes: base PDP (shared across all variants) and variant-scoped PDP. Each output row contains: productCode, insurerCode, optionCode, planType, payoutTypes, planFeatureList, riders, etc. |
| `fetchEnabledProductMasters()` | `LifeRequestValidationService.java` | 578 | Applies 6 sequential filters to normalized rows: planType, policyType (when no planType), categories (categoryTags), paymentFrequency (paymentFrequencyModes), currency, selectedPlans, insurerCode filter |
| `applyLifePostFiltersWithDefaults()` | `LifeRequestValidationService.java` | 346 | Applies post-catalogue filters: `applyPayoutDefaults` (turtlemint only — sets payoutType from first matching product if not in request), `filterByPayoutSettings` (payoutType + payoutTerm + payoutFrequency), `filterByPlanFeatures` (jointLife, returnOfPremium) |
| `createLifeValidatorRows()` | `LifeRequestValidationService.java` | 884 | Transforms enabled product master rows into validator row format. Adds user's entryAge, paymentFrequency, maturityAge, defaultPolicyTerm, defaultPPT from the normalized request onto each row |
| `applyNearestMatch()` | `LifeRequestValidationService.java` | 986 | Calls `fetchNearestMatchMap()` → IH Life Validator `POST /api/product-management/v1/life-validator`. Sends all internalProductCodes + validation scope (entryAge, policyTerm, PPT, coverage, planType). IH returns nearest valid PT/PPT per product. Products with no matching entry are dropped. Updates rows with nearest policyTerm + premiumPaymentTerm + score |
| `fetchNearestMatchMap()` | `LifeRequestValidationService.java` | 1093 | HTTP POST to IH `/api/product-management/v1/life-validator` with `x-api-key` header. Parses response: for each product, picks the option with score=0 (exact match) or lowest-score nearest match. Returns Map of `internalProductCode[-optionCode]` → `NearestMatchValue { policyTerm, premiumPaymentTerm, score }` |
| `toLifeProductQueryResponse()` | `LifeProductQueryService.java` | 48 | Iterates all rows per provider. De-duplicates using key `insurerCode|productCode`. First occurrence → `initializeResponse()` (base fields). Subsequent rows → `mergeResponseFields()` to union values |
| `initializeResponse()` | `LifeProductQueryService.java` | 87 | Creates new `LifeProductQueryResponse`. Sets productCode, productName, insurerCode, insurerName (falls back to insurerCode if missing), planType |
| `mergeResponseFields()` | `LifeProductQueryService.java` | 97 | Unions into existing response: `incomeTerms` (from `payoutBenefitTerms` + `payoutTerm`), `payoutTypes` (normalized uppercase), `defermentPeriods` (from `defermentPeriod` + `calculatedDefermentPeriod`). Calls `mergePlanFeatures()` |
| `mergePlanFeatures()` | `LifeProductQueryService.java` | 116 | Reads `planFeatureList[]` first (each entry: `code` + `active` flag; skips inactive). Falls back to `planFeatureDetailsList[]` if `planFeatureList` is empty. Calls `applyFeatureCode()` for each entry |
| `applyFeatureCode()` | `LifeProductQueryService.java` | 141 | Normalizes feature code (lowercase, strip `_`, `-`, spaces). Sets `jointLife=true` if code is `"jointlife"`; sets `returnOfPremium=true` if code is `"returnofpremium"` |
| `normalizePayoutTypes()` | `LifeProductQueryService.java` | 155 | Trims and uppercases each payout type string (e.g., `"lump_sum"` → `"LUMP_SUM"`) |

---

## Database / External Calls

| System | Endpoint / Key | Operation | When |
|---|---|---|---|
| **Redis** | `LifeProductCatalogue_<broker>` (TTL: 4h) | Read — product catalogue cache | Every request (cache hit avoids IH call) |
| **IH Product Management API** | `POST /api/product-management/v1/products/details/filters` | Read — fetch all products + PDP data for broker | On Redis cache miss |
| **IH Life Validator API** | `POST /api/product-management/v1/life-validator` | Read — nearest-match PT/PPT/score per product | Every request (not cached) |
| **MongoDB** | — | Not used | Never |

---

## Error Codes

| Scenario | HTTP | Error Code |
|---|---|---|
| Body `data` is null / invalid JSON | 400 | `INVALID_REQUEST` |
| IH returns no data | 200 | Empty `[]` array |
