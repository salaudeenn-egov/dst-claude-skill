# Transformer End-to-End Scenario Testing

Full happy/sad/exception/edge scenario testing of the health-campaign transformer, run live
(Kafka -> transformer -> egov-indexer -> Elasticsearch) against dev data. Complements
`transformer-error-handling.md` (that = HDDF-5226 12-TC error record; this = full flow coverage +
the environment/fixes that make end-to-end indexing actually work). Full artifacts live in
`C:\Users\Salaudeen N\Downloads\` (README_Transformer_Scenario_Testing.md, TEST_SCENARIO_CATALOG.md,
Transformer_Test_Report.xlsx, transformer_30_scenarios.sh, transformer_error_topic_validation.sh, make_report.py).

## PASS criterion (the mental model)
We test whether the transformer **captures & sends errors**, NOT data completeness. A case PASSES if the
transformer either (a) transforms & indexes the record, OR (b) catches the failure and sends it to
`transformer-error-topic`. Null/missing enriched fields (age, individualId, boundary) are **service-layer
data issues -> PASS**. FAIL only if an error occurred but was not captured/sent (silent loss).

## Pipeline & ownership
`save-*-topic (JSON)` -> `XxxConsumer` (manual objectMapper.readValue, one per entity) -> `XxxTransformationService`
(enrich via REST) -> `producer.push(transformer-producer-*-index-v1-topic)` -> egov-indexer (yml) -> ES `*-index-v1`.
On any thrown exception: consumer catch -> `TransformerErrorProducer` -> `transformer-error-topic` -> error index.
**Transformer-owned indexes:** household, household-member, project, project-staff, project-task, stock,
side-effect, referral, service(v2), stock-reconciliation, user-action, transformer-error.
**NOT owned** (separate direct service->indexer path): `individual-index-v1`, `project-beneficiary-index-v1`
(transformer only READS them as enrichment).

## Prerequisites (check before testing)
- Infra docker containers UP: `kafka`(9092), `elasticsearch`(9200), `redis`, `kibana`. They get SIGKILLed
  (exit 137) on host sleep -> `docker start elasticsearch kafka redis kibana`; ES cold-start ~30-60s.
- Transformer running (RITVIK codebase `C:\Users\Salaudeen N\Downloads\RITVIK TRANSFROMER\...\transformer`,
  port 8180, run from IDE) + egov-indexer (8095). Restart transformer after properties change; restart
  indexer after any yml change.
- 14 dependency port-forwards up (kubectl -> unified-dev): project 8980, household 8081, individual 8983,
  product 8986, servicedef 8280, location 8901, project-factory 8801, facility 8982, user 8284, workflow 8283,
  boundary 8902, bill 8085, device-token 8082, attendance 8800. Triage: `000`=down, `404`=wrong path, `400/415`=reachable.
- `boundary-service` pod must be `1/1` (it OOM-cycles; when down: `:8902 -> 000`, 0/1, empty endpoints ->
  `kubectl delete pod -n egov boundary-service-...`).
- Always `export MSYS_NO_PATHCONV=1` for `docker exec ... kafka /opt/kafka/...` calls.

## Fixes that made end-to-end work (root causes, in order found)
1. **Service URL paths** — health-namespace pods serve under `/health-*` (context-path == service name).
   Fix transformer application.properties: project=`/health-project/v1/_search` (+beneficiary/task/staff),
   individual=`/health-individual/v1/_search`, servicedefinition=`/health-service-request/service/definition/v1/_search`,
   attendance=`/health-attendance/v1/_search`. household/product/facility/project-factory/user/workflow/
   boundary/expense keep their paths. Leave MDMS(bauchi-hcm) + localization(oyo-hcm) pointing to their domains.
2. **Error id** — `transformer-error-indexer.yml` uses `id: $.id`; transformer must emit `id` or every error
   record is dropped (`PathNotFoundException $['id']`). Fix: `ErrorMessage.id` + `TransformerErrorProducer`
   sets `.id(buildDeterministicId(topic,payload))` = `UUID.nameUUIDFromBytes((topic+"|"+payload)...)` (dedup-safe).
3. **Indexer ymls** — 12 registered in `egov.indexer.yml.repo.path` (comma-sep file:/// list). Use UNPREFIXED
   index names (`project-task-index-v1`, not `so-...`). Wrapper docs id = nested clientReferenceId+tenantId;
   flat docs id = `$.id`+`$.tenantId`. Data-wrap `customJsonMapping`.

## Dev reference data (tenant `dev`; refetch if aged)
project `79061c3c-4ef3-411f-88a8-81053fa92034` (SMS_CONFIG_2025, MR-DN); project-staff user
`769812db-89ed-4f93-a071-80fb35b2b1f1`; beneficiary `5860e8e4...` & `cf2f78e0...`; individual `b037cee8...`;
household `2b639a10-a359-11f0-ad6d-ada985cd3a25`; real task cRef `d8d6a210...` (resource `d8d73e50...`,
product `PVAR-2025-01-03-000143`); locality `NEWTEST00222_MO_11_06_05_14_03_HOSPITAL_CAMP_CAMP_5_VILLAGE`.
**Household & household-member resolve projectId via record user -> project-staff** — MUST use a real staff
uuid as createdBy/lastModifiedBy or projectId is null -> NPE. Task/stock/referral resolve projectId directly.
Get a staff user: `POST :8980/health-project/staff/v1/_search {"ProjectStaff":{"projectId":["79061c3c..."]}}`.
Get a real task: `POST :8980/health-project/task/v1/_search`.

## Behavior taxonomy (verified live, ~42 cases)
| Behavior | Trigger | Lands | Error sent? | Verdict |
|---|---|---|---|---|
| Happy | valid, links resolve, staff user | main *-index-v1 | N/A | PASS |
| Silent ghost | valid, a link missing | main index, null fields | No | PASS (data gap upstream) |
| Degraded | facility/product/campaign miss (caught) | main index, raw-id/null | No | PASS |
| Crash NPE | projectId null -> `ConcurrentHashMap.get(null)` `CommonUtils:225` | error index | Yes | PASS (captured) |
| Malformed/wrong-shape | bad JSON / object-not-array / wrong wrapper | error index (UUID _id) | Yes | PASS (captured) |
| Empty `[]` | no records | no-op | N/A | PASS |
| Indexer drop | main doc missing top-level id | dropped | No | note (data: missing id) |
| Batch mid-fail | one bad record in array | whole batch -> error, none indexed | Yes | PASS (no per-record isolation) |
| Duplicate | same payload redelivered | 1 error doc (deterministic id) | Yes | PASS (dedup) |
| Multi-resource | task w/ N resources | N project-task docs (flatMap) | N/A | PASS (fan-out) |
| Numeric parse fail | additionalFields non-numeric | indexed, field null | No (NumberFormatException caught) | PASS |

## Evidence trail (localise ANY failure fast — "how far did it get?")
1. On `transformer-error-topic`? read `errorMessage`+`stackTrace` (record carries both). `errorMessage:null`=NPE.
2. Not on error topic, not indexed? did transformer PRODUCE? (`GetOffsetShell` on the producer topic). If yes,
   the INDEXER dropped it -> indexer log `PathNotFoundException $['id']` = missing id.
3. Indexer consuming? `kafka-consumer-groups --describe --group egov-indexer-consumer-grp` (lag).
4. Dependency? probe `:port` (000/404/400), `kubectl get pods` (0/1), `logs` (OOM), `get endpoints` (empty).
Trust **count deltas + exact _id**, not full-text `q=` hits (they false-match common words like "malformed").

## Known defects surfaced (log as findings, not test fails)
- Silent ghosts (TC-08): broken links -> degraded doc, no error. Core data-integrity gap.
- NPE on null projectId: `CommonUtils.fetchCycleIndexFromProjectAdditionalDetails:225` (ConcurrentHashMap null key). Guard: `if(projectId==null)return null;`.
- Indexer silently drops main-index docs lacking top-level id.
- ProjectTask individualId/householdId null for valid INDIVIDUAL beneficiary — key mismatch (`INDIVIDUAL_ID` vs `INDIVIDUAL_CLIENT_REFERENCE_ID`).
- Legacy dispatch (`TransformationHandler`/`Operation`/`Transformer`) is dead code; routing is direct DI.
- No batch isolation: one bad record fails the whole Kafka batch.

## Gotchas checklist
MSYS_NO_PATHCONV=1 on kafka docker calls · restart transformer after properties, indexer after ymls ·
boundary pod 1/1 + forwards alive · staff user for household/member · top-level id required or indexer drops ·
never write tenant `so` (Sokoto prod) — use `dev` · project consumer wants `{Projects:[...]}` wrapper ·
individual/beneficiary indexes are NOT transformer-owned.

## Not-yet-run (for exhaustive coverage)
facility, hf-referral, pgr, device-token, attendance(log/register/attendee/staff), bill/bill-detail/bill-report,
muster-roll; update-topic variants (stock update emits a 2nd doc to a topic with no yml); the successful index
path for project & user-action; dependency-outage & retry/offset behavior. Catalog has ~90 IDs total.
