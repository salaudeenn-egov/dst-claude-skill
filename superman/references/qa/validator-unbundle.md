# Validator-Unbundle Testing (cross-entity existence validator unbundling)

QA of the cross-entity EXISTENCE validator unbundling — the code-side mitigation for the persister-lag / Redis-cache-bypass ghost-records issue. This file is the test-side record; the code fix itself belongs to hcm-dev.

## Fix under test
Per-service boolean feature flags disable the persister-lag-prone cross-entity existence checks, so a child whose parent is still queued PERSISTS instead of being hard-rejected. Structural / uniqueness / self-existence / MDMS / enum checks and id back-fill are PRESERVED. Flags default OFF in this build (validators disabled) for the unified-dev rollout test.
- Branch: `master-nigeria-finalpull-validator-unbundle`. Env: `https://unified-dev.digit.org`, tenant `dev`.
- Clean fix evidence signal: `hasErrors` flips true (pre-fix, rejected) -> false (post-fix, persisted).

## Feature flags (default OFF = validation removed)
- household: `household.member.relationship.validation`
- health-project: `project.relationship.validation.enabled`
- referralmanagement: `referralmanagement.relationship.validation`

F2 (flag ON re-rejects) requires setting the 3 keys true + a rolling restart (config change).

## Auth & environment facts
- Auth: `<REDACTED-USER — dev superuser account>` / `<REDACTED — dev password>` / tenantId `dev` / userType EMPLOYEE at `POST /user/oauth/token` (`Authorization: Basic <REDACTED — OAuth client header>`).
- oauth 500 "Unknown error occurred in encryption process" = egov-enc-service pod not ready (NOT a bad credential); check `kubectl get pods -n egov | grep enc-service`.
- Cluster: kubectl context `unified-dev`; user/enc/persister in ns `egov`, HCM services in ns `health`. `/v1/_search` reads DB — use it as the persistence (DB) check.
- HH member search wrapper key = `HouseholdMember` (NOT HouseholdMemberSearch); field `clientReferenceId` is a list.
- ENDPOINT: individual create/search MUST use `/health-individual/v1/...` (HCM service), NOT `/individual/v1/...` (dev routes that to the crash-looping egov CORE service, different schema).
- Dev churn (waves of Pending pods, TLS handshake, 401 authz blips, persister rebalancing) makes green windows short — wait for all egov/health pods 1/1, persister not rebalancing, enc-service + accesscontrol ready before a verification suite.
- Windows gotcha: bash-tool `/tmp` != Windows-python `/tmp`; pipe curl -> `python -c` via stdin, don't write /tmp then read in python.
- Real wrapper keys: HouseholdMember / Task / ProjectBeneficiary / Referral / SideEffect. Real Referral model fields: `recipientType` (STAFF/FACILITY) + `recipientId` + `referrerId` + `projectBeneficiaryId/ClientReferenceId` + `reasons` + optional `sideEffect`. There is NO `taskId` on the referral model (task is on the side-effect). Project-task REQUIRES an `address` object (else ProjectTaskEnrichmentService NPEs). name.familyName must be 2-200 chars. RequestInfo.apiId is REQUIRED.

## Test plan (26 cases)
Plan doc: `C:\Users\Salaudeen N\Downloads\HCM-Validator-Unbundle-Test-Plan.docx`.
- U1-U9 — persist (child with bogus/non-existent parent should be accepted + persist)
- B1-B4 — back-fill preserved
- S1-S6 — regression (kept-ON validators must still reject)
- D1-D4 — downsync
- F1-F2 — flag toggle
- E1-E3 — end-to-end RCA (DB-vs-ES parity)

## Harnesses
Python harness: `C:\Users\Salaudeen N\Downloads\HCM-Validator-Test\`
- `run_tests.py` — clones a known-good payload, points parent at a random UUID, fires single (sync) + optional `--bulk` (async), checks response codes; optional ES tracer + Postgres row verification via config; writes `results.md` sign-off table.
- `config.example.json` (copy to config.json; needs token/tenant/user uuid), `base_payloads/*.json` (5 entities; replace REPLACE_* with real existing ids).
- Automates U1-U9, S1-S6, F1. Manual: B/D/E series and F2 (needs flag ON + restart). Bulk is async -> a PASS needs tracer/DB confirmation.

Postman collection: `HCM-Validator-Unbundle-Tests.postman_collection.json` (built by `build_postman.py` from the user's `Downloads\HCM Collection.postman_collection.json`). 27 requests in 4 folders (U single, U bulk, S regression, F toggle). Assertions in a collection-level test event; each request pre-request sets tc_id/tc_expect/tc_codes. Parents swapped to `{{$guid}}`; good parents are `{{good_*}}` collection vars the user fills once (+ authToken/tenantId/userUuid).

Scratchpad scripts: `u_series.py` (auth-retry-hardened, accepts TC-id args e.g. `python u_series.py U4 U5`), `verify_all.py` (searches each U clientRef, reports IN DB / MISSING by TotalCount), `bs_series.py`, `reverify_all.py`, `build_repro_doc.py`, `phase2.py`, `link_null.py`, and the null-sweep set (see below).

## Results by series

### U-series (persist) — ALL PASS on fix behaviour
Child with bogus parent ACCEPTED (202), never rejected with an existence error. DB-COMMITTED post-TTL: U1 (bulk 66a338a1 + single e1074955), U2 (3765f504), U3 (cacf46cf) members; U6 (f3b0413f), U7 (195326ec) beneficiaries; U8 (a9e2472f, corrected fields) referral; U9 (5c4dc7f1) side-effect.
- U8 referral = PASS with CORRECT field names (earlier "referral NPE bug" was a stale-field-name test-data mistake — see verification-methodology Rule 4). Minor genuine nit: service throws a raw NPE on null recipientType instead of a clean validation error.
- U4/U5 (project-task): accepted (202, no existence rejection = fix behaviour correct) but did NOT persist — the earlier "persists" was a Redis-cache false positive (id=None). Real cause = task payload had no `address` -> enrichment NPE. With address, task persists. U4/U5 also need Task.status = valid enum `ADMINISTRATION_SUCCESS` (NOT "ADMINISTERED"/"ADMINISTERED_SUCCESS") else the always-ON structural NotNull fires (correct behaviour).

### S-series (regression, kept-ON validators) — ALL PASS
Use SINGLE `_create` for S-series (synchronous 400 errors; bulk defers).
- S1 self-relationship rejected (VALIDATION_ERROR).
- S2 duplicate clientReferenceId (1st 202, 2nd 400) — uniqueness ON (timing-flaky across calls due to persister lag).
- S3 unknown relationshipType rejected — MDMS ON.
- S4 originally "reject" then CHANGED to 202 ACCEPTED: task with both projectBeneficiary ids null + valid address is accepted — earlier "reject" was the missing-address NPE, not a null-id check. With flag OFF this reference check is effectively disabled. (Sometimes returns raw NPE 500 instead of clean INVALID_RELATED_ENTITY_ID = a graceful-handling gap.)
- S5 bad recipientType -> clean "Invalid Recipient Type" (enum ON).
- S6 `_update` non-existent id rejected (self-existence ON).
So structural/uniqueness/MDMS/enum/self checks all still enforced with the flag OFF.

### B-series (back-fill) — B1/B2 PASS (DB-verified)
Test back-fill via the synchronous single `_create` RESPONSE (enrichment-time, lag-independent), referencing an EXISTING parent (dev has ~568 households; valid locality `NEWTEST00222_MO_11_06_05_14_03_HOSPITAL_CAMP_CAMP_5_VILLAGE`).
- B1 = PASS for BOTH householdId AND individualId back-fill: member referencing an existing household + a committed /health-individual (clientRef 7d33ee80, id 822e09c1) back-filled individualId = 822e09c1; and householdId back-fills (e.g. H-2026-07-05-010160). Earlier "B1 inconclusive" was purely individuals not persisting during egov-user flap / wrong /individual/ endpoint — NOT a back-fill bug.
- B3 covered by U1 (householdId null, no crash); B4 covered by U6. B2 (relativeId, relative in same batch) still NOT verified.

### F-series (toggle)
- F1 = PASS (flag OFF persists = the U-series).
- F2 (flag ON re-rejects) NOT run — needs the 3 keys set true + rolling restart.

### D-series (downsync) — BLOCKED
Endpoint `POST /referralmanagement/beneficiary-downsync/v1/_get`, body `{RequestInfo, DownsyncCriteria:{locality, projectId, tenantId, lastSyncedTime, offset, limit, includeDeleted}}` (needs BOTH locality AND projectId). Call reaches the service but ALWAYS fails `UNKNOWN_ERROR: Unknown error occurred in decryption process` on individual PII — root cause: egov-enc-service throws "Key not found in the database" = a MISSING ENCRYPTION KEY in the dev enc-service DB (dev key-seeding gap), even with enc-service 1/1 READY. NOT the fix. Code-verified only (DownsyncService.java walks the chain by clientReferenceId, so null-server-id records aren't dropped). No successful downsync observed -> D1-D4 BLOCKED. To close: restore/seed the enc key in dev, then re-run.

### E-series & F2 — removed from scope
F2 (confirmatory toggle only) and E1-E3 (DB-vs-ES parity, an RCA-outcome test out of this fix's direct scope, and blocked by the enc-key issue) were removed from the results doc. Only D1-D4 remains in-scope-but-blocked.

### Field simulation / prod-vs-dev
Simulated CDD end-of-day batch (10 households, registration+distribution) via BULK APIs in DataModelType order, flag OFF. With corrected payloads (task with address, /health-individual endpoint): the full CDD chain (household -> member -> individual -> beneficiary -> task) persists with flag OFF; hasErrors flips true->false = clean fix evidence. The historical "Individuals 0/10" and "dev missing hashedmobilenumber column" were the WRONG /individual/ endpoint (crash-looping core service) — retracted, not action items. No prod data pushed to dev; test data is synthetic (random UUIDs, 'Field/Person' names, NEWTEST00222 locality, project 79061c3c). CDD user <REDACTED-USER>/<REDACTED>, project-staff PTS-2026-07-05-014887.

## Null-relation-link sweep (2026-07-07, DB-verified post-90s)
Injected null into each of the 5 relation join fields in isolation. The service ACCEPTS null in ALL of them (202, no validation error) — relation integrity is NOT enforced at validation. Downstream persist differs:
- member.householdClientReferenceId = null -> 202, DROPPED (in_db=0).
- member.individualClientReferenceId = null -> 202, DROPPED.
- beneficiary.beneficiaryClientReferenceId = null -> 202, DROPPED. (CHANGED from 2026-07-06 where it persisted as L2 — doc line now STALE.)
- referral null parent link -> 202, DROPPED. (CHANGED from earlier O4 "persists" — doc D4 line now STALE.)
- task.projectBeneficiaryClientReferenceId = null -> 202, persists as durable ORPHAN but NON-DETERMINISTIC: one clean run left durable orphan `PT-2026-07-07-001622` (pbClientRef+pbId null, still in DB), two identical later runs returned 202 and vanished. Accepted every time; outcome = coin-flip orphan-vs-lost.
- side_effect.projectBeneficiaryClientReferenceId = null -> 202, persists as ORPHAN, DETERMINISTIC 3/3 (ids 1e053900 / 30165b92 / 840d796a, all pbClientRef+pbId null). More reliably broken than the task.
- NotNull-guarded fields correctly REJECT 400: task.projectId, side_effect.taskClientReferenceId, task.resource.productVariantId. task.address=null / task.resources=null -> 202 then dropped (NPE path).
- Poison-batch = FIXED (3+ runs): 1 null-own-clientRef member + 3 good in a bulk -> the 3 good now persist 3/3 (previously the bad one aborted the whole batch = RCA Root Cause B, data-loss path now closed).
- Reco to dev: enforce linking clientReferenceId as required at the service layer — else silent data loss (member/beneficiary/referral) or untraceable orphans (task/side-effect).

Null-sweep scratchpad scripts: null_sweep.py, relations_check.py, relations_recheck.py, ind_isolate.py, ind_byid.py, task_recheck.py, relation_nulls.py, d2_clean.py, d3_clean.py, se_null_clean.py, read_docx.py.

## The 5 reconciliation relations link correctly (clientRefs present, DB-verified)
- R1 member.householdClientReferenceId -> household.clientReferenceId (householdId back-fills; H-2026-07-07-010211).
- R2 reverse household.clientReferenceId -> member (reverse search works).
- R3 member.individualClientReferenceId -> individual.clientReferenceId (individualId back-fills to a real server id; 6c4497af).
- R4 beneficiary.beneficiaryClientReferenceId -> individual.clientReferenceId.
- R5 beneficiary.clientReferenceId -> task.projectBeneficiaryClientReferenceId (reverse search works).
- DEV QUIRK (not the fix): `/health-individual/v1/_search` returns 0 for a resolvable individual by clientReferenceId/id/individualId even though member.individualId back-fills to a real id. Individual search/indexing is unreliable in dev — don't treat "individual _search empty" as non-persist; check the member back-fill instead.

Chain join keys (for reconciliation): task.projectBeneficiaryClientReferenceId -> beneficiary.clientReferenceId; beneficiary.beneficiaryClientReferenceId -> individual.clientReferenceId; member.individualClientReferenceId -> individual; member.householdClientReferenceId -> household.

## Persister batch/duplicate test (unified-dev) — PENDING
User picked: batch-vs-single + duplicate handling.
- egov-persister on unified-dev is in CrashLoopBackOff (~4h, both replicas): `ConfigException: No resolvable bootstrap urls given in bootstrap.servers` at `PersisterConsumerConfig.startContainer:184` (env/DNS misconfig, unrelated to RCA tuning).
- Yet HCM records persist fine on unified-dev (household/member/beneficiary/task DB-verified) -> HCM entity persistence does NOT go through egov-persister on this build; the health services persist via their own path (no persister pod in ns `health`). Confirm the actual HCM persistence mechanism before designing the persister test.
- Test plan (API-level, DB-verified evidence): P1 batch no-loss (bulk N members -> all in DB); P2 poison record in batch (already: good survive); P3 duplicate handling — create X, then bulk [X-dup, Y, Z] (do Y/Z persist, is batch aborted, is dup absorbed via ON CONFLICT DO NOTHING vs service-uniqueness-rejected 400?) plus in-batch dup [W, W, V]. Two layers: service-layer uniqueness validation (400) vs persister-layer ON CONFLICT DO NOTHING.
- Constraints: synthetic API creates OK; do NOT push prod data to dev; do NOT modify cluster config (kubectl logs/get read-only fine).

### Related RCA context (Kafka consumer lag & DB persistence — on-prem PROD, doc `Downloads\RCA Kafka Lags Database Persistence.docx`)
Incident on Salama Rancher on-prem PROD (NOT dev). Persister consumes 101 topics -> Postgres; lag peaked 64 Lakhs (6.4M) msgs; insertion capped 2-3k/s vs 20k/s target; duplicate-key exceptions. Probable root cause (pending infra validation): PostgreSQL params not tuned for scale (shared_buffers 128MB->8GB, effective_cache_size 4GB->24GB, work_mem 4MB->64-128MB, wal_buffers ->64-128MB, max_wal_size 1GB->8GB, effective_io_concurrency 1->200, random_page_cost 4->1.1). Fixes done: batch processing, ON CONFLICT DO NOTHING, 20-pod scaling, read/write DB split, DLQ + dynamic batch/single switching + secondary error queue, removed unused topics/indexes. TODO: apply PG tuning, enable pg_stat_statements, pgbench. (Context for the persister test's duplicate-handling behaviour.)

## Results doc
`C:\Users\Salaudeen N\Downloads\HCM-Validator-Unbundle-Test-Results.docx` (CDD-framed; all cases pass/fail + blocked ones + needs). The plan doc `HCM-Validator-Unbundle-Test-Plan.docx` has the per-case execution log. Repro-doc builder `scratchpad\build_repro_doc.py`.
- Doc style: state ONCE at the top that testing is done with the cross-entity existence validation REMOVED (disabled); do NOT repeat "flag OFF / flag ON" per row; mention the flag ONLY in the toggle test; describe behaviour directly ("child with missing parent persists", "still rejected").
- STANDING INSTRUCTION: do NOT update the results docx until the user explicitly says "update". The doc is frequently open in Word (save -> PermissionError); ask the user to close it before updating.

## Deliverables (Downloads)
- HCM-Null-Relation-Curls.txt — 5 null-relation-link curls (fresh UUIDs, create+verify).
- HCM-Dummy-Data-AllIndexes.txt — valid fully-linked dummy record per index (create in order 1..7).

## mhbase run (2026-08-19) - unbundled AND orphans PERSIST (worse than UAT)
Adapted harness: scratchpad mhbase/test_unbundle_mhbase.py (built by line-patching the UAT script;
live deployment probes; stock sample fetched live). Tally 20 pass-class / 1 FINDING.
- **Key difference vs mc-nigeria-uat:** ghost-parent creates are accepted AND LAND IN DB - member
  w/ ghost individual, task w/ ghost PROJECT, ghost productVariant, ghost beneficiary, referral w/
  ghost PB, side-effect w/ ghost task all persisted 1/1 as orphans. On UAT they were dropped
  downstream. This build actively manufactures ghost-record divergence (the SMC ES-vs-DB pattern).
- Always-on validators all hold: RequiredLink null-PB 400, delivered-task-empty-resources 400,
  update-non-existent 400, dup-across-calls 202/400 + 1 row, missing tenantId 400, in-batch dedup
  [X,X,U1,U2] -> 3 distinct (household + task).
- FINDING: household _delete no-op reproduced (UNB-003).
- Harness artifact to remember: member cases with isHeadOfHousehold=false on empty households show
  0-persist because of the HEAD RULE, not existence validation (ghost member with head=true persists).
