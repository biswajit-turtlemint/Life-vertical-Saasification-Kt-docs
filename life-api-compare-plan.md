# Life API — Compare Plans (3 APIs)

| API | Full URL |
|---|---|
| Run comparison | `POST /api/minterprise/v2/products/life/compare-plans` |
| Generate PDF | `POST /api/minterprise/v2/products/life/get-compare-plans-pdf` |
| Email comparison | `POST /api/minterprise/v2/products/life/compare-plans/share` |

**Service:** `transactional-flows`
**Controller:** `SachetController.java` (lines 676, 734, 792)
**Aggregator:** `SachetLifeAggregator.java` (lines 947, 1028, 1143)
**Core service:** `PlanComparisonServiceImpl.java`

---

## Purpose

These three APIs form a comparison feature for life insurance plans. They share the same underlying comparison engine — they only differ in the final output step:

- `/compare-plans` → returns structured comparison data (for UI rendering)
- `/get-compare-plans-pdf` → generates a PDF file and returns its URL
- `/compare-plans/share` → emails the comparison with a PDF attachment to the customer

---

## Sample cURLs

### compare-plans

```bash
curl --location 'http://localhost:9098/api/minterprise/v2/products/life/compare-plans' \
--header 'content-type: application/json' \
--header 'x-tenant: axisbank' \
--header 'x-broker: axisbank' \
--header 'x-partner-id: 647f458828effa0001044980' \
--header 'Authorization: <token>' \
--data '{
    "requestId": "AHW4TF8UADD",
    "customerName": "ABID AHMAD SHAH",
    "selectedPlans": [
        { "quoteId": "cf0703da-dd04-4180-b14f-a76e44c05357", "isRecommended": true },
        { "quoteId": "cd346f96-cb44-48af-bc59-567961893b5e", "isRecommended": false },
        { "quoteId": "3777f6a9-0e0a-4ee2-b391-25019db0bd2d", "isRecommended": false }
    ]
}'
```

### get-compare-plans-pdf

```bash
curl --location 'http://localhost:9098/api/minterprise/v2/products/life/get-compare-plans-pdf' \
--header 'content-type: application/json' \
--header 'x-tenant: axisbank' \
--header 'x-broker: axisbank' \
--header 'Authorization: <token>' \
--data '{
    "requestId": "AHW4TF8UADD",
    "partnerId": "647f458828effa0001044980",
    "customerName": "ANSARI ALI IMRAN AHMED",
    "selectedAttributes": null,
    "selectedPlans": [
        { "quoteId": "cf0703da-dd04-4180-b14f-a76e44c05357", "isRecommended": true },
        { "quoteId": "cd346f96-cb44-48af-bc59-567961893b5e", "isRecommended": false },
        { "quoteId": "3777f6a9-0e0a-4ee2-b391-25019db0bd2d", "isRecommended": false }
    ]
}'
```

### compare-plans/share

```bash
curl --location 'https://pro.axisbank.saas-sanity.turtle-feature.com/api/minterprise/v2/products/life/compare-plans/share' \
--header 'content-type: application/json' \
--header 'x-tenant: axisbank' \
--header 'x-broker: axisbank' \
--header 'Authorization: <token>' \
--data-raw '{
    "planComparisonRequest": {
        "requestId": "AHW4TF8UADD",
        "partnerId": "647f458828effa0001044980",
        "customerName": "ANSARI ALI IMRAN AHMED",
        "selectedAttributes": null,
        "selectedPlans": [
            { "quoteId": "131e387c-cc66-4f8e-a1cc-616ea7312a94", "isRecommended": true },
            { "quoteId": "52eb341b-124f-45a2-a375-9d22eda18bd1", "isRecommended": false },
            { "quoteId": "8989f734-3609-4082-babd-2c4fc21e02f9", "isRecommended": false }
        ]
    },
    "type": "EMAIL",
    "customerEmail": "biswajit.rout@turtlemint.com",
    "fileName": "LIFE_PLANS_COMPARE.pdf",
    "fileId": "795766f5-1f37-4a33-ac44-7055df2a9bab"
}'
```

---

## Common Request — `PlanComparisonRequest`

All three APIs accept the same core request shape (**no** `PayloadWrapper` envelope). Note: `requestId` and `referenceId` are interchangeable — the backend accepts both via `@JsonAlias`.

```json
{
  "requestId": "AHW4TF8UADD",
  "uniqueId": "optional-session-id",
  "selectedPlans": [
    { "quoteId": "cf0703da-dd04-4180-b14f-a76e44c05357", "isRecommended": true },
    { "quoteId": "cd346f96-cb44-48af-bc59-567961893b5e", "isRecommended": false },
    { "quoteId": "3777f6a9-0e0a-4ee2-b391-25019db0bd2d", "isRecommended": false }
  ],
  "selectedAttributes": null,
  "partnerId": "647f458828effa0001044980",
  "customerName": "ANSARI ALI IMRAN AHMED",
  "verticalSpecificInfo": {}
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `requestId` / `referenceId` | String | **Yes** | Quote session ID — both field names accepted |
| `uniqueId` | String | No | Optional session sub-identifier |
| `selectedPlans` | Array | **Yes** | Plans to compare. Each must have `quoteId` **OR** `resultId` **OR** (`productCode` + `optionCode`) |
| `selectedAttributes` | Array\<String\> | No | Attribute codes to include; `null` / absent = all attributes returned |
| `partnerId` | String | No | Fetches partner details for branding on PDF/email |
| `customerName` | String | No | Shown on PDF and email |
| `verticalSpecificInfo` | Map | No | Reserved for future vertical-specific overrides |

### `PlanRequestInfo` fields (each item in `selectedPlans`)

| Field | Type | Description |
|---|---|---|
| `quoteId` | String | **Preferred** — UUID returned by the quote engine, matched against `_id`, `quoteId`, `insurerQuoteId`, `resultId` in the DB row |
| `resultId` | String | Alternative identifier for the result document |
| `productCode` | String | Legacy: insurer product code (e.g., `P126`) |
| `optionCode` | Integer | Legacy: used with `productCode` for plan variant selection |
| `isRecommended` | Boolean | Marks this plan as recommended (highlighted on UI) |

---

## Common Headers

| Header | Default | Description |
|---|---|---|
| `x-tenant` | `turtlemint` | Tenant identifier |
| `x-broker` | `turtlemint` | Broker/channel code |
| `x-partner-id` | `""` | Partner ID (falls back to `partnerId` in body) |
| `vertical` | *(query param, optional)* | Vertical hint |

---

## Core Comparison Flow (`PlanComparisonServiceImpl.comparePlans`)

All three APIs run this flow first. The PDF and share APIs branch after step 5.

```
1. Guard checks
   ├─ productCode must be "life" (via aggregator.supportsPlanComparison())
   ├─ referenceId must not be blank
   ├─ selectedPlans must not be empty
   └─ Each plan must have quoteId OR resultId OR (productCode + optionCode)

2. fetchQuoteRowsForComparison(referenceId)
   └─ quotationService.fetchLifeQuotesFromDBForComparison(referenceId)
        MongoDB: grid_based_quotation_result
        Returns raw quote rows (Map<String,Object>) for the session

3. resolveSelectedPlans(selectedPlans, quoteRows)
   For each plan in selectedPlans, find matching quote row + variant:
     Priority a → match by quoteId
                  (checks fields: _id, quoteId, insurerQuoteId, resultId)
     Priority b → match by resultId + productCode + optionCode
     Priority c → legacy: match by productCode + optionCode only
   Each quote row may have a base entry + responseOptions[] (variants).
   Resolves final productCode + optionCode for each matched plan.

4. Parallel data fetch (Mono.zip — all 4 run concurrently):
   │
   ├─ a. fetchLifeProductMasters(broker)
   │       LifeProductCatalogueService.fetchProductMasters(broker)
   │       MongoDB: life_product_catalogue_master
   │       → Map<"productCode|optionCode", LifeProductCatalogueMaster>
   │         Contains: minEntryAge, maxEntryAge, minPpt, maxPpt,
   │                   sumAssuredLimitOnDeath, maturityAge, payoutTypes,
   │                   paymentFrequencies, specialFeatures[]
   │
   ├─ b. fetchFeatureMap(productMasterMap, resolvedPlans)
   │       Collects all specialFeature IDs from masters of resolved plans
   │       MongoDB: life_product_special_features
   │         Query: { _id: { $in: [featureIds] } }
   │       Sorts each plan's features by priority (asc, nulls last),
   │         then by displayText (asc, case-insensitive)
   │       → Map<"productCode|optionCode", List<LifeProductSpecialFeature>>
   │
   ├─ c. fetchRiderPrices(request, resolvedPlans)
   │       Keys: "{productCode}_{optionCode}" per resolved plan (distinct)
   │       SachetLifeAggregator.getRiderPrices(referenceId, uniqueId, keys)
   │       → LifeRiderPricesResponse { riderMetaResponseList }
   │         Each entry: { key, riderInfoList[{ riderCategory, riderName,
   │                       riderSumAssured, riderPremium, inBuilt, status }] }
   │       Errors swallowed → returns empty response
   │
   └─ d. fetchPartnerDetails(partnerId)
           PartnerIntegrationService.getPartnerDetailsByPartnerId(partnerId)
           → PartnerDetails { partnerId, partnerName, partnerEmail,
                              partnerMobile, panNo, aadhaarNo (decrypted),
                              branchCode, branchName, dpNo, businessChannel }
           Errors swallowed → returns empty PartnerDetails

5. buildResponse(request, resolvedPlans, masterMap, featureMap, riderPrices, partnerDetails)
   │
   For each resolved plan (only "SUCCESS" variants included):
   │
   ├─ buildPlanResultInfo()
   │    Header fields:
   │      id, quoteId, resultId, key, productCode, optionCode, insurerCode
   │      displayName = "<insurerName> <productName>"
   │      insurerLogo = "<insurerCode>.png"
   │    Premium:
   │      premiumAmount  ← premium / premiumAnnual / basePremium
   │      displayAmount  ← premiumWithTax / totalPremium / premium / basePremium
   │      premiumFrequency: 12→Yearly, 6→Semi-Annual, 3→Quarterly, 1→Monthly, -1→Single Pay
   │    options[] ← all successful variants of this quote (same plan, different option codes)
   │
   └─ buildPlanSections() → 5 fixed sections per plan:
        ┌───────────────────┬─────┬──────────────────────────────────────────────────┐
        │ Section code      │ Seq │ Attributes (code → source)                       │
        ├───────────────────┼─────┼──────────────────────────────────────────────────┤
        │ basicDetails      │  1  │ premiumPaymentTerm → variant.premiumPaymentTerm  │
        │                   │     │ policyTerm → variant.policyTerm                  │
        │                   │     │ lifeCover → sumAssured/deathBenefit              │
        │                   │     │ coverUptoAge → entryAge + policyTerm             │
        │                   │     │ payoutOptions → variant / master.payoutTypes     │
        │                   │     │ selectedRider → riderList[isSelected=true].name  │
        ├───────────────────┼─────┼──────────────────────────────────────────────────┤
        │ keyFeatures       │  2  │ From LifeProductSpecialFeature (icon type)       │
        │                   │     │ showInitialCount = 5                             │
        │                   │     │ Populated by populateKeyFeatureSections()        │
        │                   │     │ value = true/false (plan has/lacks that feature) │
        ├───────────────────┼─────┼──────────────────────────────────────────────────┤
        │ advancedDetails   │  3  │ minMaxEntryAge → master.minEntryAge/maxEntryAge  │
        │                   │     │ minMaxPPT → master.minPpt/maxPpt                 │
        │                   │     │ paymentFrequencyOption → master / variant modes  │
        │                   │     │ minMaxSumAssured → master.sumAssuredLimitOnDeath │
        │                   │     │ minMaxMaturityAge → master.maturityAge           │
        │                   │     │ freelookPeriod → companyDetails.freelookPeriod   │
        ├───────────────────┼─────┼──────────────────────────────────────────────────┤
        │ insurerDetails    │  4  │ inceptionYear, claimSettlementRatio,             │
        │                   │     │ solvencyRatio, numberOfLivesInsured,             │
        │                   │     │ assetUnderManagement, branchesAcrossIndia        │
        │                   │     │ (all from variant/parentQuote.companyDetails     │
        │                   │     │  .lifeCompanyDetails)                            │
        ├───────────────────┼─────┼──────────────────────────────────────────────────┤
        │ availableAddons   │  5  │ One attribute per rider category                 │
        │                   │     │ value = { cover: riderSumAssured,               │
        │                   │     │           for: riderPremium | "Free" (inBuilt) } │
        │                   │     │ "-" if status != SUCCESS                         │
        │                   │     │ "NA" if no data                                  │
        └───────────────────┴─────┴──────────────────────────────────────────────────┘

   populateKeyFeatureSections():
     Merges all plans' special features into one ordered list (union)
     Sorted by: priority ASC (nulls last) → displayText ASC
     Each plan gets true/false per feature (has it or not)

   filterPlanResultSections():
     If selectedAttributes provided → remove attributes not in that set from all sections

   buildPlanSectionRows(planResults):
     Transposes data from "plan → section → attribute"
                       to "section → attribute → [value per plan]"
     Used by the UI table/grid rendering
     Empty-value rows (all NA / "-" / null) are omitted
```

---

## API 1: POST `/compare-plans`

**Controller:** `SachetController.java:676`
**Aggregator:** `SachetLifeAggregator.java:947`
**Service:** `PlanComparisonServiceImpl.java:112`

Runs the full core comparison flow and returns the structured data.

### Request fields (actual usage)

| Field | Example | Notes |
|---|---|---|
| `requestId` | `"AHW4TF8UADD"` | Quote session ID — `x-partner-id` header is the partner context |
| `customerName` | `"ABID AHMAD SHAH"` | Shown on the comparison UI |
| `selectedPlans[].quoteId` | `"cf0703da-..."` | UUID from quote engine |
| `selectedPlans[].isRecommended` | `true` | First plan typically marked recommended |
| `partnerId` | not sent in body — taken from `x-partner-id` header | |

### Response

```json
{
  "data": {
    "planComparisonRequest": { /* echo of input */ },
    "planResultInfoList": [
      {
        "id": "mongo_doc_id",
        "quoteId": "QT_HDFC_001",
        "resultId": "...",
        "productCode": "HDFC_CLICK_2_PROTECT_LIFE",
        "optionCode": 1,
        "insurerCode": "HDFC",
        "displayName": "HDFC Life Click 2 Protect Life",
        "insurerLogo": "HDFC.png",
        "planName": "Click 2 Protect Life",
        "optionName": "Life Option",
        "premiumAmount": 12500.0,
        "premiumFrequency": "Yearly",
        "displayAmount": 14750.0,
        "isRecommended": true,
        "options": [ /* other option variants */ ],
        "sections": [
          {
            "code": "basicDetails",
            "name": "Basic Details",
            "sequence": 1,
            "attributes": [
              {
                "code": "lifeCover",
                "name": "Life Cover/Death Benefit",
                "description": "The guaranteed amount payable in case of the insured's demise",
                "type": "currency",
                "value": 5000000,
                "isRecommended": true,
                "planIdentifier": "HDFC_CLICK_2_PROTECT_LIFE_1"
              }
            ]
          }
        ]
      }
    ],
    "planSectionRowInfoList": [
      {
        "code": "basicDetails",
        "name": "Basic Details",
        "sequence": 1,
        "features": [
          {
            "title": { "code": "lifeCover", "name": "Life Cover/Death Benefit", "description": "..." },
            "planFeatures": [ /* one PlanAttribute per compared plan */ ]
          }
        ]
      }
    ],
    "partnerDetails": {
      "partnerId": "PARTNER123",
      "partnerName": "Rajesh Kumar",
      "partnerEmail": "rajesh@partner.com",
      "partnerMobile": "9876543210"
    },
    "customerName": "John Doe",
    "createdOn": "13 May 2026"
  },
  "meta": { "status": "SUCCESS", "error": false }
}
```

---

## API 2: POST `/get-compare-plans-pdf`

**Controller:** `SachetController.java:734`
**Aggregator:** `SachetLifeAggregator.java:1143`
**Service:** `PlanComparisonServiceImpl.java:193`

### Request

Same `PlanComparisonRequest` shape (no wrapper). In practice, `partnerId` is sent **in the body** (not via `x-partner-id` header) and `selectedAttributes` is explicitly `null`:

```json
{
    "requestId": "AHW4TF8UADD",
    "partnerId": "647f458828effa0001044980",
    "customerName": "ANSARI ALI IMRAN AHMED",
    "selectedAttributes": null,
    "selectedPlans": [
        { "quoteId": "cf0703da-dd04-4180-b14f-a76e44c05357", "isRecommended": true },
        { "quoteId": "cd346f96-cb44-48af-bc59-567961893b5e", "isRecommended": false },
        { "quoteId": "3777f6a9-0e0a-4ee2-b391-25019db0bd2d", "isRecommended": false }
    ]
}
```

### What it does (beyond the core comparison)

```
Core comparison flow (same as /compare-plans)
  │
  └─ generatePlanComparisonPdf(productCode, broker, partnerId, request)
       │
       ├─ comparePlans() → PlanComparisonResponse
       │
       ├─ buildPlanComparisonPdfRequest(compareResponse, broker)
       │    template:      "LIFE_PLANS_COMPARE"
       │    templateType:  "pdf"
       │    data:          PlanComparisonResponse
       │    pdfOptions:
       │      orientation: "Portrait"
       │      format:      "A4"
       │      margin:      "0"
       │      height:      680, width: 500
       │      header:      { height: "0mm" }
       │      footer:      page X of Y + turtlemint logo
       │
       └─ templateGeneratorUtil.generateTemplate(request, Map.class)
            └─ Template Generator Service → renders PDF
               Returns fileId (stored in file service)
```

### Response

```json
{
  "data": {
    "fileId": "abc123xyz",
    "fileName": "LIFE_PLANS_COMPARE.pdf",
    "url": "https://files.turtlemint.com/abc123xyz?broker=turtlemint"
  },
  "meta": { "status": "SUCCESS", "error": false }
}
```

| Field | Description |
|---|---|
| `fileId` | File service identifier — use this in `/compare-plans/share` |
| `fileName` | Always `"LIFE_PLANS_COMPARE.pdf"` |
| `url` | View URL: `{fileViewUrl}/{fileId}?broker={broker}` |

---

## API 3: POST `/compare-plans/share`

**Controller:** `SachetController.java:792`
**Aggregator:** `SachetLifeAggregator.java:1028`
**Service:** `PlanComparisonServiceImpl.java:164`

### Request (wraps `PlanComparisonRequest`)

```json
{
    "planComparisonRequest": {
        "requestId": "AHW4TF8UADD",
        "partnerId": "647f458828effa0001044980",
        "customerName": "ANSARI ALI IMRAN AHMED",
        "selectedAttributes": null,
        "selectedPlans": [
            { "quoteId": "131e387c-cc66-4f8e-a1cc-616ea7312a94", "isRecommended": true },
            { "quoteId": "52eb341b-124f-45a2-a375-9d22eda18bd1", "isRecommended": false },
            { "quoteId": "8989f734-3609-4082-babd-2c4fc21e02f9", "isRecommended": false }
        ]
    },
    "type": "EMAIL",
    "customerEmail": "biswajit.rout@turtlemint.com",
    "fileName": "LIFE_PLANS_COMPARE.pdf",
    "fileId": "795766f5-1f37-4a33-ac44-7055df2a9bab"
}
```

| Field | Required | Description |
|---|---|---|
| `planComparisonRequest` | Yes | Full `PlanComparisonRequest` — uses `requestId` alias |
| `type` | Yes | Must be `"EMAIL"` — only supported channel |
| `customerEmail` | Yes | Recipient email |
| `customerMobile` | No | Added to notification if present |
| `fileId` | No | PDF file ID from `/get-compare-plans-pdf` — used as email attachment |
| `fileName` | No | PDF filename (e.g., `"LIFE_PLANS_COMPARE.pdf"`) |

> **Note:** `fileId` is obtained by calling `/get-compare-plans-pdf` first. The share API can also be called without `fileId` (no attachment) but the PDF URL is still included in the email body via `comparisonPdfUrl`.

### What it does (beyond the core comparison)

```
Core comparison flow (same as /compare-plans)
  │
  └─ sharePlanComparison(productCode, broker, partnerId, shareRequest)
       │
       ├─ comparePlans() → PlanComparisonResponse
       │
       ├─ generatePreviewUrl(compareResponse, broker)
       │    Template: "LIFE_PLANS_COMPARE_PREVIEW"
       │    Type: "image", format: "jpeg", width: 600
       │    templateGeneratorUtil.generateTemplate(...)
       │    → previewUrl = {fileViewUrl}/{fileId}?broker={broker}
       │
       ├─ buildAttachments(shareRequest, broker)
       │    ├─ downloadUrl = {fileDownloadUrl}/{fileId}?broker={broker}
       │    ├─ fileService.getFileResponseEntity(downloadUrl)
       │    │    → Downloads PDF bytes
       │    │    → Base64 encode → Attachment { name, type:"application/pdf", content }
       │    └─ On error/empty: fallback = Attachment with URL (not bytes)
       │
       ├─ buildShareNotificationRequest(shareRequest, compareResponse, broker, previewUrl, attachments)
       │    template:  "LIFE_PLAN_COMPARISON_EMAIL_{broker}"
       │    subject:   "Your life plan comparison"
       │    from:      emailCreds.mail.from.email.val
       │    to:        [customerEmail]
       │    cc:        [partnerDetails.partnerEmail]  (if present)
       │    mappings:
       │      comparisonPdfUrl     → {fileViewUrl}/{fileId}?broker={broker}
       │      comparisonPreviewUrl → generated preview image URL
       │      customerName         → planComparisonRequest.customerName
       │      partnerName          → partnerDetails.partnerName
       │      partnerMobile        → partnerDetails.partnerMobile
       │    attachments: [PDF attachment]
       │
       └─ notificationService.sendEmail(notificationRequestMapper, true)
            → Notification Service sends email
```

### Response

```json
{
  "data": "Email sent successfully",
  "meta": { "status": "SUCCESS", "error": false }
}
```

---

## Database / External Calls Summary (all 3 APIs)

| System | Collection / Endpoint | Operation | Used by |
|---|---|---|---|
| **MongoDB** | `grid_based_quotation_result` | Read — fetch quote rows by referenceId | All 3 |
| **MongoDB** | `life_product_catalogue_master` | Read — product master by productCode+optionCode | All 3 |
| **MongoDB** | `life_product_special_features` | Read — `_id IN featureIds` | All 3 |
| **SachetLifeAggregator** | `/rider/prices` (internal call) | Read — rider prices per plan key | All 3 |
| **Partner Service** | Partner details by partnerId | Read | All 3 |
| **Template Generator Service** | `LIFE_PLANS_COMPARE` template | POST — render PDF | PDF + Share |
| **Template Generator Service** | `LIFE_PLANS_COMPARE_PREVIEW` template | POST — render JPEG | Share only |
| **File Service** | `GET /{fileId}?broker=` | Download PDF bytes | Share only |
| **Notification Service** | Email send | POST | Share only |

---

## Functions & Where Things Happen

| Function | File | Line | What it does |
|---|---|---|---|
| `comparePlans()` (controller) | `SachetController.java` | 676 | Validates product support, delegates to aggregator |
| `getPlanComparisonPdf()` (controller) | `SachetController.java` | 734 | Validates PDF support, delegates to aggregator |
| `sharePlanComparison()` (controller) | `SachetController.java` | 792 | Validates share support, delegates to aggregator |
| `comparePlans()` (aggregator) | `SachetLifeAggregator.java` | 947 | Input validation, delegates to service |
| `sharePlanComparison()` (aggregator) | `SachetLifeAggregator.java` | 1028 | Input validation, delegates to service |
| `comparePlans()` (service) | `PlanComparisonServiceImpl.java` | 112 | Core comparison logic |
| `generatePlanComparisonPdf()` (service) | `PlanComparisonServiceImpl.java` | 193 | Calls comparison + triggers PDF generation |
| `sharePlanComparison()` (service) | `PlanComparisonServiceImpl.java` | 164 | Calls comparison + preview + email |
| `fetchQuoteRowsForComparison()` | `PlanComparisonServiceImpl.java` | 210 | Loads quote rows from MongoDB |
| `resolveSelectedPlans()` | `PlanComparisonServiceImpl.java` | 1167 | Matches selectedPlans to DB rows by quoteId/resultId/product |
| `fetchLifeProductMasters()` | `PlanComparisonServiceImpl.java` | 226 | Loads `LifeProductCatalogueMaster` map |
| `fetchFeatureMap()` | `PlanComparisonServiceImpl.java` | 246 | Loads and sorts `LifeProductSpecialFeature` per plan |
| `fetchRiderPrices()` | `PlanComparisonServiceImpl.java` | 310 | Fetches rider pricing for addons section |
| `fetchPartnerDetails()` | `PlanComparisonServiceImpl.java` | 334 | Loads partner info for branding |
| `buildResponse()` | `PlanComparisonServiceImpl.java` | 543 | Assembles full `PlanComparisonResponse` |
| `buildPlanResultInfo()` | `PlanComparisonServiceImpl.java` | 588 | Builds per-plan result with all sections |
| `buildPlanSections()` | `PlanComparisonServiceImpl.java` | 687 | Creates the 5 comparison sections per plan |
| `populateKeyFeatureSections()` | `PlanComparisonServiceImpl.java` | 872 | Fills icon-type key features across all plans |
| `filterPlanResultSections()` | `PlanComparisonServiceImpl.java` | 922 | Filters to selectedAttributes if provided |
| `buildPlanSectionRows()` | `PlanComparisonServiceImpl.java` | 946 | Transposes to row format for UI grid |
| `buildAttachments()` | `PlanComparisonServiceImpl.java` | 415 | Downloads PDF + Base64 encodes for email |
| `buildShareNotificationRequest()` | `PlanComparisonServiceImpl.java` | 439 | Builds email notification payload |

---

## Error Codes

| Scenario | HTTP | Error Code |
|---|---|---|
| Invalid `productCode` | 400 | `INVALID_INPUT` |
| `referenceId` blank | 400 | `MISSING_FIELD` |
| `selectedPlans` empty | 400 | `MISSING_FIELD` |
| Plan has no `quoteId`/`resultId`/product | 400 | `MISSING_FIELD` |
| No quotes found in DB | 400 | `NO_DATA_FOUND` |
| `type` is not `"EMAIL"` (share only) | 500 | `IllegalArgumentException` |
| Comparison data not found (PDF/share) | 500 | `IllegalStateException` |
