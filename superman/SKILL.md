---
name: superman
description: >-
  The eGov DST team's complete DIGIT HCM (Health Campaign Management) knowledge base —
  one skill covering platform architecture, code development, QA/testing, DST data-support
  work, and code review. Invoke for ANYTHING HCM-related: campaign data investigation,
  ES-vs-DB (Elasticsearch vs PostgreSQL) ghost-record RCA, reconciliation/recovery scripts,
  daily campaign reporting (children-treated, CDD sync, Word reports), writing/reviewing
  fixes to HCM Java services (egov-transformer, egov-indexer, egov-persister, validators),
  validator-unbundle / transformer error-handling testing, and the automation pipeline's
  Airflow migration (Scheduler+Processor DAGs, orchestration only). Triggers: any tenant code
  (so/ba/oy/na/ko/ab/od/kd/tg/cg/pl/ke), "ghost record", "ES vs DB", "clientReferenceId",
  "project-task index", "egov-tracer", "reconciliation", "chain recovery", "children treated",
  "CDD sync", "householdId null", "transformer fix", "indexer yml", "validator", "does it
  persist", "review this HCM change", "ITN", "LLIN", "bed-net", "bednet distribution",
  "Airflow", "DAG", "scheduler.py", "dst_campaign_scheduler", "dst_dynamic_campaigns",
  or any campaign name (Sokoto, Bauchi, Oyo, Ondo, Kaduna, Togo, FCT, Plateau, Nasarawa,
  Kogi, Chad).
---

# Superman — DIGIT HCM Master Knowledge Base

One skill holding the DST team's accumulated HCM knowledge at eGov, organized into five
work-streams. Read the house rules and the core model below, then open the reference
file(s) for the task at hand.

> Living skill. Last synced 2026-08-04. Point-in-time: file:line citations, tokens,
> and counts may be stale — verify against current code / live systems before asserting
> as fact. Tokens rotate; treat any token as last-known. See "Keeping this skill current".

## House rules (apply in every session)

- **Audience:** DST / HCM engineers at eGov. Data investigation + fixes across ~27 health
  campaigns (Nigeria, Mozambique, Senegal, Togo, Chad, Burundi, Congo). Typical work: ES
  queries, SQL, Python scripts. Maintainer: Salaudeen (L3 SWE); original work dir `D:\DST`.
- **Doc/response style:** no emojis, no decorative unicode (no arrows, no rule lines). No
  bookmarks in Word docs. Plain text + standard markdown.
- **Numbers rule:** in reports use the analysis-script numbers, never silently swap to
  dashboard/Metabase figures. Note a discrepancy if asked; keep conclusions on analysis numbers.
- **Honesty rule:** report only verified facts (for persistence, DB-verified past the 60s
  Redis TTL). Never present a single run or a cache hit as proof. Say what is reliable vs noise.
- **Sensitive data:** Nigeria DB is production, read-only. Never write test data to production
  tenants (e.g. `so`). Use tenant `dev` for testing.
- **No credentials in this skill:** never write passwords, tokens, usernames, or Basic-auth
  headers into any skill file or example — use `<REDACTED>` / `<REDACTED-USER>` / `{{variable}}`
  placeholders; real values stay in the local `.env` / secret store only (see `SECURITY.md`).
- **Encrypt saved data:** campaign extracts (ES/DB dumps, Excel/CSV, tracer payloads) contain
  beneficiary PII — store them on an encrypted volume or in password-protected archives, delete
  when done, and never put raw campaign data into this skill or its repo.

## The one model to hold

```
Individual -> Household -> HouseholdMember -> ProjectBeneficiary -> ProjectTask
```
Every entity references its parent. **DB writes validate parent existence; ES writes do NOT.**
DB (PostgreSQL, schema = tenant) and ES (Elasticsearch, `{tenant}-{entity}-index-v1`) are
written by INDEPENDENT Kafka consumers: egov-persister -> DB (can fail silently),
egov-transformer -> ES (always succeeds). Persister fails but transformer succeeds =>
**ghost record** (in ES, not DB). This single fact underlies nearly all HCM work here.

## Work-streams and their references

Each work-stream has an `overview.md` (its full picture + Problem->Approach->Solution
playbook) plus topic references.

### 1. Platform context (read first for any HCM task) — `references/context/`
The stable core everything else builds on.
- `context/overview.md` — orientation, skill map, fast facts.
- `context/architecture.md` — offline-sync flow, Kafka two-topic pattern, GenericRepository
  fire-and-forget save, Redis cache, the 4 ghost-record root causes, critical code file:line.
- `context/data-reference.md` — ES index + DB table structures, household->task linkage, HCM
  API patterns (202 not 200), production DB/ES credentials, tenant/token map, KD/AZM SQL,
  tracer patterns, ES scroll/update snippets.
- `context/gotchas.md` — the 21 hard-won gotchas + investigation dead ends. Read before ES/tracer work.
- `context/postman-api-index.md` — 183 HCM API endpoints across 23 folders. Full importable
  collection bundled at `context/assets/HCM_Collection.postman_collection.json` (2.3 MB — import
  into Postman; do not read raw).

### 2. Development (HCM Java code fixes) — `references/dev/`
- `dev/overview.md` — dev playbook: the Problem->Approach->Fix->Status for each fix.
- `dev/ghost-record-fixes.md` — validator/persister/DLQ fixes (file:line, before/after),
  architectural options (uForwarder/Outbox/wave-sync), Salama-prod fixes, priority matrix.
- `dev/transformer-fixes.md` — UUID error-id fix, null-topic ThreadLocal fix, indexer ymls,
  /health-* endpoint fixes, JUnit harness, JDK8/maven build + env facts.

### 3. QA / testing — `references/qa/`
- `qa/overview.md` — QA playbook + the core verify-past-TTL rule up front.
- `qa/verification-methodology.md` — the honest-verification loop (verify past 60s Redis TTL;
  id=None = cache false positive; isolate runs; separate harness-bug from defect).
- `qa/validator-unbundle.md` — the unbundle fix under test, feature flags, U/S/B/D/F/E test
  series, Python harness + Postman collection, null-relation findings, persister test plan.
- `qa/transformer-error-handling.md` — HDDF-5226 12 test cases, setup, defects (TC-08/09/10), docs.
- `qa/persister-regression.md` — persister/bulk-sync regression: search-clone-mutate method, working
  create payloads (real refs), verified persister behaviours (batch/DLQ/idempotency), and the
  evidence-backed edge cases NOT handled (in-batch dup batch-loss, household update no-op,
  numeric/date validation gaps, whitespace clientRef, malformed-JSON 500). Kafka/cluster access recipe.

### 4. DST data support (RCA, recovery, reporting, automation) — `references/dst/`
- `dst/overview.md` — DST playbook: per-campaign Problem->Approach->Root cause->Remediation->Status.
- `dst/es-vs-db-rca.md` — 6-phase RCA SOP, tracer queries, the 90,143-ghost RCA, "what proven means".
- `dst/recovery-scripts.md` — db<->ES reconciliation, relational chain check (5-link legacy scripts; 6-link A-F in the workbench), 12M-record perf
  optimizations, checkpoint pattern, tracer regex, CREATE vs UPDATE rule, dependency order.
- `dst/campaigns.md` — per-tenant config/API/token/findings/status/scripts (so/ba/oy/na/ko/ab/od/kd/tg/cg/ke).
- `dst/reporting.md` — daily pipeline, 4-slot structure, formulas, per-drug age bands, per-day
  numbers, python-docx helpers, DQ rules.
- `dst/automation-and-structure.md` — `D:\DST\automation` pipeline + `D:\DST` folder conventions.
- `dst/itn-llin-reporting.md` — ITN/LLIN (bed-net) household-based campaign reporting
  built alongside the SMC/SPAQ pipeline (Chad): separate analyze_itn/cdd_sync_itn/
  report_itn.py files, LGA-code target matching, LGA-not-facility aggregation grain,
  real per-day ES date scoping, the stale-historical-snapshot class of bug, and the
  chart-vs-table formula-consistency fix.
- `dst/chad-smc-beneficiary-id-rca.md` — the Chad SMC generated-beneficiary-ID defect (tenant ch):
  304,831 INVALID_ID errors explained AND fully reproduced live in production; disabled pool gate +
  unconditional 7-digit fallback + daily dispatch quota; 392,829 silent ledger collisions; the
  duplicate-children proof; the app-vs-individual-service-vs-idgen call chain; detection heuristics
  and the live-device + pod-log reproduction method. Quantified 2026-08-10: 21,179 realized
  duplicates (same ID consumed by a different CDD) + 9,961 phone-vs-phone collisions, via the
  reusable tracer-population -> id_pool cross-check -> classify script (ch_dup_id_crosscheck.py).
  Read for anything beneficiary-ID / id_pool / INVALID_ID / duplicate-beneficiary related.
- `dst/dst-team-and-hcm-modules.md` — what DST (DIGIT Success Team) is + the full HCM module surface.
- `dst/airflow-migration.md` — migrating the automation pipeline's orchestration
  (scheduler.py) to Apache Airflow: two architecture iterations (per-slot DAG
  factory, then a Scheduler+Processor split adopted from a real production
  reference doc), the Airflow-3 `CronTriggerTimetable` zero-width-interval
  gotcha, Postgres-lock per-tenant concurrency (replaces Airflow Pools once one
  DAG serves every tenant dynamically), the marker-dict success/no-op/failure
  protocol, and the "grouped by credential set, not by country" tenant-grouping
  discovery. `pipeline/*.py` is never touched — orchestration layer only.
- `dst/airflow-dag-patterns.md` — generic production Airflow DAG patterns (design
  principles, operators/sensors, testing, deployment) supporting the migration
  work. Formerly the standalone `airflow-dag-patterns` skill.

### 5. Code review (HCM-specific hazards) — `references/review/`
- `review/overview.md` — the 5 hazards that matter most + how to review.
- `review/review-checklist.md` — per-layer failure modes (controller, validator, persister,
  transformer, indexer yml, recovery scripts). Pair with the generic `/code-review`.

## Fast orientation facts

- **Tenants** (DB schema + ES index prefix): `so`=Sokoto, `ba`=Bauchi, `oy`=Oyo, `na`=Nasarawa,
  `ko`=Kogi, `ab`=FCT/Abuja, `od`=Ondo, `kd`=Kaduna, `tg`=Togo, `cg`=Congo, `mz`=Mozambique,
  `pl`=Plateau, `ke`=Kebbi, `si`=Sierra Leone.
- **Drugs/age bands:** SMC=SPAQ (SPAQ2 3-11m, SPAQ1 12-59m; treated 3-59m). AZM=Azithromycin
  (1-11m, 12-59m; treated 1-59m). Togo MRDN/SPAQ 0-3m/3-11m/12-59m.
- **DIGIT writes return HTTP 202, not 200** — check `in (200, 202)`.
- **egov-tracer `requestBody` is unquoted (not JSON)** — `json.loads()` fails silently; use regex.
- **`enable.auto.commit=true` is a platform-wide constraint** — never propose changing it.
- **Verify persistence past the 60s Redis TTL** — `id=None` in a search result = cache false positive.

## Keeping this skill current

- Raw per-session facts live in the maintainer's local Claude auto-memory;
  this skill is the curated, task-oriented layer synced from it.
- When the user says "update skills" or at the end of a work-stream, fold new durable learnings
  into the matching `references/<stream>/` file and add a dated Changelog entry below. Mark
  superseded findings STALE rather than deleting history.
- **Problem -> Approach rule (MANDATORY for every skill update).** Never record only the bare
  fact/fix. Each new entry must capture, in order: (1) **Problem** — the symptom exactly as it
  presented, including what was misleading about it (the wrong first hypothesis); (2) **Approach**
  — how it was investigated and reasoned to root cause (the diagnostic move that actually worked,
  and the evidence that settled it); (3) **Root cause**; (4) **Fix/Outcome** — what changed +
  how to verify. Rationale: a knowledge base that carries the problem context and the method is
  reusable — future-me can recognize the situation and re-run the approach — whereas a bare fact
  only helps if the exact same question recurs. Keep it compressed but keep all four beats.
- Each `references/<stream>/overview.md` lists its source memories.

## Changelog

- 2026-08-10 — `dst/chad-smc-beneficiary-id-rca.md`: added the duplicate cross-check section
  ("how many more like 3975695?"). Problem: team lead's pushback "there is an explicit validator
  to block this" + ask to quantify — misleading premise, since the 304,831 errors ARE the
  validator firing (it blocks the ledger write, not the registration; the pool generator can't
  see rejected numbers and re-issues them). Approach: standalone reusable script
  (ch_dup_id_crosscheck.py) — parallel sliced tracer scroll -> distinct rejected IDs with first
  error ts + rejecting users -> LEFT JOIN ch.id_pool -> classify by createdtime-vs-rejection and
  status, with two built-in validations (anomaly bucket pool-row-older-than-rejection = 0;
  consuming-username==rejecting-username = 0) + username resolution chain (ch.individual ->
  eg_user -> user-sync index, 100% resolved). Result: 306,929 distinct rejected IDs; 21,179
  realized duplicates (different CDD consumed the re-issued number; median 3.8d
  rejection->reuse; the Jul 23/25 pool batches that collapsed the error curve re-issued
  16,434+4,638 of them); 9,961 phone-vs-phone collisions (overlap 529); 133,937 dispatched
  at-risk; 59,224 unassigned to quarantine; 92,578 to exclude from future generation. New
  gotchas in context/gotchas.md: idgen _update tracer bodies ARE valid JSON; beneficiary ID not
  on task docs (reporting unaffected — but never claim zero data impact); ch.individual
  usernames populated / mobilenumber encrypted. Deliverable pattern: humanized-sheet Excel +
  re-runnable script; keep message numbers window-consistent with the attached file.
- 2026-08-07 — New `dst/chad-smc-beneficiary-id-rca.md` + Chad-SMC gotcha block in
  `context/gotchas.md`: the Chad SMC beneficiary-ID investigation, from tracer RCA to a live
  end-to-end reproduction in production. Problem: 304,831 INVALID_ID errors on id_pool/_update
  (misread three ways: persist-lag, app-side calls, "known requirement"). Approach: confirmed the
  REAL repo/branch/APK with the implementing dev (first RCA had cited the wrong codebase), code
  trace, three live device-test rounds (prod APK 2.2.75 + UAT user, adb logcat streaming, airplane
  race to keep the pool empty), DB forensics on id_pool + append-only id_transaction_log, and
  prod pod-log capture. Root causes: hasMinimumBeneficiaryId hard-coded true ("TEMPORARY
  OVERRIDE", commit e0671efbb); unconditional `?? generateRandomAlphanumeric(7)` fallback
  (format-identical to pool IDs); single silent no-retry pool download; DAILY per-user+device
  dispatch quota stranding reinstalled devices; _update validates existence only (no
  dispatch/ownership check). Mechanism: app never calls _update — individual service does,
  server-side. Quantified: pool covers 87.8% of the 7-digit space -> most generated IDs are
  silently ASSIGNED (392,829 assigned-never-dispatched; est. 750k-850k generated total; the 304k
  errors are only the misses); duplicate-children case proven (ID 3975695). Clincher: our own
  generated ID 8490579 rejected by prod beneficiary-idgen with the exact campaign error, 1s after
  our controlled sync. Bonus findings: tracer index stalls for hours (not real-time), identifierId
  encrypted in the individual index, 3 idgen replicas (search all pods), interrupted first
  bootstrap bricks the app.
- 2026-08-04 — New `dst/airflow-migration.md`: migrated `D:\DST\automation`'s
  orchestration (scheduler.py) to Apache Airflow, on branch `airflow-migration`
  (not yet committed). Built and live-tested two architecture iterations: a
  per-(tenant,time,mode) dynamic DAG factory first (found and fixed a real bug
  where adding TaskGroups silently broke hardcoded XCom task_id lookups), then
  — after the user shared a real production architecture doc for a different
  eGov HCM reporting system — rebuilt as a Scheduler+Processor DAG split
  (adopting only the DAG-architecture pattern, keeping Google Sheet config
  rather than migrating to their MDMS/Kafka/FileStore stack). Key durable
  findings: Airflow 3's `CronTriggerTimetable` gives a zero-width
  `data_interval` for plain cron-string schedules (confirmed via direct
  DagRun query) — switched to wall-clock windowing; a single static Processor
  DAG serving every tenant via dynamic conf can't use static per-tenant
  Airflow Pools, replaced with a Postgres advisory-style lock table (verified
  live against a real concurrent-trigger race); and the DST tenant-grouping
  reality is "grouped by which ES credentials a tenant needs," not by country
  (Chad shares Nigeria's "central" credential set; Taraba/Togo were split out
  only because their ES logins differ, not for network reasons — all three
  actually share one ES host).
- 2026-08-02 — New `dst/itn-llin-reporting.md`: built the full ITN/LLIN (bed-net)
  reporting pipeline for Chad as 3 new files (`analyze_itn.py`/`cdd_sync_itn.py`/
  `report_itn.py`) never touching the SPAQ originals, plus a small drug-type dispatch
  in `run.py`/`scheduler.py`. Key findings, all Problem->Approach->Root cause->Fix in
  the file: target-matching must join on the LGA CODE (`boundaryHierarchyCode.district`,
  e.g. `ADMIN_TC_16_03_ADRE`), never the plain name — chad's DB reuses Nigeria-style
  boundarytype labels (`LGA`, not `District`/`Province`) for the same ES "district"
  tier; aggregation grain moved to LGA (not 1,000+ facilities) since target data only
  ever exists at LGA grain; a zero-activity-LGA fill-in bug where LGAs with no records
  that day silently dropped out of target totals, making daily figures incomparable
  day to day; real per-day ES date scoping added (`Data.taskDates` GTE/LTE range
  filter, same field/mechanism as `analyze.py`, confirmed live), replacing an earlier
  delta-of-snapshots workaround the user correctly flagged as unreliable; and a
  stale-historical-snapshot class of bug (an old day's frozen `itn_history/` file can
  carry a wrong target forever) fixed by never reading a historical day's own stored
  target at all, always using today's fresh one — including a chart-vs-table
  formula-consistency fix, since `report.py` itself uses two different target-base
  formulas (chart = full campaign target; table = daily-target x elapsed-days) that
  read as contradictory once shown side by side for ITN.
- 2026-07-28 (3) — CORRECTION to the (2) entry below: the Gap D relink and HFReferral comment-out
  were made in the recon-recovery working tree but never committed, and were wiped when the user
  reset back to git branch `main` (clean, matching `origin/main`). When asked to re-apply "the
  meaningful fixes", the user explicitly excluded those two — **only the tracer parser/phase-4 fix
  and the error-response truncation fix are on `main`** (as uncommitted working-tree edits across
  `core/chain.py`, `core/api_client.py`, `core/utils/processor.py` — verify `git status` before
  assuming persistence). `dst/recovery-scripts.md` now has explicit STATUS notes on each of the 4
  entries below stating which are/aren't live. Also added: RCA of the chad push's 100%-failure gaps
  (HouseholdMember/Gap B all `INDIVIDUAL_NOT_FOUND` + all household heads; ProjectBeneficiary/Gap D
  all `DUPLICATE_ENTITY`, suspected ES/DB-lag in `check_exists`) — both hypotheses, neither DB-confirmed
  yet. Don't assume any of this session's chain.py/entities.py changes are live without checking the
  actual file content first.
- 2026-07-28 (2) — `dst/recovery-scripts.md`, tracer recovery false-negatives on chad, all fixed
  live in `D:\DST\scripts\recon-recovery`: (1) `core/chain.py` — new `_loose_json()` (regex-quote
  bare keys/values then `json.loads()`, ~20 lines, not a hand-rolled parser) as a fallback when
  `json.loads()` throws on a Java-toString-style unquoted tracer body; `cb()` now also accepts a
  bare-array body with no `{"<key>": [...]}` wrapper; new **phase 4** (`PHASE4_MAX_IDS=1000`) —
  per-id `match_phrase` with NO tenant/date/url/body_key constraint, for ids phase 3 still misses
  (outside date window or non-matching tenant phrase), capped to avoid a per-id scroll storm on a
  large residual gap. All inside the shared `_tracer_scan()`, so it applies to all 5 gaps, not just
  the Individual case that surfaced it (`chad` id `958946c0-...`, `UPLOAD_ERROR_FROM_APP`). Push
  needed no changes — `chain.py` always re-serializes via `json.dumps()` before the recovered
  payload hits CSV, so push only ever sees clean JSON. (2) Two independent hard truncations were
  hiding push failure reasons — `api_client.py:classify_error()` `resp_text[:400]` and
  `processor.py`'s `record()` `str(resp)[:500]` — both raised to `[:4000]`. Full details + the
  Problem->Approach->Root cause->Fix writeup in `dst/recovery-scripts.md`.
- 2026-07-28 — `dst/recovery-scripts.md` (recon-recovery, `D:\DST\scripts\recon-recovery`):
  (1) Gap D relinked for SMC per user domain correction — beneficiary =
  `HouseholdMember.householdClientReferenceId` (== household's own crid), not
  `Individual.clientReferenceId`; `chain.py`/README updated in lockstep. OPEN: this silently
  breaks the ch/Chad Gap-E head-exclusion filter (compares the now-household-id
  `beneficiaryClientReferenceId` against a set of individual head-ids) — flagged to user, not
  fixed, needs a domain answer on PB granularity before touching it. (2) HFReferral commented out
  of `entities.py` — SMC-only, ITN tenants (e.g. chad) 404 on `hf-referral-index-v1` every run.
  (3) Re-confirmed the still-unfixed `build_household()` locality/localityCode field-name bug
  (silently nulls address on every reconciliation-ghost Household push) — not applied pending
  user go-ahead.
- 2026-07-21 — Adopted the **Problem -> Approach rule** for all skill updates (see "Keeping this
  skill current"): every entry records Problem (+ misleading first read) -> Approach (diagnostic
  method + settling evidence) -> Root cause -> Fix/verify, not just the bare fact. Applied it to
  `dst/recovery-scripts.md`: the recon-recovery date-field alignment fix (ProjectTask
  createdtime->clientcreatedtime; Individual/ProjectBeneficiary @timestamp->auditDetails.createdTime),
  confirmed on ch/Chad (9,738 task false ghosts; 25 Individual false ghosts).
- 2026-07-08 — Initial build: consolidated the auto-memory (27 fact files) into five
  work-streams (context, dev, qa, dst, review); bundled the HCM Postman collection + endpoint
  index; added DST-team/HCM-module context from the training schedule.
- 2026-07-08 — Combined the five separate skills into this single `superman` skill; each former
  skill's SKILL.md is now its work-stream `overview.md`.
- 2026-07-08 — Hybrid layout: re-added hcm-context / hcm-dev / hcm-qa / dst / hcm-code-reviewer
  as thin router skills that list on their own and point into `superman/references/<stream>/`.
  `superman` remains the single source of truth for content; the five are lightweight entry points.
- 2026-07-08 — Added `qa/persister-regression.md`: persister reliability + bulk-sync regression on
  unified-dev (search-clone-mutate method; working create payloads with real refs; verified
  batch/DLQ-retry-to-exhaust/redelivery-idempotency; evidence-backed NOT-HANDLED edges — in-batch
  duplicate clientRef drops the whole batch, Household update is a silent no-op (household-specific;
  other entities update fine), missing numeric/date validation, whitespace clientRef persists,
  malformed JSON->500, mixed-tenant batch drops good record; Kafka CLI access recipe). Result doc:
  Downloads\Persister-Bulk-Regression-Results.xlsx. Cross-refs memory hcm_working_payloads_dev.md +
  session_persister_rca_null_relations.md.
- 2026-07-13 — Persister fix-validation retest folded into `qa/persister-regression.md`: full 76-TC
  re-run (2026-07-10, fixed branch, <REDACTED-USER>) confirms RECORD-LEVEL in-batch-duplicate/poison-batch
  isolation is FIXED (EDGE-1 3 uniques survive; poison 5/5 on HH/member/beneficiary); EDGE-2
  resource-level dup within a task still a FINDING; orphan-null-PB now sync-rejected (IMPROVED);
  bad-ref single creates shifted 400-sync -> 202-async-drop (data integrity intact, dev note).
  Result doc Downloads\Persister-Bulk-Regression-Retest-2026-07-10.xlsx. Added dev test-data recipe
  (POLIO project ad5d07be + productVariant PVAR-2026-06-10-000214 + "Gedero Koniso" locality
  NEWTEST00222_MO_11_04_01_01_02_GEDERO_KONUSO + <REDACTED-USER> single-session token) and the
  beneficiary-downsync verification + its empty-result gotchas (locality whitespace / wrong projectId).
