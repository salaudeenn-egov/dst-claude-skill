# Campaigns — per-tenant config, findings, recovery status, script inventory

All auth tokens are LAST-KNOWN and rotate — re-verify before use. Production DB and
ES credentials live in the `hcm-context` skill (do not duplicate). Counts and file
paths are point-in-time; re-run the comparison/chain scripts to refresh.

Tenant codes at a glance: `so` Sokoto, `ba` Bauchi, `oy` Oyo, `na` Nasarawa,
`ko` Kogi, `ab` FCT/Abuja, `od` Ondo, `kd` Kaduna, `tg` Togo, `cg` Congo (recon test),
`ke` Kebbi (automation), `pl` Plateau (reporting), `bo` Borno (SMC multi-cycle).

---

## Sokoto SMC — `so`

| Field | Value |
|---|---|
| Campaign | Nigeria Sokoto SMC (Seasonal Malaria Chemoprevention) |
| Tenant / DB schema | `so` / `so` |
| DB | `<REDACTED-DB-HOST>` (`ngcentralprd`) |
| ES index | `so-project-task-index-v1` |
| Investigation done | 28 May 2026 |

Findings:
- 4,645,970 project_task in DB (doseIndex=01, ADMINISTRATION_SUCCESS)
- 4,664,185 in ES → ES higher by 18,526 ghost records
- 311 in DB missing from ES (Jul 28 indexer failure)

Root causes: beneficiary race condition (~8,244; 12,189 unique beneficiary UUIDs in
"Related entity does not exist" tracer errors), idgen outage Sep 21 (~282; 166+37+79
idgen errors unique to Sep 21), indexer failure Jul 28 (311 DB-only, 308/311 same day, all rowversion=1).

Round breakdown of ghosts: R1 Jul 24–27 = 733 | R2 Aug 21–26 = 2,735 |
R3 Sep 18–21 = 11,795 | R4 Oct 13–20 = 2,793.

Recovery status (28 May 2026): push service built + tested locally
(`reconcile_es_not_in_db_local.py`); production push PENDING beneficiary dependency
check; 311 DB-only re-index pending. Before pushing the 18,526 tasks, verify the
12,189 beneficiary UUIDs exist in `so.project_beneficiary`; if missing push
beneficiaries first via `so-save-project-beneficiary-topic`.

Files: notes `D:\DST\campaigns\nigeria-sokoto-smc\notes\sokoto_rca_investigation_notes.md`;
RCA `C:\Users\Salaudeen N\Downloads\sokoto\Sokoto_SMC_RCA_Report.docx`;
data `C:\Users\Salaudeen N\Downloads\sokoto\so_*.csv`;
push `C:\Users\Salaudeen N\Downloads\sokoto\push_es_not_in_db.py`.

Tracer: index `egov-tracer-error-details`; tenant filter `Data.apiDetails.requestBody`
match_phrase `"tenantId":"so"` (with quotes); timestamp `Data.@timestamp`; error agg
`Data.errors.errorMessage.keyword`.

---

## Bauchi SMC — `ba`

| Field | Value |
|---|---|
| Tenant | `ba` |
| API base | `https://bauchi-hcm.digit.org` |
| Auth token (last-known, verified 2026-07-01) | `<REDACTED-TOKEN>` |
| Earlier token (last-known, rotated) | `<REDACTED-TOKEN>` |
| Campaign period | 2026-06-22 → 2026-07-01 |
| GTE / LTE | `2026-06-22T00:00:00.000Z` / `2026-07-01T23:59:59.999Z` |

Chain recovery results (`ba_chain_recovery_20260701_1013.xlsx`) — ES entity counts:
Households 467,325 | HH Members 2,285,351 | Individuals 2,307,321 |
Project Beneficiaries 2,981,053 | Project Tasks unique PB refs 2,173,263 (from 8,194,811 task docs).

Gaps: A (HHMember→Household) 22,093 missing, 0 in tracer, 0% — unrecoverable via this
method | B (Household→HHMember) 2,126 missing, 305 in tracer, 14.3% | C (HHMember→
Individual) 8 missing, 1, 12.5% | D (Individual→PB) 2,828 missing, 449, 15.9% |
E (PB→Task) 808,291 missing, 3, 0% — likely campaign-incomplete (undelivered as of Jul 1).

Pushable: Gap B (305) + Gap D (449) = 754 records via `ba_chain_push.py`
(`USER_UUID` must be filled first).

Scripts (Downloads): `ba_chain_recovery.py`, `ba_chain_push.py`, `ba_recon.py`.
`ba_recon.py`: input folder `ba_recon (1)`; checkpoint `ba_reconciliation_push_checkpoint.json`;
prior run 190 PT FAILED (`NotNull...resources[0].tenantId`, fixed by adding tenantId
to resources); 233 HHM FAILED (all `isHeadOfHousehold=True`, unresolved).
Task index gotchas: see recovery-scripts.md (taskDates works; auditDetails.createdTime
and isDeleted do NOT exist on task docs). `ba_recon` USER_UUID still needed.
Also see reporting.md — Bauchi reporting config was still pending as of 2026-06-30.

---

## Oyo — `oy` (chain recovery + SMC individual recovery)

| Field | Value |
|---|---|
| API base | `https://oyo-hcm.digit.org` |
| Auth token (last-known) | `<REDACTED-TOKEN>` |
| Chain recovery script | `C:\Users\Salaudeen N\Downloads\extraction_scripts\oy\oy_chain_recovery.py` |
| Chain config | GTE `2026-06-04T00:00:00.000Z`, LTE `2026-06-08T23:59:59.999Z` |

Chain: GAP A run (`oy_chain_recovery_20260624_0614.xlsx`) flagged 40 unique households
(118 member rows) — verify they are not `isDeleted=True` / sub-tenant before assuming loss.

SMC individual recovery: 4,182 individuals missing → `oy_individual_create_push.py`
(input `oy_individual_4k_missing.xlsx`), ran to ~34% when last observed. After recovery
DQ: Child Name 1 missing, HH Head Name 302, Beneficiary ID 5,526, Lat/Long 614.
Project-task search: 6,922 in error extract; 6,836 already in HCM; 86 missing (none have
lat/lon). Lat/lon (614) CONFIRMED UNRECOVERABLE — individual index address lat/lon is
null; task GPS is the only source; CDDs submitted without GPS lock.

---

## Nasarawa — `na` (SMC individual recovery)

| Field | Value |
|---|---|
| API base | `https://nasarawa-hcm.digit.org` |
| Auth token (last-known) | `<REDACTED-TOKEN>` |

29,049 individuals missing → `na_individual_create_push.py` (input `na_individual_29k_missing.xlsx`).
DQ after recovery: Child Name 13 missing, HH Head Name 638, Beneficiary ID 44,545.

### Nasarawa SMC 2026 — reporting reconciliation (campaign `CMP-2026-06-03-000316`)
Verified 2026-07-13/14: children-treated report **matches** dashboard. Same shape as Kogi.
- Covered/treated = ADMINISTRATION_SUCCESS + `additionalDetails.doseIndex==1` = 1,058,861 (report)
  vs dashboard 1,059,288. Gap −427 = **before-Jul-2 clip**: 427 cycle-2 dose-1 administrations
  synced before the report `start_date=2026-07-02` (dashboard scopes cycle from Jun 4; 0 after
  end). ES-confirmed count 427. Lands in 4 early-sync LGAs: Toto −198, Nasarawa Eggon −143,
  Karu −85, Doma −1. Fix: widen start_date to 2026-06-04 OR scope by `cycleIndex.keyword="02"`.
  This is the SECOND confirmed case of the start_date-clips-early-syncs gotcha (Kogi was 65).
- Redose SPAQ1 1,195 / SPAQ2 2,427 exact. Ineligible 2,834 (dash 2,835). SPAQ↔age same as ko
  (SPAQ1=3-11, SPAQ2=12-59). SPAQ swaps 13 -> 2 after fix (age-60 SPAQ1 + age-6 SPAQ2, both
  Lafia; age-60 is out-of-window so the DOB fix logic skips it — needs manual).
- DQ: **51,624 administered children >59mo (4.9%)** — the standout flag, far higher than ko
  (0.7%); zero under-3. Missing GPS 5,624 administered. Missing HH-head 142 = 134 OIC referrals
  (valid) + 8 CDD referrals, **0 on treated children** (cleaner than ko). Missing child name 3
  ("nan" junk, 2 administered). Name junk (administered): child ~413 (placeholder/digit/short),
  head ~580 (digit/short/placeholder); names ~99.95% clean.
Extract path `Downloads\nas cycle 2\` (files `CHILDREN_TREATED_part1.xlsx` + `_part2.xlsx`).

---

## Kogi — `ko` (SMC individual recovery)

| Field | Value |
|---|---|
| API base | `https://kogi-hcm.digit.org` |
| Auth token (last-known) | `<REDACTED-TOKEN>` |

30,151 individuals in error extract; most DUPLICATE_ENTITY (individuals exist under a
DIFFERENT clientReferenceId — not genuinely missing). Script `ko_individual_create_push.py`
(input `ko_individual_30k_missing.xlsx`, 30 workers, duplicate OK). DQ: HH Head Name
5,265 missing, Beneficiary ID 62,192, Lat/Long 74 (unrecoverable). Open: why HHMember
links break if individuals exist under different clientReferenceIds. Also
`ko_project_task_create_errors.py`.

### Kogi SMC 2026 — reporting reconciliation (campaign `CMP-2026-06-03-000315`)
Verified 2026-07-13: children-treated custom report **matches** the DIGIT dashboard.
- Covered/treated = `ADMINISTRATION_SUCCESS` AND `Data.additionalDetails.doseIndex==1`
  = 1,266,299 (report) vs dashboard 1,266,364. The persistent ~65 gap = success docs with
  `doseIndex` null/≠1 the dashboard counts but the dose-1 filter drops (NOT timing/dupes).
- `campaignNumber` alone spans BOTH cycles. Filter cycle 2 on
  **`Data.additionalDetails.cycleIndex.keyword` = "02"** (NOT `Data.cycleIndex`). Cycle-2 raw
  index = 3,915,664 docs.
- Index carries status **`DELIVERED`** (~2.53M ≈ 2 per treated child) = resource-delivery task
  type; the children-treated report correctly excludes it. Treated metric = ADMINISTRATION_SUCCESS.
- Redose SPAQ1 354 / SPAQ2 492 exact. Ineligible 104,089. Not-administered categories tiny.
- SPAQ↔age (ko): **SPAQ 1 = 3-11mo, SPAQ 2 = 12-59mo**; productVariants
  SPAQ1 `PVAR-2025-05-28-000023`, SPAQ2 `PVAR-2025-05-28-000024`. Found 7 drug/age swaps → 0
  after fix; plus 8,950 SPAQ2 given to >59mo (out-of-window).
- Missing-HH-head rule (44 blanks): OIC referrals = VALID (facility, no household context);
  CDD referrals = unlinked at source (unrecoverable); CDD administrations = recoverable via the
  household-member chain. Missing child name "nan" = source junk (empty name serialized as "nan").
- Read the huge 2-part exports with **python-calamine** (openpyxl too slow); pad rows to 25.
- Report ran short of dashboard by 65 because its `@timestamp` window `start_date=2026-07-02`
  clipped 65 cycle-2 dose-1 records synced before Jul 2 (dashboard scopes by cycle from Jun 4).
  ES cycle-2 dose-1 = 1,266,375. Fix: widen start_date to 2026-06-04 OR scope by
  `cycleIndex.keyword="02"` and drop the date window. See reporting.md date-window gotcha.
Scripts (`Downloads\kogi`): `children_treated_ko.py` (report, with the join fix),
`age_product_mismatch_ko.py` (detect swaps), `update_age_variant_ko.py` (fix age or variant).

---

## FCT / Abuja — `ab` (SMC individual recovery; also reporting)

| Field | Value |
|---|---|
| API base | `https://fct-abuja-hcm.digit.org` |
| Auth token (last-known) | `<REDACTED-TOKEN>` |

Tracer extraction: 28,406 unique individuals from 43,633 tracer docs. Error breakdown:
None 15,164 | INVALID_BENEFICIARY_ID 10,293 | INVALID_USER_ID 8,574 |
DUPLICATE_SUB_ENTITY 6,694 | DUPLICATE_ENTITY 2,902. Search: 2,926 already in HCM,
25,480 missing. Scripts: `ab_individual_create_errors.py`, `ab_individual_search.py`,
`ab_individual_create_push.py` (50 workers, input `ab_individual_search_20260619_1545.xlsx`
sheet=Missing). Note: `ab` is also the FCT reporting tenant (see reporting.md).

### Nigeria SMC individual-recovery shared facts (oy/na/ko/ab)
Goal: recover missing Individual records to fix null HH-head names and missing
Beneficiary IDs in the CHILDREN_TREATED report.
- Lat/lon source is `Data.latitude`/`Data.longitude` in the project-task index only
  (CDD GPS at submission). Individual `address.latitude` is null in Oyo. Unrecoverable
  if null at submission.
- DUPLICATE_ENTITY on create = individual exists under a different clientReferenceId
  (same UNIQUE_BENEFICIARY_ID taken) — treat as already-exists, not failure.
- clientReferenceId search returning 0 does NOT mean missing (Kogi).
- Missing Beneficiary ID = null `UNIQUE_BENEFICIARY_ID` in the tracer payload —
  re-creating the individual cannot recover it.
- Step 2 still needed: creating individuals alone does NOT fix HH-head names — a
  HouseholdMember create push is a separate required step for all states.
- Tracer URL filters: individuals `/individual/v1/bulk/_create`; project tasks
  `/project/task/v1/bulk/_create`.

---

## Ondo ITN — `od`

| Field | Value |
|---|---|
| Campaign | Ondo State ITN distribution |
| API base | `https://ondo-hcm.digit.org` |
| Auth token (last-known) | `<REDACTED-TOKEN>` |
| Date range | 2025-09-13 → 2025-10-01 |

The gap: dashboard bednets 3,562,724 vs report 3,560,866 = -1,858.

Root cause: 3,718 ADMINISTRATION_SUCCESS tasks in `od-project-task-index-v1` have
`Data.householdId = null` (Sep 20-21; indexer processed tasks before household DB
writes completed, wrote null, never backfilled). `project_task` has NO `householdid`
column in DB; linkage is
`project_task.projectBeneficiaryClientReferenceId → od.project_beneficiary.beneficiaryclientreferenceid = household clientReferenceId`.

Recovery of the 3,718: householdId recovered (PB in DB) = 3,356 | still null (no PB in
DB) = 362 (= 1,081 bednets, permanently unrecoverable). Of the 3,356: household in DB
= 36; household in ES only (Sep 21 outage) = 3,320. Sep 21 pattern: `od.project_beneficiary`
wrote OK; `od.household` DB write FAILED → household in `od-household-index-v1` only.

Scripts (`ondo issue\`): `OD_project_task_dump_and_patch.py`, `OD_verify_households_api.py`,
`OD_household_search_and_update.py`, `OD_household_create_missing.py`,
`OD_project_task_bulk_create.py`, `OD_es_update_householdId_project_task.py` (DRY_RUN flag);
plus `C:\Users\Salaudeen N\Downloads\ondo data fix\ondo household memner\OD_household_member_create.py`
(bulk creates 20,637 members).

Key files: `OD_project_task_patched_dump_20260615_084650.csv` (3,718 rows);
`OD_HH_ES_details_not_in_DB_20260609_104751.csv` (23,366 HH in ES not DB);
`OD_HM_ES_details_not_in_DB_20260609_105722.csv` (20,637 members);
`OD_PB_ES_details_not_in_DB_20260609_102353 (1).csv` (20,924 PBs in ES not DB).

Execution status (2026-06-16): 36 DB households update DONE (all 202); 3,267 households
create INCOMPLETE (~640 done, ~2,627 remaining); 3,356 project tasks bulk create DONE
(3,356/3,356 OK 202, 0 fail); 20,637 HH members bulk create NOT YET RUN; ES project-task
householdId patch NOT YET RUN (DRY_RUN=True).
Remaining order: (1) finish `OD_household_create_missing.py` (~2,627); (2) run
`OD_household_member_create.py`; (3) set DRY_RUN=False in the ES householdId patch.

Separate issues: 20,924 ES-only PBs (Sep 21; zero overlap with the 3,718; not actioned);
612 households in project-task missing from both DB AND ES entirely (contribute the -1,858).

Ondo HCM API facts: `/household/v1/_search` → `"Households"` (plural);
`/household/v1/_update` → HTTP 202; `/household/member/v1/bulk/_create` → array
`"HouseholdMembers"`; `/project/task/v1/bulk/_create` → array `"Tasks"`, returns 202;
`/project/beneficiary/v1/_search` → `"ProjectBeneficiaries"`; member create requires
parent household in DB first (else VALIDATION_ERROR).

---

## Kaduna AZM — `kd`

| Field | Value |
|---|---|
| Campaign | AZM (Azithromycin), Cycle 1, 2026 — NOT SMC/SPAQ |
| Tenant | `kd` |
| Campaign ref ID | `CMP-2026-06-09-000345` |
| Campaign dates | 11–19 Jun 2026 (distribution 12–16 Jun, 5 days) |
| ES task index | `kd-project-task-index-v1` |
| ES sync index | `kd-user-sync-index-v1` |
| productName / projectType | `AZM` / `AZT` |
| Total CDDs (SQL from `kd.project_staff` by ref ID) | 8,487 |
| Total CDDs (dashboard, active only) | 7,402 |

Primarily a reporting campaign — see reporting.md for age bands (1-11 / 12-59 months,
NO SPAQ split), Day 1/2/5 numbers, and the CDD-sync analysis (`kd_never_synced.py`,
join `kd.individual.useruuid = kd.project_staff.staffid`, username NULL in DB → use ES
`syncedUserName`). Also see project_oy_chain_recovery — the `kd` `data_Recovery_Script (1)`
is the 12M+ record chain-recovery template.

---

## Togo MRDN — `tg`

| Field | Value |
|---|---|
| Campaign | MRDN / SPAQ (Cycle 01), 4-day |
| Start | 2026-06-13 |
| ES auth (header, not tuple) | `Basic <REDACTED>` (last-known) |
| ES base | `<REDACTED-ES-URL>` |
| Project type ID | `644c4356-5214-11ee-be56-0242ac120003` |

Separate ES cluster, Basic auth, NO tenant prefix on indices (`project-task-index-v1`,
`project-staff-index-v1`, `user-sync-index-v1`, `individual-index-v1`,
`household-member-index-v1`). Field paths differ from Nigeria — full detail in reporting.md.
Scripts in `C:\Users\Salaudeen N\Downloads\extraction_scripts\tg\`
(`children_treated_tg.py` is the reference implementation of the full name-based DQ chain).

---

## Congo — `cg` (reconciliation test tenant)

Congo UAT (`azure-congo-uat-db`) was the test bed for the reconciliation scripts —
household DB=ES=9,598 after pushing 45 missing. See recovery-scripts.md.

---

## Kebbi — `ke` (automation)

Kebbi (SPAQ drug despite the `nigeria-kebbi-azm` folder name) also delivers ORS-Zinc as
a secondary product. Gender stored in `additionalDetails.gender` (admin-console tenant),
not `Data.gender`. Details in automation-and-structure.md.

---

## Borno — `bo` (SMC, multi-cycle; recon-recovery tool test, Jul 2026)

| Field | Value |
|---|---|
| Campaign | Borno SMC-Nig (multiple cycles across 2025 + 2026 in same ES indices) |
| Tenant / DB | `bo` / `<REDACTED-DB-HOST>` (`ngcentralprd`) |
| 2026 project | `projectId 529ac0ad…` (registration + distribution Jul 2026) |
| 2025 project | `projectId ec6efaa8…` (SMC cycle 03, taskDates Aug 2025) |
| Tool | `D:\DST\scripts\recon-recovery` — see recovery-scripts.md |

Key facts (why bo is the reference case for multi-cycle correctness):
- ES indices hold BOTH the 2025 and 2026 campaigns → date windows do NOT separate
  campaigns; `projectId` is the real key.
- Master data reused: 2026 HouseholdMembers reference 2025 Households (register-once,
  distribute-per-cycle). Confirmed live (member 2026-07-09 → household 2025-07-05, present).
- Single-day recon run (2026-07-09): ProjectTask ghosts=28,246 but ALL `syncedDate`=07-10
  → false ghosts (synced next day, DB createdtime outside the 1-day window). Individual
  ghosts=2,043, index-failed 100; Task index-failed 265; HHMember index-failed 1.
- Chain (all-dates existence confirm): Gap A 3,355→0 real, C 83→0 real (master reuse),
  B 225 real (12 recovered via tracer), D 192 real (25 recovered), E 106,403 real / 0
  recovered = pending/untreated coverage, NOT recoverable — do not push.

---

## Bauchi SMC+VAS — `ba` (dashboard-vs-report, Jul 2026)

Combined **SPAQ + Vitamin A (VAS)** campaign, `CMP-2026-06-22-000401`. Report = 27-col extract
(standard 25 + `Red VAS` + `Blue VAS` yes/no; has HH-head + lat/lon + Administration Status).
FULLY reconciled 2026-07-17:
- SPAQ matches (SPAQ1 409,831 exact / SPAQ2 −1 / total 2,164,461 vs dash 2,164,462).
- VAS matches (total −252); **Red VAS = 12-59, Blue VAS = 6-11** (reverse of WHO — verified by counting).
- SPAQ+VAS both 1,682,413 (+118). Redose EXACT (SPAQ1 4,138 / SPAQ2 15,678, recorded as VISITED status).
- Gender/coverage%/all-20-LGAs match. SPAQ↔age swaps = 0 (cleanest). VAS mismatch ~6.
- Ineligible & Referred cracked via **`additionalDetails.flow`**: dashboard Ineligible = flow `smcDone`
  (53,321 ≈ 53,320); Referred = flow `vasDone` (18,917 exact). Report counted both flows -> over-count.
  Generator fix: scope INELIGIBLE to flow=smcDone (pull it out of the blanket other_statuses clause);
  add flow=vasDone to the hf-referral query. See reporting.md.
- DQ clean: HH-head 30, child name 68 (0 truly empty), GPS 66,801 (3.0%), dupes 73,150 (3.1%).
Scripts (Downloads): `ba_full.py`, `ba_kpi.py`, `ba_inel.py`, `ba_calibrate.py`, `ba_childtreat_FIXED.py`.

## Kebbi SPAQ+ORS — `ke` (dashboard-vs-report + generator fixes, Jul 2026)

SPAQ (SMC) + **ORS-Zinc** (co-administered secondary; appears Argungu-only in ke). Campaign
`CMP-2026-06-29-000421`. Admin-console tenant (gender in `additionalDetails.gender`). Auth:
`kebbi-hcm-central.digit.org`; task-write needs an admin-class (`<REDACTED-USER>`-class) role (the dashboard-only account `<REDACTED-USER>` won't do,
401 on task APIs). API base for recovery = `kebbi-hcm-central.digit.org`.
- Dashboard SPAQ = 1,441,600 (dose-1 DOCS). Report distinct children ~1,400,584 ≈ ES distinct
  1,390,630 -> the −2.7% is DUPLICATE dose-1 docs (~3.7% same-child re-doses), NOT timing/clip/LGA;
  report is the more accurate unique-child count. (I mislabeled it "timing"/"Argungu" twice before
  verifying docs-vs-cardinality — always run that check.)
- ORS-Zinc = separate `DSS_HEALTH_ORS` tab; the ORS Summary CSV export was PARTIAL (40 Argungu HFs =
  40,292) so ORS not fully reconciled; report ORS = 80,440 (Argungu-only). Need full ORS total.
- SPAQ↔age swaps = 16 (ES productName query; corrected extractor `ke_spaq_mismatch_FIXED.py` excludes
  ORS-Zinc which the broad query wrongly counted as 68,400). DOB-derived age decides UPDATE_AGE vs
  UPDATE_VARIANT; record with DOB<3 = manual.
- Generator (`ke_children_treated_FIXED.py`) fixes: (1) multi-product flag-reset bug -> create-once +
  accumulate (SPAQ+ORS overlap 0 -> 66,135); (2) added HH-head + lat/lon + Child Beneficiary ID
  (field = `beneficiaryId` NOT `uniqueBeneficiaryId`); (3) Child Name/HH-head = individual-index FIRST
  then `additionalDetails.childName`/`headName` fallback (don't let the chain overwrite with blank);
  (4) Excel >1M rows -> split across sheets. DQ (pre-fix file): HH-head 713 all CDD_KB, child name 34
  (31 empty), GPS 21,105 (1.5%), dupes 5,027 — re-run fills HH-head/child-name/beneficiary-ID.
Scripts (Downloads): `ke_check.py`, `ke_full.py`, `ke_final.py`, `ke_ors_recon.py`, `ke_children_treated_FIXED.py`,
`ke_spaq_mismatch_FIXED.py`.

### Bauchi (ba) SMC + VAS — additions (Jul 2026)
- **VAS variants:** Blue VAS `PVAR-2025-05-22-000009` (6-11), Red VAS `PVAR-2025-05-22-000010` (12-59);
  parent product `P-2025-05-22-000008` ("VAS"). BOTH variants are **soft-deleted (isDeleted=true)** ->
  every VAS task `_update` fails `NON_EXISTENT_RELATED_ENTITY` (see context/gotchas). Restore first to fix.
  API base `bauchi-hcm.digit.org` (public); token resolves to an admin account (`<REDACTED-USER>`).
- `ba_childtreat_FIXED.py` now: **VAS counted PER-DOC** (each Red/Blue VAS doc = its own row, duplicates
  kept -> Red 1,573,967≈dash 1,573,958, Blue 166,710≈166,703; VAS total exact 1,740,677) + **`Household ID`**
  column (`additionalDetails.householdClientReferenceId` -> resolved to server `id` via ba-household-index-v1,
  HH-member index fallback). Report ~4.0M rows (SPAQ 2.16M + VAS 1.74M as own rows). SPAQ+VAS "both" =
  triple-grain (individual+user+cycle) 1,682,478 ≈ dash 1,682,295.
- **Recon 2026-07-20:** SPAQ, VAS (total+gender exact), redose, not-administered (exact), ineligible 53,321,
  referred 18,917 all match within same-day drift. Per-LGA SPAQ outliers Toro/Darazo/Dambam/Zaki =
  `@timestamp`(report) vs `taskDates`(dashboard) sync-lag, not error.
- **VAS age mismatch = 16 age typos** (9 Red + 7 Blue), DOB-confirmed valid deliveries (0 mis-dose); after
  age-fix re-run, Red/Blue outside-band -> 0. DQ (SPAQ rows): HH-head 30, child-name 65 (all `null`/`nan`
  source-nulls from a few CDDs), GPS 0.41%, gender 0.
- Scripts (Downloads): `ba_childtreat_FIXED.py` (report gen, VAS per-doc + Household ID), `ba_vas_age_mismatch.py`
  (DOB verify), `ba_vas_mismatch_fix.py` (UPDATE_AGE/UPDATE_VARIANT), `Bauchi SMC Reconciliation Report.xlsx`.

---

## Chad POLIO (`chad` — SIA enumeration + vaccination, 2026-07 analysis)

DISTINCT from Chad SMC (`ch`). Polio uses the **`chad-`** index prefix (chad-project-beneficiary-index-v1,
chad-project-task-index-v1, chad-individual-index-v1, chad-household-member-index-v1,
chad-user-sync-index-v1). Standalone scripts hit `https://elasticsearch-data.es-cluster:9200` with Basic auth
(`elastic:...`) — a different host string than the SMC `es-cluster-v8`. All `chad-` docs nest under `Data.*`
(NOT flat like the SMC `ch-` project-beneficiary).

**Campaign:** `CMP-2026-05-29-000091` ("POLIO 20th May Campaign", the June-3..9 wave). `projectTypeId
2a1bb2e7-06d8-4fe4-ba1e-f4a6363a2344` is shared by ~5 POLIO rounds (also CMP-...090/083/093/097) — so scope
enumeration by **campaignNumber**; vaccination by **projectTypeId** only if you want cross-round doses (else
campaignNumber). project-task DOES carry campaignNumber for this campaign.

**Data model (two activities, different indices):** ENUMERATION = `project-beneficiary` (registration);
VACCINATION = `project-task` (delivery). Link = `task.projectBeneficiaryClientReferenceId` <->
`beneficiary.projectBeneficiary.clientReferenceId`; `task.individualId` == child individual ref ==
`beneficiary.projectBeneficiary.beneficiaryClientReferenceId`. project-beneficiary holds the WHOLE household
(heads + members); `additionalDetails.isHeadOfHousehold`, `ageInMonths`. **Eligible child = non-head AND
ageInMonths<=59.** Counts: all beneficiaries 111,530; non-head members 73,752; eligible 0-59m **52,944**.
Vaccination target = 0-59m only, so enumeration >> vaccination is by design (adults enumerated, not vaccinated).

**Date fields (critical):** `syncedDate` (sync day, string) ; `syncedTimeStamp` (ISO, date-typed -> use for
hourly `date_histogram` with `time_zone`) ; `createdTime` (epoch ms, mapped **long** -> `date_histogram`
works but REJECTS a custom time_zone) ; `projectBeneficiary.clientAuditDetails.createdTime` = DEVICE
record-created time (~= activity day) ; `auditDetails.createdTime` = SERVER = ~= syncedDate ; `taskDates` =
activity date. Offline: device createdTime != server createdTime. **Sync-date analysis -> syncedDate;
device/"when created" analysis -> clientAuditDetails.createdTime.** Chad = WAT (+01:00).

**Statuses:** ADMINISTRATION_SUCCESS, VISITED (also delivered), ADMINISTRATION_FAILED (refused), DELIVERED,
CLOSED_HOUSEHOLD, NOT_ADMINISTERED. **`Data.isDelivered` (bool) = the clean "dose given" flag** (VISITED &
SUCCESS both true; FAILED false). **Count "vaccinated" as DISTINCT children with isDelivered=true, NOT task
records.** Geo: household-member `Data.geoPoint` `[lon,lat]` = enumeration location (beneficiary has none in
additionalDetails); task has additionalDetails.latitude/longitude/lat/lng + geoPoint + `latLng` string
("lat, lon, accuracy") used on ADMINISTRATION_FAILED (top-level lat/long null there).

**The 3 analyses + reconciliation (proven):**
- Analysis 1 = by **sync date** ; Analysis 2 = by **device/created date** ; must use the SAME cohort + same
  "vaccinated" definition or they won't reconcile. Cohort 52,944 = Vaccinated 40,309 + Failed 152 + Not-done 12,483.
- **Record-vs-child gap = 2,787**: old raw task count 43,096 = 40,309 distinct delivered + 743 duplicate
  delivered records (same child >1 task) + 1,732 deliveries to non-cohort children + 312 refused RECORDS.
- **Failed 312 vs 152**: 312 = raw ADMINISTRATION_FAILED records; **152 = distinct cohort children refused &
  never vaccinated** (the actionable coverage loss). Use 152 in both reports to match. The 743 duplicate + 1,732
  non-cohort are record-level (A1) only — they CANNOT go in A2's per-child matrix (non-cohort kids have no row;
  duplicates are real doses -> would mislabel/double-count).
- Analysis 3 (duplicates) key = `username|age|hh-head-name|child-name` (+ optional `|geolocation`); split
  enumeration-dups vs vaccination-dups; "duplicate" = 2nd+ occurrence of a key.

**Standalone script pattern (fast):** self-contained ES_BASE+AUTH_HEADER+scroll (no COMMON_UTILS), **NO
per-record decrypt** (egov-enc-service call is the big slowdown; skip it — names only), bulk `terms` 8000/batch,
distinct via `cardinality`, hourly via `date_histogram` on `syncedTimeStamp`. Scripts in Downloads:
`chad_polio_analysis1_fast.py` (sync date), `chad_polio_analysis2_fast.py` (device date, Done/Failed per date),
`chad_polio_analysis3_fast.py` (dups + 3.1 detail), `chad_polio_hourly_3to10june.py` (per-day hourly sheets).
