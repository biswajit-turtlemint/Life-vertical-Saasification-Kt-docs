# Life API — POST `/singleresult` (Fetch Single Life Quote)

**Full URL:** `POST /api/minterprise/v2/products/life/singleresult`
**Service:** `transactional-flows`
**Controller:** `SachetController.java:130` → `getSingleResult()`
**Aggregator:** `SachetLifeAggregator.java:879` → `getSingleResult()`

---

## Purpose

Re-fetches a premium quote for one specific life plan variant the user has selected (e.g., after changing riders or sum assured). Unlike `/query` which returns all available plans, this re-quotes a single product by calling the insurer's premium API again via Integration Hub, then saves the updated result to MongoDB. The result is used downstream for comparison, proposal, and issuance.

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
| `x-provider` | *(optional)* | Filter to a specific insurer |
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
| `requestId` | String | **Yes** | Quote session reference ID — same as `referenceId` in other APIs. Also accepted as `referenceId` |
| `vertical` | String | No | Always `"LIFE"` — informational only |
| `keys` | Array\<String\> | **Yes** | Product code(s) to re-quote (e.g., `["P126"]`). If absent, derived from `riderMeta[].productCode` |
| `subKey` | Integer | No | Option code / variant index — informs which variant to fetch |
| `riderMeta` | Array | No | Selected rider configuration per product — see below |
| `offerMeta` | Array | No | Offer metadata (usually empty `[]`) |

> **Key normalization:** Keys like `"P126_2"` are normalized by stripping everything after the first `_` → `"P126"`. If `keys[]` is empty, the aggregator derives keys from `riderMeta[].productCode`, `offerMeta[].productCode`, `ulipFundAllocationInfos[].productCode`.

#### `riderMeta[]` fields

| Field | Type | Description |
|---|---|---|
| `subKey` | Integer | Option code — matches the top-level `subKey` |
| `productCode` | String | Insurer product code (e.g., `P126`) |
| `optionCode` | Integer | Plan variant/option code |
| `premiumPaymentTerm` | Integer | PPT in years |
| `policyTerm` | Integer | Policy term in years |
| `sumAssured` | Long | Sum assured in INR |
| `basePlanPremium` | Long | Base plan premium (without riders) |
| `riderInfoList` | Array | List of rider selections — see below |

#### `riderMeta[].riderInfoList[]` fields

| Field | Type | Description |
|---|---|---|
| `riderCode` | String | Internal rider code (e.g., `R75`) |
| `riderName` | String | Display name of rider |
| `riderCategory` | String | Category label (e.g., `Critical Illness`) |
| `riderApiCode` | String | Insurer's API code for the rider |
| `isCoverAmountEditable` | Boolean | Can the rider SA be changed by user |
| `isCoverAmountIncludedInBasePlan` | Boolean | Whether rider SA is part of base SA |
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

Returns a `QuotationResponse` wrapped in `PayloadWrapper`. The shape mirrors the `/query` response but contains only the re-quoted plan:

```json
{
  "data": {
    "referenceId": "AHX5828XJB7",
    "quotes": [
      {
        "quoteId": "some-uuid",
        "productCode": "P126",
        "productName": "HDFC Click 2 Protect Life",
        "insurerCode": "HDFCLI",
        "premium": 139956,
        "premiumWithTax": 165148.08,
        "sumAssured": 10000000,
        "policyTerm": 30,
        "premiumPaymentTerm": 25,
        "riderList": [ ... ],
        "status": "SUCCESS"
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
    "errors": [{ "field": "referenceId", "message": "referenceId or requestId is mandatory", "errorCode": "MISSING_FIELD" }]
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
  │  Body: { "data": { requestId, keys, subKey, riderMeta, offerMeta } }
  ▼
SachetController.getSingleResult()
  │
  └─ ISachetProductAggregatorFactory.getPremiumAggregator("life")
       └─ Returns SachetLifeAggregator
  │
  └─ SachetLifeAggregator.getSingleResult(payload, tenant, broker, partnerId, provider)
       │                                                            [line 879]
       │
       ├─ resolveSingleResultPayload(requestPayload)
       │    Handles double-wrapped payloads: if payload.data contains requestId/keys/riderMeta,
       │    unwraps the inner data object (handles client sending data.data edge case)
       │
       ├─ referenceId = requestId OR referenceId from payload (either field accepted)
       │
       ├─ resolveSingleResultProductKeys(payload)
       │    1st: reads payload.keys[] (normalized: "P126_2" → "P126", deduplicated)
       │    Fallback: if keys[] empty → derives from riderMeta[].productCode,
       │              offerMeta[].productCode, ulipFundAllocationInfos[].productCode
       │
       ├─ Guard: referenceId blank
       │    └─► 400 MISSING_FIELD "referenceId or requestId is mandatory"
       │
       ├─ Guard: requestedProductKeys empty
       │    └─► 400 MISSING_FIELD "keys are mandatory"
       │
       ├─ quotationService.getRequestFromDB("life", referenceId)
       │    MongoDB: sachetPremiumRequest (by referenceId)
       │    → QuotationRequest (original user inputs — age, gender, coverage, plan details)
       │    switchIfEmpty → 400 NO_DATA_FOUND "quote not found"
       │
       └─ triggerLifeSingleResultQuote(storedRequest, payload, requestedKeys, ...)
              │                                                    [line 1231]
              │
              ├─ Clone storedRequest → singleResultRequest
              │    Override: referenceId, tenant, broker, partnerId, provider (from headers)
              │
              ├─ buildSingleResultSelectionPayload(payload, requestedKeys)
              │    Extracts riderMeta and offerMeta rows whose productCode matches requestedKeys
              │    → selections map with rider/offer choices for the re-quote
              │
              ├─ enrichSingleResultSelectionsWithStoredQuoteIds(storedRequest, requestedKeys, ...)
              │    Loads existing QuotationResult rows for this referenceId from MongoDB
              │    For each requested product key, finds the stored quoteId (UUID)
              │    Adds stored quoteId to selections so IH knows which prior quote to re-quote
              │    → singleResultSelections stored as _lifeSingleResultSelections in otherDetails
              │
              ├─ buildSingleResultInsurerSpecificInfo(storedRequest, singleResultRequest, keys)
              │    Validates each requested key exists in stored quotes (has insurerCode, productCode)
              │    Assembles insurer-specific metadata needed for IH re-quote call
              │    (insurerCode, internalProductCode, fieldMappings per product)
              │    If empty → 400 NO_DATA_FOUND "quote not found" (product not in session)
              │    → stored as _lifeRequoteInsurerSpecificInfo in otherDetails
              │
              ├─ buildLifeConfiguredProviders(singleResultRequest)
              │    Determines which insurer providers to include in the re-quote call
              │    (Uses provider header or derives from stored insurer info)
              │
              ├─ lifeRequestValidationService.resolveValidatedProviders(request, providers)
              │    Integration Hub call: validates providers and fetches product configuration rows
              │    → LifeValidationResult { rowsByProvider }
              │
              ├─ filterValidationResultForRequote(singleResultRequest, validationResult)
              │    Filters IH validation result to only include rows matching the requested keys
              │
              ├─ buildLifeServiceRequests(singleResultRequest, effectiveValidationResult)
              │    Creates one QuotationRequest per product×provider combination
              │    Embeds rider selections, sum assured, policyTerm, PPT into each request
              │    If empty → 400 NO_DATA_FOUND (no valid products found after filtering)
              │
              ├─ premiumServiceUtil.getIHPremiumResponse(serviceRequests, singleResultRequest)
              │    Fires IH premium API calls for each service request (reactive, parallel)
              │    IH calls insurer's premium endpoint with the updated selections
              │    Saves updated QuotationResult to MongoDB (sachetPremiumResponse)
              │
              ├─ saveLifeLeadForQuoteJourney()
              │    Updates the lead stage to PROPOSAL in the lead/order tracking system
              │
              ├─ fetchLifeAggregateResponse(referenceId, runId, null, true, requestedKeys)
              │    Re-reads the freshly saved results from MongoDB
              │    Merges and enriches the quote response (companyDetails, riders, offers, etc.)
              │    → QuotationResponse with full enriched quote data
              │
              └─ filterLifeSingleResultResponse(singleResultRequest, response)
                   Filters the aggregated response to only include the requested product keys
                   Returns the final single-plan QuotationResponse
  │
  ├─[QuotationResponse] → ResponseEntity 200 + PayloadWrapper.generateResponse(response, null)
  ├─[ErrorResponseData] → ResponseEntity 400 + PayloadWrapper.generateResponse(error, FAILURE)
  └─[empty]             → ResponseEntity 200 empty
```

---

## Functions & Where Things Happen

| Function | File | Line | What it does |
|---|---|---|---|
| `getSingleResult()` (controller) | `SachetController.java` | 130 | Entry point — reads headers, calls aggregator, wraps response in ResponseEntity |
| `getSingleResult()` (aggregator) | `SachetLifeAggregator.java` | 879 | Input validation, DB lookup, delegates to triggerLifeSingleResultQuote |
| `resolveSingleResultPayload()` | `SachetLifeAggregator.java` | 1456 | Unwraps doubly-wrapped payloads (handles `data.data` edge case) |
| `resolveSingleResultProductKeys()` | `SachetLifeAggregator.java` | 1487 | Reads `keys[]`; if empty falls back to productCodes in riderMeta/offerMeta |
| `normalizeRequestedProductKey()` | `SachetLifeAggregator.java` | 1513 | Strips option-code suffix from key (e.g., `P126_2` → `P126`) |
| `getRequestFromDB()` | `IQuotationService` impl | — | MongoDB: queries `sachetPremiumRequest` by `referenceId` to load original user inputs |
| `triggerLifeSingleResultQuote()` | `SachetLifeAggregator.java` | 1231 | Orchestrates the full re-quote: clone → enrich → IH call → save → fetch |
| `buildSingleResultSelectionPayload()` | `SachetLifeAggregator.java` | — | Extracts rider/offer choices from request body for the requested product keys |
| `enrichSingleResultSelectionsWithStoredQuoteIds()` | `SachetLifeAggregator.java` | — | Loads prior quoteIds from stored results to include in re-quote request to IH |
| `buildSingleResultInsurerSpecificInfo()` | `SachetLifeAggregator.java` | — | Assembles insurer metadata (insurerCode, internalProductCode) from stored quotes |
| `resolveValidatedProviders()` | `LifeRequestValidationService` | — | IH call: validates providers and fetches product config rows for broker |
| `filterValidationResultForRequote()` | `SachetLifeAggregator.java` | — | Narrows IH validation rows to only the requested product keys |
| `buildLifeServiceRequests()` | `SachetLifeAggregator.java` | — | Creates per-product QuotationRequest objects with all IH fields embedded |
| `getIHPremiumResponse()` | `PremiumServiceUtil` | — | Fires IH premium calls, saves QuotationResult to `sachetPremiumResponse` |
| `saveLifeLeadForQuoteJourney()` | `SachetLifeAggregator.java` | — | Updates lead stage to PROPOSAL in lead tracking |
| `fetchLifeAggregateResponse()` | `SachetLifeAggregator.java` | — | Reads saved results from MongoDB and builds enriched QuotationResponse |
| `filterLifeSingleResultResponse()` | `SachetLifeAggregator.java` | — | Filters aggregated response to only the requested keys |

---

## Database / External Calls

| System | Collection / Endpoint | Operation | When |
|---|---|---|---|
| **MongoDB** | `sachetPremiumRequest` | Read — load original quote inputs by `referenceId` | Always |
| **MongoDB** | `sachetPremiumResponse` | Read — load prior results to find stored quoteIds | During enrichment |
| **Integration Hub** | Provider validation API | GET — fetch product config rows for broker | Always |
| **Integration Hub** | Insurer premium endpoint (per provider) | POST — compute updated premium with rider selections | Always |
| **MongoDB** | `sachetPremiumResponse` | Write — persist updated quote result | After IH success |

---

## Error Codes

| Scenario | HTTP | Error Code | Message |
|---|---|---|---|
| `requestId`/`referenceId` blank | 400 | `MISSING_FIELD` | `referenceId or requestId is mandatory` |
| `keys` empty (and no riderMeta productCodes) | 400 | `MISSING_FIELD` | `keys are mandatory` |
| Quote request not found in DB | 400 | `NO_DATA_FOUND` | `quote not found` |
| Requested product key not in stored quotes | 400 | `NO_DATA_FOUND` | `quote not found` |
| IH validation returns no matching products | 400 | `NO_DATA_FOUND` | `quote not found` |
| IH / insurer premium error | 400 | `SERVER_ERROR` | insurer-specific message |
