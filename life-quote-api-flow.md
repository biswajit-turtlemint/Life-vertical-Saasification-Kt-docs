# Life Insurance Quotes API — Detailed Flow Documentation

**API:** `POST /api/minterprise/v2/products/life/quotes`

---

## Table of Contents

1. [Overview](#overview)
2. [Step 1 — Controller Entry Point](#step-1--controller-entry-point)
3. [Step 2 — Quotation Service: validatePremium](#step-2--quotation-service-validatepremium)
4. [Step 3 — Life Aggregator: generateQuote (Orchestration)](#step-3--life-aggregator-generatequote-orchestration)
5. [Step 4 — Applicable Products (fetchEnabledProductMasters)](#step-4--applicable-products-fetchenabledproductmasters)
    - [4a — Category & PlanType Normalization](#4a--category--plantype-normalization)
6. [Step 5 — Post-Filters Before Nearest Match](#step-5--post-filters-before-nearest-match)
7. [Step 6 — Nearest Match (life-validator)](#step-6--nearest-match-life-validator)
8. [Step 7 — ValidationMap Creation](#step-7--validationmap-creation)
9. [Step 8 — Building Per-Insurer Service Requests](#step-8--building-per-insurer-service-requests)
10. [Step 9 — Pre-Enrichment (Rider & Offer Metadata)](#step-9--pre-enrichment-rider--offer-metadata)
11. [Step 10 — Async IH Premium Dispatch](#step-10--async-ih-premium-dispatch)
12. [Step 11 — IH Request Building (LifeQuoteIntegrationRequestBuilder)](#step-11--ih-request-building-lifequoteintegrationrequestbuilder)
13. [Step 12 — IH Response Normalization (Per-Insurer Normalizers)](#step-12--ih-response-normalization-per-insurer-normalizers)
14. [Step 13 — Post-IH Enrichment & DB Persistence](#step-13--post-ih-enrichment--db-persistence)
15. [Step 14 — Async Rider Prices Flow](#step-14--async-rider-prices-flow)
16. [Step 15 — Initial Response to FE](#step-15--initial-response-to-fe)
17. [Step 16 — GET /quotes & /poll — Response Formation](#step-16--get-quotes--poll--response-formation)
18. [Database Collections Summary](#database-collections-summary)
19. [Modify Flow (same referenceId, new quoteId)](#modify-flow-same-referenceid-new-quoteid)
20. [Requote Flow (same referenceId, same quoteId)](#requote-flow-same-referenceid-same-quoteid)
21. [Internal Metadata Keys Reference](#internal-metadata-keys-reference)
22. [Flowchart](#flowchart)

---

## Overview

The life quotes API is a **fire-and-return** async system. When a quote is requested:

- The system synchronously validates the request, determines eligible products via two external calls (Applicable Products + Nearest Match), builds per-insurer payloads, then **fires the IH calls asynchronously** and **immediately returns a `pending` response** to the FE.
- The FE polls using `GET /api/minterprise/v2/products/life/quotes/poll` until all quotes arrive.
- Rider prices for each successful quote are also computed asynchronously and stored separately.

---

## Step 1 — Controller Entry Point

**File:** `SachetController.java` (line 109)
**Base URL mapping:** `@RequestMapping("/api/minterprise/v2")`

```
POST /api/minterprise/v2/products/life/quotes
```

### What happens:

1. The incoming `PayloadWrapper` body has a `data` field; this is deserialized into a `QuotationRequest` object.
2. The following headers are read and set on the request:

| Header | Field Set | Default |
|--------|-----------|---------|
| `x-tenant` | `request.tenant` | `turtlemint` |
| `x-broker` | `request.broker` | `turtlemint` |
| `x-partner-id` | `request.partnerId` | `NULL_STRING` |
| `x-provider` | `request.provider` | (none) |

3. `productCode = "life"` is set from the URL path variable.
4. `quotationService.generateQuote(premiumRequest)` is called.
5. The response is wrapped in `PayloadWrapper` and returned as `200 OK` or `400 Bad Request`.

---

## Step 2 — Quotation Service: validatePremium

**File:** `IQuotationServiceImpl.java` (lines 110–169, 433–460)

### Check for Custom Quote Flow

```java
ISachetProductAggregator quoteFlowAggregator = getQuoteFlowAggregator("life");
// SachetLifeAggregator.supportsCustomQuoteFlow() returns true → life uses custom flow
```

Since `SachetLifeAggregator.supportsCustomQuoteFlow() = true`, the standard validation aggregator call is **bypassed** for life. The system skips `validationAggregator.validateRequest()` and goes directly to `validatePremium()`.

### validatePremium Logic (line 433)

**Pre-normalize:** `quoteFlowAggregator.normalizeQuoteRequestForStorage(request)` is called first. This:
- Extracts `planDetails` derived fields (`policyTerm`, `premiumPaymentTerm`, `maturityAge`) and puts them on `premiumRequest.planDetails`
- Moves insurer-specific info from `premiumRequest` to `otherDetails`
- Stashes the original quoteId into `otherDetails._lifeRequestedQuoteId`

**referenceId handling:**
```
if referenceId is blank:
    isInitial = true
    generate new referenceId (UUID)
else:
    isInitial = false (this is a modify or requote)
```

**quoteId handling:**
```
if isInitial OR incomingQuoteId is blank:
    generate NEW quoteId (UUID)     ← MODIFY FLOW gets new quoteId here
else:
    use incomingQuoteId as-is       ← REQUOTE FLOW keeps existing quoteId
```

**DB lookup for non-initial:**
If `isInitial = false`, the existing `QuotationRequest` is fetched from DB:
- `quoteFlowAggregator.prepareQuoteRequestForValidation(storedRequest, newRequest)` merges stored+new:
  - Copies `_id` and `createdAt` from stored request
  - Calls `mergeStoredLifePremiumRequest()` (carry forward stored planDetails)
  - Calls `preserveStoredLifeInsurerSpecificInfo()` (carry forward insurer-specific data)
  - Calls `prepareLifeRequoteMetadata()` to check if this is a requote (looks up DB for existing quotes)

**Persistence:** `saveAPIPremiumRequest()` saves to `sachetPremiumRequest` collection.
After save, `savePremiumRequestHistory()` saves a copy to `sachetPremiumRequestHistory` (fire-and-forget `.subscribe()`).

---

## Step 3 — Life Aggregator: generateQuote (Orchestration)

**File:** `SachetLifeAggregator.java` (line 290)

This is the main orchestrator. The sequence is:

```
1. buildLifeConfiguredProviders(request)
   → Extracts requested insurers from premiumRequest.planDetails.insurers or premiumRequest.insurers
   → Returns [{provider: "ICICI"}, {provider: "HDFC"}, ...] or [] if not specified

2. lifeRequestValidationService.resolveValidatedProviders(request, configuredProviders)
   → Returns LifeValidationResult {providers, validationMap, rowsByProvider}

3. filterValidationResultForRequote(request, validationResult)
   → Only runs if _lifeRequoteSelection or _lifeRequoteInsurerSpecificInfo is in otherDetails
   → Filters to only the requested insurer+product combinations

4. buildLifeServiceRequests(request, effectiveValidationResult)
   → Creates one QuotationRequest per validation row (one per plan/variant per insurer)

5. If serviceRequests is empty:
   → If requote: return existing aggregate
   → Else: return error "No eligible life providers found"

6. attachLifeRunStateToRequest(request, validationResult)
   → Sets lifeValidationMap, lifePendingKeyList, lifeValidationRowsByProvider on otherDetails

7. persistLifeRunState(request)
   → Updates the stored QuotationRequest with new quoteId and merged otherDetails

8. fetchCustomPremiumResponse(serviceRequests, request)   ← FRESH FLOW
   OR
   premiumServiceUtil.getIHPremiumResponse(serviceRequests, request) → fetchLifeAggregateResponse  ← REQUOTE FLOW
```

---

## Step 4 — Applicable Products (fetchEnabledProductMasters)

**File:** `LifeRequestValidationService.java` (line 578)
**Called by:** `resolveValidatedProviders()`

### URL

```
POST {integration-hub.base-url}/api/product-management/v1/products/details/filters
```

### Request Payload (built internally)

```json
{
  "broker": "turtlemint",
  "businessCategory": "LIFE",
  "businessSubCategory": "LIFE",
  "productStatus": "LIVE",
  "inclusions": ["PDP", "INTEGRATION_INFO"]
}
```

**Note:** This call is **Redis-cached** for 4 hours per broker. Cache key: `LifeProductCatalogue_{broker}`.

### What the Response Contains (per product)

Each product in the response has:
- `productCode` — internal product code
- `providerCode` — insurer code (e.g., `ICICILI`)
- `providerName` — insurer name
- `productName` — product display name
- `tmId` — TM plan ID
- `productStatus` — `LIVE`
- `integrationInfo` → `variants[]` — list of variant objects with `variantCode`, `variantName`
- `pdp[]` — base product detail page fields (flattened)
- `variants[]` (at product level) — variant-level pdp

### How the Response is Processed

The `normalizeProductDetailRows()` method iterates each product:

1. **Validates** each product: skips if `productStatus` missing or `integrationInfo` missing.
2. **Flattens PDP fields** (`flattenProductDetailPage()`): converts the PDP array `[{fieldName, fieldType, data, value}]` into a flat map.
3. For each variant in `integrationInfo.variants`:
   - Builds a **row** containing all PDP fields + identity fields.

Key PDP fields extracted into each row:

| Field | Source |
|-------|--------|
| `productCode` | PDP field (maps to plan variant code) |
| `internalProductCode` | `productCode` at product level |
| `insurerCode` | `providerCode` at product level |
| `optionCode` | `variantCode` from integration variant |
| `planType` | PDP `planType` |
| `policyType` | PDP `policyType` |
| `categoryTags` | PDP `categoryTags` (list) |
| `paymentFrequencyModes` | PDP field (list of supported modes) |
| `payoutTypes` | PDP field (list: `lumpsum`, `income`) |
| `payoutFrequencies` | PDP field |
| `planFeatureList` | PDP field (joint life, ROP features) |
| `riders` | PDP field — configured rider codes |
| `offers` | PDP field — configured offer codes |
| `showRider` | PDP field — boolean |
| `defaultOption` | PDP field — boolean |
| `currency` | PDP field |
| `isActive` | PDP field |

### Filters Applied After Fetching

Once products are retrieved, they are filtered by these criteria (in `fetchEnabledProductMasters()`):

| Filter | How |
|--------|-----|
| `planType` | `row.planType == request.planDetails.planType` (if planType present) |
| `policyType` | `row.policyType == request.policyType` (only if planType is absent) |
| `category` | `row.categoryTags.containsAll(requestedCategories)` |
| `paymentFrequency` | `row.paymentFrequencyModes.contains(request.paymentFrequency)` |
| `currency` | `row.currency == request.currency` (defaults to `INR`) |
| `selectedPlans` | `row.productCode in request.selectedPlans` |
| `insurerCodes` | `row.insurerCode in configuredProviders` |

---

## 4a — Category & PlanType Normalization

**File:** `LifeCategorySupport.java` (`transactional-flows/.../service/aggregator/life/LifeCategorySupport.java`)

Before filters are applied, `planType` and `category` values are normalized and derived. This affects which products pass the filter.

### planType → category derivation (`deriveCategoryFromPlanType()`)

| planType (request) | Derived Category |
|--------------------|-----------------|
| `term` | `term` |
| `non-participating` | `guaranteed` |
| `participating` | `participating` |
| `ulip` | `ulip` |
| `pension` | `retirement` |
| `whole-life` | `whole-life` |

If `planType` is set, this category is used for filtering unless `FORCE_INCLUDE_CATEGORY` applies.

### planType → policyType derivation (`derivePolicyType()`)

| planType | Derived policyType |
|----------|--------------------|
| `term` | `TERM` |
| `ulip` | `ULIP` |
| `pension` | `PENSION` |
| all others | `TRADITIONAL` |

When `planType` is absent, `policyType` from the request is used directly for filtering.

### Category normalization (`normalizeCategory()`)

All requested category strings are lowercased and normalized:
- `whole-of-life`, `whole-life`, `wholelife` → all become `"whole-life"`
- Separator normalized to `-`

### FORCE_INCLUDE_CATEGORY

`Set.of("child", "retirement")` — these two categories **always force the category filter to be applied**, even when `planType` is also set. Without this, setting a `planType` would skip the category filter and include products that don't have the `child` or `retirement` tag.

---

## Step 5 — Post-Filters Before Nearest Match

**File:** `LifeRequestValidationService.java` (line 346)
**Method:** `applyLifePostFiltersWithDefaults()`

After the basic product fetch and filter, two more filtering passes happen:

### 5a. Payout Defaults (broker = turtlemint only)

If `payoutType` is not set and `planType != TERM`:
- For ULIP/Participating: default `payoutType = LUMPSUM`
- For others: default `payoutType = INCOME`
- If no match: use first product's first payout type

If `payoutType = INCOME` and `payoutFrequency` not set: default `payoutFrequency = YEARLY`

### 5b. filterByPayoutSettings

If `planType` and `payoutType` are set:
- Keep only rows where `row.payoutTypes.contains(payoutType)`
- If `payoutTerm` specified: further filter by `row.payoutBenefitTerms.contains(payoutTerm)`
- If `payoutType = INCOME` and `payoutFrequency` specified: filter by `row.payoutFrequencies.contains(payoutFrequency)`

### 5c. filterByPlanFeatures

Requested plan features are extracted from the request:
- `isJointLife = true` → filter for rows with `planFeatureList` containing feature code `jointlife`
- `returnOfPremiumDetails.status = true` → filter for rows with feature code `returnofpremium`

---

## Step 6 — Nearest Match (life-validator)

**File:** `LifeRequestValidationService.java` (line 986)
**Method:** `applyNearestMatch()`

### URL

```
POST {integration-hub.nearest-match-base-url}/api/product-management/v1/life-validator
```

### Scope Formation

**File:** `LifeCategorySupport.java` (line 122) — `createNearestMatchScope()`

Before calling nearest match, a `scope` map is built from the `validationRequest`. The **exact fields** placed in the scope are:

```java
scope.put("age",                 getInteger(validationRequest.get("entryAge")));
scope.put("policyTerm",          getInteger(validationRequest.get("policyTerm")));
scope.put("premiumPaymentTerm",  getInteger(validationRequest.get("premiumPaymentTerm")));
scope.put("premiumPaymentMode",  getInteger(validationRequest.get("paymentFrequency")));
scope.put("income",              getLong(validationRequest.get("minIncome")));
scope.put("premium",             getDouble(validationRequest.get("premium")));
scope.put("sumAssuredOnDeath",   getLong(validationRequest.get("sumAssured")));
```

Field name mapping from request to scope:

| Scope Field | Source Field | Type | Notes |
|-------------|-------------|------|-------|
| `age` | `entryAge` | Integer | Insured's age at entry |
| `policyTerm` | `policyTerm` | Integer | Requested or derived |
| `premiumPaymentTerm` | `premiumPaymentTerm` | Integer | Requested or derived |
| `premiumPaymentMode` | `paymentFrequency` | Integer | 1=annual, 4=monthly, etc. |
| `income` | `minIncome` | Long | For income-replacement plans |
| `premium` | `premium` | Double | For premium-first plans (non-TERM) |
| `sumAssuredOnDeath` | `sumAssured` | Long | For cover-first plans (TERM) |

Note: `null` fields are included in the scope map — the nearest match API handles null values by ignoring them.

### Request Payload

```json
{
  "productCode": ["ICICILI_IPROTECT_SMART", "HDFCLI_CLICK2PROTECT", ...],
  "scope": {
    "age": 32,
    "policyTerm": 30,
    "premiumPaymentTerm": 30,
    "premiumPaymentMode": 1,
    "income": null,
    "premium": null,
    "sumAssuredOnDeath": 10000000
  }
}
```

The `productCode` array contains **internal product codes** from all eligible rows after step 5.

### Response Structure

```json
[
  {
    "productCode": "ICICILI_IPROTECT_SMART",
    "lifeValidatorOptions": [
      {
        "score": 0,
        "evaluatedAttributes": {
          "policyTerm": {"simpleValue": 30},
          "premiumPaymentTerm": {"simpleValue": 30}
        }
      }
    ],
    "productVariants": [
      {
        "variantCode": "1",
        "lifeValidatorOptions": [
          {
            "score": 5,
            "evaluatedAttributes": {
              "policyTerm": {"min": 10, "max": 40},
              "premiumPaymentTerm": {"simpleValue": 15}
            }
          }
        ]
      }
    ]
  }
]
```

### How the Response is Used — Score Logic

For each product + each variant, the system runs `selectNearestMatch()`:

1. **Filter** only options with a `score` field present.
2. **If any option has score = 0** (exact match) → immediately use that option with the user's own `policyTerm` and `premiumPaymentTerm`.
3. **If no exact match** → sort remaining options ascending by score, then by how close `premiumPaymentTerm` is to the user's requested value.
4. Pick the top option (`selected`).
5. Extract `nearestPolicyTerm` and `nearestPremiumPaymentTerm` from `evaluatedAttributes`:
   - If `simpleValue` present → use it.
   - If `min`/`max` range present:
     - If user value is within range → use user value.
     - If below min → use min; if above max → use max.
6. The row in our `validatorRows` list gets updated with `policyTerm`, `premiumPaymentTerm`, and `score`.

**Rows that have no match in the nearest match response are dropped** (they go into `unmatchedKeys` and are excluded from `updatedRows`).

This is the final list of rows that represent eligible plan+variant combinations.

---

## Step 7 — ValidationMap Creation

**File:** `LifeRequestValidationService.java` (line 1327)
**Method:** `createValidationMap()`

After nearest match, one `validationMap` entry is created **per unique `productCode`** from the survivor rows:

```json
{
  "TM_TERM_ICICILI_IPROTECT_SMART": {
    "valid": true,
    "key": "TM_TERM_ICICILI_IPROTECT_SMART",
    "insurerCode": "ICICILI",
    "internalProductCode": "ICICILI_IPROTECT_SMART",
    "optionCode": 1,
    "policyTerm": 30,
    "premiumPaymentTerm": 30,
    "score": 0
  },
  ...
}
```

**Note:** Only the first row per `productCode` gets an entry (duplicate productCodes are skipped). This map is returned as part of the API response as `validationMap` so FE knows which plans were validated and their resolved terms.

The final `LifeValidationResult` also contains:
- `rowsByProvider`: `{insurerCode → [validatorRows]}` — grouped for building service requests
- `providers`: filtered list of configured providers that have at least one valid row

---

## Step 8 — Building Per-Insurer Service Requests

**File:** `SachetLifeAggregator.java` (line 535)
**Method:** `buildLifeServiceRequests()`

For each `(insurerCode → rows)` entry in `validationResult.rowsByProvider`, and for each row:

1. **Optional filter by `planCode`**: If `request.planCode` is set, only rows matching that `planCode` are included.
2. **Optional filter for single-result flow**: If `_lifeSingleResultSelections` is in `otherDetails`, only rows matching the selected `productCode + optionCode` are included.
3. A new `QuotationRequest` is cloned from the main request with:
   - `provider = insurerCode`, `insurerCode = insurerCode`
   - `planCode = row.productCode`
   - `category = row.category`
   - `riskInsured.insurerCode = insurerCode`, `riskInsured.tenant = request.tenant.toUpperCase`
4. `otherDetails` is enriched with:

| Key | Value |
|-----|-------|
| `lifeRequestValidatorRows` | `[row]` — the single row for this request |
| `lifeScopedValidatorRows` | all rows for this insurer (all variants) |
| `lifeValidationMap` | the full validationMap from step 7 |
| `lifePendingKeyList` | list of all productCodes (derived from validationMap keys) |
| `lifeValidationRowsByProvider` | full rowsByProvider map |
| `uniqueId` | the current quoteId |

5. `attachLifePreEnrichmentMeta()` is called to fetch rider+offer metadata **synchronously before** the IH call.

---

## Step 9 — Pre-Enrichment (Rider & Offer Metadata)

**File:** `SachetLifeAggregator.java` (line 618) → `LifeQuoteMasterDataEnrichmentService.java` (line 262)

### Purpose

Before calling the Integration Hub, we need to know which riders and offers are eligible for each plan variant. This is fetched from MongoDB and validated via the Rule Engine.

### validationContext Formed

A `validationContext` map is built from the row + request:

```json
{
  "broker": "turtlemint",
  "productCode": "TM_TERM_ICICILI_IPROTECT_SMART",
  "internalProductCode": "ICICILI_IPROTECT_SMART",
  "optionCode": 1,
  "entryAge": 32,
  "policyTerm": 30,
  "premiumPaymentTerm": 30,
  "sumAssured": 10000000,
  "coverAmount": 10000000,
  "premium": null,
  "paymentFrequency": 1,
  "maturityAge": 62,
  "payoutType": null,
  "currency": "INR",
  "insurerCode": "ICICILI",
  "category": "term",
  "planType": "TERM",
  "offers": [...configured offer codes from PDP],
  "riders": [...configured rider codes from PDP],
  "showRider": true
}
```

### fetchRiderOfferMetaForPreEnrichment

**DB Lookup — lifeRiderMaster collection:**
- Queries by `riderCode in configured rider codes` + `insurerCode` + `currency`
- Fallback: queries by `insurerCode + broker + internalProductCode` scope if code-based lookup returns empty

**Rule Engine Call 1 — Rider Slabs:**
```
POST {rule-engine.base-url}/api/rules/v0/life/riders/slab
```
Payload: `{productCode, optionCode, broker, entryAge, policyTerm, premiumPaymentTerm, ruleId: "slab", riderInfoList: [...]}`

Response: each rider row gets slab ranges (min/max sumAssured, min/max premium, applicable policyTerms).

**Rule Engine Call 2 — Rider Validation:**
```
POST {rule-engine.base-url}/api/rules/v0/life/riders/validate
```
Payload: `{ruleId: "validateRiders", riderRequests: [{productCode, optionCode, riderCode, ...}]}`

Response: valid/invalid flag + resolved slab for each rider.

**DB Lookup — lifeOfferMaster collection:**
- Similar to riders: queries by `offerCode in configured offer codes` + `insurerCode`

### Output

After pre-enrichment, these keys are added to `otherDetails`:

| Key | Description |
|-----|-------------|
| `riderMasterList` | raw rider master rows from DB |
| `offerMasterList` | raw offer master rows from DB |
| `riderInfoList` | final list of validated riders (with slab info, isSelected flags) |
| `offerInfoList` | final list of validated offers |

If `showRider = false` on the PDP row, rider lookup is **skipped entirely**.

For `_lifeSingleResultFlow`, if `_lifeSingleResultSelections` already has specific rider/offer selections, those override the fetched `riderInfoList`/`offerInfoList` (via `applySingleResultSelectionOverrides()`).

---

## Step 10 — Async IH Premium Dispatch

**File:** `SachetLifeAggregator.java` (line 2551)
**Method:** `fetchCustomPremiumResponse()`

This is the core async pattern for the **fresh quote flow**:

```java
// 1. Trigger lead save (fire-and-forget)
saveLifeLeadForQuoteJourney(quotationRequest, QUOTE_STAGE, null).subscribe();

// 2. Fire all IH calls asynchronously — DO NOT WAIT
premiumServiceUtil.getIHPremiumResponse(serviceRequests, quotationRequest)
    .subscribeOn(Schedulers.boundedElastic())
    .subscribe(
        response -> { /* after all IH calls done */ saveLifeLead(QUOTE_STAGE, results) },
        error -> { /* log error */ }
    );

// 3. Return IMMEDIATELY with a PENDING response
return Mono.just(buildInitialLifeQuoteResponse(quotationRequest))
    .map(filterLifeRequoteResponse(...))
    .map(enrichLifeQuotesCurrency(...));
```

The **initial pending response** returned to FE is built by `buildInitialLifeQuoteResponse()`:
```json
{
  "productCode": "life",
  "referenceId": "ref-uuid",
  "quoteId": "quote-uuid",
  "uniqueId": "quote-uuid",
  "status": "pending",
  "pendingKeyList": ["TM_TERM_ICICILI_IPROTECT_SMART", "TM_TERM_HDFCLI_CLICK2PROTECT"],
  "validationMap": { ... },
  "quotes": [],
  "premiumRequest": { ...normalized request... }
}
```

The `pendingKeyList` is the list of all `productCode` keys from the validationMap — one per plan variant being fetched.

---

## Step 11 — IH Request Building (LifeQuoteIntegrationRequestBuilder)

**File:** `LifeQuoteIntegrationRequestBuilder.java` (`transactional-flows/.../service/aggregator/life/LifeQuoteIntegrationRequestBuilder.java`)

**Called by:** `LifePremiumDispatchService.fetchPreparedLifeQuotationResult()` (line 56)

`LifeQuoteIntegrationRequestBuilder.build()` (line 96) wraps `buildRequest()` in `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())` to keep blocking DB calls off the reactive thread.

---

### a) Build Request Context (`buildLifeRequestContext()`, line 957)

The request context is assembled by **merging multiple data sources in layered priority order** — each layer can override fields from the layer below:

```
Priority (lowest → highest, each overwrites previous):
  1. planDetails         (request.planDetails)
  2. personalDetails     (request.personalDetails)
  3. proposerDetails     (request.proposerDetails)
  4. insuredMember       (request.riskInsured.insuredMembers[0])
  5. riskInsured         (request.riskInsured — base fields)
  6. premiumRequest      (root request fields)
  7. requestMap          (otherDetails overrides — HIGHEST priority)
```

**Field aliases** resolved after merge (first non-blank wins):
| Normalized Field | Candidate Sources |
|-----------------|-------------------|
| `entryAge` | `entryAge` → `age` |
| `coverAmount` | `coverAmount` → `sumAssured` → `sumAssuredOnDeath` |
| `annualIncome` | `annualIncome` → `income` |
| `occupation` | `occupation` → `tmOccupation` |

---

### b) Insurer Constants & Broker Config Lookups

Before building the mapper, two DB lookups are performed:

**Insurer Constants (`fetchInsurerConstants()`, line 631):**
- **MongoDB Collection:** `lifeInsurerProviderMeta` (`@Document` via `LifeFieldMapper.LifeInsuranceCollections.LIFE_INSURER_PROVIDER_META`)
- **Query:** `insurerCode = insurerCode AND stage = "PREMIUM" AND broker = broker`
- **Returns:** `constants` field — `Map<String, Map<String, String>>` of insurer-specific key-value configs (API credentials, agent codes, fixed multipliers)

**Broker Config (`fetchInsurerConfig()`, line 667):**
- **MongoDB Collection:** `brokerConfig` (queried via broker-specific `ReactiveMongoTemplate` from `reactiveMongoTemplateMap`)
- **Query:** `broker = request.broker`
- **Returns:** `insurerConfig.{insurerCode}` — broker-level overrides (e.g., broker-specific API params for a given insurer)

---

### c) Building LifeIntegrationRequestMapper (`buildIntegrationRequestMapper()`, line 155)

The `LifeIntegrationRequestMapper` is the central POJO sent to IH. All fields come from the merged request context, validator row, and pre-enrichment data:

| Field | Source | Notes |
|-------|--------|-------|
| `age` | `requestContext.entryAge` | Integer |
| `category` | validator row | e.g., `term`, `ulip`, `guaranteed` |
| `payoutType` | `requestContext.payoutType` | |
| `sumAssured` | `requestContext.coverAmount` | **TERM category only** |
| `premium` | `requestContext.premium` | **Non-TERM category only** |
| `policyTerm` | validator row (nearest match resolved) | NOT raw request value |
| `premiumPaymentTerm` | validator row (nearest match resolved) | NOT raw request value |
| `productCode` | `internalProductCode` from validator row | IH-level code |
| `optionCode` | `optionCode` from validator row | |
| `productCodeOptionCode` | `productCode + "_" + optionCode` | Combined key |
| `premiumPaymentFrequency` | `paymentFrequency` via `IntegrationConstantUtils.getPPTKey()` | e.g., 1→`"Annual"`, 4→`"Monthly"` |
| `premiumPaymentOption` | derived from PT/PPT (see below) | `Single Pay` / `Regular Pay` / `Limited Pay` |
| `gender` | `requestContext.gender` | M/F |
| `dob` | `requestContext.dob` | Format varies by insurer (see below) |
| `pincode` | `requestContext.pincode` | |
| `tmPincode` | `requestContext.pincode` | Same value |
| `isSmoking` | `tobaccoStatus=Y` OR `smoker=true` → `"Y"`, else `"N"` | |
| `state` | `requestContext.state` | |
| `bqpState` | `requestContext.state` | Same as state |
| `rider` | only riders where `isSelected=true` | From pre-enrichment `riderInfoList` |
| `offer` | only offers where `isSelected=true` | From pre-enrichment `offerInfoList` |
| `followUpRequest` | `true` if any rider or offer is selected | |

**premiumPaymentOption derivation:**
```
PPT == 1            →  "Single Pay"
PPT == policyTerm   →  "Regular Pay"
PPT < policyTerm    →  "Limited Pay"
```

**DOB format by insurer:**
```
ICICIPRULI only:  yyyy-MM-dd
All others:       dd/MM/yyyy
```

---

### d) MappingQuery — IH Routing Key

The `MappingQuery` tells Integration Hub which config to use:

```json
{
  "integrationProvider": "ICICILI",
  "tmRequestType": "PREMIUM_REQUEST",
  "vertical": "LIFE",
  "broker": "turtlemint",
  "productCode": "ICICILI_IPROTECT_SMART"
}
```

IH uses this 5-tuple to look up the right transformation config, endpoint URL, and field mappings for the insurer.

---

### e) UID Generation (`buildUid()`, line 1273)

The UID for each IH call is an MD5 hash for idempotency (same inputs → same UID):

```
uid = MD5(
  requestId + "_" + productCode + "_" + policyTerm + "_" + PPT
  + "_" + optionCode + "_" + paymentFreq + "_" + age + "_" + gender
  + "_" + coverAmount + "_" + inputBonus + "_" + smoker
) + "__" + productCode + "__" + optionCode
```

---

### f) Requote Flag Resolution (`resolveInsurerSpecificInfo()`, line 1023)

If `_lifeRequoteInsurerSpecificInfo` is present in `otherDetails`:
- Searches for a key matching prefix `insurerCode_productCode_`
- If found → merges the stored insurer-specific fields and sets `Requote = "Yes"`
- If not found → sets `Requote = "No"`

This `Requote` flag is forwarded to IH so the insurer API can handle the requote differently (e.g., reuse a cached session, skip re-underwriting).

---

### g) Master Mapper for TERM — Occupation & Qualification

**File:** `LifeQuoteIntegrationRequestBuilder.java` (line 703)

**Only for `category = "term"`**, two external lookups map TM codes to insurer-specific codes:

```
GET {masterServiceV2Host}/api/v2/masters/master-mappings/turtlemint/{insurerCode}/OCCUPATION/LIFE/filter
    ?occupation_code={tmOccupation}

GET {masterServiceV2Host}/api/v2/masters/master-mappings/turtlemint/{insurerCode}/QUALIFICATION/LIFE/filter
    ?qualification_code={tmQualification}
```

The returned insurer-specific occupation and qualification codes are added to the mapper.

---

### h) Insurer-Specific Enrichment (applied AFTER base mapper)

---

**BAJAJ ALLIANZ (`BAJAJLI`) — Cover Codes**

BAJAJ requires a proprietary "cover code" rider system. `enrichBajajCoverDetailsRider()` maps internal `productCode_optionCode` to BAJAJ rider API codes:

```java
BALIC_COVER_CODES = {
  "P94_1" → "L176A01",   // Accidental Death Cover
  "P94_2" → "L176B01",
  "P91_1" → "L190B01",   // Critical Illness
  "P91_2" → "L190B01",
  "P76_1" → "L177A04",   // Waiver of Premium
  "P76_2" → "L177A06",
  "P76_3" → "L177A05"
}
```

For each rider with `isSelected=true`, if a matching cover code exists, the rider is enriched with:
- `riderApiCode` ← BAJAJ's cover code (e.g., `"L176A01"`)
- `riderSumAssured`, `riderPolicyTerm`, `riderPPT`

---

**ICICI PRUDENTIAL (`ICICIPRULI`) — SA Multiple**

ICICI requires a `saMultiple` field for premium-to-SA ratio:

```java
if (age < 54 || productCode in ["P64", "P66", "P70"])  saMultiple = 10
else if (productCode == "P67")                          saMultiple = 10
else                                                    saMultiple = 0
```

---

**MAX LIFE (`MAXLIFELI`) — Model Factors & Committed Premium**

`applyMaxLifeInsurerSpecificFieldsFallback()` adds:
- `productName`, `optionCode`, `benefitPayoutFetched`, `planType`
- `vestingAge` / `retirementAge` (for pension plans)
- `committedPremium` — computed for product P80 using **model factors**:
  ```java
  MODEL_FACTORS = {12 → 1.0, 1 → 0.09}
  committedPremium = premium × MODEL_FACTORS[paymentFrequency]
  ```

---

**TATA AIA (`TATAAIALI`) and DIGIT LIFE (`DIGITLI`) — Plan Codes from Rule Engine**

`enrichPlanCodeFromRuleEngine()` calls:

```
POST {ruleEngineBaseUrl}/api/rules/v0/life/planCodes
```
```json
{
  "productCode": "TATAAIALI_MAHA_LIFE_GOLD",
  "optionCode": "1",
  "policyTerm": 20,
  "premiumPaymentTerm": 20,
  "age": 35,
  "paymentFrequency": 1,
  "smoker": "N"
}
```

Note: `policyTerm`/`premiumPaymentTerm` may be adjusted by `ptNvestFlag`/`pptNvestFlag` on the product row.

Response: `planCode` + `baseSumAssured` — both applied to the request mapper.

---

**HDFC LIFE (`HDFCLI`) — Mintpro Partner Details**

`fetchPartnerDetails()` calls:

```
GET {mintproPartnerApiUrl}/{clientId}
Headers: x-broker: {broker}
```

Response fields added to the mapper:
- `dpNo` — distributor point number
- `branchCode`
- `partnerName`, `partnerEmail`, `partnerMobile`
- `spCompositeLicense` — composite license number

---

## Step 12 — IH Response Normalization (Per-Insurer Normalizers)

**File:** `LifePremiumDispatchService.java` → `LifeResponseNormalizerFactory` + insurer normalizer files
**Location:** `transactional-flows/.../service/aggregator/life/normalizer/`

### Factory Pattern

`LifeResponseNormalizerFactory` (line 18) is a Spring `@Component` that builds a `Map<insurerCode → LifeResponseNormalizer>` from all `@Component` implementations of `LifeResponseNormalizer`. It falls back to `GenericResponseNormalizer` for unknown insurers.

```
ICICIPRULI  → IciciResponseNormalizer
BAJAJLI     → BajajResponseNormalizer
HDFCLI      → HdfcResponseNormalizer
MAXLIFELI   → MaxLifeResponseNormalizer
TATAAIALI   → TataResponseNormalizer
DIGITLI     → DigitResponseNormalizer
ABSLIFELI   → AdityaBirlaResponseNormalizer
AEGONLI     → AegonResponseNormalizer
(all others) → GenericResponseNormalizer
```

### What a Normalizer Does

Each normalizer implements `normalize(ihData, requestContext)` and:
1. **Unwraps** the insurer-specific wrapper key from `ihData` (e.g., ICICI uses `"icicipruliResponse"` key; generic uses `"genericResponse"` or falls back to root)
2. **Maps insurer field names → standard fields**:
   - `basePremium` → `premium` (ICICI terminology)
   - `premiumWithoutTax` → `premium` (BAJAJ/generic terminology)
   - `serviceTax` → `tax` (ICICI); `tax` (others)
3. **Applies modal factor** — annual premium/tax derived from per-frequency values: `premiumAnnual = premium × (12/paymentFrequency)`
4. **Sets death/survival benefit fields**: `deathBenefitGuaranteed`, `deathBenefitTotal`, `survivalBenefitTotal`
5. **Sets status fields**: `status`, `insurerStatus`, `insurerMessage`, `insurerBusinessFlowType`
6. **Normalizes riderList**: maps `rider` (ICICI key) or `riderList` (others) → standard `riderList`
7. **Sets** `policyType`, `isBIPdfAvailable = true`

### Standard Normalized Response Fields

After normalization, these fields are always present (if available from IH):

| Field | Type | Description |
|-------|------|-------------|
| `status` | String | `SUCCESS`/`ERROR`/`PENDING` |
| `insurerStatus` | String | Raw insurer status string |
| `premium` | Double | Per-frequency premium (excl. tax) |
| `premiumAnnual` | Double | Annual premium (excl. tax) |
| `tax` | Double | Per-frequency tax |
| `taxAnnual` | Double | Annual tax |
| `premiumWithTax` | Double | Per-frequency total (premium + tax) |
| `sumAssured` | Double | Cover amount |
| `deathBenefitGuaranteed` | Long | Death benefit |
| `deathBenefitTotal` | Long | Death benefit incl. bonuses |
| `survivalBenefitTotal` | Long | Total maturity/survival benefit |
| `riderList` | List | Riders from IH response |
| `policyType` | String | TERM/ULIP/PENSION/TRADITIONAL |
| `isBIPdfAvailable` | Boolean | Always `true` |

### After Normalization (back in `LifePremiumDispatchService`)

After normalization, `buildQuoteResultFromIHResponse()` does:

1. **`enrichLifePremiumResponseWithValidatorRow()`** (line 330): Adds fields from the matched validator row to the normalized response:
   - `optionCode`, `optionName`, `productCode`, `internalProductCode`, `insurerCode`, `tmPlanId`
   - `planType`, `policyType`, `categoryTags`, `paymentFrequencyModes`, `payoutTypes`, `payoutFrequencies`, `planFeatureList`
   - `policyTerm`, `premiumPaymentTerm`, `score`, `defaultOption`/`isDefaultOption`
   - All fields from `LIFE_VALIDATOR_METADATA_FIELDS` list

2. **`carryLifePreEnrichmentMeta()`** (line 392): Transfers internal keys from `request.otherDetails` into the premiumResponse (for use during enrichment):
   - `_preValidatedRiderInfoList` ← `otherDetails.riderInfoList`
   - `_preValidatedOfferInfoList` ← `otherDetails.offerInfoList`
   - `_lifeScopedValidatorRows` ← `otherDetails.lifeScopedValidatorRows`
   - `_lifeBroker` ← `request.broker`
   - Calculates `taxSavingAmount` based on premium and maxIncome

3. **Build `LifeQuotationResult`**:

```java
LifeQuotationResult {
  planCode        ← premiumResponse.productCode or request.planCode
  planName        ← request.planName
  policyTerm      ← riskInsured.policyTerm
  riskInsured     ← with insurerCode set
  status          ← "success" / "pending" / "failed"
  premium         ← premiumResponse.premium
  tax             ← premiumResponse.taxRate
  totalPremium    ← premiumResponse.premiumWithTax
  premiumResponse ← enriched normalized response map
  validationMap   ← from IH response or otherDetails.lifeValidationMap (on error)
  pendingKeyList  ← from IH response data.pendingKeyList
  errorMessage    ← from IH or error
}
```

Status mapping:
- `SUCCESS` or `OK` → `"success"`
- `PENDING` or `IN_PROGRESS` → `"pending"`
- Anything else → `"failed"`

---

## Step 13 — Post-IH Enrichment & DB Persistence

**File:** `LifePremiumDispatchService.java` (line 232) → `LifeQuoteMasterDataEnrichmentService.java` (line 98)

### enrichQuoteResponse (called on the premiumResponse before save)

This is a `Mono.zip()` of 8 parallel enrichments:

| Enrichment | DB/Source | What it adds |
|-----------|-----------|--------------|
| `enrichCompanyDetails` | MongoDB: `CompanyDetails` collection (query by `insurerCode`) | `companyDetails: {InsurerCode, companyName, logo, website, claimSettlementRatio, solvencyRatio, ...}` |
| `enrichRiderList` | In-memory: `_preValidatedRiderInfoList` already in premiumResponse (fetched in Step 9 from `lifeRiderMaster`) | `riderList: [...]` — finalized rider list with premium, slab info |
| `enrichOfferList` | In-memory: `_preValidatedOfferInfoList` already in premiumResponse (fetched in Step 9 from `lifeOfferMaster`) | `offerList: [...]` |
| `enrichUlipFundAllocation` | MongoDB: `lifeUlipFundAllocationMaster` collection (query by `productCode`) — ULIP plans only | `ulipFundAllocation: {...}` |
| `enrichResponseOptions` | In-memory: splits multi-option IH responses | `responseOptions: [...]` |
| `enrichErrorCategory` | MongoDB: `LifeResponseAttributionRules` collection (query by `insurerCode + vertical=LIFE + stage=QUOTE`) | Error categorization fields (`errorCategory`, `errorSubCategory`) |
| `enrichTaxSavingsInfo` | Classpath: `income-tax-savings-config.json` + `_preValidatedInBuiltRiderCodes` (no DB call) | `taxSavingsInfo: {...}` |
| `enrichResultCardsInfo` | Computed from response fields (no DB call) | `resultCardsInfo: {...}` |

After `Mono.zip()`:
- `applyDerivedQuoteFields()` — computes derived fields
- `buildResultCardsInfo()` + `mergeResultCardsInfo()` — result card data for FE
- `applyBiTimeline()` — benefit illustration timeline
- `applyRiderPremiumFields()` — applies rider premium amounts to the rider list
- **Removes all internal keys** (`_preValidatedRiderInfoList`, `_preValidatedOfferInfoList`, `_lifeBroker`, `_lifeScopedValidatorRows`, `_lifeSingleResultFlow`, `_lifeVariantRiderEnrichment`)

### DB Persistence

After enrichment, `defaultAggregatorService.savePremiumResult()` is called, which saves to:

**Collection: `sachetPremiumResponse`**

Key fields stored:
- `referenceId`, `quoteId`, `uniqueId` (all set to the same quoteId)
- `productCode = "life"`
- `provider = insurerCode`
- `planCode`
- `status` (`"success"` / `"pending"` / `"failed"`)
- `premium`, `tax`, `totalPremium`
- `premiumResponse` — full enriched insurer response map
- `validationMap`
- `pendingKeyList`
- `errorMessage`
- `riskInsured`
- `createdAt`, `updatedAt`

For requote flow, `preserveLifeRequoteQuoteId()` ensures the result is saved with the correct quoteId.

---

## Step 14 — Async Rider Prices Flow

**File:** `PremiumServiceUtil.java` (line 1021) → `LifeRiderPricesAsyncService.java`

After a **successful** (status = `"success"`) quote result is saved, rider prices are computed asynchronously (only for non-single-result flows):

```java
lifeRiderPricesAsyncService.generateAndSaveRiderPrices(serviceRequest)
    .subscribeOn(Schedulers.parallel())
    .subscribe(...)
```

### How Rider Prices are Generated

1. **Read** `riderInfoList` from the service request's `otherDetails` (the pre-validated rider list).
2. **Get compatible rider groups** via Rule Engine:
   ```
   POST {rule-engine.base-url}/api/rules/v0/life/riders/interdependency
   ```
   Payload: `{riderRequestList: [{productCode, optionCode, riderCode, ruleId: "interdependentRiders", broker}]}`
   
   Response: groups of mutually compatible rider combinations.
3. For each rider combination group, **fire a new IH quote call** (`lifePremiumDispatchService.fetchPreparedLifeQuotationResult()`) with that specific rider combination set as `isSelected=true`.
4. Read `riderList` from the IH response.

### DB Persistence

**Collection: `sachetLifeRiderPrices`**

Per rider group, a `LifeRiderPrices` object is upserted:
- `requestId = referenceId`
- `uniqueId = quoteId`
- `productCode`
- `optionCode`
- `riderCodes` (list of selected rider codes in this group)
- `riderList` (full rider response from IH)

---

## Step 15 — Initial Response to FE

The FE immediately receives (synchronously, before IH calls complete):

```json
{
  "meta": { "status": "SUCCESS", "error": false },
  "data": {
    "productCode": "life",
    "referenceId": "REF-UUID",
    "quoteId": "QUOTE-UUID",
    "uniqueId": "QUOTE-UUID",
    "status": "pending",
    "pendingKeyList": ["TM_TERM_ICICILI_IPROTECT_SMART", "TM_TERM_HDFCLI_CLICK2PROTECT"],
    "validationMap": {
      "TM_TERM_ICICILI_IPROTECT_SMART": {
        "valid": true,
        "key": "TM_TERM_ICICILI_IPROTECT_SMART",
        "insurerCode": "ICICILI",
        "policyTerm": 30,
        "premiumPaymentTerm": 30,
        "score": 0
      }
    },
    "quotes": [],
    "premiumRequest": { ...normalized input request... }
  }
}
```

The FE then polls: `GET /api/minterprise/v2/products/life/quotes/poll?referenceId=...&uniqueId=...&keyList=...`

---

## Step 16 — GET /quotes & /poll — Response Formation

**Files:** `SachetController.java` → `IQuotationServiceImpl.java` → `SachetLifeAggregator.java`

There are **two GET endpoints** for reading quote results, both ultimately converging on the same `buildQuoteResponse()` method:

---

### GET /api/minterprise/v2/products/life/quotes

```
GET /api/minterprise/v2/products/{productCode}/quotes
    ?referenceId=...
    &includeRequest=true|false     (default: false)
    &filterValue=...               (optional)
Headers: x-provider (optional insurer filter)
```

**Flow:**

1. **`IQuotationServiceImpl.fetchResultFromDB()`** (line 300): Calls `quoteFlowAggregator.buildQuoteApiResult(referenceId, insurerCode, includeRequest, filterValue)`
2. **`SachetLifeAggregator.buildQuoteApiResult()`** (line 6332): Calls `fetchLifeAggregateResponse(referenceId, null, insurerCode, includeRequest, null)` — note `uniqueId=null` means use the latest quoteId from the stored request
3. **If `includeRequest=true`**: After building the response, fetches the stored `QuotationRequest` from `sachetPremiumRequest` by `referenceId` and attaches it as `premiumRequest` in the returned `QuotationResult`

**Redis Cache:** For the legacy single-result path (when `quoteId` is provided in properties), the result is first looked up in Redis cache (`key = quoteId + insurerCode + referenceId`), then falls back to DB, then falls back to `buildQuoteApiResult()`.

---

### GET /api/minterprise/v2/products/life/quotes/poll

```
GET /api/minterprise/v2/products/{productCode}/quotes/poll
    ?referenceId=...
    &uniqueId=...         (the quoteId returned in the POST response)
    &keyList=...          (comma-separated list of productCode keys still pending)
```

**Validation:** Returns `400 Bad Request` immediately if `keyList` is empty or `referenceId`/`uniqueId` is blank.

**Flow:** `IQuotationServiceImpl.fetchResultsByReferenceIdAndUniqueIdAndKeyList()` (line 546) → `SachetLifeAggregator.fetchResultsByReferenceIdAndUniqueIdAndKeyList()` (line 6346) → `fetchLifeAggregateResponse(referenceId, uniqueId, null, false, keyList)`

---

### fetchLifeAggregateResponse (common to both endpoints, line 6386)

```java
1. Find stored QuotationRequest by referenceId from sachetPremiumRequest
   → If not found, build fallback request from {referenceId, uniqueId}
2. buildEffectiveLifeAggregateRequest():
   → Sets productCode="life", referenceId
   → Resolves uniqueId: firstNonBlank(urlParam uniqueId, storedRequest.quoteId)
   → Puts uniqueId into otherDetails for runId resolution
3. fetchLifeQuoteRowsForRun():
   → MongoDB query on sachetPremiumResponse:
     - referenceId = referenceId
     - productCode = "life"
     - uniqueId = uniqueId      (from param or stored request)
     - provider = insurerCode    (if x-provider header set)
     - planCode/premiumResponse.productCode IN keyList  (if keyList provided)
     - Sort: updatedAt DESC
4. buildQuoteResponse(referenceId, effectiveRequest, quoteRows, includeRequest, requestedKeys)
```

---

### buildQuoteResponse (line 6041)

This method builds the final `QuotationResponse` from a list of raw DB rows. Steps in order:

#### Step 1 — Resolve Current Validation Result

`resolveCurrentLifeValidationResult()` — tries to get the validation context in priority order:
1. From `otherDetails.lifeValidationRowsByProvider` + `otherDetails.lifeValidationMap` in the stored request (fast path — already computed during POST)
2. If not present, re-runs `lifeRequestValidationService.resolveValidatedProviders()` live (slow path — applicable products + nearest match again)

#### Step 2 — Filter Quote Rows (4 passes)

```
Pass 1: filterLifeQuoteRowsByRequestedInsurers()
  → If planDetails.insurers was set, keep only rows for those insurers
  → Matches by resolveLifeQuoteRowInsurer(row)

Pass 2: filterLifeQuoteRowsByCurrentRunId()
  → Keep only rows where row.uniqueId == resolveLifeRunId(premiumRequest)
  → resolveLifeRunId() = premiumRequest.quoteId or otherDetails.uniqueId

Pass 3: filterLifeQuoteRowsByCurrentValidation()
  → Builds allowedProductKeys = { insurerCode|productCode } from validationResult.rowsByProvider
  → Keep only rows whose (insurerCode|productCode) key is in allowedProductKeys

Pass 4: flattenLifeQuoteRowsForResponse() [compaction + flattening]
  → compactLifeQuoteRowsForResponse(): groups rows by {referenceId|provider|planCode} key
    - If multiple rows for same group: picks the most recently updated, merges responseOptions
    - Builds unified variants list from all rows in the group
  → For each compacted row: mergeLifeResponseFieldsFromStoredProductDetails()
```

#### Step 3 — computePendingKeyList()

```
basePendingKeys = from otherDetails.lifePendingKeyList (set during POST)
                  or from validationMap keys (fallback)
                  or from requestedKeys (poll keyList param)

For each quote row:
  - Resolve the productCode key for that row
  - If row is still pending (status=pending OR premiumResponse.status=PENDING): add key back
  - If row is complete (success/failed): remove from pending set
```

A quote row is considered "pending" if:
- `row.status == "pending"`, OR
- `premiumResponse.status == "PENDING"` or `premiumResponse.insurerStatus == "PENDING"`

#### Step 4 — resolveAggregateStatus()

```
if pendingKeyList is non-empty → "pending"
else if any row has status == "success" or premiumResponse.status == "SUCCESS" → "success"
else if any row has status == "pending" → "pending"
else → "failed"

Special case: if safeQuoteRows is empty AND pendingKeyList is non-empty → "pending"
              if safeQuoteRows is empty AND pendingKeyList is empty → "failed"
```

#### Step 5 — Build Response Object

```java
QuotationResponse {
  productCode  = "life"
  referenceId  = referenceId
  quoteId      = resolveLifeRunId(premiumRequest)
  uniqueId     = resolveLifeRunId(premiumRequest)
  status       = "success" | "pending" | "failed"
  pendingKeyList = [...remaining products not yet returned]
  validationMap  = from otherDetails or re-resolved validation
  error          = first non-blank errorMessage from failed rows (if not success)
  transactionSource / transactionMode  = from premiumRequest or first quote row
}
```

#### Step 6 — buildLifeApiQuotes() (line 6616)

For each row in `responseQuoteRows`:
1. Converts `LifeQuotationResult` → raw `premiumResponse` map via `normalizeLifeQuoteMap()`
2. `ensureLifeResponseCoreFields()` — ensures `insurerCode`, `productCode`, `policyTerm`, `premiumPaymentTerm` are present at root level
3. Calls `lifeQuoteMasterDataEnrichmentService.enrichQuoteResponse(premiumResponse, true)` — applies the **same 8-way parallel enrichment** used after IH (company details, riders, offers, ULIP fund allocation, response options, error category, tax savings, result cards)

**Note:** enrichment here is applied again at read time (`buildLifeApiQuotes`) to ensure the latest master data is always returned, even if the stored response is stale.

#### Step 7 — applyCurrentLifeValidationToQuotes()

After building the quote maps, each quote map gets validation metadata merged in from the current `validationResult`. This ensures that even if the stored premiumResponse is missing some metadata fields, the latest validated terms (policyTerm, PPT, score) are applied.

#### Step 8 — Final Ordering and Request Attachment

```
reorderLifeDefaultQuoteRoot() → for each quote, moves defaultOption=true variant to first position in responseOptions

normalizeLifeRequestForResponse() → strips internal keys (_*) from the premiumRequest before attaching

enrichLifeQuotesCurrency() → applies currency conversion if needed
```

#### Final Response Shape

```json
{
  "meta": {"status": "SUCCESS", "error": false},
  "data": {
    "productCode": "life",
    "referenceId": "REF-UUID",
    "quoteId": "QUOTE-UUID",
    "uniqueId": "QUOTE-UUID",
    "status": "success",
    "pendingKeyList": [],
    "validationMap": {
      "TM_TERM_ICICILI_IPROTECT_SMART": {
        "valid": true, "key": "TM_TERM_ICICILI_IPROTECT_SMART",
        "insurerCode": "ICICILI", "policyTerm": 30, "premiumPaymentTerm": 30, "score": 0
      }
    },
    "quotes": [
      {
        "productCode": "TM_TERM_ICICILI_IPROTECT_SMART",
        "insurerCode": "ICICILI",
        "optionCode": 1,
        "policyTerm": 30,
        "premiumPaymentTerm": 30,
        "premium": 8500.0,
        "premiumAnnual": 8500.0,
        "tax": 1530.0,
        "premiumWithTax": 10030.0,
        "sumAssured": 10000000,
        "deathBenefitGuaranteed": 10000000,
        "status": "SUCCESS",
        "companyDetails": {...},
        "riderList": [...],
        "offerList": [...],
        "taxSavingsInfo": {...},
        "resultCardsInfo": {...},
        "responseOptions": [...other variants...],
        "score": 0,
        "defaultOption": true
      }
    ],
    "premiumRequest": { ...normalized input request (with _internal keys stripped)... }
  }
}
```

---

## Database Collections Summary

### Write Collections (transactional data)

| Collection | Constant | Purpose | When Written |
|------------|----------|---------|--------------|
| `sachetPremiumRequest` | `DBColConstants.GRID_BASED.QUOTATION_REQUEST` | Stores quote requests | Every POST /quotes call (upsert by referenceId) |
| `sachetPremiumRequestHistory` | `DBColConstants.GRID_BASED.QUOTATION_REQUEST_HISTORY` | Audit trail of all request saves | Fire-and-forget after every request save |
| `sachetPremiumResponse` | `DBColConstants.GRID_BASED.QUOTATION_RESPONSE` | Stores enriched quote results per insurer | After each IH call completes (one doc per insurer) |
| `sachetLifeRiderPrices` | `DBColConstants.GRID_BASED.LIFE_RIDER_PRICES` | Pre-computed rider price combinations per plan | Async after successful non-single-result quote |

### Read-Only Collections (master / config data — MongoDB)

| Collection | Query Key | Purpose | Step Used |
|------------|-----------|---------|-----------|
| `lifeRiderMaster` | `riderCode IN codes + insurerCode + currency` | Rider master definitions — slab ranges, isSelected flags, categories | Step 9 (pre-enrichment before IH call) |
| `lifeOfferMaster` | `offerCode IN codes + insurerCode` | Offer master definitions | Step 9 (pre-enrichment before IH call) |
| `lifeInsurerProviderMeta` | `insurerCode + stage="PREMIUM" + broker` | Insurer-specific API constants (credentials, fixed params) needed for IH scope | Step 11 (`fetchInsurerConstants()` — building IH scope) |
| `brokerConfig` | `broker` | Broker-level insurer overrides (`insurerConfig.{insurerCode}`) | Step 11 (`fetchInsurerConfig()` — uses broker-specific ReactiveMongoTemplate) |
| `CompanyDetails` | `insurerCode` | Insurer company details (name, logo, CSR, solvency ratio) | Step 13 (`enrichCompanyDetails()` — post-IH enrichment) |
| `lifeUlipFundAllocationMaster` | `productCode` | ULIP fund allocation options | Step 13 (`enrichUlipFundAllocation()` — ULIP plans only) |
| `LifeResponseAttributionRules` | `insurerCode + vertical=LIFE + stage=QUOTE` | Error categorization rules for response attribution | Step 13 (`enrichErrorCategory()`) |

---

## Database / External Calls

| System | Collection / Endpoint | Operation | When |
|---|---|---|---|
| **MongoDB** | `sachetPremiumRequest` | Read — load stored request on modify/requote (by `referenceId`) | Non-initial calls (referenceId already exists) |
| **MongoDB** | `sachetPremiumRequest` | Write — upsert quote request (by `referenceId`) | Every POST /quotes call |
| **MongoDB** | `sachetPremiumRequestHistory` | Write — fire-and-forget audit copy | After every request save |
| **MongoDB** | `sachetPremiumResponse` | Write — persist enriched quote result per insurer | After each IH premium call completes |
| **MongoDB** | `sachetPremiumResponse` | Read — fetch results for poll/GET response | Step 16 (GET /quotes, /poll) |
| **MongoDB** | `sachetLifeRiderPrices` | Write — store pre-computed rider price combinations | Async after successful non-single-result quote (Step 14) |
| **MongoDB** | `lifeRiderMaster` | Read — rider definitions by `riderCode + insurerCode + currency` | Step 9 — pre-enrichment for each insurer request |
| **MongoDB** | `lifeOfferMaster` | Read — offer definitions by `offerCode + insurerCode` | Step 9 — pre-enrichment for each insurer request |
| **MongoDB** | `lifeInsurerProviderMeta` | Read — insurer API constants by `insurerCode + stage="PREMIUM" + broker` | Step 11 — building IH scope (`fetchInsurerConstants()`) |
| **MongoDB** | `brokerConfig` | Read — broker-level insurer overrides by `broker` | Step 11 — building IH scope (`fetchInsurerConfig()`) |
| **MongoDB** | `CompanyDetails` | Read — insurer info by `insurerCode` | Step 13 — post-IH enrichment (`enrichCompanyDetails()`) |
| **MongoDB** | `lifeUlipFundAllocationMaster` | Read — fund allocation by `productCode` | Step 13 — post-IH enrichment, ULIP plans only |
| **MongoDB** | `LifeResponseAttributionRules` | Read — error rules by `insurerCode + vertical=LIFE + stage=QUOTE` | Step 13 — post-IH enrichment (`enrichErrorCategory()`) |
| **Redis** | `LifeProductCatalogue_<broker>` (TTL: 4h) | Read — product catalogue cache | Step 4 — cache hit skips IH Product Management call |
| **IH Product Management API** | `POST /api/product-management/v1/products/details/filters` | Read — all broker-configured life products + PDP fields | Step 4 — on Redis cache miss only |
| **IH Life Validator API** | `POST /api/product-management/v1/life-validator` | Read — nearest valid PT/PPT/score per product for user inputs | Step 6 — every request (not cached) |
| **Rule Engine** | `POST {rule-engine.base-url}/api/rules/v0/life/riders/slab` | Read — rider slab ranges (min/max SA, premium, applicable terms) | Step 9 — per insurer, if `showRider=true` |
| **Rule Engine** | `POST {rule-engine.base-url}/api/rules/v0/life/riders/validate` | Read — rider validity + resolved slab per rider | Step 9 — per insurer, if `showRider=true` |
| **Integration Hub** | Insurer premium endpoint (`TMRequestType.PREMIUM_REQUEST`) | POST — compute premium for each insurer product | Step 10 — async per insurer (parallel) |

---

## Modify Flow (same referenceId, new quoteId)

**Scenario:** User changes some parameters (age, cover, term) and hits the quotes API again with the same `referenceId` but **either no quoteId or a new quoteId**.

### How It Works

**In `validatePremium()`:**

1. `referenceId` is present → `isInitial = false`
2. `incomingQuoteId` is **blank** (or a new value the FE sends) → system generates a **fresh `quoteId`**
3. Fetches the stored `QuotationRequest` from `sachetPremiumRequest`
4. Calls `prepareQuoteRequestForValidation(storedRequest, newRequest)`:
   - Copies `_id` and `createdAt` from stored request (so the same DB document is updated, not a new one)
   - Calls `mergeStoredLifePremiumRequest()` — carries forward any stored fields the new request doesn't override
   - Calls `preserveStoredLifeInsurerSpecificInfo()` — preserves insurer-specific info from stored request

**Result:** The stored `sachetPremiumRequest` document is **updated** (same `_id`) with the new quoteId and new request data.

**In `generateQuote()`:** runs the complete fresh flow — applicable products, nearest match, builds new service requests, fires new IH calls asynchronously.

**In `fetchCustomPremiumResponse()`:** returns a new `pending` response with the new `quoteId` as `uniqueId`.

**Key difference from fresh:** The DB document is updated in-place (same `_id`), but the new `quoteId` means all new `sachetPremiumResponse` rows are tagged with this new quoteId. Old quote rows (from previous quoteId) are not deleted — they simply won't match the `filterLifeQuoteRowsByCurrentRunId()` filter in `buildQuoteResponse()`.

---

## Requote Flow (same referenceId, same quoteId)

**Scenario:** User has already seen quotes and wants to re-fetch the quote for a specific plan (e.g., changed riders, offers, or cover amount for a selected insurer). Both `referenceId` AND `quoteId` are kept the same.

### How It Differs

**Identification:** The request contains `_lifeRequoteSelection` OR `_lifeRequoteInsurerSpecificInfo` in `otherDetails` (or nested under `premiumRequest.otherDetails`). This is the signal that this is a requote.

```json
// Option 1: single insurer/product selection
"otherDetails": {
  "_lifeRequoteSelection": {
    "insurerCode": "ICICILI",
    "productCode": "TM_TERM_ICICILI_IPROTECT_SMART"
  }
}

// Option 2: multi-insurer insurer-specific info
"otherDetails": {
  "_lifeRequoteInsurerSpecificInfo": {
    "ICICILI_TM_TERM_ICICILI_IPROTECT_SMART": { ...insurer specific fields... }
  }
}
```

**In `validatePremium()`:**
1. `incomingQuoteId` is **present** (not blank) → `request.setQuoteId(incomingQuoteId)` — **quoteId is preserved as-is**
2. Fetches stored request, calls `prepareQuoteRequestForValidation()` → `prepareLifeRequoteMetadata()` (looks up existing quotes from DB to carry forward requote metadata)

**In `generateQuote()`:**
```java
LifeValidationResult effectiveValidationResult = filterValidationResultForRequote(request, validationResult);
```

`filterValidationResultForRequote()` restricts the validation rows to ONLY the rows matching the requested `insurerCode + productCode`. All other insurers/products are stripped from `rowsByProvider` and `validationMap`.

**IH Call — Synchronous for Requote:**
```java
if (isLifeRequoteRequest(request)) {
    return persistLifeRunState(request)
        .then(premiumServiceUtil.getIHPremiumResponse(serviceRequests, request))  // SYNC - waits
        .flatMap(response -> saveLifeLead(...))
        .then(fetchLifeAggregateResponse(...))  // reads ALL rows for this referenceId + quoteId
        .map(response -> filterLifeRequoteResponse(request, response))  // filter to only requoted plan
        .map(response -> enrichLifeQuotesCurrency(request, response));
}
```

Unlike the fresh flow:
- IH calls are **awaited** (not fire-and-forget)
- After IH completes, `fetchLifeAggregateResponse()` reads all rows for `referenceId + quoteId` from DB
- `filterLifeRequoteResponse()` then filters the aggregate response to only include the requoted plan's quotes
- **Full response** (not pending) is returned to FE

**DB:** New `sachetPremiumResponse` rows are created with the same `quoteId` (overwriting or adding to the same run). The old rows from other insurers are still there and included in `buildQuoteResponse()` for non-requote reads.

---

## Internal Metadata Keys Reference

These keys are used internally in `otherDetails` and/or `premiumResponse` during the quote flow. They are prefixed with `_` to distinguish from FE-facing fields:

| Key | Set By | Purpose |
|-----|--------|---------|
| `_lifeRequoteSelection` | FE/request | Signals requote; identifies specific insurer+product |
| `_lifeRequoteInsurerSpecificInfo` | FE/request | Multi-insurer requote with insurer-specific data |
| `_lifeRequestedQuoteId` | `normalizeQuoteRequestForStorage` | Stashes the original quoteId sent by FE |
| `_lifeSingleResultFlow` | Set if `_lifeSingleResultSelections` present | Signals single-result mode (no async, waits for all) |
| `_lifeSingleResultSelections` | FE/request | Specific productCode+optionCode+rider/offer selections |
| `_lifeModifyFlow` | Set on modify | Signals this is a modification of existing request |
| `_lifeModifyRunId` | Set on modify | The runId being modified |
| `_lifeBroker` | `carryLifePreEnrichmentMeta` | Broker carried into premiumResponse |
| `_lifeScopedValidatorRows` | `carryLifePreEnrichmentMeta` | All variants for the insurer in this run |
| `_preValidatedRiderInfoList` | `carryLifePreEnrichmentMeta` | Pre-validated rider list (removed after enrichment) |
| `_preValidatedOfferInfoList` | `carryLifePreEnrichmentMeta` | Pre-validated offer list (removed after enrichment) |
| `_preValidatedInBuiltRiderCodes` | `fetchRiderOfferMetaForPreEnrichment` | Inbuilt rider codes from PDP |
| `_lifeVariantRiderEnrichment` | `enrichQuoteResponse` | Variant-wise rider enrichment flag |

---

## Flowchart

```
POST /api/minterprise/v2/products/life/quotes
               │
               ▼
   SachetController.addQuotation()
   [Extract QuotationRequest from PayloadWrapper.data]
   [Set tenant, broker, partnerId, provider, productCode="life"]
               │
               ▼
   IQuotationServiceImpl.generateQuote()
   [SachetLifeAggregator.supportsCustomQuoteFlow() = true → custom flow]
               │
               ▼
   validatePremium()
   ┌──────────────────────────────────────────────────────────────────┐
   │  normalizeQuoteRequestForStorage()                               │
   │  ├─ Flatten premiumRequest.planDetails derived fields             │
   │  ├─ Move insurerSpecificInfo → otherDetails                      │
   │  └─ Stash original quoteId → otherDetails._lifeRequestedQuoteId  │
   │                                                                  │
   │  referenceId missing?  → generate new referenceId (UUID)         │
   │  quoteId missing / fresh?  → generate new quoteId (UUID)         │
   │  quoteId present (requote)? → keep existing quoteId              │
   │                                                                  │
   │  non-initial? → fetch stored request from sachetPremiumRequest   │
   │    └─ prepareQuoteRequestForValidation()                         │
   │       ├─ copy _id, createdAt from stored                         │
   │       ├─ mergeStoredLifePremiumRequest()                         │
   │       ├─ preserveStoredLifeInsurerSpecificInfo()                 │
   │       └─ prepareLifeRequoteMetadata()                            │
   │                                                                  │
   │  saveAPIPremiumRequest() → sachetPremiumRequest (upsert)         │
   │  savePremiumRequestHistory() → sachetPremiumRequestHistory [async]│
   └──────────────────────────────────────────────────────────────────┘
               │
               ▼
   SachetLifeAggregator.generateQuote()
               │
               ▼
   buildLifeConfiguredProviders()
   [Extract requested insurers from planDetails.insurers]
               │
               ▼
   LifeRequestValidationService.resolveValidatedProviders()
   ┌────────────────────────────────────────────────────────────────────┐
   │  Extract planDetails from premiumRequest                           │
   │  normalizeLifeRequestForValidation()                               │
   │  ├─ entryAge from riskInsured.insuredMembers[0] or proposer        │
   │  ├─ derive missing policyTerm, premiumPaymentTerm, maturityAge     │
   │  └─ investmentTermCode (short/medium/long)                         │
   │                                                                    │
   │  ┌─── Applicable Products API ───────────────────────────────┐     │
   │  │ POST {IH_BASE}/api/product-management/v1/products/details/filters │
   │  │ Payload: {broker, businessCategory:"LIFE",                │     │
   │  │           businessSubCategory:"LIFE", productStatus:"LIVE",│     │
   │  │           inclusions:["PDP","INTEGRATION_INFO"]}           │     │
   │  │ [Redis cached 4 hours per broker]                         │     │
   │  │                                                           │     │
   │  │ Response: list of products with PDP fields + variants     │     │
   │  │   normalizeProductDetailRows() → one row per variant      │     │
   │  └───────────────────────────────────────────────────────────┘     │
   │                                                                    │
   │  Filter rows by: planType, policyType, category,                   │
   │                  paymentFrequency, currency,                        │
   │                  selectedPlans, insurerCodes                        │
   │                                                                    │
   │  applyLifePostFiltersWithDefaults()                                 │
   │  ├─ applyPayoutDefaults() (default payoutType for non-TERM)        │
   │  ├─ filterByPayoutSettings() (payoutType, payoutTerm, payoutFreq)  │
   │  └─ filterByPlanFeatures() (jointLife, returnOfPremium)            │
   │                                                                    │
   │  createLifeValidatorRows() → build structured rows from survivors  │
   │                                                                    │
   │  ┌─── Nearest Match API ─────────────────────────────────────┐     │
   │  │ POST {IH_BASE}/api/product-management/v1/life-validator    │     │
   │  │ Payload: {productCode: [...internalProductCodes],          │     │
   │  │           scope: {entryAge, policyTerm, premiumPaymentTerm,│     │
   │  │                   sumAssured, categories, ...}}            │     │
   │  │                                                           │     │
   │  │ Response: per product → lifeValidatorOptions (with score) │     │
   │  │           + productVariants → per variant options          │     │
   │  │                                                           │     │
   │  │ selectNearestMatch():                                     │     │
   │  │   score=0 → exact match, use user's term                  │     │
   │  │   score>0 → sort by score asc, pick nearest terms         │     │
   │  │   unmatched → drop row                                    │     │
   │  │                                                           │     │
   │  │ Updates row: policyTerm, premiumPaymentTerm, score        │     │
   │  └───────────────────────────────────────────────────────────┘     │
   │                                                                    │
   │  groupRowsByProvider() → {insurerCode → [rows]}                    │
   │  filterProviders() → filtered configured providers                 │
   │  createValidationMap() → {productCode → {valid, key, policyTerm,...}}│
   │                                                                    │
   │  Returns: LifeValidationResult {                                    │
   │    providers, validationMap, rowsByProvider                         │
   │  }                                                                  │
   └────────────────────────────────────────────────────────────────────┘
               │
               ▼
   filterValidationResultForRequote() [if _lifeRequoteSelection present]
   [Filter rowsByProvider to only requested insurer+productCode]
               │
               ▼
   buildLifeServiceRequests()
   [One QuotationRequest per validation row, per insurer]
   [Sets: provider, insurerCode, planCode, category, riskInsured.insurerCode]
   [OtherDetails: lifeRequestValidatorRows, lifeScopedValidatorRows,]
   [             lifeValidationMap, lifePendingKeyList, lifeValidationRowsByProvider, uniqueId]
               │
               ▼
   For each service request: attachLifePreEnrichmentMeta() [SYNC]
   ┌────────────────────────────────────────────────────────────────────┐
   │  fetchRiderOfferMetaForPreEnrichment(insurerCode, validationContext)│
   │  ├─ Query lifeRiderMaster (by riderCode + insurerCode + currency)   │
   │  ├─ Rule Engine: /api/rules/v0/life/riders/slab (slab ranges)      │
   │  ├─ Rule Engine: /api/rules/v0/life/riders/validate (valid riders) │
   │  └─ Query lifeOfferMaster (by offerCode + insurerCode)             │
   │  → otherDetails.riderMasterList, riderInfoList, offerMasterList,   │
   │    offerInfoList                                                    │
   └────────────────────────────────────────────────────────────────────┘
               │
               ▼
   attachLifeRunStateToRequest() + persistLifeRunState()
   [Update sachetPremiumRequest with new quoteId + validationMap + rowsByProvider]
               │
               ▼
        ┌──────────────────────────────────────────────────────────┐
        │              FLOW BRANCHING                              │
        ├─── FRESH FLOW ────────────────────────────────────────── │
        │  fetchCustomPremiumResponse()                            │
        │  ├─ saveLifeLeadForQuoteJourney() [async]                │
        │  ├─ getIHPremiumResponse() ──── FIRE AND FORGET ──────── │
        │  └─ Return PENDING response immediately to FE            │
        │                                                          │
        ├─── REQUOTE FLOW ──────────────────────────────────────── │
        │  persistLifeRunState()                                   │
        │  .then(getIHPremiumResponse()) ── SYNC WAIT ──────────── │
        │  .then(fetchLifeAggregateResponse()) ─ Read all DB rows  │
        │  .map(filterLifeRequoteResponse()) ─ Filter to requoted  │
        └──────────────────────────────────────────────────────────┘
               │ (async, in background for fresh flow)
               ▼
   getIHPremiumResponse() [PremiumServiceUtil]
   [Concurrent IH dispatch with ihMaxConcurrency]
   For each serviceRequest:
   │
   ├── LifePremiumDispatchService.fetchPreparedLifeQuotationResult()
   │   │
   │   ├─ [STEP 11] LifeQuoteIntegrationRequestBuilder.build()
   │   │  ├─ buildLifeRequestContext()
   │   │  │  [Merge: planDetails→personalDetails→proposerDetails→insuredMember→riskInsured→premiumRequest→requestMap]
   │   │  ├─ fetchInsurerConstants() → lifeInsurerProviderMeta collection
   │   │  ├─ fetchInsurerConfig() → brokerConfig collection
   │   │  ├─ buildIntegrationRequestMapper() [all IH fields]
   │   │  │  ├─ age, gender, DOB (dd/MM/yyyy or yyyy-MM-dd for ICICI)
   │   │  │  ├─ sumAssured (TERM) or premium (non-TERM)
   │   │  │  ├─ policyTerm, PPT from validator row (NOT raw request)
   │   │  │  ├─ premiumPaymentOption: Single/Regular/Limited Pay
   │   │  │  └─ isSmoking: Y/N from tobaccoStatus or smoker flag
   │   │  ├─ fetchMasterMapper() [TERM only: occupation + qualification codes]
   │   │  ├─ Insurer-specific enrichments:
   │   │  │  ├─ BAJAJLI: enrichBajajCoverDetailsRider() [cover codes map]
   │   │  │  ├─ ICICIPRULI: saMultiple (age/product based)
   │   │  │  ├─ MAXLIFELI: model factors, committedPremium
   │   │  │  ├─ TATAAIALI/DIGITLI: enrichPlanCodeFromRuleEngine()
   │   │  │  │  └─ POST /api/rules/v0/life/planCodes → planCode + baseSumAssured
   │   │  │  └─ HDFCLI: fetchPartnerDetails() → Mintpro partner API
   │   │  ├─ resolveInsurerSpecificInfo() → sets Requote=Yes/No
   │   │  ├─ buildUid() → MD5 hash of key fields
   │   │  └─ MappingQuery {integrationProvider, tmRequestType, vertical, broker, productCode}
   │   │
   │   ├─ ihService.sendIntegrationRequestToIH(payload, "LIFE")
   │   │
   │   └─ [STEP 12] buildQuoteResultFromIHResponse()
   │      ├─ LifeResponseNormalizerFactory.getNormalizer(insurerCode).normalize()
   │      │  [Per-insurer normalizer: unwrap key, map fields, compute annual premium]
   │      │  ├─ ICICIPRULI: unwrap "icicipruliResponse", map basePremium→premium
   │      │  ├─ BAJAJLI:    unwrap "bajajResponse", map fields
   │      │  ├─ HDFCLI:     unwrap "hdfcResponse", map fields
   │      │  └─ others:     GenericResponseNormalizer (unwrap "genericResponse")
   │      │  [All: setStatusFields, computeAnnualPremium, normalizeRiderList]
   │      ├─ enrichLifePremiumResponseWithValidatorRow()
   │      │  [Add: optionCode, policyTerm, premiumPaymentTerm, score, all validator metadata]
   │      └─ carryLifePreEnrichmentMeta()
   │         [Add: _preValidatedRiderInfoList, _preValidatedOfferInfoList, _lifeBroker]
   │
   └── [STEP 13] prepareLifePremiumResultForPersistence()
       └─ LifeQuoteMasterDataEnrichmentService.enrichQuoteResponse()
          [Mono.zip of 8 parallel enrichments]
          ├─ enrichCompanyDetails() → CompanyDetails collection
          ├─ enrichRiderList() → from _preValidatedRiderInfoList
          ├─ enrichOfferList() → from _preValidatedOfferInfoList
          ├─ enrichUlipFundAllocation() → lifeUlipFundAllocationMaster collection
          ├─ enrichResponseOptions() → split multi-option responses
          ├─ enrichErrorCategory() → LifeResponseAttributionRules collection
          ├─ enrichTaxSavingsInfo() → income-tax-savings-config.json
          └─ enrichResultCardsInfo() → computed
          [Also: applyBiTimeline(), applyRiderPremiumFields()]
          [Removes all _internal keys from final response]
          │
          └── defaultAggregatorService.savePremiumResult()
              [Save to sachetPremiumResponse]
              │
              └── [STEP 14] triggerAsyncLifeRiderPrices() [if status=success, non-single-result]
                  ├─ Rule Engine: /api/rules/v0/life/riders/interdependency
                  │  [Get compatible rider combination groups]
                  ├─ For each rider group:
                  │  └─ lifePremiumDispatchService.fetchPreparedLifeQuotationResult()
                  │     [IH call with specific rider combination → Steps 11+12+13]
                  └─ Save to sachetLifeRiderPrices (upsert)

               │
               ▼ (FE polls after receiving pending response)

   [STEP 16] GET /quotes?referenceId=...&includeRequest=true
         OR  GET /quotes/poll?referenceId=...&uniqueId=...&keyList=...
               │
               ▼
   fetchLifeAggregateResponse()
   ├─ Find stored request from sachetPremiumRequest by referenceId
   │  (or build fallback if not found)
   ├─ fetchLifeQuoteRowsForRun():
   │  MongoDB query: referenceId + productCode="life" + uniqueId + [provider] + [keyList]
   │  Sort: updatedAt DESC
   └─ buildQuoteResponse()
      ├─ resolveCurrentLifeValidationResult()
      │  [From stored otherDetails (fast) or re-run validation (slow)]
      ├─ filterLifeQuoteRowsByRequestedInsurers() [if insurers in request]
      ├─ filterLifeQuoteRowsByCurrentRunId() [row.uniqueId == quoteId]
      ├─ filterLifeQuoteRowsByCurrentValidation() [insurerCode|productCode in validationMap]
      ├─ flattenLifeQuoteRowsForResponse()
      │  └─ compactLifeQuoteRowsForResponse(): group by {referenceId|provider|planCode}
      │     [Multiple rows for same group → merge variants, keep latest primary]
      ├─ computePendingKeyList() [start from lifePendingKeyList, remove completed, add back pending]
      ├─ resolveAggregateStatus()
      │  [pending if pendingKeyList non-empty; success if any row=success; else failed]
      ├─ buildLifeApiQuotes()
      │  [for each row: normalizeLifeQuoteMap → enrichQuoteResponse (8-way Mono.zip)]
      ├─ applyCurrentLifeValidationToQuotes() [merge validator metadata into quotes]
      └─ reorderLifeDefaultQuoteRoot() [defaultOption=true variant first]
               │
               ▼
   Return QuotationResponse to FE
   {
     productCode, referenceId, quoteId, uniqueId,
     status: "success"|"pending"|"failed",
     pendingKeyList: [...remaining products],
     validationMap: {...},
     quotes: [{
       productCode, insurerCode, optionCode, policyTerm, premiumPaymentTerm,
       premium, tax, premiumWithTax, sumAssured, deathBenefitGuaranteed,
       status, companyDetails, riderList, offerList, taxSavingsInfo,
       resultCardsInfo, responseOptions, score, defaultOption
     }],
     premiumRequest: {...normalized request (internal keys stripped)...}
   }
```

---

*This document was generated from source code analysis of the `minterprise` codebase.*

*Key files:*
- *`SachetController.java` — API entry point (GET + POST endpoints)*
- *`IQuotationServiceImpl.java` — Request normalization, referenceId/quoteId assignment, DB save*
- *`SachetLifeAggregator.java` — Main orchestrator: validation → IH dispatch → response building*
- *`LifeRequestValidationService.java` — Applicable products, filters, nearest match, validationMap*
- *`LifeCategorySupport.java` — Category/planType normalization, nearest match scope fields*
- *`LifeQuoteIntegrationRequestBuilder.java` — IH request building, insurer-specific enrichments*
- *`LifeResponseNormalizerFactory.java` + normalizer files — Per-insurer IH response extraction*
- *`LifePremiumDispatchService.java` — IH call, validator row enrichment, pre-enrichment meta carry*
- *`LifeQuoteMasterDataEnrichmentService.java` — 8-way post-IH enrichment, pre-enrichment rider/offer fetch*
- *`LifeRiderPricesAsyncService.java` — Async rider price computation*
- *`PremiumServiceUtil.java` — Concurrent IH dispatch, rider price trigger*
