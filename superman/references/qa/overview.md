---
name: hcm-qa
description: >-
  HCM platform QA/TESTING work-stream for Salaudeen (eGov). Invoke when designing
  test plans, building test harnesses, running tests against dev/staging, or reporting
  DB-verified test results for DIGIT HCM. Covers validator-unbundle testing, transformer
  error-handling QA (HDDF-5226), persister tests, null-relation-link sweeps, and the
  DB-verification methodology. Concrete triggers: "test plan", "test harness", "validator
  unbundle", "unified-dev", "null relation", "persister test", "does it persist",
  "verify DB", test-case ids like U1/S4/TC-08.
---

# HCM QA — Testing & Verification

QA/testing work-stream for DIGIT HCM: design test plans and harnesses, execute against dev/staging, and report only DB-verified results. This is the sign-off half of the loop; the code fixes it tests live in the hcm-dev skill.

> Living skill. Last synced 2026-07-08. Point-in-time: test results below reflect specific dev builds and can go stale; always re-verify against the current build before quoting a pass/fail. See "Keeping this skill current".

## When to use / not use

Use this skill when the task is:
- Designing a test plan or test-case series for an HCM change.
- Building or running a test harness (Python runner, Postman collection, kafka-console-producer injection, JUnit flow tests).
- Executing tests against dev/staging (unified-dev) and reporting outcomes.
- Answering "does it persist?", "is it in the DB?", verifying a relation link, or checking a ghost-record path.
- Reporting or updating a results doc.

Do NOT use this skill for:
- Writing the actual code fix / null-guard / null-topic-fallback implementation — that is the hcm-dev work-stream (referenced here only to say what is under test).
- Data recovery / reconciliation of production campaigns — that is the DST work-stream.
- Platform architecture background (offline sync, Kafka two-topic, Redis, ES/DB divergence) — see the hcm-context memory (digit_hcm_architecture / digit_hcm_global_context).

## Core rule: how to verify a "persisted" claim

The single most important rule. A create does `putInCache` in Redis with a 60s TTL. A `_search` a few seconds after create hits the REDIS CACHE and returns a row whose `id=None` — a FALSE-POSITIVE "persisted". During persister lag the DB write never happens and after the TTL the row VANISHES (this is the live ghost mechanism).

Therefore:
1. Verify every "persisted" claim with a `_search` by clientReferenceId AFTER 90-100s (past the 60s TTL). `id=None` (no server id) is the tell it is cache-only, not committed.
2. Prefer the authoritative embedded back-fill signal over an unfiltered reverse-search: fetch the child by its own clientRef and check whether the server id back-filled (e.g. member.individualId non-null proves the individual exists).
3. Isolate single runs. Parallel suites create false negatives under persister lag. Run one isolated suite, get a tiebreaker before concluding.
4. Separate a harness bug (wrong wrapper key, unfiltered filter, wrong endpoint) from a real defect — re-query the same already-created ids before claiming a defect.

Full playbook: references/verification-methodology.md.

## Reference files

| File | Read it when |
| --- | --- |
| references/verification-methodology.md | Before reporting ANY persist/relation result. The reusable honest-verification playbook (TTL rule, embedded-signal rule, isolation, harness-bug-vs-defect, corrections-over-time). |
| references/validator-unbundle.md | Testing the cross-entity existence validator unbundling: fix under test, feature flags, U/S/B/D/F/E test series and outcomes, Python harness + Postman collection, null-relation-link findings, the 5 reconciliation relations, persister test plan. |
| references/transformer-error-handling.md | Transformer error-handling QA (HDDF-5226): the 12 test cases, Kafka+ES injection setup, the pass/fail matrix, the three High defects, JUnit flow harness, the three result docs. |
| references/chad-upload-error-from-app.md | Chad ITN upload-error-from-app live-device QA: mobile app `dumpError()`/retry-threshold code path, the result matrix (total-outage vs mid-transfer-cutoff signatures), the two confirmed false-positive `UPLOAD_ERROR_FROM_APP` cases (exact IDs/timestamps), gating implication for the Gap B/E tracer counts. |

## Problem -> Approach -> Solution playbook

### Validator-unbundle test (existence-validator unbundling, unified-dev)
- Goal: prove that with the persister-lag-prone cross-entity EXISTENCE checks disabled (flags default OFF), a child whose parent is still queued PERSISTS instead of being hard-rejected, while structural/uniqueness/self-existence/MDMS/enum checks stay ON, and id back-fill is preserved.
- Approach: 26-case plan (U1-U9 persist, B1-B4 back-fill, S1-S6 regression, D1-D4 downsync, F1-F2 toggle, E1-E3 e2e RCA). Python harness + importable Postman collection. Every persist claim DB-verified post-TTL.
- What we found:
  - U1-U9 = PASS on fix behaviour (child with bogus parent ACCEPTED 202, never existence-rejected). DB-committed post-TTL for U1(bulk+single),U2,U3,U6,U7,U9; U8 referral PASS with CORRECT field names; U4/U5 accepted-but-not-persisted was a missing-`address` enrichment NPE (task requires an `address` object), not a fix bug.
  - S1-S6 regression = kept-ON validators still enforce (self-relationship, uniqueness, MDMS relationshipType, enum recipientType, self-existence on _update). S4 later CHANGED to 202-accepted (both-projectBeneficiary-null with valid address) — earlier "reject" was the missing-address NPE.
  - B1/B2 back-fill = PASS (DB-verified): member referencing existing household+individual by clientRef only back-fills BOTH householdId and individualId. Test back-fill via the synchronous single `_create` RESPONSE (enrichment-time, lag-independent).
  - F1 = PASS (flag OFF persists). F2 (flag ON re-reject) and E1-E3 removed from scope. D1-D4 downsync = BLOCKED (dev enc-key missing, not the fix).
- Status: fix behaviour validated; results doc built (CDD-framed). D1-D4 blocked on dev enc-key. Do NOT update the results docx until the user says "update".
- Reliable vs noise: reliable = post-TTL DB-verified U/S/B outcomes, the hasErrors true->false flip. Noise = cache false-positives (id=None), egov-user/enc-service/accesscontrol churn causing 401/500/persist-lag, the `/individual/` vs `/health-individual/` endpoint mistake (use /health-individual), dev individual-search returning 0 for a resolvable individual.

### Null-relation-link sweep (validator-unbundle build, 2026-07-07)
- Goal: with existence validation removed, does the system enforce relation integrity, and what happens downstream to a child with a null link?
- Approach: inject null into each of the 5 relation join fields in isolation (others valid), DB-verify post-90s; plus poison-batch and NotNull-guard checks.
- What we found: the service ACCEPTS null in ALL 5 relation link fields (202, no validation error) — relation integrity NOT enforced at validation. Downstream persist differs: member.householdClientReferenceId / member.individualClientReferenceId / beneficiary.beneficiaryClientReferenceId / referral null-link all 202 then DROPPED (in_db=0); task.projectBeneficiaryClientReferenceId=null 202 -> orphan but NON-DETERMINISTIC (coin-flip orphan-vs-lost); side_effect.projectBeneficiaryClientReferenceId=null 202 -> DETERMINISTIC orphan 3/3. NotNull-guarded fields correctly REJECT 400. Poison-batch FIXED (1 bad + 3 good -> 3 good persist). All 5 reconciliation relations link correctly when clientRefs present.
- Status: findings solid (DB-verified, isolated, tiebroken). Recommendation to dev: enforce linking clientReferenceId as required at service layer.
- Reliable vs noise: reliable = the deterministic side-effect orphan (3/3), the drops, poison-batch fix. Noise/corrections = beneficiary & referral null-link used to persist on 2026-07-06 (L2/O4) but now drop -> doc D3/D4 lines flagged STALE; task orphan non-determinism reported honestly rather than as clean pass/fail.

### Persister batch/duplicate test (unified-dev, PENDING)
- Goal: DB-verified evidence of batch no-loss, poison-record survival, and duplicate handling (service-layer uniqueness 400 vs persister-layer ON CONFLICT DO NOTHING).
- Approach: API-level plan P1 (bulk N members -> all in DB), P2 (poison record in batch -> good survive, already observed), P3 (create X, then bulk [X-dup, Y, Z] and in-batch [W, W, V] -> what persists, is batch aborted, is dup absorbed vs rejected).
- What we found so far: egov-persister on unified-dev is in CrashLoopBackOff (ConfigException: no resolvable bootstrap urls) — yet HCM records persist fine, so HCM entity persistence does NOT go through egov-persister on this build; health services persist via their own path. Confirm the actual HCM persistence mechanism before designing the persister test.
- Status: PENDING (test plan drafted, not executed). Read-only cluster access only; do not modify cluster config; do not push prod data to dev.

### Transformer error-handling QA (HDDF-5226)
- Goal: verify the transformer error-handling enhancement captures and persists failures to the error index, and identify which failure classes (especially the silent ghost-record class) are still unhandled.
- Approach: egov-transformer + egov-indexer against same Kafka+ES, tenant dev. Inject failures via kafka-console-producer to source topics; verify on transformer-error-topic and ES index transformer-error-index-v1. 12 test cases.
- What we found: Pass = TC-01/02/03/06/07/11 (exception-based failures captured+persisted). Fail = TC-04/05/08/09/10/12. Three High defects: TC-08 silent empty-enrichment indexed with null links = THE ghost mechanism (Bauchi 89/89 ES project_task with householdId=null, absent from DB); TC-09 indexer casts every id JsonPath value to String, ErrorMessage.timestamp is Long -> ClassCastException -> index never created, ALL error records dropped; TC-07 no retry for transient dependency outage (offset auto-committed, not reprocessed). TC-10 non-unique doc id (`id: $.topic`) -> overwrite. A JUnit flow-test harness (14 tests, happy+sad per flow) codifies the ghost/degraded behaviour.
- Status: QA reported (3 docs). The UUID/id fix (fixes TC-09+TC-10) and null-topic fallback (TC-06) are hcm-dev implementation items; the silent-enrichment ghost cause (TC-08) is still unfixed. Gating reco: fix TC-09, then TC-08, then TC-07, then the rest, before relying on this as a production data-integrity safeguard.
- Reliable vs noise: reliable = the pass/fail matrix and code-located root causes. Noise/corrections = the earlier "referral NPE bug" that turned out to be stale field names (test-data mistake); happy-path 404s that were dependency context-path misconfig (/project vs /health-project), not code.

### mc-nigeria-uat release QA (persister + unbundle + transformer + correlation/tenant, API-only, 2026-07-21)
- Env: `https://mc-nigeria-uat.digit.org`, tenant `ba`, auth = raw `access_token` in `RequestInfo.authToken`. API-only (no cluster/Kafka/ES).
- **CRITICAL path gotcha — UN-prefixed here (opposite of unified-dev):** use `/project`, `/individual`, `/project/beneficiary`, `/project/task`, `/project/staff`, `/referralmanagement`, `/household`, `/household/member`, `/stock`. The `/health-*` prefix returns 503 on this env. Always confirm the prefix per env before building payloads.
- **Reconcile failures against deployment state FIRST.** On this UAT: deployed = project/household/individual/stock/benf-idgen/facility/notification-push/worker-registry; bauchi-ns = transformer/excel-ingestion; NOT deployed = product, referralmanagement (=> side-effect + hf-referral 503, productVariant validation can't run). A 202-but-0-persist or 503 on referral/side-effect/hf-referral/productVariant is a DEPLOYMENT gap, not a defect — mark BLOCKED-DEPLOY, re-test after deploy.
- Persister (51 API cases): batch no-loss to 500, per-record isolation (any poison position, all-poison=0 no-crash, liveness), ON CONFLICT idempotency (re-create/in-batch-dup/concurrent/replay), update-applies, zero-loss accounting, data-integrity — all PASS. DLQ/retry/offset internals NOT API-testable.
- Unbundle (26 API cases): flags default FALSE = ghost-parent create ACCEPTED (202) not blocked = fix works; member SINGLE still bundled (400) vs BULK unbundled; always-on validators (uniqueness/enum/self-existence/RequiredLink/missing-tenantId) still reject. Flag-ON toggle BLOCKED (needs restart).
- Transformer error-capture (#1): NOT API-testable — needs Kafka/ES. Standalone Jupyter script `Downloads\test_transformer_error_capture.py` (kafka-python producer + ES basic-auth, marker-based verify); Kafka-push pattern from `D:\DST\scripts\script-reconciliation\db not in es .py`.
- Correlation ID: NOT echoed in ResponseInfo (internal/tracer only, not API-visible). Tenant isolation: bogus tenant -> 401 (PASS).
- **Two GENUINE findings (deployed services, not deployment gaps):** (1) `POST .../_delete` returns 202 but `isDeleted` not applied in DB (household, single+bulk); (2) HouseholdMember needs a RESOLVABLE household to persist — unbundle accepts the ghost-household member (202) but the row is DROPPED (householdId can't back-fill); member+real-household persists (ghost individual is fine). So unbundle removes the validation BLOCK but member persistence keeps a household-linkage dependency at persist time.
- Full detail: [[project_mc_nigeria_uat_release_testing]]. Deliverables in Downloads: `Release-UAT-TestCases.xlsx` (3 tabs), `test_unbundle_uat.py`, `test_transformer_error_capture.py`.

### Chad ITN upload-error-from-app live-device QA (2026-08-05)
- Goal: prove whether the mobile app's error-report pipe (`dumpError()` -> `error-handler/handle-error` -> `egov-tracer-error-details`, `errorMessage: UPLOAD_ERROR_FROM_APP`) actually works on the real PROD APK, and explain why household-member `UPLOAD_ERROR_FROM_APP` volume (267) is far below household's (3,526) despite Gap B having 3,673 affected households.
- Approach: real device + `adb logcat` (scoped to the app's own `flutter` tag — unscoped is near-100% noise) against the actual `chad` PROD APK/env supplied by the implementation team. Deliberately cut/restore network at different points (before sync, mid-transfer) and cross-checked every claim against ES (`chad-household-index-v1` etc.) and the tracer (`egov-tracer-error-details`) by exact `clientReferenceId`, never by locality/name text alone (those fields don't exist on every entity's payload — household-member and project-beneficiary payloads carry NO address/locality data at all, so a locality-text search will silently miss them).
- What we found: the report mechanism genuinely works (confirmed tracer docs for household-member AND household) when given a working connection to retry into. A total outage from the very first attempt does NOT reach the retry-increment code path at all (per code, that needs a server response) — yet the original create still eventually recovers once reconnected, because the unsynced record just keeps retrying indefinitely regardless of the separate error-report bookkeeping. **Bigger finding: two independent, fully-timestamped cases of the app reporting `UPLOAD_ERROR_FROM_APP` for a record that had ALREADY synced successfully minutes earlier** (household `26a5ffd0` synced 12:52 UTC, reported failed 13:29 UTC; household `c0511ff0` synced 13:37:57 UTC, reported failed 13:40:22 UTC) — a confirmed false-positive reporting defect, not a hypothesis.
- Status: reported to the team (thread "Chad ITN: Critical Data Loss"). Root cause of the false-positive (theory: server writes succeed but the app loses the response mid-read, matching `Connection reset by peer`/`Software caused connection abort` signatures) not yet traced to an exact code line. Reproducibility is inconsistent — most short interruptions self-recover with no error at all.
- Reliable vs noise: reliable = the two false-positive cases (exact IDs + ES-confirmed persist timestamps + fetched tracer docs, not inference) and the code-confirmed retry-increment condition. Noise/corrections = an early "DOH_CENTRE" locality-text tracer search that returned 0 hits not because nothing existed but because household-member/PB payloads don't carry locality fields at all — always search by ID for non-household entities; a `GET` request with a JSON body that silently matched ~10,000 unrelated docs (use `POST` for `_search` with a body, not `GET`).
- Full detail: references/chad-upload-error-from-app.md.

Corrections-over-time is the method here, not an accident: every strong claim above survived at least one wrong first read that was DB-verified and reversed. Keep doing that.

## Keeping this skill current
- After a QA session, add a dated Changelog entry and update the relevant reference file with the new DB-verified outcomes. Mark superseded findings STALE (do not delete the history — the contradiction trail is evidence).
- When the user says "update skills", sync from auto-memory.
- Source memories: project_validator_unbundle_test, session_persister_rca_null_relations, project_transformer_error_handling_qa (test portions only; code-fix details belong to hcm-dev).
- Standing user instruction: do NOT update HCM-Validator-Unbundle-Test-Results.docx (or other results docs) until the user explicitly says "update".

## Changelog
- 2026-08-05 — Added Chad ITN upload-error-from-app live-device QA: `dumpError()`/retry-threshold code path, result matrix (total-outage vs mid-transfer-cutoff error signatures), two confirmed false-positive `UPLOAD_ERROR_FROM_APP` cases with exact IDs/timestamps, gating implication for Gap B/E tracer counts.
- 2026-07-21 — Added mc-nigeria-uat release QA playbook (persister/unbundle/transformer/correlation-tenant, API-only): UN-prefixed path gotcha, deployment-state reconciliation rule, delete-not-applying + member-household-linkage findings, standalone Jupyter test scripts. Source: project_mc_nigeria_uat_release_testing.
- 2026-07-08 — Initial consolidation from memory files.
