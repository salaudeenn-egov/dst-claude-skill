# Ghost-Record Fixes — Validators, Persister, DLQ, Architecture

Code-level and architectural fixes for the DIGIT HCM ghost-record problem (ES records whose parent never committed to DB). Source: hcm_fix_recommendations, redis_bypass_proof.

## Hard constraint
- `enable.auto.commit=true` CANNOT be changed — platform-wide DIGIT constraint. Every fix below works within that.

## Short-term code fixes (ship this week)

### Fix 1 — Stop Redis bypass (1 line)
File: `PtProjectBeneficiaryIdValidator.java:70`

```java
// CHANGE:
projectBeneficiaryRepository.validateIds(tenantId, getIdList(entities, idMethod), columnName);
// TO:
projectBeneficiaryRepository.validateClientReferenceIdsFromDB(tenantId, getIdList(entities, idMethod), true);
```

`validateClientReferenceIdsFromDB` exists in `GenericRepository.java:392` — direct SQL, no cache.
Same fix needed in: PtProjectIdValidator, HmHouseholdValidator, HmIndividualValidator, PbIndividualValidator.

Why it matters (mechanism):
- `PtProjectBeneficiaryIdValidator` -> `validateIds()` -> `GenericRepository.findById()` -> `findInCache(tenantId, ids)` checks REDIS first; on HIT it returns without a DB query (BYPASS), on MISS it runs `SELECT * FROM project_beneficiary`.
- The beneficiary controller populates Redis (TTL 60s) BEFORE the Kafka consumer commits to DB, opening a 60s window where a task validates against a beneficiary that is not yet (or never) in DB.
- Live proof (2026-06-02, localhost:8080): after clearing Redis, calling the beneficiary API fired putInCache (Redis=YES, DB=0); the task API 0.3s later logged `o.e.c.d.r.GenericRepository: Cache hit, 1 items found` and returned HTTP 202; beneficiary in DB after = 0 (ghost condition).
- Production Record 10: beneficiary `e16df990` put in Redis 08:50:46, task validator ran 08:50:48 (within 60s TTL), API TotalCount 0 for beneficiary / 1 for individual (parent in DB). 6/10 sampled records confirmed Redis bypass by timing (e.g. aa9a9f80 19s gap, ff0cd480 30s, 2da51750 15s all YES; d20b0370 13min gap = NO, transformer read Topic A instead).
- TTL reset gotcha: TTL is on the entire Redis hash, not per field. A later batch's `putAll()` + `expire()` resets TTL on the whole hash, so during an active sync session the bypass window stays open indefinitely.
- Industry framing: "Write-Before-Durability Hazard" / cache poisoning in a validation context. Uber/Netflix/CommCare populate cache only after DB write confirmation.

### Fix 2 — Enable retry for race condition (1 line)
File: `ValidatorUtils.java` — method `getErrorForNonExistentRelatedEntity()`

```java
.type(Error.ErrorType.RECOVERABLE)  // was NON_RECOVERABLE
```

### Fix 3 — Add retry BackOff (3 lines)
File: `KafkaConsumerErrorHandler.java`

```java
public KafkaConsumerErrorHandler() {
    super(new FixedBackOff(10000L, 3L)); // 3 retries, 10s apart
}
```

### Fix 4 — Enable DLQ (remove comment)
File: `DeadLetterQueueConfiguration.java`
Remove `/*` at line 1 and `*/` at line 39 — the DLQ is already fully implemented, just commented out.

### Fix 5 — Fix KafkaConsumerErrorHandler to correctly route persister errors
File: `KafkaConsumerErrorHandler.java`
The persister DOES publish errors to egov-tracer. The issue: KafkaConsumerErrorHandler (changed to DefaultErrorHandler in the Feb 2024 commit 877d7daa01) is not configured to catch and route these errors to egov-tracer-error-details. Non-retryable exceptions were never configured, so permanent errors (e.g. duplicate key) retry 10 times and never reach the dead-letter queue cleanly.
Fix: configure non-retryable exception types and correct the error routing so persister failures surface in egov-tracer-error-details.

### Fix 6 — Log empty-row skip (1 line)
File: `PersistRepository.java:41`

```java
} else { log.warn("No rows extracted for query: {} — possible JSON path mismatch", query); }
```

## Medium-term fix (1-4 weeks)

### Remove controller-level putInCache
File: `ProjectApiController.java:115`

```java
// REMOVE:
projectBeneficiaryService.putInCache(beneficiaryRequest.getProjectBeneficiaries());
```

Cache should be populated only AFTER DB write is confirmed, not before. This is the companion to Fix 1.

## Architectural fixes (long-term)

### Option A — uForwarder (Uber open source)
- Sits between Kafka and consumers; commits offset ONLY after DB write confirmed.
- Eliminates both silent persister failure AND enables retry for the race condition.
- Requires converting @KafkaListener to gRPC endpoint; 15+ consumer classes across all HCM services to convert.
- Effort: 2-3 months.

### Option B — Outbox pattern
- Write to DB + outbox table in ONE transaction; CDC publishes to Kafka only AFTER DB commit.
- Transformer only ever sees confirmed data -> ghost records structurally impossible.
- Effort: 1-2 months.

### Option C — Wave-based mobile sync (DHIS2 pattern)
- Mobile sends in 5 waves with server confirmation between waves: 1. individuals -> confirm -> 2. households -> confirm -> 3. members -> confirm -> 4. beneficiaries -> confirm -> 5. tasks.
- Eliminates the race condition at source; no server change needed beyond a confirmation endpoint.
- Requires mobile app change + 1 confirmation endpoint on server.
- Effort: 2-3 weeks (mobile) + 1 week (server endpoint).

### Option D — Remove parent validation on bulk async (CommCare pattern)
- Mobile app data is trusted (self-consistent offline batch).
- Skip `PtProjectBeneficiaryIdValidator` from `isApplicableForCreate` for bulk=true.
- Run an hourly SQL job to flag orphaned records.
- Simplest server-side change, no retry complexity.
- Effort: 1 day.

## Fixes already implemented in Salama production
Branch: `2.9-LTS-persister-changes`

| # | Fix | Files |
|---|---|---|
| S1 | Transaction failures caught and sent to error queue one by one | TransactionInterceptorAspect.java, PersisterMessageListener.java |
| S2 | Infinite retry loop broken — failed -> error queue, still-failing -> dead-letter | PersisterConsumerConfig.java, PersisterMessageListener.java |
| S3 | batchUpdate failure -> retry records one by one in non-bulk mode | PersisterBatchListner.java, PersistService.java |
| S4 | Real batch write: records sharing same query combined into one batch | PersistService.java |
| S5 | Bulk/non-bulk selected per existing topic via config (persister.batch.topics) — no new topic needed | PersisterConsumerConfig.java, PersisterBatchConsumerConfig.java |
| T1 | Tracer: error queue still needs explicit per-service handling — not automatic | KafkaConsumerErrorHandler.java |
| T2 | Tracer: Kafka logs still carry no tenant ID and often no correlation ID | KafkaTemplateLoggingInterceptors.java |

Notes: S3 is the direct fix for the batchUpdate root cause (GOTCHA 14). S2 is the fix for the infinite retry loop (Section 3.3 of RCA).

## Priority order

```
Day 1:   Fix 1 (validator DB query) + Fix 4 (uncomment DLQ)
Week 1:  Fix 2 + Fix 3 (retry for race condition)
Week 2:  Fix 5 + Fix 6 (visibility)
Month 1: Option C (wave-based mobile sync) — structural fix
Month 3: Option A or B (uForwarder or Outbox) — permanent fix
```

## What each fix addresses

| Fix | Stops Redis bypass | Stops race condition | Stops silent failure | Makes failures visible |
|---|---|---|---|---|
| Fix 1 validateClientReferenceIdsFromDB | YES | NO | NO | NO |
| Fix 2 RECOVERABLE error | NO | Enables retry | NO | NO |
| Fix 3 BackOff | NO | YES (with Fix 2) | NO | NO |
| Fix 4 DLQ | NO | Captures failures | Captures failures | YES |
| Fix 5 persister error publish | NO | NO | Partial | YES |
| uForwarder | YES (indirectly) | YES | YES | YES |
| Wave sync | YES (eliminates it) | YES (eliminates it) | NO | NO |
