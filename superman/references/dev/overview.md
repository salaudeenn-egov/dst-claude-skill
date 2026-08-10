---
name: hcm-dev
description: >-
  HCM platform CODE DEVELOPMENT work-stream for Salaudeen (eGov). Invoke when
  writing, reviewing, building, or shipping code FIXES to DIGIT HCM Java
  services: egov-transformer, egov-indexer, egov-persister, and the cross-entity
  validators. Covers the ghost-record fixes (Redis-bypass validator, DLQ,
  RECOVERABLE retry, persister error routing), the transformer error-handling
  fixes (HDDF-5226: UUID/deterministic error-record id, null-topic ThreadLocal
  fallback, indexer ymls, endpoint /health-* path fixes, JUnit flow harness),
  and the JDK8/maven build+env facts. Concrete triggers: "transformer fix",
  "indexer yml", "validator fix", "DLQ", "persister error routing", "UUID error
  id", "null-topic fix", "ghost record fix", class/file names like
  TransformerErrorProducer, ErrorMessage, IndexerUtils.buildIndexId,
  PtProjectBeneficiaryIdValidator, KafkaConsumerErrorHandler, and branch names
  like 2.9-LTS-persister-changes or HDDF-5226-uuid-error-record-doc-id.
---

# HCM Dev — Platform Code Fixes

Development work-stream for writing and shipping code fixes to the DIGIT HCM Java services (transformer, egov-indexer, egov-persister, cross-entity validators). This is the "how do I fix the code" companion to the data-recovery and QA work-streams.

> Living skill. Last synced 2026-07-08. Point-in-time: file:line/branch/build facts may be stale — verify against current code before editing. See "Keeping this skill current" at the bottom.

## When to use / not use

Use this skill when:
- Writing, reviewing, or building a code fix to egov-transformer, egov-indexer, egov-persister, or the HCM cross-entity validators.
- Working on the ghost-record fixes (validator Redis bypass, DLQ, RECOVERABLE retry, persister error routing) or the transformer error-handling fixes (HDDF-5226).
- Creating or editing egov-indexer ymls for transformer flows.
- Needing the JDK8/maven build commands, env/port-forward facts, or which transformer clone is the actually-run one.

Do NOT use this skill for:
- Data-recovery / reconciliation scripts (ES<->DB fixes, ghost-record cleanup) — that is the `dst` work-stream.
- Test execution or QA methodology (the 12 test cases, kafka-console-producer repro commands, pass/fail matrix) — that is the `hcm-qa` work-stream.
- Platform architecture background (entity chain, ES/DB divergence point, Kafka two-topic pattern) — see the `hcm-context` skill / `digit_hcm_architecture` memory.

## Reference files

| File | Read it when |
| --- | --- |
| references/ghost-record-fixes.md | Fixing the ghost-record root causes in validators / persister / DLQ: the Redis-bypass one-liner, RECOVERABLE + FixedBackOff retry, DLQ uncomment, persister error routing, remove controller putInCache; plus the architectural options (uForwarder / Outbox / wave sync / remove-validation) and the Salama-production fixes (branch 2.9-LTS-persister-changes, S1-S5/T1-T2). |
| references/transformer-fixes.md | Working on HDDF-5226 transformer error-handling: the UUID/deterministic error-record id fix, the null-topic ThreadLocal fallback fix, the indexer ymls for all transformer flows, the /health-* endpoint path fixes, the JUnit flow-test harness, and the JDK8/maven build + port-forward env facts. Also identifies which transformer clone is the actually-run one. |

## Problem -> Approach -> Solution playbook

THE most important section. Compact entries; exact identifiers preserved verbatim.

### 1. Redis-bypass validator fix (the ghost-record trigger)
**Problem:** Task validator `PtProjectBeneficiaryIdValidator` calls `validateIds()`, which checks Redis before DB. The beneficiary controller puts beneficiaries in Redis (TTL 60s) BEFORE the Kafka consumer commits them to DB. In that window the task validator gets a cache HIT, marks the task valid, transformer writes the task to ES — but the beneficiary DB write can fail. Result: ES task with a parent that never landed in DB (ghost record). Live proof 2026-06-02 on localhost:8080: log line `o.e.c.d.r.GenericRepository: Cache hit, 1 items found`, beneficiary in DB after = 0, task returned HTTP 202. Production Record 10: beneficiary `e16df990` in Redis 08:50:46, validator ran 08:50:48, DB TotalCount 0. 6/10 sampled records confirmed via timing.
**Approach:** Trace `validateIds()` -> `GenericRepository.findById()` -> `findInCache()` returning on cache hit without a DB query. `validateClientReferenceIdsFromDB` already exists (`GenericRepository.java:392`) as a direct-SQL, no-cache path.
**Fix:** `PtProjectBeneficiaryIdValidator.java:70` — change `projectBeneficiaryRepository.validateIds(tenantId, getIdList(entities, idMethod), columnName);` to `projectBeneficiaryRepository.validateClientReferenceIdsFromDB(tenantId, getIdList(entities, idMethod), true);`. Same fix needed in PtProjectIdValidator, HmHouseholdValidator, HmIndividualValidator, PbIndividualValidator. Companion: remove `projectBeneficiaryService.putInCache(...)` at `ProjectApiController.java:115` so cache is populated only AFTER DB write.
**Status:** Proposed + proven; not yet pushed. See references/ghost-record-fixes.md.

### 2. Transformer UUID / deterministic error-record id fix (TC-09 + TC-10)
**Problem:** egov-indexer `transformer-error-indexer.yml` used `id: $.topic` (later `id: $.topic,$.payload`) as the ES doc id. `IndexerUtils.buildIndexId` (`IndexerUtils.java:365`) casts each id JsonPath value to String; on the composite/keyless error record this either collapsed distinct failures onto one doc (TC-10 overwrite) or, when a Long field like `ErrorMessage.timestamp` was in the id, threw a ClassCastException -> "Indexing will not be done" -> every error record dropped (TC-09). In the actually-run codebase `ErrorMessage` had no `id` at all, so the indexer threw `com.jayway.jsonpath.PathNotFoundException: $['id']` and the ES error index stuck at 452.
**Approach:** Give the error record its own String id and index on it. Deterministic (hash of topic+payload) so Kafka redelivery overwrites instead of duplicating (satisfies TC-11).
**Fix:** three edits — (1) add `private String id;` to `ErrorMessage.java`; (2) in `TransformerErrorProducer.sendToErrorTopic` compute `payloadString` once and set `.id(buildDeterministicId(sourceTopic, payloadString))`, where `buildDeterministicId` = `UUID.nameUUIDFromBytes((topic+"|"+payload).getBytes(UTF_8))` (imports java.util.UUID, java.nio.charset.StandardCharsets); (3) set indexer yml `id: $.id`. An earlier variant used `.id(UUID.randomUUID().toString())` with `id: $.topic,$.payload` -> `id: $.id`.
**Status:** Implemented + validated. `mvn -o compile` BUILD SUCCESS. Validated locally: 2 distinct + 2 identical malformed payloads produced 4 distinct ES docs, each `_id = id = UUID` (old composite id would collapse identical to 1). PENDING: user must restart their IDE transformer to load it. See references/transformer-fixes.md.

### 3. Transformer null-topic fix (TC-06)
**Problem:** 25 service-layer sites call `sendToErrorTopic(x, null, e)` with no source topic; 2 of them (`BoundaryService.java:183`, `ProjectService.java:124`) ALSO rethrow, so the consumer emits a second record. Net effect: a duplicate error record plus a null-topic record that cannot be replayed.
**Approach:** Thread-scoped topic fallback that is behavior-preserving (no RECEIVED_TOPIC threading through every signature).
**Fix:** (1) `TransformerErrorProducer` adds `ThreadLocal<String> CURRENT_SOURCE_TOPIC` plus `setSourceTopic`/`clearSourceTopic`, and `sendToErrorTopic` resolves `topic = passed if non-blank else ThreadLocal`; (2) all 22 consumers call `errorQueueProducer.setSourceTopic(topic)` at start of try and `clearSourceTopic()` in finally; (3) `BoundaryService:183` + `ProjectService:124` drop their own emit (keep the rethrow) so the consumer emits the single correct record.
**Status:** Implemented + validated in hcm-fork clone (`D:\DST\scripts\codebase\hcm-fork`), branch `HDDF-5226-uuid-error-record-doc-id`, NOT pushed. Validated by killing the boundary port-forward (8902) and pushing to save-project-task-topic: error doc had topic=save-project-task-topic (not null), single doc, UUID _id, original payload. Keep application.properties LOCAL drift out of the commit. See references/transformer-fixes.md.

### 4. TC-08 silent-enrichment ghost mechanism (STILL UNFIXED)
**Problem:** `ProjectTaskTransformationService.getProjectBeneficiaryDetails` builds the ES doc with null householdId / memberCount when `searchHousehold` / `searchBeneficiary` return empty — no exception is thrown, so the doc is indexed silently with null links. This is THE ghost mechanism (proven: Bauchi 89/89 ES project_task with householdId=null, absent from DB). The error-handling work only captures exception-based failures; this silent-loss class is not caught.
**Approach:** Treat empty enrichment as an error / data-quality flag rather than silently indexing a null-linked doc.
**Fix:** not yet implemented — recommended as a gating fix (after the id fix). Options include raising on empty enrichment or flagging the record.
**Status:** UNFIXED. Highest-value remaining transformer fix.

### 5. Indexer ymls for all transformer flows
**Problem:** egov-indexer (`D:\services\Digit-Core\core-services\egov-indexer`) only had transformer-error + project-task ymls; the other transformer flows had no consumer, so their topics were never indexed.
**Approach:** Add one yml per flow using UNPREFIXED index names (matching the live ES: household-index-v1 etc.) and a Data-wrapping customJsonMapping template, then register every yml.
**Fix:** added 10 ymls (household, household-member, project, project-staff, stock, side-effect, referral, service [service-index-v2], stock-reconciliation, user-action) and registered all in `application.properties:118` `egov.indexer.yml.repo.path` (now 12 `file:///` entries). Indexer restarted and confirmed consuming all new topics.
**Status:** Implemented + confirmed. See references/transformer-fixes.md.

### 6. Endpoint /health-* path fix (RITVIK transformer)
**Problem:** Happy-path transforms failed with NPE — root cause was dependency MISCONFIG, not code. health-* service pods serve under a `/health-*` context path but the transformer application.properties pointed at `/project`, `/individual`, `/service-request`, `/attendance` -> 404 -> projectId resolved null -> `CommonUtils.fetchCycleIndexFromProjectAdditionalDetails` line 225 `projectIdVsCycleInfoCache.get(null)` NPE (ConcurrentHashMap forbids null key, line 44).
**Approach:** Verify each service URL against unified-dev via kubectl, then correct the property paths.
**Fix:** in the RITVIK transformer application.properties set egov.search.project.url=/health-project/v1/_search, project.beneficiary=/health-project/beneficiary/v1/_search, project.task=/health-project/task/v1/_search, project.staff=/health-project/staff/v1/_search, individual=/health-individual/v1/_search, servicedefinition=/health-service-request/service/definition/v1/_search, attendance.register=/health-attendance/v1/_search. LEFT AS-IS intentionally: MDMS=bauchi-hcm.digit.org, localization=oyo-hcm.digit.org.
**Status:** Implemented + validated. All 4 happy flows persisted to ES (household-index-v1, household-member-index-v1, referral-index-v1, stock-index-v1). See references/transformer-fixes.md.

### 7. Architectural options (long-term, summary)
**Problem:** The short-term fixes reduce the ghost-record window but do not structurally eliminate the write-before-durability hazard. `enable.auto.commit=true` cannot be changed (platform-wide DIGIT constraint).
**Options:**
- Option A — uForwarder (Uber OSS): commits offset only after DB write confirmed; kills silent persister failure AND enables retry. Requires converting @KafkaListener to gRPC endpoints across 15+ consumer classes. Effort 2-3 months.
- Option B — Outbox pattern: DB + outbox table in one transaction, CDC publishes to Kafka only after commit; transformer only sees confirmed data. Effort 1-2 months.
- Option C — Wave-based mobile sync (DHIS2 pattern): 5 confirmed waves (individuals -> households -> members -> beneficiaries -> tasks); eliminates the race at source. Effort ~2-3 weeks mobile + 1 week server endpoint.
- Option D — Remove parent validation on bulk async (CommCare pattern): trust the self-consistent offline batch, skip PtProjectBeneficiaryIdValidator for bulk=true, run an hourly orphan-flagging SQL job. Effort ~1 day.
**Status:** Proposed. See references/ghost-record-fixes.md for tradeoff table and priority order.

## Keeping this skill current
- After a dev session on HCM services, add a dated Changelog entry below and fold durable facts (file:line, branch, build commands) into the reference files.
- When the user says "update skills", sync new learnings here from the auto-memory.
- Source memories: hcm_fix_recommendations, project_transformer_error_handling_qa (dev portions), redis_bypass_proof (fix portion).

## Changelog
- 2026-07-08 — Initial consolidation from memory files (hcm_fix_recommendations, project_transformer_error_handling_qa, redis_bypass_proof).
