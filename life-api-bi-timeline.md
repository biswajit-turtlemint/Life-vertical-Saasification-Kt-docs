# Life API — POST `/createBITimeLine` (Benefit Illustration Timeline Image)

**Full URL:** `POST /api/minterprise/v2/life/createBITimeLine`
**Service:** `transactional-flows`
**Controller:** `VerticalController.java:36` → `createBITimeLine()`

---

## Purpose

Generates a visual **timeline image** (JPEG) for a selected life quote's benefit illustration. Unlike [`/getBenefitIllustration`](./life-api-benefit-illustration.md) which calls the insurer's API via Integration Hub to fetch year-wise data, this API works entirely from data **already stored in MongoDB** — it enriches the quote with master data and renders a timeline graphic via the Template Generator Service.

### Key difference from `/getBenefitIllustration`

| | `/getBenefitIllustration` | `/createBITimeLine` |
|---|---|---|
| Data source | Calls insurer via Integration Hub | Uses existing quote from MongoDB |
| Output | Insurer's BI data (year-wise table) | JPEG image of a timeline |
| External call | Integration Hub → Insurer BI API | Template Generator Service |
| When to use | Need insurer-computed year-wise figures | Need a visual timeline image for UI / sharing |

---

## Sample cURL

```bash
curl --location 'http://localhost:9098/api/minterprise/v2/life/createBITimeLine' \
--header 'Content-Type: application/json' \
--data '{
    "data": {
        "referenceId": "AHXKBEGFTOG",
        "quoteId": "5a940510-7ec4-4486-8eff-dfab4d6253e3"
    }
}'
```

---

## Request

### Headers

| Header | Example Value | Description |
|---|---|---|
| `Content-Type` | `application/json` | Required |
| `x-broker` | `axisbank` | Broker/channel code |
| `x-tenant` | `axisbank` | Tenant identifier |
| `Authorization` | `<token>` | Auth token |

### Body

Wrapped in `PayloadWrapper`:

```json
{
  "data": {
    "referenceId": "AHXKBEGFTOG",
    "quoteId": "5a940510-7ec4-4486-8eff-dfab4d6253e3"
  }
}
```

Deserialized into `BITimeLineRequest`:

| Field | Type | Required | Description |
|---|---|---|---|
| `quoteId` | String | **Always mandatory** | UUID of the specific quote variant. Must match a quote stored in the result document |
| `referenceId` | String | One of | Quote session ID — used to look up the `QuotationResult` document |
| `premiumResultId` | String | One of | Direct MongoDB `_id` of the `QuotationResult` — alternative when `referenceId` is unavailable |

If `quoteId` is blank → `400 MISSING_FIELD`.
If both `referenceId` and `premiumResultId` are blank → `400 MISSING_FIELD`.

---

## Response

### Success (HTTP 200)

```json
{
  "data": {
    "fileName": "LIFE_BI_TIMELINE_TEMPLATE.jpeg",
    "fileId": "a3c9f1e2-8b44-47dc-bf91-12f3e9abc001"
  },
  "meta": { "status": "SUCCESS", "error": false }
}
```

| Field | Description |
|---|---|
| `fileName` | Always `"LIFE_BI_TIMELINE_TEMPLATE.jpeg"` |
| `fileId` | File ID from the Template Generator / File Service — use to build the view URL |

### Failure (HTTP 400)

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
  │  POST /api/minterprise/v2/life/createBITimeLine
  │  Body: { "data": { referenceId, quoteId } }
  ▼
VerticalController.createBITimeLine()
  │
  ├─ Parse PayloadWrapper.data → BITimeLineRequest
  │
  └─ IVerticalService.createBITimeLine(biTimeLineRequest, "life")
       │
       └─ VerticalServiceImpl → SachetLifeAggregator.createBITimeLine(request)
              │                                               [line 3345]
              │
              ├─ Guard: quoteId blank
              │    └─► 400 MISSING_FIELD "quoteId is mandatory"
              │
              ├─ Guard: referenceId AND premiumResultId both blank
              │    └─► 400 MISSING_FIELD "referenceId is mandatory"
              │
              ├─[referenceId present]
              │    quotationResultMono =
              │      quotationService.fetchResultFromDB("life", referenceId, quoteId, null)
              │      MongoDB: sachetPremiumResponse
              │
              └─[premiumResultId present]
                   quotationResultMono =
                     quotationResultRepository.findById(premiumResultId)
                     MongoDB: sachetPremiumResponse (by _id)
                     → NoRecordFoundException if not found
              │
              └─ quotationResultMono.flatMap(quotationResult → )
                   │
                   ├─ resolveSelectedLifeQuotePayload(quotationResult, quoteId)
                   │    Extracts the matching quote Map from the result document
                   │    Matches quoteId against stored quote identifiers
                   │    → empty Map if quoteId not found
                   │    └─► empty → 400 "Life quote not found for referenceId=... quoteId=..."
                   │
                   ├─ selectedQuote.remove("responseOptions")
                   │    Strip nested variants — only base quote is needed for timeline
                   │
                   ├─ lifeQuoteMasterDataEnrichmentService.enrichQuoteResponse(selectedQuote, true)
                   │    Parallel fetch (Mono.zip — all run concurrently):
                   │    │
                   │    ├─ enrichCompanyDetails(insurerCode)
                   │    │    MongoDB: CompanyDetails
                   │    │    → companyDetails { insurerName, claimSettlementRatio,
                   │    │                       solvencyRatio, lifeCompanyDetails { ... } }
                   │    │
                   │    ├─ enrichRiderList(insurerCode)
                   │    │    MongoDB: lifeRiderMaster
                   │    │    → riderList [ { riderCode, riderName, riderCategory, ... } ]
                   │    │
                   │    ├─ enrichOfferList(insurerCode)
                   │    │    MongoDB: lifeOfferMaster
                   │    │    → offerList [ { offerCode, offerName, ... } ]
                   │    │
                   │    ├─ enrichUlipFundAllocation(productCode)
                   │    │    MongoDB: lifeUlipFundAllocationMaster
                   │    │    → ulipFundAllocation { ... } (ULIP plans only)
                   │    │
                   │    ├─ enrichResponseOptions(baseResponse)
                   │    │    MongoDB: sachetPremiumResponse (responseOptions)
                   │    │    → responseOptions [ { variant quotes } ]
                   │    │
                   │    ├─ enrichErrorCategory(insurerCode)
                   │    │    MongoDB: LifeResponseAttributionRules
                   │    │    → errorCategory, errorSubCategory patches
                   │    │
                   │    ├─ enrichTaxSavingsInfo(baseResponse)
                   │    │    Computed from quote fields → taxSavingsInfo { ... }
                   │    │
                   │    └─ enrichResultCardsInfo(baseResponse)
                   │         Computed from quote fields → resultCardsInfo { ... }
                   │
                   └─ generateLifeBITimelineFile(enrichedQuote, broker)
                        │                          [line 3431]
                        │
                        ├─ Build template generator request:
                        │    template:   "LIFE_BI_TIMELINE_TEMPLATE"
                        │    fileType:   "image"
                        │    broker:     broker (fallback: "turtlemint")
                        │    data:       enrichedQuote (responseOptions stripped)
                        │    fileOptions:
                        │      format:  "jpeg"
                        │      height:  100
                        │      width:   330
                        │
                        └─ templateGeneratorUtil.generateTemplate(request, Map.class)
                             └─ Template Generator Service renders JPEG image
                                Returns fileId
                                Response: { fileName: "LIFE_BI_TIMELINE_TEMPLATE.jpeg", fileId }
  │
  ├─[ErrorResponseData] → PayloadWrapper.generateResponse(error, MetaInfo("FAILURE", true))
  └─[success]           → PayloadWrapper.generateResponse({ fileName, fileId }, null)
                              │
                              ▼
                         HTTP 200
```

---

## Functions & Where Things Happen

| Function | File | Line | What it does |
|---|---|---|---|
| `createBITimeLine()` (controller) | `VerticalController.java` | 36 | Entry point: unwraps `PayloadWrapper.data` → `BITimeLineRequest`, delegates to `VerticalServiceImpl`, maps error/success response |
| `createBITimeLine()` (service) | `VerticalServiceImpl.java` | 27 | Thin delegation: calls `ISachetProductAggregatorFactory.getPremiumAggregator("life")` → `SachetLifeAggregator.createBITimeLine()` |
| `createBITimeLine()` (aggregator) | `SachetLifeAggregator.java` | 3344 | Core logic: guards (quoteId, referenceId) → DB fetch (two paths: by referenceId or premiumResultId) → `resolveSelectedLifeQuotePayload()` → strip responseOptions → `enrichQuoteResponse()` → `generateLifeBITimelineFile()` |
| `resolveSelectedLifeQuotePayload()` | `SachetLifeAggregator.java` | — | Extracts the matching quote as `Map<String,Object>` from the `QuotationResult` document. Matches `quoteId` against the root entry and each `responseOptions[]` variant by checking `_id`, `quoteId`, `insurerQuoteId`, `resultId`. Returns empty map if not found |
| `enrichQuoteResponse()` | `LifeQuoteMasterDataEnrichmentService.java` | 98 | Runs 7 parallel enrichments via `Mono.zip`. Adds master data onto the quote map before it is sent to the template renderer. The `true` parameter means ULIP fund allocation is included |
| `enrichCompanyDetails()` | `LifeQuoteMasterDataEnrichmentService.java` | — | MongoDB `CompanyDetails` collection: queries by `insurerCode`. Adds `companyDetails` map to quote (includes `insurerName`, `claimSettlementRatio`, `solvencyRatio`, `freelookPeriod`, `lifeCompanyDetails`) |
| `enrichRiderList()` | `LifeQuoteMasterDataEnrichmentService.java` | — | MongoDB `lifeRiderMaster`: queries by `insurerCode`. Adds `riderList[]` to quote for the template to render available rider info |
| `enrichOfferList()` | `LifeQuoteMasterDataEnrichmentService.java` | — | MongoDB `lifeOfferMaster`: queries by `insurerCode`, filters active offers. Adds `offerList[]` to quote |
| `enrichUlipFundAllocation()` | `LifeQuoteMasterDataEnrichmentService.java` | — | MongoDB `lifeUlipFundAllocationMaster`: queries by `productCode`. Adds `ulipFundAllocation` to quote — only populated for ULIP-type plans |
| `enrichErrorCategory()` | `LifeQuoteMasterDataEnrichmentService.java` | — | MongoDB `LifeResponseAttributionRules`: queries by `insurerCode + stage=QUOTE`. Patches `errorCategory` and `errorSubCategory` onto the quote for error attribution context |
| `enrichTaxSavingsInfo()` | `LifeQuoteMasterDataEnrichmentService.java` | — | Computed (no DB call): derives tax savings info from the quote's premium/coverage fields → adds `taxSavingsInfo` map to quote |
| `enrichResultCardsInfo()` | `LifeQuoteMasterDataEnrichmentService.java` | — | Computed (no DB call): derives result card summary fields (death benefit, maturity benefit labels, etc.) from quote → adds `resultCardsInfo` map |
| `generateLifeBITimelineFile()` | `SachetLifeAggregator.java` | 3431 | Builds the Template Generator request: `template="LIFE_BI_TIMELINE_TEMPLATE"`, `fileType="image"`, `fileOptions={format:"jpeg", height:100, width:330}`, `data=enrichedQuote`. Strips `responseOptions` again before sending. Calls `templateGeneratorUtil.generateTemplate()` → returns `{fileName, fileId}` |

---

## Database / External Calls

| System | Collection / Endpoint | Operation | When |
|---|---|---|---|
| **MongoDB** | `sachetPremiumResponse` | Read — by `referenceId + quoteId` or by `_id` | Always |
| **MongoDB** | `CompanyDetails` | Read — by `insurerCode` | Always (enrichment) |
| **MongoDB** | `lifeRiderMaster` | Read — by `insurerCode` | Always (enrichment) |
| **MongoDB** | `lifeOfferMaster` | Read — by `insurerCode` | Always (enrichment) |
| **MongoDB** | `lifeUlipFundAllocationMaster` | Read — by `productCode` | ULIP plans only |
| **MongoDB** | `LifeResponseAttributionRules` | Read — by `insurerCode`, stage=`QUOTE` | Always (enrichment) |
| **Template Generator Service** | `LIFE_BI_TIMELINE_TEMPLATE` | POST — render JPEG image (100×330) | Always |
| **Integration Hub** | Not called | — | Never — no IH call in this API |

---

## Error Handling

| Exception | Response |
|---|---|
| `quoteId` blank | 400 — "quoteId is mandatory" |
| Both `referenceId` and `premiumResultId` blank | 400 — "referenceId is mandatory" |
| `NoRecordFoundException` (result not found by `premiumResultId`) | 400 — message from exception |
| `quoteId` not found in result document | 400 — "Life quote not found for referenceId=... quoteId=..." |
| Template Generator / File Service failure | 400 — "Unable to create benefit illustration timeline right now" |

---

## Error Codes

| Scenario | HTTP | Error Code |
|---|---|---|
| `quoteId` blank | 400 | `MISSING_FIELD` |
| Both `referenceId` and `premiumResultId` blank | 400 | `MISSING_FIELD` |
| Quote result not found | 400 | `INVALID_REQUEST` |
| `quoteId` not in result | 400 | `INVALID_REQUEST` |
| Template generation failure | 400 | `INVALID_REQUEST` |
