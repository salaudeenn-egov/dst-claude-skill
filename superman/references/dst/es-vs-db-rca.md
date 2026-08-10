# ES vs DB RCA — the 6-phase SOP and the 90,143-ghost-record investigation

Proven end-to-end on Sokoto SMC (`so`), May 2026, and Bauchi MR-DN (`ba`). This is
the fixed order for investigating a PostgreSQL-vs-Elasticsearch count mismatch in
DIGIT HCM. Follow the order — it eliminates the wrong hypotheses (persister downtime,
soft-delete, wrong tenant) in phases 1-3 before you touch infra logs.

For platform architecture (Kafka two-topic pattern, ES/DB divergence point, Redis,
entity chain), credentials, and the full gotcha list, see the `hcm-context` skill —
do not duplicate it here.

---

## The 6-phase SOP

### Phase 1 — Run the comparison script first
Use the `OY_VERSION` / `so_*` / `BA_comparison_script.py` pattern that:
- Pulls IDs from DB with the right filters (status, doseIndex, isdeleted=false)
- Scrolls all IDs from ES with matching filters
- Outputs: Summary CSV, DB-only IDs, ES-only IDs, detail files

Match key is `clientReferenceId` (mobile-generated, present in both ES and DB), NOT
the server-generated ID (PTB-xxx, PT-xxx) which is absent from the Kafka message
before enrichment.

Standard filters: `status=ADMINISTRATION_SUCCESS`, `doseIndex=01`, `isdeleted=false`.

Key outputs to read immediately:
- `soft_deleted_count = 0` → rules out soft-delete hypothesis
- `rowversion = 1` for all DB-only → single Kafka event, indexer missed it
- `createdTime == lastModifiedTime` in ES-only docs → ES never updated after initial index

### Phase 2 — Characterise the DB-only records (the smaller anomaly)
```sql
SELECT date_trunc('day', to_timestamp(createdtime/1000)) AS day,
       COUNT(*), MIN(rowversion), MAX(rowversion)
FROM <schema>.project_task
WHERE clientreferenceid IN (<ids>)
GROUP BY 1 ORDER BY 1;
```
- All same day + all rowversion=1 → indexer failure on that date (data safe, re-index)
- Spread across days → different issue (re-check indexer config)

### Phase 3 — Characterise the ES-only records (the main anomaly = ghosts)
3a. Existence check (no filters):
```sql
SELECT * FROM <schema>.project_task WHERE clientreferenceid IN (<sample ids>);
```
- 0 rows → never written to DB (consumer failure, not delete)
- Rows returned → filter mismatch in the comparison script

3b. Date cluster analysis (Python):
```python
es_df['cd'] = pd.to_datetime(es_df['Data.createdTime'], unit='ms')
print(es_df['cd'].dt.date.value_counts().sort_index())
```
- Clusters at round-end dates → offline sync race condition
- Random distribution → different cause

3c. ES field checks:
```python
print([c for c in es_df.columns if 'dose' in c.lower()])
print(es_df['Data.additionalDetails.doseIndex'].value_counts())
print(es_df['Data.createdTime'].equals(es_df['Data.lastModifiedTime']))  # True = never updated
```

### Phase 4 — Tracer evidence (confirm the consumer failure)
Working Kibana query pattern (tested on Nigeria cluster):
```json
{
  "query": {
    "bool": {
      "must": [
        { "match_phrase": { "Data.apiDetails.requestBody": "\"tenantId\":\"<tenant>\"" }},
        { "match_phrase": { "Data.apiDetails.url": "project/task" }}
      ],
      "filter": [{ "range": { "Data.@timestamp": { "gte": "<start>", "lte": "<end>" }}}]
    }
  },
  "aggs": {
    "by_error_message": { "terms": { "field": "Data.errors.errorMessage.keyword", "size": 30 }},
    "by_day": {
      "date_histogram": { "field": "Data.@timestamp", "calendar_interval": "day" },
      "aggs": { "errors": { "terms": { "field": "Data.errors.errorMessage.keyword", "size": 10 }}}
    }
  }
}
```

Tracer gotchas:
- Tracer index: `egov-tracer-error-details`
- `requestBody` uses `"tenantId":"so"` WITH quotes — confirmed working format
- Timestamp field is `Data.@timestamp`, NOT `Data.auditDetails.createdTime` (can be null)
- Error field partial match: use `wildcard` on `.keyword`, NOT `match_phrase` (arrays fail)
- Exact buckets: `Data.errors.errorMessage.keyword`
- `total_unique` cardinality agg → count unique beneficiary UUIDs in "Related entity" errors
- Task-consumer error URL: `/project/task/v1/bulk/_create`
- Individual-consumer error URL: `/individual/v1/bulk/_create`
- The "Related entity does not exist in db <id>" message contains the PARENT
  (beneficiary) clientReferenceId, not the task's ID.

Find one ghost's error in tracer:
```json
GET egov-tracer-error-details/_search
{"size":1,"query":{"bool":{"must":[
  {"match_phrase":{"Data.apiDetails.url":"/project/task/v1/bulk/_create"}},
  {"match_phrase":{"Data.apiDetails.requestBody":"<clientReferenceId>"}}
]}}}
```

What the error messages mean:
- `"Related entity does not exist in db"` → beneficiary race condition (Root Cause C)
- `"Connection refused to egov-idgen"` → idgen outage (check if unique to one day → D)
- `"DUPLICATE_ENTITY"` from `PtExistentEntityValidator` → normal, ignore
- `"UPLOAD_ERROR_FROM_APP"` → mobile retry indicator (Root Cause A), see below

### Phase 5 — Root cause classification
| Pattern | Root cause | Fix |
|---|---|---|
| Records missing from DB, 0 tracer errors | Kafka event loss / silent persister failure | Re-push via save topic |
| "Related entity does not exist" | Beneficiary race condition | Push beneficiaries first, then tasks |
| idgen errors unique to one date | Service outage that date | Push pipeline covers it |
| rowversion=1 all same day (DB-only) | Indexer failure that day | Targeted ES re-index |

### Phase 6 — Remediation order
1. Check parent entities (beneficiaries) are in DB
2. Push beneficiaries first if missing → `<schema>-save-project-beneficiary-topic`
3. Push tasks → `<schema>-save-project-task-topic`
4. Re-index DB-only records to ES
5. Re-run the comparison script to verify

---

## The 90,143-ghost-record RCA (Bauchi MR-DN + Sokoto SMC)

### Scale
- Total ghost records (ES not in DB): 90,143 across 5 entity types
- DB not in ES (missing index): 747 records (data safe, needs re-index)
- Investigation period: Jun 2025 – Nov 2025 (tracer window Aug 1 – Nov 14; data before Aug 1 purged)

| Entity | Ghost (ES not DB) | DB not ES |
|---|---|---|
| Bauchi ProjectTask | 57,594 | 311 |
| Sokoto ProjectTask | 18,526 | 311 |
| Bauchi HHM | 7,207 | 45 |
| Bauchi ProjectBeneficiary | 6,167 | 78 |
| Bauchi Household | 649 | 2 |
| TOTAL | 90,143 | 747 |

### Root causes (tracer-verified)
| Root cause | Count | % | Evidence |
|---|---|---|---|
| A — UPLOAD_ERROR_FROM_APP (mobile retry indicator) | 62,958 | 69.8% | egov-tracer dumpError() entries |
| B — Silent persister failure (no tracer entry) | 26,533 | 29.4% | Zero tracer entries for records |
| C — Race condition | 424 | 0.5% | NON_EXISTENT_RELATED_ENTITY in tracer |
| D — idgen outage Sep 21 | 34 | 0.04% | Connection Refused in tracer, Sep 21 only |

### Why UPLOAD_ERROR_FROM_APP is a retry indicator, not a cause
- NOT published by the server consumer checking hasErrors=true.
- Published by the mobile app's `dumpError()` method (Flutter `data_repository.dart`)
  on a retry attempt; mobile sets `"errorMessage": "UPLOAD_ERROR_FROM_APP"`.
- The string exists in NO Java/Digit-Core code — only Flutter. That search
  (Java→not found, Digit-Core→not found, Flutter→found) is how it was reclassified.
- So 69.8% is a retry indicator; the original ghost was created by Root Cause B or
  Redis bypass. Consumer silently skips hasErrors=true records via `notHavingErrors()`.

### Sep 21 spike is UPLOAD_ERROR_FROM_APP, not idgen
- Initial assumption: Sep 21 spike = idgen outage. Tracer verification of 5 random
  Sep 21 HHM records → all UPLOAD_ERROR_FROM_APP (CDDs retrying failed syncs).
- idgen outage caused only 34 confirmed ghosts across all entities.

### Redis cache bypass (Scenario A only creates ghosts) — proven live 2026-06-02
- Controller: `putInCache(beneficiaries)` runs BEFORE `producer.push(Kafka)` (ProjectApiController.java:115)
- Beneficiary lands in Redis but not yet in DB (within 60s TTL window)
- Task validator `PtProjectBeneficiaryIdValidator.validateIds()` → `findInCache()` → REDIS HIT
- Task marked valid without DB query → pushed to Topic B → transformer writes to ES
- Persister DB write fails; error published to tracer but KafkaConsumerErrorHandler
  not configured to catch/route → task NOT in DB
- Live proof log line: `"Cache hit, 1 items found"` in local project service
- Scenario B (cache MISS) does NOT create ghosts — rejected at service layer, counted under Race Condition (C)

### Redis TTL detail
- TTL = 60s hardcoded (GenericRepository.java:62). Set on the ENTIRE hash, not per
  field → later sync batches `putAll()` + `expire()` RESET the TTL for all records
  → during an active sync session the bypass window stays open beyond 60s.
- Redis expires when Kafka consumer lag > 60s (high load, pod restart, rebalance).

### batchUpdate = confirmed trigger for Root Cause B
- Persister uses JDBC batchUpdate for ALL records in a Kafka message regardless of
  `bulk.enabled=false`. 1 Kafka message = array of N records = 1 batchUpdate = 1 DB txn.
- 1 bad record → entire batch rolled back → all N dropped. Written to pod log +
  tracer, but KafkaConsumerErrorHandler misconfigured → no `egov-tracer-error-details`
  entry → invisible. `enable.auto.commit=true` is a platform-wide constraint → the
  offset is committed anyway → permanent loss. This is the trigger for the 26,533.

### 747 DB-not-in-ES records
- All rowVersion=1 → no second Kafka message → egov-indexer missed the initial message.
- egov-indexer has no tracer error publishing (CoreIndexMessageListener catches + logs only).
- Fix: re-index. Data is safe in DB.

### Live proof — Record 10 (Bauchi Sep 21)
```
Individual  e1663160  IN DB     API TotalCount: 1  (08:47:23 committed)
Beneficiary e16df990  NOT IN DB  API TotalCount: 0  (ghost — 08:50:46 synced)
Task        2da51750  NOT IN DB  (ghost)
Redis bypass window: beneficiary API 08:50:46 → task API ~08:50:48 = 2s, well within 60s TTL
```

### Investigation methodology (the reusable SOP)
1. DB/ES comparison scripts → ghost record IDs
2. ES detail files → slim files (clientReferenceId only)
3. Tracer proof script → classify each record A/B/C/D
4. API timestamp comparison → prove race condition timing
5. Live local service → prove Redis bypass in running code
6. DB verification → API TotalCount: 0 confirms ghost

### Key scripts (Downloads)
- `prove_race_condition.py` — API timestamp proof (calls beneficiary search for each ghost task's parent; positive gap = race)
- `prove_all_ghost_tracer.py` — tracer classification for all 90k (20 parallel threads)
- `BA_comparison_script.py` — DB vs ES comparison for Bauchi PT
- Slim files: `ba_PT_ids_only.csv`, `ba_HHM_ids_only.csv`, `ba_PB_ids_only.csv`, `so_PT_ids_only.csv`
- RCA doc: `C:\Users\Salaudeen N\Downloads\rca.docx`

### What "proven" means in this investigation
- "Confirmed" = verified by tracer search finding the exact record
- "Proved" = API call returning TotalCount confirming DB state
- "Inferred" = concluded from timing evidence (Redis bypass) without direct Redis-state proof
- "Assumed" = date-based classification (Sep 21 = idgen) — later disproved by tracer sampling

### Short-term code fixes proposed (no architecture change) — belong in hcm-dev, listed for reference
1. `PtProjectBeneficiaryIdValidator.java:70` — `validateIds()` → `validateClientReferenceIdsFromDB()`
2. `ValidatorUtils.java` — NON_RECOVERABLE → RECOVERABLE for parent-not-found
3. `KafkaConsumerErrorHandler.java` — add `super(new FixedBackOff(10000L, 3L))`
4. `DeadLetterQueueConfiguration.java` — uncomment (already written, commented out)
5. Cannot change `enable.auto.commit` — platform-wide
6. `PersistRepository.java:45` — publish error to tracer on DB write fail

Long-term correct fix: mobile-side wave-based sync with server confirmation
(eliminates ordering at source, like DHIS2). Not retry-with-backoff (too slow),
not uForwarder alone (doesn't fix ordering).

## Relational-gap triage: "is a big Gap REAL?" — the verification ladder
When a chain/relational gap shows a scary count (proven on ch/Chad: Gap E "148K project-beneficiaries with no project-task"), do NOT treat it as real/recoverable until the cheap explanations are disproved IN THIS ORDER, each by a direct query against source (never trust the tool's own count):

1. **Is the fixed code even running?** Check the log format or a KNOWN false positive. On ch, the household `e720b930` (exists in ES, created outside the window) kept appearing in Gap A and the log lacked the pass-2 "confirmed/dropped" lines → the run was an OLD copy. (The edit tree `D:\DST\scripts\recon-recovery` is NOT the runtime `/home/jovyan/dst/Data Recovery Script/v2/` — deploy first.)
2. **Sample 10 flagged records; check the OTHER store UNWINDOWED.** ES: `terms` filter + `terms`-agg on the id field, no date filter. Any that come back present = **date-window/lag artifact** (the counterpart was just outside the audited window). On ch this was ~2/10.
3. **Join-key check.** LEFT JOIN the two tables on BOTH candidate keys (client-ref AND server id) and count which link column is NULL (`crid_link_null` vs `serverid_link_null`). If the tool joins on the key that's NULL → false gap = join bug. ch result: all 830,128 tasks have `projectbeneficiaryclientreferenceid`, `projectbeneficiaryid` is NULL on ALL of them → the chain's crid join was correct (not a bug).
4. **Duplicate/twin check.** Group task-less parents by shared beneficiary — is the link satisfied under a DIFFERENT record? ch: only 5 of 173,952 → not duplicates.
5. **Quantify at scale — don't extrapolate from 10.** Campaign-wide `NOT EXISTS` anti-join by the correct key. ch: pb_total 501,274; dosed 327,695; task-less 173,947.
6. **Once confirmed real, find WHY: profile flagged vs known-good on a discriminating attribute** (status / age / DOB / **role**). Compare the "bad" set to the "good" set. First cut on ch: un-dosed = 98% placeholder DOB (`dateofbirth` NULL or `1970-01-01`/epoch-0), dosed = real 3-59mo ages. **Then the sharper cut (a colleague's field insight — always solicit these): the task-less PBs are HOUSEHOLD HEADS.** Cross-tab dosed × `householdMember.isHeadOfHousehold`: un-dosed **182,180/186,170 = 97.8% heads**; dosed **333,862/351,859 = 95% non-head members**. The head is the adult respondent registered per household, NOT a dosing target → no task BY DESIGN (placeholder DOB was just the symptom — heads' DOB isn't collected). Real residual = ~2,461 un-dosed NON-head members. Lesson: keep cutting until the "bad" population has a crisp, benign identity — "placeholder DOB" was true but "household heads" is the actual reason.
7. **Anchor to the domain assumption.** For SMC/MRDN, registration + dosing happen in the SAME visit, so "38% registered-but-untreated" is implausible for REAL children — that contradiction is the thread to pull. It resolved cleanly: register=dose HELD for the children; the gap was household heads (+ placeholder-DOB shells), benign and non-recoverable. Placeholder-DOB signature (NULL / `1970-01-01` / epoch-0 = the app's "not entered" default) is how we first SPOTTED it, but it is NOT the criterion. FIX: chain Gap E excludes head beneficiaries ONLY — `isHeadOfHousehold==true` from the member load, matched on the PB beneficiary. **DOB was rejected as the filter** (it's a symptom; keying on it would wrongly drop a real eligible child with an unentered DOB). Lesson: filter on the DIRECT semantic flag (isHeadOfHousehold = "not a dosing target"), never on a correlated symptom (DOB).

FIX shipped to chain.py: pass-2 unwindowed existence confirm (drops window/lag artifacts) + head-only Gap E filter (isHeadOfHousehold, NOT DOB) → Gap E ~148K collapses to the ~2.5K genuinely-un-dosed. See [[project_recon_recovery_tool]] for the code detail. Other gotchas: ES indices can transiently 404 during rotation (retry); run recon on a SETTLED window (≥1 day old) or lag inflates "index-failed".

## Error-tracer pattern triage (one aggregation query, any tenant)
To see WHAT is failing and HOW MUCH for a tenant/date range, aggregate the tracer index in one call. `egov-tracer-error-details` is SHARED across tenants (no `<t>-` prefix) → scope by `match_phrase` on the request body `"tenantId":"<t>"`; date field is `Data.@timestamp`.
```
GET egov-tracer-error-details/_search
{ "size":0, "track_total_hits":true,
  "query":{"bool":{"must":[
    {"match_phrase":{"Data.apiDetails.requestBody":"\"tenantId\":\"ch\""}},
    {"range":{"Data.@timestamp":{"gte":"2026-07-20","lte":"now"}}}]}},
  "aggs":{
    "by_error_code":{"terms":{"field":"Data.errors.errorCode.keyword","size":50}},
    "by_api_url":{"terms":{"field":"Data.apiDetails.url.keyword","size":50}},
    "by_day":{"date_histogram":{"field":"Data.@timestamp","calendar_interval":"day"}},
    "processed_vs_not":{"terms":{"field":"Data.isProcessed","size":2}}}}
```
Total = `hits.total.value`. **Cross-read `by_error_code` against `by_api_url`** — when one error code's count equals one url's count, you've pinpointed a single root cause. Error-code buckets usually sum to LESS than the total (~10% of docs carry no parsed `errorCode`) — report that gap honestly.
Error-code catalog (meanings): `INVALID_ID` = idgen rejected the beneficiary id-pool update (ID-generation failure); `DUPLICATE_ENTITY` = record already exists, offline retry (usually benign); `NO_PROJECT_FACILITY_MAPPING_EXISTS` = facility not mapped to project (config gap); `MISMATCHED_ROW_VERSION` = concurrent/stale update; `NON_EXISTENT_ENTITY`/`_RELATED_ENTITY`/`_SUB_ENTITY` = sync-ordering / parent-not-found; `INDIVIDUAL_ALREADY_MEMBER_OF_HOUSEHOLD` / `HOUSEHOLD_ALREADY_HAS_HEAD` = duplicate membership; `INVALID_HOUSEHOLD` / `INDIVIDUAL_NOT_FOUND` = bad/missing reference; `QUERY_EXECUTION_ERROR` = DB error; `NETWORK_ERROR` = transient.
**ch example (2026-07-20..23):** 271,726 errors; **81% = `INVALID_ID`, ALL on `/beneficiary-idgen/id/id_pool/_update`** (the idgen id-pool failure — one root cause, not many), 7% `DUPLICATE_ENTITY` (benign retries), rest tiny; rising daily (59k→75k→99k). idgen is a PLATFORM issue, not data-entry.
