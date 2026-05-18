# Life APIs — Multitenancy Changes Required

## Background

When a detached `.subscribe()` is called (i.e., a subscription that fires and-forget, not part of the main reactive chain), it creates a new subscription boundary. The Micrometer baggage carrying `TENANT_DB_BAGGAGE` (the broker/tenant routing key) is **not automatically propagated** across this boundary. As a result, `MultiTenantMongoFactory` cannot resolve the correct broker-specific MongoDB and falls back to the default tenant.

The fix is `baggageUtil.subscribeWithBaggage(operationName, context, publisher)`, which:
1. Captures the current broker from baggage **before** the detach (`captureCurrentBroker()`)
2. Wraps the publisher in `withBaggage(TENANT_DB_BAGGAGE, broker, publisher)` — opens a new baggage scope via `Mono.using()` / `Flux.using()`
3. Calls `.contextCapture()` so the Reactor context propagates across thread hops
4. Calls `.subscribe()` with logging callbacks

**Reference implementation:** `SachetHealthAggregator.java` (lines 381, 391, 806, 4627, 4924, 5056, 5065)

---

## API → File Mapping

| Life API | Primary Classes |
|---|---|
| POST /quotes | `SachetLifeAggregator.fetchCustomPremiumResponse()` → `PremiumServiceUtil.getIHPremiumResponse()` → `IQuotationServiceImpl.saveAPIPremiumRequest()` |
| POST /singleresult | `SachetLifeAggregator.triggerLifeSingleResultQuote()` → `PremiumServiceUtil.getIHPremiumResponse()` |
| POST /proposal | `SachetLifeAggregator.createProposal()` → `ProposalServiceV2Impl.saveProposalRequest()` |
| POST /proposal (reject payment) | `ProposalServiceV2Impl.rejectProposal()` |
| POST /payment DD callback | `SachetLifeAggregator.postProcessingDd()` |
| POST /issuance / policy save | `SachetLifeAggregator.saveLifeIssuanceResult()` + `SachetLifeAggregator.updateLifeProposal()` |
| KYC callback (shared) | `DefaultAggregatorServiceImpl.handleKycCallback()` + `handleCkycType()` |
| Issuance request save (shared) | `DefaultAggregatorServiceImpl.saveIssuanceRequest()` |
| POST /quotes (request history) | `IQuotationServiceImpl.saveAPIPremiumRequest()` |

---

## Changes Required — File by File

---

### 1. `SachetLifeAggregator.java`
**Path:** `transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java`

`BaggageUtil baggageUtil` is already `@Autowired` at line 271. No new injection needed.

---

#### Change 1 — Line 2491 | API: POST /proposal

**Context:** Inside `createProposal()` — fires after proposal is created to update lead stage.

**Current:**
```java
.doOnNext(result -> {
    if (!(result instanceof ErrorResponseData)) {
        saveLifeLeadForProposalJourney(lifeProposalRequest).subscribe();
    }
})
```

**Replace with:**
```java
.doOnNext(result -> {
    if (!(result instanceof ErrorResponseData)) {
        baggageUtil.subscribeWithBaggage(
            "LifeProposal.LeadJourney",
            "referenceId=" + lifeProposalRequest.getReferenceId(),
            saveLifeLeadForProposalJourney(lifeProposalRequest));
    }
})
```

---

#### Change 2 — Lines 2592–2596 | API: POST /quotes

**Context:** Inside `fetchCustomPremiumResponse()` — fires before the async IH call to record QUOTE lead stage.

**Current:**
```java
saveLifeLeadForQuoteJourney(
        quotationRequest,
        LeadConstants.GROUP_PRODUCT_LEAD_STAGES.QUOTE,
        null)
        .subscribe();
```

**Replace with:**
```java
baggageUtil.subscribeWithBaggage(
    "LifeQuotes.LeadJourney.pre",
    "referenceId=" + (quotationRequest == null ? null : quotationRequest.getReferenceId()),
    saveLifeLeadForQuoteJourney(quotationRequest, LeadConstants.GROUP_PRODUCT_LEAD_STAGES.QUOTE, null));
```

---

#### Change 3 — Lines 2598–2614 | API: POST /quotes

**Context:** Inside `fetchCustomPremiumResponse()` — the main async IH call that saves `QuotationResult` to MongoDB, then fires another nested subscribe to update the lead.

**Current:**
```java
premiumServiceUtil.getIHPremiumResponse(serviceRequests, quotationRequest)
        .subscribeOn(Schedulers.boundedElastic())
        .subscribe(
                response -> {
                    log.info("[SachetLifeAggregator] Async life quote dispatch completed referenceId={} uniqueId={}",
                            quotationRequest == null ? null : quotationRequest.getReferenceId(),
                            quotationRequest == null ? null : quotationRequest.getQuoteId());
                    saveLifeLeadForQuoteJourney(
                            quotationRequest,
                            LeadConstants.GROUP_PRODUCT_LEAD_STAGES.QUOTE,
                            extractLifeQuotationResults(response))
                            .subscribe();  // ← nested raw subscribe at line 2609
                },
                error -> log.warn("[SachetLifeAggregator] Async life quote dispatch failed referenceId={} uniqueId={} cause={}",
                        quotationRequest == null ? null : quotationRequest.getReferenceId(),
                        quotationRequest == null ? null : quotationRequest.getQuoteId(),
                        error.getMessage()));
```

**Replace with:**
```java
baggageUtil.subscribeWithBaggage(
    "LifeQuotes.IHDispatch",
    "referenceId=" + (quotationRequest == null ? null : quotationRequest.getReferenceId()),
    premiumServiceUtil.getIHPremiumResponse(serviceRequests, quotationRequest)
            .subscribeOn(Schedulers.boundedElastic())
            .flatMap(response -> {
                log.info("[SachetLifeAggregator] Async life quote dispatch completed referenceId={} uniqueId={}",
                        quotationRequest == null ? null : quotationRequest.getReferenceId(),
                        quotationRequest == null ? null : quotationRequest.getQuoteId());
                return saveLifeLeadForQuoteJourney(
                        quotationRequest,
                        LeadConstants.GROUP_PRODUCT_LEAD_STAGES.QUOTE,
                        extractLifeQuotationResults(response)).then(Mono.just(response));
            }),
    ignored -> {},
    error -> log.warn("[SachetLifeAggregator] Async life quote dispatch failed referenceId={} uniqueId={} cause={}",
            quotationRequest == null ? null : quotationRequest.getReferenceId(),
            quotationRequest == null ? null : quotationRequest.getQuoteId(),
            error.getMessage()));
```

> **Note on nested `.subscribe()` at line 2609:** By converting the outer `.subscribe()` to `subscribeWithBaggage`, the inner `saveLifeLeadForQuoteJourney` is pulled into the same reactive chain via `.flatMap()`. This eliminates the nested subscribe entirely and keeps baggage propagation in one place.

---

#### Change 4 — Line 10215 | API: Issuance (life policy save)

**Context:** Inside `saveLifeIssuanceResult()` — saves a history record after updating the main issuance result.

**Current:**
```java
.map(savedResult -> {
    savedResult.set_id(null);
    issuanceResultHistoryRepository.save(
            JsonUtils.fromJsonObject(savedResult, IssuanceResultHistory.class)).subscribe();
    return savedResult;
});
```

**Replace with:**
```java
.map(savedResult -> {
    savedResult.set_id(null);
    baggageUtil.subscribeWithBaggage(
        "LifeIssuance.ResultHistory",
        "referenceId=" + savedResult.getReferenceId(),
        issuanceResultHistoryRepository.save(
                JsonUtils.fromJsonObject(savedResult, IssuanceResultHistory.class)));
    return savedResult;
});
```

---

#### Change 5 — Lines 10291–10294 | API: POST /payment DD callback

**Context:** Inside `postProcessingDd()` — fires async issuance after DD payment confirmation.

**Current:**
```java
initiateIssuance(request, responseMap, ddRequest)
        .subscribeOn(Schedulers.boundedElastic())
        .subscribe(unused -> {
        }, error -> log.error("[LifePostPayment] Async initiateIssuance failed", error));
```

**Replace with:**
```java
baggageUtil.subscribeWithBaggage(
    "LifePostPayment.initiateIssuance",
    "referenceId=" + request.getReferenceId(),
    initiateIssuance(request, responseMap, ddRequest)
            .subscribeOn(Schedulers.boundedElastic()),
    ignored -> {},
    error -> log.error("[LifePostPayment] Async initiateIssuance failed", error));
```

---

#### Change 6 — Line 10342 | API: Issuance (proposal update)

**Context:** Inside `updateLifeProposal()` — updates proposal result properties after payment/issuance.

**Current:**
```java
private void updateLifeProposal(String referenceId, Map<String, Object> updatePropertiesMap) {
    lifeProposalResultRepository.updateProperties(REFERENCEID, referenceId, updatePropertiesMap, false).subscribe();
}
```

**Replace with:**
```java
private void updateLifeProposal(String referenceId, Map<String, Object> updatePropertiesMap) {
    baggageUtil.subscribeWithBaggage(
        "LifeProposal.update",
        "referenceId=" + referenceId,
        lifeProposalResultRepository.updateProperties(REFERENCEID, referenceId, updatePropertiesMap, false));
}
```

---

### 2. `PremiumServiceUtil.java`
**Path:** `transactional-flows/src/main/java/com/sachetProduct/util/PremiumServiceUtil.java`

**Action required:** Add `@Autowired BaggageUtil baggageUtil;` field.

---

#### Change 7 — Line 975 | API: POST /quotes (aggregated result save)

**Context:** Inside `getIHPremiumResponseV2()` — saves aggregated quote result to MongoDB.

**Current:**
```java
.map(aggregatedQuotes -> {
    QuotationResponse quotationResponse = new QuotationResponse();
    quotationResponse.setAggregateQuoteId(uniqueIdGenerator.generateNextUUID());
    quotationResponse.setProductCode(quotationRequest.getProductCode());
    quotationResponse.setQuotes(aggregatedQuotes);
    defaultAggregatorService.saveAggregatorPremiumResult(quotationResponse).subscribe();
    return quotationResponse;
});
```

**Replace with:**
```java
.map(aggregatedQuotes -> {
    QuotationResponse quotationResponse = new QuotationResponse();
    quotationResponse.setAggregateQuoteId(uniqueIdGenerator.generateNextUUID());
    quotationResponse.setProductCode(quotationRequest.getProductCode());
    quotationResponse.setQuotes(aggregatedQuotes);
    baggageUtil.subscribeWithBaggage(
        "LifeQuotes.AggregatorResult",
        "referenceId=" + quotationRequest.getReferenceId(),
        defaultAggregatorService.saveAggregatorPremiumResult(quotationResponse));
    return quotationResponse;
});
```

---

#### Change 8 — Lines 1030–1039 | API: POST /quotes (async rider prices)

**Context:** Inside `triggerAsyncLifeRiderPrices()` — fires async rider price generation after successful life quote. Rider prices are saved to MongoDB. This is explicitly skipped for singleresult requests (line 1024 guard).

**Current:**
```java
lifeRiderPricesAsyncService.generateAndSaveRiderPrices(JsonUtils.fromJsonObject(serviceRequest, QuotationRequest.class))
        .subscribeOn(reactor.core.scheduler.Schedulers.parallel())
        .subscribe(
                ignore -> log.info("[LifeRiderPrices] Async generation completed for referenceId={} planCode={} provider={}",
                        serviceRequest.getReferenceId(), serviceRequest.getPlanCode(), serviceRequest.getProvider()),
                error -> log.warn("[LifeRiderPrices] Async generation failed for referenceId={} planCode={} provider={} cause={}",
                        serviceRequest.getReferenceId(),
                        serviceRequest.getPlanCode(),
                        serviceRequest.getProvider(),
                        error.getMessage()));
```

**Replace with:**
```java
baggageUtil.subscribeWithBaggage(
    "LifeQuotes.RiderPrices",
    "referenceId=" + serviceRequest.getReferenceId() + ", planCode=" + serviceRequest.getPlanCode(),
    lifeRiderPricesAsyncService.generateAndSaveRiderPrices(
                    JsonUtils.fromJsonObject(serviceRequest, QuotationRequest.class))
            .subscribeOn(reactor.core.scheduler.Schedulers.parallel()),
    ignore -> log.info("[LifeRiderPrices] Async generation completed for referenceId={} planCode={} provider={}",
            serviceRequest.getReferenceId(), serviceRequest.getPlanCode(), serviceRequest.getProvider()),
    error -> log.warn("[LifeRiderPrices] Async generation failed for referenceId={} planCode={} provider={} cause={}",
            serviceRequest.getReferenceId(),
            serviceRequest.getPlanCode(),
            serviceRequest.getProvider(),
            error.getMessage()));
```

---

### 3. `IQuotationServiceImpl.java`
**Path:** `transactional-flows/src/main/java/com/sachetProduct/service/impl/IQuotationServiceImpl.java`

**Action required:** Add `@Autowired BaggageUtil baggageUtil;` field.

---

#### Change 9 — Line 333 | API: All quote fetch calls (Redis cache write)

**Context:** Inside `fetchResultFromDB()` — pushes a `QuotationResult` fetched from MongoDB into Redis cache. This writes to **Redis**, not MongoDB. Redis key already contains broker-specific identifiers (`quoteId + insurerCode + referenceId`). Technically low risk, but still best practice to wrap since the chain is within a reactive context that may hop threads.

**Current:**
```java
.switchIfEmpty(quotationResultRepository.findOneByProperties(properties, "createdAt").
        map(val -> {
            log.info("[Premium Service] Cache miss for fetchResultFromDB() : " + quoteId + insurerCode + referenceId);
            try {
                log.info("[Premium Service] Cache stored for fetchResultFromDB() : " + quoteId + insurerCode + referenceId);
                redisUtil.pushDataToCache(quoteId + insurerCode + referenceId, val, null).subscribe();
            } catch (Exception ex) {
                ex.printStackTrace();
            }
            return val;
        }))
```

**Replace with:**
```java
.switchIfEmpty(quotationResultRepository.findOneByProperties(properties, "createdAt").
        map(val -> {
            log.info("[Premium Service] Cache miss for fetchResultFromDB() : " + quoteId + insurerCode + referenceId);
            log.info("[Premium Service] Cache stored for fetchResultFromDB() : " + quoteId + insurerCode + referenceId);
            baggageUtil.subscribeWithBaggage(
                "QuotationResult.cacheWrite",
                "key=" + quoteId + insurerCode + referenceId,
                redisUtil.pushDataToCache(quoteId + insurerCode + referenceId, val, null));
            return val;
        }))
```

> **Note:** Remove the `try/catch` wrapper — `subscribeWithBaggage` logs errors internally via the `onError` callback.

---

#### Change 10 — Line 496 | API: POST /quotes (request history save)

**Context:** Inside `saveAPIPremiumRequest()` — saves a history copy of the quote request to MongoDB after the primary save.

**Current:**
```java
return pr.log()
        .map(r -> {
            r.set_id(null);
            savePremiumRequestHistory(r).subscribe();
            return quoteFlowAggregator != null ? req : r;
        });
```

**Replace with:**
```java
return pr.log()
        .map(r -> {
            r.set_id(null);
            baggageUtil.subscribeWithBaggage(
                "LifeQuotes.RequestHistory",
                "referenceId=" + r.getReferenceId(),
                savePremiumRequestHistory(r));
            return quoteFlowAggregator != null ? req : r;
        });
```

---

### 4. `DefaultAggregatorServiceImpl.java`
**Path:** `transactional-flows/src/main/java/com/sachetProduct/service/impl/DefaultAggregatorServiceImpl.java`

**Action required:** Add `@Autowired BaggageUtil baggageUtil;` field.

Note: This is a shared service used by all products. All changes below apply to life flow and any other product that routes through the same code paths.

---

#### Change 11 — Line 400 | API: Issuance (save issuance request)

**Context:** Inside `saveIssuanceRequest()` — fired after issuance request creation.

**Current:**
```java
public void saveIssuanceRequest(IssuanceRequest request) {
    issuanceRequestRepository.save(request).subscribeOn(Schedulers.boundedElastic()).subscribe();
}
```

**Replace with:**
```java
public void saveIssuanceRequest(IssuanceRequest request) {
    baggageUtil.subscribeWithBaggage(
        "Issuance.saveRequest",
        "referenceId=" + request.getReferenceId(),
        issuanceRequestRepository.save(request).subscribeOn(Schedulers.boundedElastic()));
}
```

---

#### Change 12 — Lines 618–623 | API: KYC callback (OVD flow)

**Context:** Inside `handleKycCallback()` — fires async IH OVD call after processing KYC document.

**Current:**
```java
insurerKycDoc.flatMap(kycDocument -> {
    log.info("[handleKycCallback] updating kycDocument in kycUserDataResponse.");
    kycUserDataResponse.setDocuments(List.of(kycDocument));
    return iSachetProductAggregatorFactory.getPremiumAggregator(productCode)
            .callInsurerForOVD(proposalResult, kycUserDataResponse);
}).subscribe();
```

**Replace with:**
```java
baggageUtil.subscribeWithBaggage(
    "KYC.OVDCallback",
    "referenceId=" + proposalResult.getReferenceId(),
    insurerKycDoc.flatMap(kycDocument -> {
        log.info("[handleKycCallback] updating kycDocument in kycUserDataResponse.");
        kycUserDataResponse.setDocuments(List.of(kycDocument));
        return iSachetProductAggregatorFactory.getPremiumAggregator(productCode)
                .callInsurerForOVD(proposalResult, kycUserDataResponse);
    }));
```

---

#### Change 13 — Line 633 | API: KYC callback (CKYC lead save)

**Context:** Inside `handleKycCallback()` — saves KYC-completed lead stage for CKYC path.

**Current:**
```java
sachetProductLeadService.saveSachetLead(sachetLeadOrderInfo).subscribe();
```

**Replace with:**
```java
baggageUtil.subscribeWithBaggage(
    "KYC.LeadSave.ckyc",
    "referenceId=" + proposalResult.getReferenceId(),
    sachetProductLeadService.saveSachetLead(sachetLeadOrderInfo));
```

---

#### Change 14 — Line 894 | API: KYC callback (CKYC lead save — second path)

**Context:** Inside `handleCkycType()` — same pattern, different code path for CKYC type.

**Current:**
```java
sachetProductLeadService.saveSachetLead(sachetLeadOrderInfo).subscribe();
```

**Replace with:**
```java
baggageUtil.subscribeWithBaggage(
    "KYC.LeadSave.ckycType",
    "referenceId=" + referenceId,
    sachetProductLeadService.saveSachetLead(sachetLeadOrderInfo));
```

---

#### Change 15 — Lines 1205, 1215 | API: CSV bulk upload (file activity update)

**Context:** Inside `processBatchedChunks()` — updates `fileProcessingActivity` records during CSV bulk order processing. This is a **batch/bulk flow**, not a user-facing life API. Low priority for life multitenancy, but same pattern applies.

**Current (both occurrences):**
```java
fileProcessingActivityRepository
        .updateProperties("_id", fileProcessingActivity.get_id(),
                JsonUtils.fromJsonObject(fileProcessingActivity, Map.class), true)
        .subscribe();
```

**Replace with:**
```java
baggageUtil.subscribeWithBaggage(
    "CSV.FileActivityUpdate",
    "activityId=" + fileProcessingActivity.get_id(),
    fileProcessingActivityRepository.updateProperties("_id", fileProcessingActivity.get_id(),
            JsonUtils.fromJsonObject(fileProcessingActivity, Map.class), true));
```

---

#### Change 16 — Line 1278 | API: CSV bulk upload (lead save)

**Context:** Inside `processCompleteFlow()` — saves PAYMENT_INITIATED lead stage during CSV bulk payment step.

**Current:**
```java
sachetProductLeadService.saveSachetLead(sachetLeadOrderInfo).subscribe();
```

**Replace with:**
```java
baggageUtil.subscribeWithBaggage(
    "CSV.LeadSave.payment",
    "referenceId=" + referenceId,
    sachetProductLeadService.saveSachetLead(sachetLeadOrderInfo));
```

---

### 5. `ProposalServiceV2Impl.java`
**Path:** `transactional-flows/src/main/java/com/sachetProduct/service/impl/ProposalServiceV2Impl.java`

**Action required:** Add `@Autowired BaggageUtil baggageUtil;` field.

---

#### Change 17 — Line 141 | API: POST /proposal (request history save)

**Context:** Inside `saveProposalRequest()` — saves a history copy of the proposal request to MongoDB after the primary save.

**Current:**
```java
return defaultAggregatorService.saveProposalRequest(proposalRequest).map(proposalRequest1 -> {
    proposalRequestHistoryRepository.saveData(proposalRequest).subscribe();
    return proposalRequest1;
});
```

**Replace with:**
```java
return defaultAggregatorService.saveProposalRequest(proposalRequest).map(proposalRequest1 -> {
    baggageUtil.subscribeWithBaggage(
        "Proposal.RequestHistory",
        "referenceId=" + proposalRequest.getReferenceId(),
        proposalRequestHistoryRepository.saveData(proposalRequest));
    return proposalRequest1;
});
```

---

#### Change 18 — Line 273 | API: POST /proposal (consent audit on reject)

**Context:** Inside `rejectProposal()` — saves a consent audit record when payment is rejected by user.

**Current:**
```java
consentAuditRepository.save(consentAudit).subscribe();
```

**Replace with:**
```java
baggageUtil.subscribeWithBaggage(
    "Proposal.ConsentAudit.reject",
    "referenceId=" + proposalResult.getReferenceId(),
    consentAuditRepository.save(consentAudit));
```

---

#### Change 19 — Line 288 | API: POST /proposal (lead save on reject)

**Context:** Inside `rejectProposal()` — saves PAYMENT_REJECTED lead stage.

**Current:**
```java
sachetProductLeadService.saveSachetLead(sachetLeadOrderInfo).subscribe();
```

**Replace with:**
```java
baggageUtil.subscribeWithBaggage(
    "Proposal.LeadSave.rejected",
    "referenceId=" + savedProposalResult.getReferenceId(),
    sachetProductLeadService.saveSachetLead(sachetLeadOrderInfo));
```

---

### 6. `IssuanceServiceV2Impl.java` — Scheduled Task (Out of Scope for Baggage)
**Path:** `transactional-flows/src/main/java/com/sachetProduct/service/impl/IssuanceServiceV2Impl.java`

**Lines 189–191** — `updatePolicyStatusExpired()` is a scheduled task that calls `.subscribe()` on `updatePolicyStatus()` and `getAllPolicyByRiskEndDate()`.

This does **not** need `subscribeWithBaggage`. Scheduled tasks run without an incoming HTTP request, so there is no baggage context to capture. MongoDB routing for scheduled tasks requires a different approach (e.g., iterate over all tenants explicitly, or use a system tenant). This is **out of scope** for the standard baggage propagation pattern and requires separate design.

---

## Summary Table

| # | File | Line(s) | API / Context | MongoDB Collections Affected | Priority |
|---|---|---|---|---|---|
| 1 | `SachetLifeAggregator` | 2491 | POST /proposal — lead save after proposal | sachetLeadOrder | High |
| 2 | `SachetLifeAggregator` | 2592–2596 | POST /quotes — pre-IH lead save | sachetLeadOrder | High |
| 3 | `SachetLifeAggregator` | 2598–2614 | POST /quotes — async IH + nested lead save | sachetPremiumResponse, sachetLeadOrder | High |
| 4 | `SachetLifeAggregator` | 10215 | Issuance — history record save | issuanceResultHistory | High |
| 5 | `SachetLifeAggregator` | 10291–10294 | POST /payment DD callback — async issuance | lifeProposalResult, issuanceResult | High |
| 6 | `SachetLifeAggregator` | 10342 | Issuance — proposal property update | lifeProposalResult | High |
| 7 | `PremiumServiceUtil` | 975 | POST /quotes — aggregated result save | sachetPremiumResponse | High |
| 8 | `PremiumServiceUtil` | 1030–1039 | POST /quotes — async rider prices | lifeRiderPrices | High |
| 9 | `IQuotationServiceImpl` | 333 | All quotes — Redis cache write (low risk) | Redis only | Low |
| 10 | `IQuotationServiceImpl` | 496 | POST /quotes — request history save | sachetPremiumRequest (history) | High |
| 11 | `DefaultAggregatorServiceImpl` | 400 | Issuance — save issuance request | issuanceRequest | High |
| 12 | `DefaultAggregatorServiceImpl` | 618–623 | KYC callback — async OVD call | kycResult, (IH external) | High |
| 13 | `DefaultAggregatorServiceImpl` | 633 | KYC callback — CKYC lead save | sachetLeadOrder | High |
| 14 | `DefaultAggregatorServiceImpl` | 894 | KYC callback — CKYC type lead save | sachetLeadOrder | High |
| 15 | `DefaultAggregatorServiceImpl` | 1205, 1215 | CSV bulk — file activity update | fileProcessingActivity | Medium |
| 16 | `DefaultAggregatorServiceImpl` | 1278 | CSV bulk — lead save | sachetLeadOrder | Medium |
| 17 | `ProposalServiceV2Impl` | 141 | POST /proposal — request history save | proposalRequestHistory | High |
| 18 | `ProposalServiceV2Impl` | 273 | POST /proposal — consent audit on reject | consentAudit | High |
| 19 | `ProposalServiceV2Impl` | 288 | POST /proposal — lead save on reject | sachetLeadOrder | High |
| — | `IssuanceServiceV2Impl` | 189–191 | Scheduled task — policy expiry update | issuanceResult | Out of scope |

## Injection Changes Needed

| File | Change |
|---|---|
| `SachetLifeAggregator` | Already has `@Autowired BaggageUtil baggageUtil` — no change |
| `PremiumServiceUtil` | Add `@Autowired BaggageUtil baggageUtil;` |
| `IQuotationServiceImpl` | Add `@Autowired BaggageUtil baggageUtil;` |
| `DefaultAggregatorServiceImpl` | Add `@Autowired BaggageUtil baggageUtil;` |
| `ProposalServiceV2Impl` | Add `@Autowired BaggageUtil baggageUtil;` |
