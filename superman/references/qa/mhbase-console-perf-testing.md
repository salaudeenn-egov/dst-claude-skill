# mhbase-hcm-dev: Admin Console perf benchmarking + env facts (2026-08-17/19)

Env `https://mhbase-hcm-dev.digit.org`, tenant `mhbase`. UN-prefixed HCM paths (`/household`,
`/individual`, `/project/*`, `/referralmanagement`; `/health-*` = 404). All services deployed
(referralmanagement + side-effect + stock + product all live). Auth tokens are single-session
(a portal re-login as the same user kills the API token mid-suite - coordinate before long runs).
Bednet project `87c5ca86-53b4-4ee5-abb3-d40e609b664e`, ITN PVAR `PVAR-2026-02-06-000662`.
Hierarchies: legacy ITN_NI (Audi 1-5) + ADMIN_NI (19,113 Bauchi boundaries ingested 2026-08-18).
ES = es-cluster-v8 in-cluster (`elasticsearch-data.es-cluster-v8:9200`, readonly creds in .env only).

## THE root cause - householdType (Problem -> Approach -> Fix)
- **Problem:** every API household create (single + bulk, ~650 records over 2 days) returned 202,
  was searchable for 60s, then vanished; tenant count frozen. Looked exactly like a stuck persister
  consumer ("household write path down"). Misleading read #2: member.householdId back-fill seemed
  to prove the household committed - WRONG: **back-fill enrichment reads the Redis cache at create
  time, not the DB**, so children carry valid-looking parent server-ids for parents that never persisted.
- **Approach that settled it:** adb-captured the mobile app's sync - the app's households persisted
  in the same minutes API ones didn't -> payload-dependent, not infra. A/B/C/D variant isolation:
  minimal+clientAuditDetails NO, minimal+householdType YES, app-style YES, minimal NO.
- **Root cause:** `householdType` (e.g. FAMILY) is required by the async household insert but the
  API does not validate it synchronously -> 202 + silent permanent loss (records never replayed).
- **Fix/verify:** always send `householdType` in household payloads; verify persistence by _search
  after 100s (60s Redis TTL; a t+10s search returning the row proves nothing). Reported as defect:
  missing sync validation = silent data loss.

## Unified-template rules (each learned from a real console rejection)
- Pre-filled rows (Boundary List, Facilities with ROW_ID) are IMMUTABLE - deleting any ->
  `IMMUTABLE_ORPHAN_ROWS`. Tier campaign size via user/facility counts, never by trimming boundaries.
- Targets are MANDATORY on EVERY Boundary List row (2,094-row rejection proved it) - "scope by
  filling only some targets" does not work; scope = boundary selection + facility Active flags.
- Facility Type: only `Warehouse` accepted on this build ('Health Facility' rejected).
- Phones must be NUMERIC 11-digit (text '080...' rejected: "Phone Number must be a 11-digit number");
  keep a distinct range per campaign/tier (phone uniqueness is tenant-global).
- New boundaries CANNOT be created via the unified sheet here ("not a valid boundary from the
  dropdown list") - ingest via Boundary Management upload (Country..Community + code + FR/PT + lat/lon
  columns), THEN download a fresh template. Boundary names reject backtick and double-quote
  (allowed: letters any language, digits, space, ' & - / . ( ) _ , : ; @ + ! ?).
- User rows: boundary code column may be left blank (console resolves from cascading names);
  role-level mapping convention: CDD/DISTRIBUTOR at HEALTH FACILITY level (not village),
  TEAM_SUPERVISOR/ward, DISTRICT_SUPERVISOR/LGA, HELPDESK+PROVINCIAL/state, NATIONAL/country;
  warehouses at every stock-chain hop (country->state->LGA->ward->HF).
- Excel-open trap: if the user has the xlsx open while a script saves it, Excel's next save
  OVERWRITES the script's changes. Save under a NEW filename and read back from disk to verify.

## Console perf benchmark method (reference docx: "DIGIT HCM Admin Console Performance Testing
## & Benchmarking Report" in Downloads)
Tiers 5K/10K/20K (boundaries-facilities-users). Their timings assume the report's tuning (Kafka
partitions 3-4 on save/batch topics tenant-prefixed, DB max-active=30/threads=200 on 5 services,
pod sizing esp. excel-ingestion 2000m/3Gi/Xmx2550m, PF vars by user bracket: >18K users ->
BATCH_LAG=7/POLL=10/PACE=100ms, gateway codec 20MB, bulk-bcrypt-threads=4). UNTUNED mhbase:
55K-row sheet validation took 12+ min vs <1 min benchmark - apply configs before timing anything.
Mapping stages = staff mapping (1 per user), facility mapping (1 per ACTIVE facility - set
pre-existing facilities Inactive to keep the benchmark count exact), resource mapping.
Mobile-side cost at scale: facility downsync pages 200/page with ~20s CLIENT-side processing per
page -> ~25-30 min initial sync at 15K facilities; server responds instantly (not a server issue).

## Device-log QA additions (extends chad-upload-error-from-app.md)
- Persistent `adb logcat` streams get killed by this harness - use `adb logcat -G 16M` (bigger ring
  buffer) + on-demand `adb logcat -d` snapshots instead; grep '[REQUEST]/[RESPONSE'.
- Payload-extraction gotcha: `identifiers[]` inside Individual carry their own clientReferenceId -
  counting them as individuals creates false "missing" records.
- App crash found: boundary selection before boundary downsync completes -> Null check operator
  crash at digit_data_model boundary.dart:68 (BoundaryBloc._handlefind). OPPO HansManager freezes
  the app mid-sync (disable battery optimization on test devices).
- App E2E verified on mhbase: registration/administration/refused/absent flows all persist; the
  app always sends householdType=FAMILY (why app data never hit the root cause).
- DQ notes for reporting: unable-to-deliver tasks carry status=ADMINISTRATION_SUCCESS with the real
  outcome ONLY in additionalFields.deliveryComment, and a resource attached despite non-delivery -
  dashboards counting by status or summing resources overcount.

## Deliverables (Downloads)
mhbase-Persister-Suite-Results-FINAL-2026-08-18.xlsx (root-cause investigation, 36P/11F),
Persister-Regression-mhbase-2026-08-18(_v2).xlsx (46 TC regression), Unbundle-Test-mhbase-2026-08-18.xlsx
(3-env comparison), transformer_test_suite.py (standalone 12-TC suite, creds via env vars),
tier files MR-DN*_FINAL_UPLOAD*.xlsx. Harnesses in session scratchpad `mhbase/`
(persister_suite.py, persister_regression_mhbase.py, chain_sync.py, test_unbundle_mhbase.py).

## Unified template: EDITABLE vs IMMUTABLE columns (2026-08-26, learned the hard way)

**Problem.** An upload returned `Field 'Facility Type' contains invalid value 'Active'` on 2 rows.
Treating that as "a value to correct", the Facility Type cell was set to `Warehouse` and re-uploaded.
The whole sheet then failed with `IMMUTABLE_CELL_TAMPERED` — a SHEET-level rejection, strictly worse
than the 2 row-level errors it replaced.

**Root cause / rule.** On `Facilities List`, `HCM_ADMIN_CONSOLE_FACILITY_USAGE` (col M) IS editable —
flipping Inactive->Active on pre-filled rows is accepted and persists. `HCM_ADMIN_CONSOLE_FACILITY_TYPE`
(col J) is pre-filled/read-only; editing ANY pre-filled cell aborts the entire upload. Row-level
`invalid` only skips that row; `IMMUTABLE_CELL_TAMPERED` kills everything.

**Verify.** After any activation, assert that exactly the intended rows differ from the download and
that the ONLY differing column index is 12 (col M). Do not eyeball it.

## Boundary-level facility rows
"Activate the boundary/HF rows" means the facility rows whose boundary code has FEWER THAN 5 numeric
segments — Country/State/LGA/Ward/HealthFacility, not community. Identify by numeric-segment depth of
`HCM_ADMIN_CONSOLE_BOUNDARY_CODE`, NOT by name: name-matching misses sets like `DST FACILITY 001-005`
which are boundary-level but not named "... health facility". Typical result: 10 rows, 2 per level.

## Working rule: append vs activate
Only APPEND new rows when the user explicitly says append / "make Nk". Otherwise the ask is to
ACTIVATE existing rows and append nothing. Users sheet arrives empty every time so it is always
generated; the Facilities sheet is where the distinction matters.

## Carry-over growth and the 20,002 row cap
The console returns EVERY previously created facility in each download, all `Inactive`, all immutable.
Observed growth on mhbase: 0 -> 4,979 -> 7,362 -> 20,038 rows over four uploads. Data validations only
cover rows 3..20002, so appending past that is untested territory — one build reached 25,038 rows.
Once carry-over approaches 20k, either stop appending facilities or clear them from the tenant first.

## DEFECT: facility create silently corrupts facilityType
A create payload with `usage: "Active"` and NO `facilityType` returns 202 and stores the *usage* value
into the **facilityType** field. The record is then permanently un-correctable through the console,
because facilityType is read-only there — edit it and the sheet is rejected, so the only fix is the
facility `_update` API or deletion. Same family as the `householdType` silent-drop defect: the create
API validates neither field synchronously, and the damage only surfaces in a later export.
Reproduced 2026-08-26 with `F-2026-08-26-137926` / `-137927`.

## Benchmark: claim vs reality (vs DIGIT HCM Admin Console Perf Report)
| Activity | Benchmark | mhbase actual | Verdict |
|---|---|---|---|
| Sheet validation | 60 s | 0.33 / 0.41 / 0.53 s (5K/10K/20K) | PASS, ~113-180x inside |
| Campaign creation 5K / 10K | 25 min | 18 min / 20 min | PASS |
| Campaign creation 20K | 25 min | 29 min | FAIL, +4 min |
| Boundary ingestion 17,737 | 5 min | 97 min | FAIL, 21.6x |
| Users created | 5,000 / 10,000 / 20,000 | -300 / -200 / -602 | FAIL (correctness, not timing) |

User-creation failures cluster in **batch 400-499 at every tier** — a defect signature, not load.
Error logged: "Individual not searchable after 10 consistency poll attempts; worker creation deferred
for retry". Caveat before quoting any delta: the benchmark tiers SCALE boundaries (5K-5K-5K etc) while
our sheet carries a fixed 17,737 boundary rows in every tier, so our "5K" is a ~27,737-row workload —
not like-for-like, and our low tiers are pessimistic. Also unconfirmed whether mhbase carries the
report's tuned config (Kafka partitions, tomcat max-active=30, pod sizing, bulk-bcrypt-threads=4);
if it does not, the two FAILs are expected rather than regressions.
