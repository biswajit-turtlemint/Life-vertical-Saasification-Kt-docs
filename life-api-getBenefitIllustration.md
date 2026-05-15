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

The API fetches the BI PDF from the insurer (via IH), uploads it to S3 via File Service, and returns a file reference — **not** year-wise data.

```json
{
  "data": {
    "fileId": "abc123pid",
    "fileName": "<resultId>_<insurerCode>_<customerName>_<dd_MM_yyyy'T'HH_mm_ss>.pdf",
    "fileUrl": "https://s3.example.com/benefit-illustration/..."
  },
  "meta": { "status": "SUCCESS", "error": false }
}
```

| Field | Description |
|---|---|
| `fileId` | `pid` from File Service upload response — used to retrieve or display the PDF |
| `fileName` | Auto-generated: `{resultId}_{insurerCode}_{customerName}_{timestamp}.pdf` |
| `fileUrl` | Pre-signed / view URL returned by `fileService.getViewFileUrl()` |

**How the PDF gets to S3:**
1. IH calls the insurer's BI API.
2. Insurer responds with either:
   - A `pdfResponse.url` (BAJAJLI, HDFCLI, ICICIPRULI) — service downloads the PDF bytes directly from that URL.
   - A `data` field containing base64-encoded PDF bytes (Federal, MaxLife, ISHDFCLI).
3. The raw PDF bytes are uploaded to S3 via File Service (`multipart/form-data`).
4. File Service returns `processInfo.pid` (the `fileId`) and a view URL.

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
                   │    │    MongoDB: sachetPremiumResponse
                   │    └─ quotationRequestMono =
                   │         quotationService.getRequestFromDB("life", referenceId)
                   │         MongoDB: sachetPremiumRequest
                   │
                   ├─[premiumResultId present]
                   │    ├─ quotationResultMono =
                   │    │    quotationResultRepository.findById(premiumResultId)
                   │    │    MongoDB: sachetPremiumResponse (by _id)
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
| `getBenefitIllustration()` (controller) | `VerticalController.java` | 24 | Entry point: unwraps `PayloadWrapper.data` → `BenefitIllustrationRequest`, delegates to service, maps `ErrorResponseData` to FAILURE meta |
| `getBenefitIllustration()` (service) | `VerticalServiceImpl.java` | 19 | Thin delegation: uses `ISachetProductAggregatorFactory.getPremiumAggregator("life")` → calls `SachetLifeAggregator.fetchBenefitIllustration()` |
| `fetchBenefitIllustration()` | `SachetLifeAggregator.java` | 3196 | All BI orchestration: guards → DB load (two paths) → quote extraction → mapper build → normalization → IH dispatch → S3 upload |
| `extractBenefitIllustrationPremiumResponse()` | `SachetLifeAggregator.java` | ~3831 | Reads `premiumResponse` ObjectNode from `QuotationResult`. Calls `reorderResponseOptionIfRequired()` which promotes a matching `responseOptions[]` variant to the root node. Returns `null` if `quoteId` not matched |
| `mapPremiumRequestDetailsForBI()` | `SachetLifeAggregator.java` | 6141 | Reads the **request** side of the stored quote (age, gender, DOB, smoker status, income, pincode, state, payment frequency, riders, payout params) from `QuotationRequest.premiumRequest` and populates `LifeIntegrationRequestMapper` |
| `mapPremiumFieldsForBI()` | `SachetLifeAggregator.java` | ~5267 | Reads the **response** side of the quote node (insurerCode → integrationProvider, internalProductCode, quoteId, premium, sumAssured, policyTerm, PPT, optionCode, payout fields, resultId → uid). BI-specific vs standard `mapPremiumFields()` which lacks integrationProvider and extra payout fields |
| `alignBenefitIllustrationRequestWithLifeService()` | `SachetLifeAggregator.java` | ~5327 | Formats DOB/propDOB for insurer: ICICIPRULI → `yyyy-MM-dd`, others → `dd/MM/yyyy`. Normalizes premium/tax/totalPremium/sumAssured strings (strips trailing `.0`) |
| `enrichBenefitIllustrationRequestForInsurer()` | `SachetLifeAggregator.java` | ~5341 | ICICIPRULI only: uppercases `insuranceType`, resolves `investmentReturns`/`inputBonus` from `benifitCalculationRate` fallback chain, defaults `businessModel` to `"B2B"`, resolves `maritalStatus` |
| `enrichBenefitIllustrationPlanCodeIfRequired()` | `SachetLifeAggregator.java` | ~5398 | Only runs for `BI_PLANCODE_INSURERS` (TATAAIALI, DIGITLI). Calls Rule Engine `POST /api/rules/v0/life/planCodes` with productCode, optionCode, policyTerm, PPT, age, paymentFrequency, premium, smoker. Sets `planCode` and optionally overrides `sumAssured` (baseSumAssured from response). Failures are swallowed |
| `fetchInsurerConstants()` | `SachetLifeAggregator.java` | ~5836 | Queries MongoDB `lifeInsurerProviderMeta` by `insurerCode + stage("PREMIUM") + broker`. Returns `constants: Map<String, Map<String,String>>` — insurer-specific key-value configs (agent codes, API credentials, partner IDs) that IH needs to authenticate with the insurer |
| `buildBenefitIllustrationMasterMapper()` | `SachetLifeAggregator.java` | ~5491 | Parallel `Mono.zip`: calls Master Service v2 for occupation code mapping (`GET /api/v2/masters/turtlemint/{insurer}/OCCUPATION/LIFE/filter?occupation_code={value}`) and qualification code mapping. Returns map with `occupationMappingResponse` and `qualificationMappingResponse`. Short-circuits if both tmOccupation and tmQualification are blank |
| `sendRequestToIH()` | `IHService` | — | Dispatches `IntegrationScopeMapper` (request + master + constants) to Integration Hub with request type `BENEFIT_ILLUSTRATION`. IH routes to the correct insurer adapter based on `integrationProvider` + `internalProductCode` |
| `processBenefitIllustrationResponse()` | `SachetLifeAggregator.java` | ~3841 | Type 1 (`pdfResponse` key): calls `downloadBIPdfFromInsurer()` → `uploadBIPdfToS3()`. Type 2 (`data` key as base64 string): decodes → `uploadBIPdfToS3()`. Fallback: returns raw data map. Error checks: null IH response, `isSuccess=false`, empty data |
| `downloadBIPdfFromInsurer()` | `SachetLifeAggregator.java` | ~3899 | Fetches PDF bytes from insurer URL via `HttpsURLConnection`. Runs on `boundedElastic` to avoid blocking reactor thread. Handles `apiParams=true` (builds query string from body). Handles ASP.NET postback for BAJAJ-style HTML-first redirect flows |
| `uploadBIPdfToS3()` | `SachetLifeAggregator.java` | 4132 | Validates bytes are valid PDF (magic bytes check). Generates filename: `{resultId}_{insurerCode}_{customerName}_{timestamp}.pdf`. Uploads to File Service via `multipart/form-data`. Calls `fileService.getViewFileUrl()` for the pre-signed URL. Returns `{fileId, fileName, fileUrl}` |

---

## Database / External Calls

| System | Collection / Endpoint | Operation | When |
|---|---|---|---|
| **MongoDB** | `sachetPremiumResponse` | Read — by `referenceId + quoteId` or by `_id` | Always |
| **MongoDB** | `sachetPremiumRequest` | Read — by `referenceId` | Always |
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
