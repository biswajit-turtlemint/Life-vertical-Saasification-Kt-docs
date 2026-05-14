# Life API — POST `/getBenefitIllustration` (Benefit Illustration)

**Full URL:** `POST /api/minterprise/v2/life/getBenefitIllustration`
**Service:** `transactional-flows`
**Controller:** `VerticalController.java:24` → `getBenefitIllustration()`

---

## Purpose

Fetches a year-by-year benefit illustration for a selected life quote — showing how death benefit, maturity benefit, and bonus accumulate over the policy term. The illustration is insurer-specific and is fetched by calling the insurer's Benefit Illustration API via Integration Hub (IH). The insurer field mappings and constants are resolved internally before dispatching to IH.

---

## Sample cURL

```bash
curl --location 'http://localhost:9098/api/minterprise/v2/life/getBenefitIllustration' \
--header 'authorization: <token>' \
--header 'content-type: application/json' \
--header 'x-broker: axisbank' \
--header 'x-tenant: axisbank' \
--data '{
    "data": {
        "referenceId": "AHXK5LLBAPU",
        "quoteId": "fc54e1df-be7a-4ed2-bf15-4a7c4380393b"
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
| `Authorization` | `<token>` | Auth token |

### Body

Wrapped in `PayloadWrapper`:

```json
{
  "data": {
    "referenceId": "AHXK5LLBAPU",
    "quoteId": "fc54e1df-be7a-4ed2-bf15-4a7c4380393b"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `quoteId` | String | **Always mandatory** | UUID of the specific quote variant — returned by the quote engine (visible in `/query` or `/singleresult` response) |
| `referenceId` | String | One of (`referenceId` OR `premiumResultId`) | Quote session ID |
| `premiumResultId` | String | One of | Direct MongoDB `_id` of the `QuotationResult` document — alternative when `referenceId` is not available |

If `quoteId` is blank → `400 MISSING_FIELD` immediately.
If both `referenceId` and `premiumResultId` are blank → `400 MISSING_FIELD`.

---

## Response

The response shape is **insurer-specific** — it is returned as-is from IH after processing. Typical shape:

```json
{
  "data": {
    "referenceId": "AHXFK8QMWHY",
    "quoteId": "QT_HDFC_001",
    "benefitIllustration": {
      "yearWiseData": [
        { "year": 1,  "deathBenefit": 5000000, "maturityBenefit": 0,       "bonus": 0 },
        { "year": 5,  "deathBenefit": 5000000, "maturityBenefit": 0,       "bonus": 25000 },
        { "year": 10, "deathBenefit": 5000000, "maturityBenefit": 1200000, "bonus": 50000 },
        { "year": 20, "deathBenefit": 5000000, "maturityBenefit": 3500000, "bonus": 120000 }
      ],
      "disclaimers": [
        "Illustrations are indicative only and are based on current rates.",
        "Bonus is not guaranteed."
      ]
    }
  },
  "meta": { "status": "SUCCESS", "error": false }
}
```

### Error Response (HTTP 400)

```json
{
  "data": {
    "errors": [{ "field": "quoteId", "message": "quoteId is mandatory", "errorCode": "MISSING_FIELD" }]
  },
  "meta": { "status": "FAILURE", "error": true }
}
```

---

## Flowchart

```
Client
  │
  │  POST /api/minterprise/v2/life/getBenefitIllustration
  │  Body: { "data": { referenceId, quoteId, premiumResultId } }
  ▼
VerticalController.getBenefitIllustration()
  │
  ├─ Parse PayloadWrapper.data → BenefitIllustrationRequest
  │
  └─ IVerticalService.getBenefitIllustration(request, "life")
       │
       └─ VerticalServiceImpl → ISachetProductAggregator.fetchBenefitIllustration(request)
              │
              └─ SachetLifeAggregator.fetchBenefitIllustration(request)  [line 3196]
                   │
                   ├─ Guard: quoteId blank
                   │    └─► 400 MISSING_FIELD "quoteId is mandatory"
                   │
                   ├─ Guard: referenceId AND premiumResultId both blank
                   │    └─► 400 MISSING_FIELD "referenceId is mandatory"
                   │
                   ├─[referenceId present]
                   │    ├─ quotationResultMono =
                   │    │    quotationService.fetchResultFromDB("life", referenceId, quoteId, null)
                   │    │    MongoDB: grid_based_quotation_result
                   │    └─ quotationRequestMono =
                   │         quotationService.getRequestFromDB("life", referenceId)
                   │         MongoDB: grid_based_quotation_request
                   │
                   ├─[premiumResultId present]
                   │    ├─ quotationResultMono =
                   │    │    quotationResultRepository.findById(premiumResultId)
                   │    │    MongoDB: grid_based_quotation_result (by _id)
                   │    │    → NoRecordFoundException if not found
                   │    └─ quotationRequestMono =
                   │         quotationService.getRequestFromDB("life", result.referenceId)
                   │
                   └─ Mono.zip(quotationRequestMono, quotationResultMono)
                        │
                        ├─ extractBenefitIllustrationPremiumResponse(quotationResult, quoteId)
                        │    Extracts the specific quote's ObjectNode from the result document
                        │    → null if quoteId not found in result
                        │    └─► null → 400 "Life quote not found for referenceId=... quoteId=..."
                        │
                        ├─ Build LifeIntegrationRequestMapper:
                        │    mapPremiumRequestDetailsForBI(quotationRequest, mapper)
                        │      Copies: broker, tenant, partnerId, age, gender, smoker status,
                        │              annualIncome, pincode, etc.
                        │    mapPremiumFieldsForBI(premiumResponseNode, mapper)
                        │      Copies: insurerCode, productCode, optionCode, sumAssured,
                        │              policyTerm, premiumPaymentTerm, paymentFrequency,
                        │              integrationProvider, internalProductCode, quoteId, uid
                        │
                        ├─ Guard: integrationProvider OR internalProductCode blank
                        │    └─► 400 "Benefit illustration is not configured for the selected life quote"
                        │
                        ├─ alignBenefitIllustrationRequestWithLifeService(mapper)
                        │    Adjusts field values to match the BI API's expected format
                        │    (e.g., frequency normalization, field renames per insurer standard)
                        │
                        ├─ enrichBenefitIllustrationRequestForInsurer(quotationRequest, mapper)
                        │    Insurer-specific enrichment (e.g., plan code aliases, extra params)
                        │
                        ├─ enrichBenefitIllustrationPlanCodeIfRequired(quotationRequest, mapper)
                        │    May override internalProductCode based on insurer config
                        │
                        └─ Mono.zip (parallel):
                             │
                             ├─ fetchInsurerConstants(integrationProvider, broker, LifeStage.PREMIUM)
                             │    IH constants for the insurer (field mappings, enum values)
                             │
                             └─ buildBenefitIllustrationMasterMapper(quotationRequest, integrationProvider)
                                  Master data for field resolution
                             │
                             └─ ihService.sendRequestToIH(
                                    integrationProvider,    ← insurer code
                                    internalProductCode,    ← insurer's internal product ID
                                    broker,
                                    "LIFE",
                                    TMRequestType.BENEFIT_ILLUSTRATION,
                                    scope { request: mapper, master, constants })
                                  │
                                  ├─► Integration Hub → Insurer BI API
                                  │
                                  └─ processBenefitIllustrationResponse(ihResp, provider, customerName, quoteId)
                                       Transforms IH response → final response object
  │
  ├─[ErrorResponseData] → PayloadWrapper.generateResponse(error, MetaInfo("FAILURE", true))
  └─[success]           → PayloadWrapper.generateResponse(response, null)
                              │
                              ▼
                         HTTP 200
```

---

## Error Handling

| Exception | Response |
|---|---|
| `NoRecordFoundException` (result not found by `premiumResultId`) | 400 — "referenceId not found" |
| `quoteId` not in the result document | 400 — "Life quote not found for referenceId=... quoteId=..." |
| `integrationProvider` or `internalProductCode` blank | 400 — "Benefit illustration is not configured" |
| Any other exception | 400 — "Unable to generate benefit illustration right now" |

---

## Functions & Where Things Happen

| Function | File | Line | What it does |
|---|---|---|---|
| `getBenefitIllustration()` (controller) | `VerticalController.java` | 24 | Entry point, parses `BenefitIllustrationRequest`, delegates |
| `getBenefitIllustration()` (service) | `VerticalServiceImpl.java` | — | Routes to correct aggregator |
| `fetchBenefitIllustration()` (aggregator) | `SachetLifeAggregator.java` | 3196 | All BI logic lives here |
| `extractBenefitIllustrationPremiumResponse()` | `SachetLifeAggregator.java` | — | Pulls the matching quote's ObjectNode from result doc |
| `mapPremiumRequestDetailsForBI()` | `SachetLifeAggregator.java` | — | Maps original request fields onto BI mapper |
| `mapPremiumFieldsForBI()` | `SachetLifeAggregator.java` | — | Maps premium response fields onto BI mapper |
| `alignBenefitIllustrationRequestWithLifeService()` | `SachetLifeAggregator.java` | — | Normalizes fields for BI API format |
| `enrichBenefitIllustrationRequestForInsurer()` | `SachetLifeAggregator.java` | — | Insurer-specific enrichment |
| `enrichBenefitIllustrationPlanCodeIfRequired()` | `SachetLifeAggregator.java` | — | May override internal product code |
| `fetchInsurerConstants()` | `SachetLifeAggregator.java` | — | Fetches IH insurer constants |
| `buildBenefitIllustrationMasterMapper()` | `SachetLifeAggregator.java` | — | Fetches master data for field resolution |
| `sendRequestToIH()` | `IHService` | — | Dispatches to Integration Hub BI endpoint |
| `processBenefitIllustrationResponse()` | `SachetLifeAggregator.java` | — | Transforms IH response to final shape |

---

## Database / External Calls

| System | Collection / Endpoint | Operation | When |
|---|---|---|---|
| **MongoDB** | `grid_based_quotation_result` | Read — by `referenceId + quoteId` or by `_id` | Always |
| **MongoDB** | `grid_based_quotation_request` | Read — by `referenceId` | Always |
| **Integration Hub** | Insurer BI endpoint (`TMRequestType.BENEFIT_ILLUSTRATION`) | POST | Always (after DB fetch) |

---

## Error Codes

| Scenario | HTTP | Error Code |
|---|---|---|
| `quoteId` blank | 400 | `MISSING_FIELD` |
| Both `referenceId` and `premiumResultId` blank | 400 | `MISSING_FIELD` |
| Result doc not found (`premiumResultId` path) | 400 | `INVALID_REQUEST` (via NoRecordFoundException) |
| `quoteId` not found in result | 400 | `INVALID_REQUEST` |
| BI not configured for plan | 400 | `INVALID_REQUEST` |
| IH / upstream failure | 400 | `INVALID_REQUEST` |
