# DIGIT HCM — Platform Architecture

Platform internals: how data moves from the offline mobile app to DB and ES, why
they diverge, and the exact code locations that matter. Point-in-time; verify
file:line against current code.

## Offline sync flow (CDD mobile app -> server)

```
CDD Mobile App (offline) -> syncs data -> Server API
                                            |
                        putInCache(records) | producer.push(TOPIC A)
                                            |
                                    return 202 immediately  (no DB confirmation)
```

## Kafka two-topic pattern

```
TOPIC A (bulk input):  save-project-task-bulk-topic
                       project-beneficiary-consumer-bulk-create-topic
                       consumed by ProjectTaskConsumer / ProjectBeneficiaryConsumer

TOPIC B (save topic):  save-project-task-topic
                       save-project-beneficiary-topic
                       pushed by the consumer AFTER validation
                       consumed by BOTH egov-persister (DB) AND egov-transformer (ES)
```

Records that fail validation never reach Topic B. Transformer reads Topic B
(save topic), NOT Topic A — see gotchas.

## Divergence point — GenericRepository.save() (health-services-common)

```java
public List<T> save(List<T> objects, String topic) {
    producer.push(tenantId, topic, objects);  // pushes to TOPIC B
    putInCache(objects);                       // cache updated immediately
    return objects;                            // returns SUCCESS — no DB confirmation
}
```

Topic B is consumed by egov-persister (DB write, may fail silently) and transformer
(ES write, always succeeds). They are INDEPENDENT consumer groups. Persister failure
does not affect transformer => the ES record exists with no DB row = GHOST RECORD.
save() is fire-and-forget: no confirmation the persister wrote to DB or the transformer
indexed to ES; the cache is updated regardless of downstream success.

## Redis cache structure

```
Redis Key  : "project_beneficiary" (or "ba-project_beneficiary" with schema)
Redis Type : HASH
Field      : clientReferenceId -> single serialized entity (Jackson type array)
TTL        : 60 seconds (spring.cache.redis.time-to-live:60, GenericRepository.java:62)
```

TTL is set on the ENTIRE hash, not per field. `redisTemplate.expire(cacheTableName, 60,
SECONDS)`. When any new record arrives, `putAll()` + `expire()` resets the TTL for ALL
records in that hash. During an active sync session the bypass window stays open
indefinitely — it never expires until Kafka consumer lag exceeds 60s (high load, pod
restart, rebalance).

## Why ghost records happen — 4 root causes

| Cause | Mechanism | Result |
|---|---|---|
| **A — Silent persister failure** | JDBC batchUpdate runs ALL records in a Kafka message as one transaction (regardless of bulk.enabled). One bad record rolls back the entire batch; all N dropped. Error published to egov-tracer but KafkaConsumerErrorHandler (DefaultErrorHandler since Feb 2024) not configured to route it, so it never surfaces in egov-tracer-error-details. | ES has record, DB does not |
| **B — Redis cache bypass** | Validator checks Redis before DB; parent record put in cache before its Kafka consumer commits to DB. Within the 60s window the child validator gets a cache HIT and passes -> child written to ES; parent DB write later fails. | child created against a parent not in DB |
| **C — Race condition** | Two sync calls arrive within the same Redis TTL window; child consumer runs before parent committed. NON_EXISTENT_RELATED_ENTITY. Blocked at service layer (does NOT itself create a ghost — it produces a tracer entry). | tracer error, retried |
| **D — idgen outage** | egov-idgen down (Connection refused) -> clientReferenceId assignment fails. | write fails; small volume |

Root-cause split observed on the 90,143-ghost Nigeria investigation:
A (UPLOAD_ERROR_FROM_APP retry indicator) 69.8%, B (silent persister failure) 29.4%,
C (race) 0.5%, D (idgen Sep 21) 0.04%. See the `dst` skill's es-vs-db-rca reference for
the full RCA. Note: UPLOAD_ERROR_FROM_APP is a mobile RETRY indicator (Flutter
`dumpError()`), not a server error class.

## Mobile app client-side mechanism (UPLOAD_ERROR_FROM_APP, confirmed via live-device QA)

The mobile app (`health-campaign-field-worker-app` Flutter codebase) has its own
retry/error-report state machine BEFORE a request ever leaves the device:

```
Sync-up attempt fails (repo layer)
  -> updateSyncDownRetry() increments syncDownRetryCount
     -- ONLY fires when the server RESPONDS successfully but THIS record's
        serverGeneratedId comes back null. A pure transport-level failure
        (no network at all) never reaches this increment.
  -> syncDownRetryCount >= SYNC_DOWN_RETRY_COUNT (env, default 3)
     -> nonRecoverableError = true
  -> next sync pass: DataRepository.dumpError() -> POST error-handler/handle-error
     -> errorDetail.errors[].errorMessage = "UPLOAD_ERROR_FROM_APP"
     -> lands in egov-tracer-error-details
```

`sync_service.dart` wraps the ENTIRE sync-up call (including the `dumpError()` call
itself) in one broad try/catch that just stores the exception and calls
`service?.stopSelf()` — no external logging unless MDMS `FIREBASE_CONFIG.
enableCrashlytics` is on for that tenant. So a transport failure during the
`dumpError()` call itself is just as silent as the original create failure.

**Confirmed false-positive defect (Chad, 2026-08-05):** the app can report
`UPLOAD_ERROR_FROM_APP` for a record that had ALREADY synced successfully server-side,
minutes earlier. Two independent live-device cases confirmed by matching exact
`clientReferenceId`s against both ES (actual persist timestamp) and a fetched tracer
document (actual `UPLOAD_ERROR_FROM_APP` doc). Working theory: the create request
succeeds and the server responds, but the connection drops while the app is reading
that response (signatures: `Connection reset by peer`, `Software caused connection
abort`, `HttpException: Write failed` — distinct from `Failed host lookup`, which means
no network at all). The app never registers the success and eventually reports the
record as permanently failed anyway. Full detail + exact IDs/timestamps: the `hcm-qa`
skill's `chad-upload-error-from-app.md` reference.

**Implication:** `UPLOAD_ERROR_FROM_APP` counts in the tracer are not a clean proxy for
genuine data loss — they can include both false positives (reported failed but actually
synced) and undercounts (a sustained total outage never reaches the report-attempt
stage at all, since the retry-increment condition needs a server response).

## Redis bypass — the exact code chain

```
ProjectApiController.java:115
    putInCache(beneficiaries)              -> Redis (TTL 60s)
    producer.push(TOPIC_A, request)        -> Kafka
    return 202

PtProjectBeneficiaryIdValidator.java:70
    projectBeneficiaryRepository.validateIds()
        -> GenericRepository.findById()
            -> findInCache(tenantId, ids)   [REDIS CHECKED FIRST]
                if HIT: return without DB query   [BYPASS]
                if MISS: SELECT * FROM project_beneficiary
```

The fix is one line: call `validateClientReferenceIdsFromDB()` (GenericRepository.java:392,
direct SQL, already exists) instead of `validateIds()`. See the `hcm-dev` skill.

## Critical code locations (file:line — verify before editing)

| File | Line | What |
|---|---|---|
| ProjectApiController.java | 115 | putInCache(beneficiaries) before Kafka push |
| ProjectApiController.java | 116, 394 | producer.push to TOPIC A |
| ProjectTaskService.java | 137 | repository.save(validTasks, TOPIC B) |
| GenericRepository.java | 186-193 | save() — fire-and-forget, no DB confirmation |
| GenericRepository.java | 147-163 | findById() -> findInCache() before DB query |
| GenericRepository.java | 392 | validateClientReferenceIdsFromDB() — direct SQL, no cache |
| PtProjectBeneficiaryIdValidator.java | 70 | validateIds() — checks Redis before DB (the bypass) |
| PersisterConsumerConfig.java | 68 | enable.auto.commit=true HARDCODED (platform constraint) |
| PersistRepository.java | 39-46 | if(!rows.isEmpty()) silent skip + log.error() only |
| DeadLetterQueueConfiguration.java | 1-39 | DLQ config fully implemented but COMMENTED OUT (/* */) |

## Config file locations

- `D:\DST\scripts\codebase\health-campaign-services\health-services\project\src\main\resources\application.properties`
- `D:\services\Digit-Core\core-services\egov-persister\src\main\java\org\egov\infra\persist\consumer\PersisterConsumerConfig.java`
- `D:\DST\scripts\health-campaign-config\egov-indexer\project-task-indexer.yml`
- `D:\DST\scripts\codebase\health-campaign-services\health-services\transformer\src\main\resources\application.properties`

## Persistence note (unified-dev, 2026-07)

On the unified-dev build, egov-persister was in CrashLoopBackOff yet HCM records still
persisted — the health services (household/health-individual/health-project/
referralmanagement) persist via their own path, not egov-persister, on that build.
Confirm the actual persistence mechanism per environment before reasoning about it.
