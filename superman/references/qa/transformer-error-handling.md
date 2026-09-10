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

## Standalone injection suite (2026-08-19): Downloads\transformer_test_suite.py
Self-contained, env-agnostic runner for the 12 HDDF-5226 TCs: kafka-python injection into
(tenant-prefixed) source topics + marker-based ES verification (unique run marker embedded per
payload so only our records match in the shared error index). Built-in defect signatures:
TC-04 blank errorMessage, TC-06 null-topic/dup, TC-08 ghost-beneficiary task silently indexed with
null links (checks BOTH error index and entity index), TC-09 error docs landing at all, TC-10/11
redelivery (distinct UUID docs vs deterministic overwrite vs lost). TC-07 = MANUAL (kill a dep).
Config via env vars TT_TENANT/TT_KAFKA/TT_ES/TT_ES_USER/TT_ES_PASS/TT_TOPIC_PREFIXED - creds NEVER
hardcoded into the KB; in-cluster DNS (elasticsearch-data.es-cluster-v8:9200, kafka headless svc)
resolves only inside the cluster -> run from a jump host or kubectl port-forward both. Unreachable
deps -> TCs report SKIPPED (never fake-fail). Writes transformer_suite_results_<tenant>_<ts>.json.

## LIVE RUN on mhbase-hcm-dev (2026-08-19) - suite v2, control-validated
Env facts that made the run work (all discovered by probing, each one had cost a failed attempt):
- Source topics ARE tenant-prefixed (`mhbase-save-project-task-topic`) but the **error topic is
  GLOBAL/UNPREFIXED: `transformer-error-topic`** (one shared topic across tenants; 429 records
  pre-existing). Watching `<tenant>-transformer-error-topic` = watching a topic that does not exist.
- Tracer topic on this cluster is `egov-tracer-error-details-enriched` (the plain name does not exist).
- Transformer consumers deserialize **List<Entity>**: a bare object payload dies at deserialization
  ("Cannot deserialize instance of `[Lorg.egov.common.models.project.Task;") BEFORE any enrichment -
  my first run's TC-04/06/07/08 verdicts were all invalid for this reason. Payloads must be `[ {...} ]`.
- Lesson institutionalised: the suite now injects a **TC-14 control** (fully valid task array) which
  must produce NO error record AND be indexed. Without it, "no error emitted" is unattributable.

Method that makes findings attributable: TWO-STAGE verification per case -
KAFKA stage (did the transformer catch it? read only records after a captured baseline offset) and
ES stage (did the indexer persist it?). Kafka-yes + ES-no isolates the indexer leg.

### Results (13 cases): 4 FINDING / 1 OBSERVATION / 6 PASS / 2 INFO
- **TC-08 CRITICAL - ghost-record mechanism REPRODUCED ON DEMAND.** Problem: valid task whose
  projectBeneficiary does not exist. Behaviour: NO error record emitted anywhere, and the doc is
  INDEXED into `mhbase-project-task-index-v1` with `householdId`, `projectBeneficiaryId`,
  `memberCount` = null. This is exactly the production Sokoto/Bauchi signature (ES rows with null
  links, unmatched in DB, nothing in any error queue). Fix: treat empty enrichment as error /
  data-quality flag instead of indexing a hollow doc.
- **TC-09 HIGH - error records never reach ES.** 437 records on `transformer-error-topic`, but NO
  transformer error index exists (only `mhbase-transformer-pgr-services` matches `*transformer*`).
  Root cause is NOT the old unified-dev missing-id bug: **TC-11 PASSED, 8/8 records carry an `id`**
  (the July id fix is present in this build). So it is an indexer subscription/config gap -
  `transformer-error-indexer.yml` not registered in `egov.indexer.yml.repo.path` / not consuming the
  global topic. Consequence: error capture works but is invisible to operators (nothing in Kibana).
  Distinguish "not subscribed" vs "subscribed but failing" by checking for a consumer group on the
  topic (kafka-python 3.x: KafkaAdminClient method names differ - introspect dir(admin)).
- **TC-04 MEDIUM - blank errorMessage** on NPE-class failures (also on TC-06): record has id +
  correct source topic but `errorMessage: null` (exception.getMessage() on an NPE). Unactionable.
- **TC-12 MEDIUM - no tracer forwarding**: enriched tracer topic offsets unchanged 7796 -> 7796.
- TC-07 OBSERVATION: ghost productVariantId -> no error emitted at all (same silent-absorption
  family as TC-08; check whether the doc still got indexed).
- PASS: TC-01 malformed JSON, TC-03 wrong shape, TC-13 household path (capture works across
  entities), TC-10 identical payload x2 -> 2 distinct records, TC-11 ids present, TC-14 control.
Deliverables: `Downloads\transformer_test_suite.py` (v2 suite), `check_kafka_es.py`,
`check_kafka_push.py`, `probe_es_indices.py`, `probe_kafka_consumers.py`, `inspect_error_topic.py`,
results `Downloads\Transformer-Test-Results-mhbase-2026-08-19.xlsx` + suite JSON.

## Round 2 on mhbase-hcm-dev (2026-08-24) — TC-09 FIXED, but the fix created new defects

**Problem.** After the 08-19 run the open question was whether the error path had been repaired.
The user's own Kibana query showed `transformer-error-index-v1` with 646 docs — so the headline
("errors are now indexed") looked like a clean pass. Misleading first read: doc-count > 0 was taken
as "error handling works". It does not follow — the docs are present but largely unusable, and the
mechanism that actually manufactures ghost records still emits nothing at all.

**Approach.** Built `Downloads\transformer_error_handling_test.py` (v3 -> v4) as an ES-ONLY
verifier: Kafka is used solely to inject, every verdict comes from a pre-injection ES baseline
(newest `@timestamp` + doc count + ids of docs already holding markerless payloads) compared with a
post-settle range query. Added a per-doc grader (D1-D7 codes) applied both to injected docs and, in
`TT_MODE=audit`, to whatever is already in the index. A control (valid task, must index + emit no
error) ran green in every round, so the other verdicts are attributable.

**Root causes / findings (all reproduced twice, runs `87e3e27c` and `8a5c28a2`):**
- **TC-09 FIXED.** Index exists and receives docs (646 -> 653 -> 661 across two runs). The 08-19
  "indexer subscription gap" is closed. Every doc carries an `id`.
- **BATCH ISOLATION — NEW, HIGH.** `[valid, bad, valid]` in ONE array: **0/2 healthy siblings
  indexed**. One bad element destroys the good records in the same batch, and the emitted error doc
  contains all three payloads with `errorMessage: null`. Silent data loss at batch granularity —
  the same class as the persister's EDGE-1, now proven on the transformer. This is the most
  significant new finding; a real device sync sends batches, so any single malformed record in a
  batch loses its neighbours.
- **D2 content-hash doc id.** `_id` is a **UUIDv3** (name-based hash of content; verified with
  `uuid.UUID(id).version == 3` on every doc). Identical payloads collapse to ONE doc: 3 identical
  pushes -> 1 doc, 1 distinct `_id`. Worse, it collapses ACROSS RUNS — doc `401501dc-...` (payload
  `{}`) was written 17:45:40.389, then **overwritten** by a later run at 17:53:44.121, destroying the
  earlier run's evidence. The July id fix cured the old `$.topic` overwrite and swung to the opposite
  failure: the index cannot represent failure FREQUENCY. N redeliveries look like 1. Kafka retains
  the truth; ES does not.
- **D1 blank `errorMessage`** on the NPE class: 6 of 31 docs on the test day (19%), all on
  `mhbase-save-project-task-topic`. Paired with D4 (truncated stack: one doc had `stackTrace` =
  the single line `java.lang.NullPointerException`, no frames) the failure is recorded but not
  diagnosable.
- **`errorMessage` embeds the payload -> unaggregatable.** Jackson inlines the offending payload
  into the message, so a `terms` agg on `errorMessage.keyword` yields one bucket per payload
  (cardinality == number of distinct payloads). Proof: the only bucket with `doc_count > 1` was the
  40KB case where Jackson wrote `(StringReader)` instead of inlining. Combined with D1, a dashboard
  built on that agg under-reports and never groups.
- **D3 keyword subfields ignored.** `payload.keyword` / `stackTrace.keyword` land in `_ignored`
  (over `ignore_above`) on nearly every doc -> those fields cannot be term-queried or aggregated,
  only full-text matched.
- **TC-08 ghost mechanism STILL PRESENT.** Valid task + ghost beneficiary -> no error doc anywhere,
  indexed into `mhbase-project-task-index-v1` with `householdId` / `projectBeneficiaryId` /
  `memberCount` = null. Unchanged from 08-19.
- **TC-12 tracer** still no forwarding.
- **Project path UNTESTED, not broken.** Neither `mhbase-save-project` nor `mhbase-save-project-topic`
  produced anything. Arithmetic closed exactly (10 Kafka records = 8 distinct payloads, one of which
  overwrote an existing doc = +7), proving the Project consumer never received the message. Do not
  file this as a defect until the live topic name is confirmed.

**KB CORRECTION — TC-02 was wrong.** The 12-TC matrix recorded empty `{}` as a "documented silent
no-op". It is NOT: `{}` produces a real error doc (`Cannot deserialize ... out of START_OBJECT
token`), because the consumers deserialize `List<Entity>` so a bare object dies at parse. Empty
array `[]` IS a genuine silent no-op (correctly produces nothing). Verified in both runs.

**Harness bugs found in our own scripts (fix before trusting a result):**
- v3 skipped every markerless case (`if not marker: continue`), so the `{}` and `[]` cases were
  never graded and never appeared in the results JSON — a silent coverage hole that made the old
  TC-02 claim look confirmed. v4 verifies markerless payloads by exact-payload match inside the
  time window.
- Kafka offset windows are unreliable for attribution because the error topic is GLOBAL/shared —
  another tenant's activity lands in the same window. v4 dropped all Kafka reads.

**Fix recommendation (one change addresses several).** In `TransformerErrorProducer`, emit the
exception CLASS and root-cause message as their own field and keep the offending payload out of the
message (`exception.getClass().getName() + ": " + rootCause`), and give the doc id an identity
component (source topic + entity clientReferenceId + timestamp) rather than a pure content hash.
That fixes D1, the cardinality problem and D2 together. Batch isolation and TC-08 are separate,
larger fixes in the consumer/transformation layer.

**Deliverables:** `Downloads\transformer_error_handling_test.py` (v4, ES-only, audit + full modes,
batch-isolation case, `TT_SKIP_BATCH=1` to omit), `Downloads\probe_e4_project_path.py` (Kafka-side
topic/consumer probe for the Project path), results `transformer_error_v3/v4_mhbase_*.json`.

### ES query recipes for this index (all verified live)
- **Today only, WAT:** `{"range":{"@timestamp":{"gte":"now/d","lt":"now+1d/d","time_zone":"+01:00"}}}`
  — without `time_zone` you lose the 00:00-01:00 WAT window into "yesterday".
- **Count blank errorMessage:** `{"bool":{"must_not":{"exists":{"field":"errorMessage"}}}}`.
  Do NOT use a `missing` agg on `errorMessage.keyword` — `missing` also counts docs whose keyword
  was dropped by `ignore_above`, which inflated our first reading from 6 to 10.
- **Grouping:** only `topic.keyword` is reliable. `match_phrase` on `errorMessage` for a class name
  (e.g. `models.project.Task`) returns ZERO — the standard analyzer does not tokenize the dotted
  FQN as assumed; run `_analyze` on the field before writing any `match` clause.
- The doc `id` field == `_id` (both the UUIDv3 content hash); there is NO field carrying the failed
  entity's id / clientReferenceId / tenantId, so an error doc cannot be correlated back to the
  record that failed, nor filtered per tenant on this GLOBAL index.
