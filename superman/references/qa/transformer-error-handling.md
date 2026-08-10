# Transformer Error-Handling QA (HDDF-5226)

QA of the Transformer Error-Handling Enhancement, ticket HDDF-5226-ERROR-CATCH. Ties to the Sokoto SMC + Bauchi MR-DN ghost-record RCAs. This file is the test-side record; the code fixes (UUID id, null-topic ThreadLocal fallback) are hcm-dev implementation items, referenced here only to say what the tests gate.

## What it does & why we test it
The enhancement is supposed to catch transform-layer exceptions and persist them to an error index so failures become visible. The QA goal: prove which failure classes are captured, and expose the SILENT-LOSS class behind the ghost records that is NOT handled.

## Setup (failure injection)
- egov-transformer + egov-indexer against the same Kafka + ES, tenant `dev`.
- Inject failures via `kafka-console-producer` to source topics (`save-project`, `save-project-task-topic`, etc.). Verify on `transformer-error-topic` and ES index `transformer-error-index-v1`.
- Flow: source topic -> transformer KafkaListener/transform -> on exception try/catch -> TransformerErrorProducer -> transformer-error-topic -> egov-indexer (`transformer-error-indexer.yml`) -> ES.
- Error model `ErrorMessage.java` fields: topic, payload, errorMessage, stackTrace, timestamp.
- Produce via `docker exec kafka /opt/kafka/bin/kafka-console-producer.sh` (use MSYS_NO_PATHCONV=1 to stop Git Bash path mangling).
- Do NOT write test data to tenant `so` (Sokoto production) — auto-mode classifier blocks it; use tenant `dev`.

## Result matrix (reproduction guide, 12 TCs)
- PASS: TC-01 malformed JSON, TC-02 empty payload, TC-03 wrong shape, TC-06 4xx, TC-07 unreachable dep, TC-11 unique-id persistence.
- FAIL: TC-04, TC-05, TC-08, TC-09, TC-10, TC-12.
Exception-based failures are captured + persisted; the silent-loss class behind the ghost records is NOT handled.

## Defects (code in `D:\DST\scripts\codebase\transformer\...\transformer` + egov-indexer)
- TC-08 (High) — empty enrichment indexed silently with null links. `ProjectTaskTransformationService.getProjectBeneficiaryDetails` builds a doc with null householdId/memberCount when `searchHousehold`/`searchBeneficiary` return empty; no error raised. THE ghost mechanism. Proven: Bauchi 89/89 ES project_task with householdId=null, absent from DB. Still unfixed.
- TC-07 (High) — NO retry for transient dependency outage. auto-commit true -> offset committed -> not reprocessed when the dependency recovers. Fix: bounded retry / dead-letter.
- TC-09 (High, config) — indexer DROPS all error records. `IndexerUtils.buildIndexId` (IndexerUtils.java:365) casts each id JsonPath value to String; `ErrorMessage.timestamp` is Long -> ClassCastException -> index never created -> records lost. Fix: String-only id or coerce.
- TC-10 (Med) — error index overwrite. Doc id not unique (`id: $.topic`); a 2nd failure overwrites the 1st. Fix: UUID as doc id.
- TC-06 (Med) — duplicate records + null source topic. `BoundaryService.java:183` (and `ProjectTransformationService.java:191`) call `sendToErrorTopic(req, null, e)`; the consumer also sends. The null-topic record can't be replayed.
- TC-04 (Med) — blank error message on NPE. `TransformerErrorProducer` uses `exception.getMessage()` (null). Fix: use toString() / class + root cause.
- TC-12 (Med) — errors not forwarded to egov-tracer (`egov-tracer-error-details`); only in the transformer's own index.
- TC-05 (Low) — empty batch `[]` = silent no-op.

## Gating recommendation
Before relying on this as a production data-integrity safeguard, fix in priority order: TC-09 (unique String UUID doc id — also fixes TC-10), then TC-08 (treat empty enrichment as error / data-quality flag), then TC-07 (retry/DLQ), then TC-06 / TC-04 / TC-12 / TC-05.

## JUnit flow-test harness (2026-07-07)
In-repo unit tests for the transform layer at `...\transformer\src\test\java\org\egov\transformer\flows\`: `TestDataFactory.java` (dummy-data builders via no-arg ctor + setters) + 7 `*FlowTest.java` (Household, HouseholdMember, SideEffect, ProjectTask, Referral, Stock, ServiceTask) = 14 tests, each a HAPPY + SAD case.
- Pattern: construct the `*TransformationService` manually with a REAL `new ObjectMapper()` (transform calls `createObjectNode()`) + Mockito mocks for all other collaborators; capture `producer.push(topic, list)` via ArgumentCaptor.
- SAD cases ASSERT the documented ghost/degraded behaviour (record still emitted with null identity fields — ProjectTask fabricates a synthetic id `status-taskId`, Stock falls back to raw facility id, Referral null project linkage); ServiceTask SAD asserts the malformed-code exception path (nothing produced).
- Run: `export PATH="/c/apache-maven-3.9.15/bin:$PATH"; mvn -o test -Dtest='*FlowTest'` (JDK8; `-o` offline, deps in ~/.m2). All 14 green.
- Gotchas: `ProjectTaskIndexV1.isDelivered()` is a primitive boolean getter (not getIsDelivered); surefire `-Dtest` needs `*FlowTest`, not a package glob. Model APIs discovered via `javap` against `health-services-models-1.0.19-irs1-SNAPSHOT.jar`.
- Not yet covered (same pattern extends): Bill*, MusterRoll, Attendance*, PGR, HfReferral, StockReconciliation, UserAction, Project, ProjectStaff, DeviceToken, AttendanceStaff, Attendee.

## Test docs (Downloads)
- `Transformer_ErrorHandling_Test_and_Reproduction_Guide.docx` — AUTHORITATIVE detailed record; 12 TCs with exact kafka-console-producer/curl repro commands + code root causes.
- `QA_Test_Report_Transformer_ErrorHandling_HDDF-5226.docx` — summarized QA report (11 TCs; drops TC-09 & TC-12).
- `Transformer_ErrorHandling_Test_Report.docx` — earlier report version.
- Error-topic validation script `C:\Users\Salaudeen N\Downloads\transformer_error_topic_validation.sh` = 10/10 source topics push malformed -> transformer-error-topic.

## Environment / corrections notes (test-side)
- The actually-run transformer varies by session: `D:\DST\scripts\codebase\transformer\health-campaign-services\health-services\transformer` (unfixed id build in one session) and `C:\Users\Salaudeen N\Downloads\RITVIK TRANSFROMER\health-campaign-services\health-services\transformer` (has the id + ThreadLocal fix). Run from IDE; server.port varies (8180/8183); context-path `/transformer`.
- Local docker Kafka(9092) + ES(9200, no auth) + Redis + Kibana; enrichment services port-forwarded to unified-dev. Real dev refs for happy-path runs: project 79061c3c-4ef3-411f-88a8-81053fa92034 (SMS_CONFIG_2025 MR-DN), beneficiary clientRef 5860e8e4-b2bb-4ad7-81bd-796b800b007e -> individual b037cee8-0861-4aa4-9abb-0a3e7eda3a61, project-staff user 769812db-89ed-4f93-a071-80fb35b2b1f1, locality NEWTEST00222_MO_11_06_05_14_03_HOSPITAL_CAMP_CAMP_5_VILLAGE.
- Happy-path 404 correction: happy-path failures were NOT code but a DEPENDENCY CONTEXT-PATH misconfig — transformer application.properties used `/project`,`/individual`,`/service-request`,`/attendance` but the health-* pods serve under `/health-*` -> 404 -> projectId resolved null -> `CommonUtils.fetchCycleIndexFromProjectAdditionalDetails:225` `projectIdVsCycleInfoCache.get(null)` NPE (ConcurrentHashMap forbids null key). After fixing the URLs (/health-project, /health-individual, /health-project/task, /health-project/beneficiary, /health-project/staff, /health-service-request/service/definition, /health-attendance), ALL 4 happy flows PERSISTED to ES: household-index-v1, household-member-index-v1, referral-index-v1, stock-index-v1 (each 1 doc, 0 errors). Household/member resolve projectId via record's user -> project-staff (needs a REAL staff uuid else projectId null -> NPE); task/stock/referral resolve from record/beneficiary directly.
- Env instability caught (non-fatal): product :8986 + project-factory :8801 port-forwards drop intermittently but are caught (ProductService returns input id, ProjectFactoryService returns null campaignId); user :8284 returns 400 "decryption process" -> caught, username degraded. Port-forwards go stale when pods restart; boundary-service was OOMKilled once (restart -> 1/1).
