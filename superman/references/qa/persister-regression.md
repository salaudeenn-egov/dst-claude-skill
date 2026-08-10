# Persister reliability & bulk-sync regression (unified-dev)

How to regression-test the egov-persister + HCM entity APIs so "nothing that worked is broken",
and the edge cases that are NOT handled. Backs two RCAs: Kafka-lag/DB-persistence (throughput,
batch, DLQ, ON CONFLICT) and Sokoto/Bauchi data-difference (silent batch rollback / ghost records).
Deliverable format: Excel (Test Cases + Summary + Findings sheets), color-coded, with an Evidence column.

## Method (senior-QA, honest)
1. **Search each entity first, clone the real record** (fresh clientReferenceId, keep real reference
   fields) for happy cases; mutate the clone for sad/exception. Real refs = payloads never break on bad data.
2. **Two signals, reported separately:** synchronous API contract (HTTP + errorCode) is the reliable
   "is it broken" verdict; async persistence is env-dependent. Verify persistence by `_search` on
   clientReferenceId AFTER the 60s Redis TTL (see [verification-methodology]). id=None/empty = not a real write.
3. **Always run a control** (e.g. N unique records → N/N) so a "0 persisted" is attributable to the
   test, not a down write-path.
4. Classify each: handled (clean 400 reject / correct persist / idempotent) vs NOT-HANDLED
   (good data lost, invalid data persisted, silent drop, crash, duplicate rows). Exclude env/egov-user
   failures as non-defects.

## Working create payloads — dev (real refs that persist)
tenant `dev`; projectId `79061c3c-4ef3-411f-88a8-81053fa92034`; locality `NEWTEST00222_MO_11_06_05_14_03_HOSPITAL_CAMP_CAMP_5_VILLAGE`.
Real reference values (searchable/committed on dev):
- household `e14e9090-2f10-11f0-9f41-cf1777c319ba`; individual clientRef `2bd94da0-7e6a-11f0-8463-13a3b25488a3` (not findable by /health-individual search — use cache path)
- committed beneficiary `e3e6a090-...` (PTB-2025-05-12-000554); task's PB `cf2f78e0-...`; real task `d8d6a210-...`
- **productVariantId `PVAR-2025-01-03-000143`** (REQUIRED for task resources); **facility recipient `F-2025-05-15-013589`** (referral recipientType=FACILITY)
- Endpoints: household `/household/v1`, member `/household/member/v1`, individual `/health-individual/v1`, beneficiary `/health-project/beneficiary/v1`, task `/health-project/task/v1`, referral `/referralmanagement/v1`, side-effect `/referralmanagement/side-effect/v1`, stock `/stock/v1`.
Recipe per entity: **Household/Individual** no parent. **Member/Beneficiary** validate parent against Redis cache -> create the individual/household first, then the child within 60s (cache path). **Task/Referral/SideEffect** validate parent against COMMITTED DB -> parent must persist first. Dependency order: individual -> household -> member -> beneficiary -> (commit) -> task/referral/side-effect.
Entity-specific must-haves: task needs `address` + valid `status` enum + `resources[]` with a REAL productVariantId; referral needs a REAL recipientId matching recipientType; side-effect needs committed task+beneficiary; individual familyName 2-200 chars, dateOfBirth dd/MM/yyyy, valid mobileNumber; first household member must be isHeadOfHousehold=true. See full payloads: memory `hcm_working_payloads_dev.md`.

## Persister behaviours — VERIFIED working
- **Batch insert:** bulk create = one batch insert; log `Persisted N row(s) to DB`.
- **Redelivery idempotency:** produce the same persister message twice -> unique-constraint hit -> caught -> log `already present (duplicate) - idempotent success`; 1 row, no DLQ growth. (App-level catch, NOT SQL `ON CONFLICT` on this build.)
- **DLQ retry->exhaust:** `egov-persister-deadletter` holds a failed record with attempts 1..4; on exhaust it moves to `egov-persister-deadletter-processed` (attempt 5). Proven from real data.
- **Two error paths:** SERVICE-layer validation failures -> `error-details-indexer-topic` (diverted before persister, with full apiDetails+stack). PERSISTER-layer failures -> `egov-persister-deadletter`.
- **Idempotency/concurrency:** same clientReferenceId across two calls OR two parallel bulk creates -> 1 row (no duplicate).
- Auth rejects invalid/empty token (401); search edges graceful (empty/mixed/neg-offset->400/huge-limit->400); oversized ~5MB payload -> 413; XSS stored as literal (safe); wrong JSON types -> clean 400; stale rowVersion -> no lost-update.

## Edge cases NOT handled (evidence-backed — baseline 2026-07-08; see FIX-VALIDATION below for what's since fixed)
Data loss (good records dropped):
- **In-batch duplicate clientReferenceId** in one bulk call -> whole batch not persisted (unrelated good records lost too). Control-proven (5 unique->5/5 vs [X,X,U1,U2,U3]->0/5). = RCA batchUpdate-rollback / Root Cause B. Same at resource level (two task resources same clientRef -> task lost). **RECORD-LEVEL now FIXED (2026-07-10 retest) — resource-level (EDGE-2) still open.**
- **Household bulk `_update` = silent no-op** (202 but field unchanged + rowVersion NOT incremented; confirmed 3x). **HOUSEHOLD-SPECIFIC** — Task/Referral/SideEffect updates DO apply (rowVersion 1->2 verified). Household single `_update` -> 400 "An unhandled exception occurred on the server".
- Mixed-tenant bulk batch (one dev + one bogus-tenant record) -> 401, valid dev record also lost.
Invalid data accepted+persisted (read-back confirmed): negative resource quantity (-9), negative memberCount (-5), quantity=0, quantity=2147483648 (int overflow), memberCount=0, plannedStartDate>plannedEndDate. No lower/upper-bound or date-order validation.
Other: whitespace-only clientReferenceId ("   ") persists though empty-string "" is rejected 400 (inconsistent); task empty `resources:[]` -> 202 silent drop; malformed JSON -> HTTP 500 (should be 400); cross-tenant record (tenantId!=token) accepted 202 (verify superuser intent); batch ~2000 -> `EVENT_BUS_FAILURE` 400 (Kafka message-size cap; 1000 works — chunk large batches); `resources:[null]` accepted 202; float quantity 1.5 stored.

## FIX-VALIDATION RETEST — full 76-TC suite re-run (2026-07-10, fixed branch deployed)
Re-ran the whole suite as <REDACTED-USER> (role DISTRIBUTOR) against unified-dev to validate the deployed persister fix. Tally: 59 PASS / 8 PASS-with-note / 1 IMPROVED / 1 FINDING / 6 ENV-blocked(Stock) / 10 RBAC-blocked / 1 gateway-N/A. Result doc: `Downloads\Persister-Bulk-Regression-Retest-2026-07-10.xlsx` (Post-Fix columns filled per TC).
- **RECORD-LEVEL batch isolation = FIXED (the genuine finding is resolved).** EDGE-1 `[X,X,U1,U2,U3]` -> 3 uniques persist (was 0/5). In-batch dup pair -> 1 persists (was 0/2). Poison batch (5 good + 1 null-clientRef) -> 5/5 survive on Household, HouseholdMember AND ProjectBeneficiary. All-bad batch -> 0, no crash. Redelivery -> idempotent 1. Peak x100 -> 100/100.
- **EDGE-2 STILL A FINDING**: duplicate resource clientReferenceId WITHIN one task -> whole task not persisted (0/1). Record-level fix did NOT cover resource-level dedup inside a task. Report to dev.
- **IMPROVED**: orphan task (null projectBeneficiary) now SYNC-rejected 400 (baseline accepted 202 as a non-deterministic orphan).
- **BEHAVIOR-CHANGE note (data integrity intact, but flag to dev)**: bad-ref/invalid SINGLE creates (bad productVariantId/PB/recipient/project, missing address, out-of-order member) now return **202 async-drop + 0 persisted** instead of baseline **400 sync-reject**. No ghost rows, but the API no longer 4xx's synchronously.
- Individual create (baseline EXCLUDED for USER_SERVICE_ERROR) now PASSES. Stock still env-blocked. StockReconciliation + HFReferral = RBAC 401 for DISTRIBUTOR — need a higher-privilege account; do NOT auto-escalate to a superuser account (auto-mode classifier correctly blocks an unauthorized superuser login).
- **Harness gotchas** (scripts: scratchpad `persister_regression.py` two-phase submit->100s->verify; `supplement.py` fresh-parent Member/PB; `final_report.py` writes the xlsx): search `_search` REQUIRES `limit`+`offset` URL params or it errors `NotNull.URLParams.offset` and looks like 0-persist (false negative). Clone-real-record reuses the parent ref -> PB bulk shares one beneficiary (duplicate-per-project reject) and Member bulk reuses a household that already has a head (VALIDATION_ERROR) -> BOTH are TEST-DATA artifacts, not persister defects; mint fresh distinct parents (single create populates cache) per record. urllib rapid-fire can transiently drop the connection -> add a retry + re-auth-on-401 in the POST helper.

## Dev test-data creation — POLIO project / "Gedero Koniso" (2026-07-09/10)
Reusable full-chain synthetic dataset on tenant `dev`, distinct from the SMS_CONFIG payloads above.
- auth **<REDACTED-USER> / `<REDACTED — dev password>` / EMPLOYEE** (uuid c8ec8ce6-521b-4e0c-95db-97c57dc47a62, role DISTRIBUTOR). Token is SINGLE-SESSION: a fresh login invalidates the prior token (downstream `/user/_details` -> InvalidAccessTokenException); re-auth before each run.
- project **`ad5d07be-62cd-4d65-8fc7-b8952a0a1b54`** = "POLIO 10th June C1" (projectType POLIO/multiround). Its productVariant **`PVAR-2026-06-10-000214`** (from projectType additionalDetails doseCriteria) is the valid one for task resources on THIS project.
- boundary "Gedero Koniso" localized name -> locality code **`NEWTEST00222_MO_11_04_01_01_02_GEDERO_KONUSO`** (code spells KONUSO). Task address carries `locality.code`; boundary/boundaryType left null; lat/lon ~12.9288,77.6279.
- Built 40 full families here (30 on 07-09 + 10 on 07-10): each = individual + household + head member + beneficiary (**registration**) + one delivery task ADMINISTRATION_SUCCESS with that PVAR (**distribution**). All DB-verified. Scripts: scratchpad `make30.py` / `make10.py` (two-phase: register all, wait 90s for commit, then distribute — task validator checks committed DB so beneficiary must commit first).
- GOTCHA: `/health-project/beneficiary/v1/_search` can return 0 for a single clientRef even when committed (dev quirk) but returns them when the whole clientRef array is passed; prove PB commit indirectly (a task referencing it passes the committed-DB validator).

## Offline downsync verification — `/referralmanagement/beneficiary-downsync/v1/_get`
Confirms the offline-sync path returns registration + distribution for a boundary. Body `DownsyncCriteria`: `locality`, `projectId`, `tenantId`, `lastSyncedTime`, `offset`, `limit`, `includeDeleted`. Response `Downsync` buckets: Households / HouseholdMembers / Individuals / ProjectBeneficiaries / Tasks / Referrals / SideEffects.
- GOTCHAS that silently return EMPTY (`Households:[]`, status still `successful`): (1) `locality` must EXACTLY match — a **leading/trailing space** in the code -> 0 rows; (2) `projectId` must be the project the data was created under (POLIO `ad5d07be...`, not SMS_CONFIG `79061c3c...`) -> wrong project = 0 rows. Verified: corrected payload returned 40 households, 49 members/individuals/beneficiaries, 31 tasks for GEDERO_KONUSO + `ad5d07be...`.

## Kafka / cluster access (read-only)
- Kafka CLI: `kubectl exec -n kafka-kraft release-name-kafka-controller-0 -c kafka -- bash -lc "/opt/bitnami/kafka/bin/<tool>.sh --bootstrap-server localhost:9092 ..."` (kafka-topics / kafka-consumer-groups / kafka-get-offsets / kafka-console-consumer).
- Persister consumer group = `egov-infra-persist`; healthy = LAG 0. `egov-persister` (ns egov) is the shared persister for HCM save-* topics; when it flaps (`No resolvable bootstrap urls` / `Node -1 disconnected`), writes 202 but don't persist. Pod is distroless (no shell); HCM DB is external (not reachable from cluster) so DB-param verification isn't possible on dev.

## NOT testable on unified-dev (needs perf-lab / prod-spec DB)
Throughput/lag/stress (2-3k -> 20k/s), pgbench, PostgreSQL parameter tuning, DB-down recovery (DbHealthMonitor), multi-instance rebalancing. Individual/Stock create intermittently blocked by egov-user (USER_SERVICE_ERROR / USER_SEARCH_ERROR) — environment, exclude as non-defect. Stock create also needs a project-facility mapping (absent on dev).
Scripts + Excel builder live in the session scratchpad; results doc: `Downloads\Persister-Bulk-Regression-Results.xlsx`.
