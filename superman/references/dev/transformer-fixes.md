# Transformer Error-Handling Fixes (HDDF-5226) + Build/Env

Development fixes to egov-transformer and egov-indexer for the error-handling enhancement (ticket HDDF-5226-ERROR-CATCH). Source: project_transformer_error_handling_qa (dev portions). The QA test cases and repro commands live in the separate hcm-qa work-stream; this file is the CODE FIXES only.

## Error-flow model (context)
Flow: source topic -> transformer KafkaListener/transform -> on exception try/catch -> `TransformerErrorProducer` -> transformer-error-topic -> egov-indexer (`transformer-error-indexer.yml`) -> ES index `transformer-error-index-v1`.
`ErrorMessage.java` fields: topic, payload, errorMessage, stackTrace, timestamp (+ the added `id`).

## Which transformer clone is the actually-run one
There are multiple clones on disk — do NOT edit the wrong one.
- ACTUALLY RUN (per the 2026-07-07 sessions): `C:\Users\Salaudeen N\Downloads\RITVIK TRANSFROMER\health-campaign-services\health-services\transformer` — run from the IDE; server.port varies 8180/8183, context-path `/transformer`; already HAS the id + ThreadLocal null-topic fix in ErrorMessage/TransformerErrorProducer.
- Earlier "actually-run" clone: `D:\DST\scripts\codebase\transformer\health-campaign-services\health-services\transformer` — run from IDE, jps PID varies, debug agent. Was the UNFIXED build; the minimal id-only fix was applied here.
- Fork clone (null-topic fix branch): `D:\DST\scripts\codebase\hcm-fork`, branch `HDDF-5226-uuid-error-record-doc-id`, NOT pushed.
- egov-indexer: `D:\services\Digit-Core\core-services\egov-indexer`.

## Fix A — UUID / deterministic error-record id (TC-09 + TC-10)

Problem: `transformer-error-indexer.yml` used `id: $.topic` (later `id: $.topic,$.payload`) as the ES doc id. `IndexerUtils.buildIndexId` (`IndexerUtils.java:365`) casts each id JsonPath value to String. On the keyless/composite error record this either collapsed distinct failures onto one doc (TC-10 overwrite) or, when a Long field like `ErrorMessage.timestamp` was in the id, threw a ClassCastException -> "Indexing will not be done" -> every error record dropped (TC-09). In the actually-run codebase `ErrorMessage` had no `id` at all and `TransformerErrorProducer` set no `.id()`, so egov-indexer (yml `id: $.id`) threw `com.jayway.jsonpath.PathNotFoundException: $['id']` at `IndexerUtils.buildIndexId` line 365 -> every error record dropped; ES error index stuck at 452 (confirmed live from the indexer's own logs, TENANTID=pb).

Minimal id-only fix (applied to the actually-run codebase, 2026-07-07):
1. Add `private String id;` to `ErrorMessage.java`.
2. In `TransformerErrorProducer.sendToErrorTopic`: compute `payloadString` once, set `.id(buildDeterministicId(sourceTopic, payloadString))`.
3. Add `buildDeterministicId` = `UUID.nameUUIDFromBytes((topic + "|" + payload).getBytes(UTF_8))` (add imports `java.util.UUID`, `java.nio.charset.StandardCharsets`).
Deterministic id so Kafka redelivery overwrites (not duplicates) — satisfies TC-11.

Earlier equivalent variant (validated 2026-07-06):
1. `ErrorMessage.java` add `private String id;`.
2. `TransformerErrorProducer.java` set `.id(UUID.randomUUID().toString())`.
3. `D:\services\Digit-Core\core-services\egov-indexer\src\main\resources\transformer-error-indexer.yml` change `id: $.topic,$.payload` -> `id: $.id`.
Rationale: other transformer indexes reuse the entity id (`.id(x.getId())`) and never generate a UUID — the error record is the only keyless one, so a generated id is correct.

Build/validation: `mvn -o compile` = BUILD SUCCESS (JDK8 + C:\apache-maven-3.9.15). Validated locally: 2 distinct + 2 identical malformed payloads -> 4 distinct ES docs, each `_id = id = UUID` (old composite id would collapse the identical two into 1). TC-09 + TC-10 confirmed fixed. Not exercised: TC-06/TC-08 enrichment paths (malformed input fails at deserialization before enrichment). PENDING: user must restart their IDE transformer to load the minimal fix, then verify a malformed push lands in transformer-error-index-v1 (count > 452, `_id` = deterministic UUID).

## Fix B — Null-topic ThreadLocal fallback (TC-06)

Problem: 25 service-layer sites call `sendToErrorTopic(x, null, e)` (no source topic); 2 of them (`BoundaryService.java:183`, `ProjectService.java:124`) ALSO rethrow, so the consumer emits a second record -> a duplicate error record plus a null-topic record that cannot be replayed.

Fix design (behavior-preserving thread-scoped fallback, no RECEIVED_TOPIC threading):
1. `TransformerErrorProducer` adds `ThreadLocal<String> CURRENT_SOURCE_TOPIC` + `setSourceTopic` / `clearSourceTopic`; `sendToErrorTopic` resolves `topic = passed if non-blank else ThreadLocal`.
2. All 22 consumers call `errorQueueProducer.setSourceTopic(topic)` at the start of the try and `clearSourceTopic()` in finally.
3. `BoundaryService:183` + `ProjectService:124` drop their own emit (keep the rethrow) so the consumer emits the single correct record.

Status: implemented + validated in hcm-fork (`D:\DST\scripts\codebase\hcm-fork`), branch `HDDF-5226-uuid-error-record-doc-id`, NOT pushed. Build: JDK8 (`/c/Program Files/Java/jdk1.8.0_202`) + maven `C:\apache-maven-3.9.15`; `mvn -f .../transformer/pom.xml clean install -DskipTests` = BUILD SUCCESS. Validated: killed boundary port-forward (8902), pushed project-task to save-project-task-topic -> error doc had topic=save-project-task-topic (not null), null-topic count stayed 3, single doc, UUID `_id`, original payload preserved.
Commit hygiene: application.properties in this clone has LOCAL drift (mozhealth -> postgres, workflow 8280 -> 8283, boundary 8901 -> 8902) — keep those OUT of the commit.

## Fix C — Indexer ymls for all transformer flows

Problem: egov-indexer (`D:\services\Digit-Core\core-services\egov-indexer`) only had transformer-error + project-task ymls (project-task wrote `so-project-task-index-v1`). Other transformer flows had no indexer consumer.

Fix: added 10 ymls — household, household-member, project, project-staff, stock, side-effect, referral, service (index `service-index-v2`), stock-reconciliation, user-action — using UNPREFIXED index names (household-index-v1 etc., matching the live ES) + a Data-wrapping customJsonMapping template. Registered all in `application.properties:118` `egov.indexer.yml.repo.path` (now 12 `file:///` entries). Indexer restarted and confirmed consuming all new topics.

Live finding while validating: happy household push -> HTTP 404 in boundary/localization/mdms enrichment path; ghost household -> NPE in project/cycle chain; BOTH routed to the error topic. So happy indexing was blocked by an enrichment-dependency 404, separate from the id fix (see Fix D).
Safety: do NOT write test data to tenant `so` (Sokoto production) — the auto-mode classifier blocks it; use tenant `dev`.

## Fix D — Endpoint /health-* path fix (RITVIK transformer application.properties)

Problem: happy-path transforms failed with NPE. Root cause was dependency MISCONFIG, not code: health-* service pods serve under a `/health-*` context path, but transformer application.properties used `/project`, `/individual`, `/service-request`, `/attendance` -> 404 -> projectId resolved null -> `CommonUtils.fetchCycleIndexFromProjectAdditionalDetails` line 225 `projectIdVsCycleInfoCache.get(null)` NPE (ConcurrentHashMap forbids null key, line 44).

Fix (in RITVIK application.properties, verified against unified-dev via kubectl):
- egov.search.project.url=/health-project/v1/_search
- project.beneficiary=/health-project/beneficiary/v1/_search
- project.task=/health-project/task/v1/_search
- project.staff=/health-project/staff/v1/_search
- individual=/health-individual/v1/_search
- servicedefinition=/health-service-request/service/definition/v1/_search
- attendance.register=/health-attendance/v1/_search

LEFT AS-IS intentionally per user: MDMS=bauchi-hcm.digit.org, localization=oyo-hcm.digit.org (point to domain on purpose).
Also: boundary-service pod was OOMKilled (0/1, empty endpoints, :8902 -> 000); user restarted it -> 1/1 -> :8902 boundary-relationships 200.

Result (tenant dev, real refs): all 4 happy flows PERSISTED to ES — household-index-v1, household-member-index-v1, referral-index-v1, stock-index-v1 (each 1 doc, 0 errors). Household needed a re-push (first attempt hit boundary mid-OOM-restart = transient Connection reset; boundary caches the tree globally after first fetch). Household/member resolve projectId via record's user -> project-staff (MUST use a real staff uuid, else projectId null -> NPE); task/stock/referral resolve projectId from record/beneficiary directly.
Real refs used: project 79061c3c-4ef3-411f-88a8-81053fa92034 (SMS_CONFIG_2025 MR-DN), beneficiary clientRef 5860e8e4-b2bb-4ad7-81bd-796b800b007e -> individual b037cee8-0861-4aa4-9abb-0a3e7eda3a61, real project-staff user 769812db-89ed-4f93-a071-80fb35b2b1f1, locality NEWTEST00222_MO_11_06_05_14_03_HOSPITAL_CAMP_CAMP_5_VILLAGE.

## Fix E — TC-08 silent-enrichment ghost mechanism (STILL UNFIXED)
`ProjectTaskTransformationService.getProjectBeneficiaryDetails` builds the ES doc with null householdId / memberCount when `searchHousehold` / `searchBeneficiary` return empty — no exception, so the doc is indexed silently with null links. THIS is the ghost mechanism (proven: Bauchi 89/89 ES project_task with householdId=null, absent from DB). Recommended gating fix: treat empty enrichment as an error / data-quality flag. Not yet implemented.

## JUnit flow-test harness

Location: `...\transformer\src\test\java\org\egov\transformer\flows\`.
Files: `TestDataFactory.java` (dummy-data builders via no-arg ctor + setters) + 7 `*FlowTest.java` (Household, HouseholdMember, SideEffect, ProjectTask, Referral, Stock, ServiceTask) = 14 tests, each a HAPPY + SAD case.
Pattern: construct the `*TransformationService` manually with a REAL `new ObjectMapper()` (transform calls `createObjectNode()`) plus Mockito mocks for all other collaborators; capture `producer.push(topic, list)` via an ArgumentCaptor. SAD cases assert the documented ghost/degraded behaviour (record still emitted with null identity fields — e.g. ProjectTask fabricates synthetic id `status-taskId`, Stock falls back to raw facility id, Referral null project linkage); ServiceTask SAD asserts the malformed-code exception path (nothing produced).
Run: `export PATH="/c/apache-maven-3.9.15/bin:$PATH"; mvn -o test -Dtest='*FlowTest'` (JDK8; `-o` offline, deps already in ~/.m2). All 14 green.
Model APIs discovered via `javap` against `health-services-models-1.0.19-irs1-SNAPSHOT.jar`.
Gotchas: `ProjectTaskIndexV1.isDelivered()` is a primitive boolean getter (not getIsDelivered); surefire `-Dtest` needs `*FlowTest`, not a package glob.
Not yet covered (same pattern extends): Bill*, MusterRoll, Attendance*, PGR, HfReferral, StockReconciliation, UserAction, Project, ProjectStaff, DeviceToken, AttendanceStaff, Attendee.

## Build / env facts developers need

- JDK: JDK8 at `/c/Program Files/Java/jdk1.8.0_202`.
- Maven: `C:\apache-maven-3.9.15` (bash: `export PATH="/c/apache-maven-3.9.15/bin:$PATH"`).
- Build commands: `mvn -o compile` (offline compile), `mvn -o test -Dtest='*FlowTest'` (offline tests), `mvn -f .../transformer/pom.xml clean install -DskipTests`. `-o` = offline (deps already in ~/.m2).
- Local infra (docker): Kafka on 9092 (container name `kafka`), ES on 9200 (no auth), Redis on 6379, Kibana. ES before this work only had so-project-task-index-v1 / so-user-sync / transformer-error.
- Produce test messages: `docker exec kafka /opt/kafka/bin/kafka-console-producer.sh ...`; prefix with `MSYS_NO_PATHCONV=1` to stop Git Bash mangling topic paths.
- Downstream services are k9s / port-forwards to unified-dev (0 kubectl.exe procs); forwards go stale when pods restart. Ports observed: health-project 8980, household 8081, health-individual 8983, product 8986, project-factory 8801, facility 8982, health-service-request 8280, health-attendance 8800, health-expense 8085, egov-notification-push 8082, boundary-service 8902, egov-location 8901, egov-user 8284, egov-workflow-v2 8283.
- Env instability tolerated by the code (caught, non-fatal): product :8986 + project-factory :8801 forwards drop intermittently (ProductService returns input id, ProjectFactoryService returns null campaignId); user :8284 returns 400 "Unknown error occurred in decryption process" (caught, username degraded).
- Error-topic validation script (QA-side, for reference): `C:\Users\Salaudeen N\Downloads\transformer_error_topic_validation.sh` (10/10 source topics push malformed -> transformer-error-topic).
