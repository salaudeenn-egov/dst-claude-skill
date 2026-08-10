# Reporting — daily campaign reporting pipeline, formulas, numbers, style rules

Covers the manual per-day pipeline (extract, combine, analyze, report), the 4-slot
Google-Docs structure, the treatment-rate / duplicate-key / performance-band formulas,
the per-drug age bands (SMC/SPAQ vs AZM vs Togo MRDN), python-docx helpers, and the
recorded per-day numbers for FCT, Plateau, Kaduna, and Togo. For the fully automated
version of this pipeline see automation-and-structure.md.

Rule of thumb: always report the analysis-script numbers, NOT the dashboard numbers
(confirmed FCT Day 1: 152,328 script vs 160,379 dashboard; dashboard is wrong).

---

## Pipeline (per state, run in order; each step a standalone Python script)
1. combine: read all per-facility xlsx into one combined xlsx (one sheet per facility)
2. analyze: iterate individual xlsx, compute metrics, write notes txt + per-facility txt
3. excel: generate performance Excel (all facilities, tabs filtered by band)
4. report: generate Word .docx (sections below)
5. (multi-state) merge state docs into one combined report

The generalized 5-script form (also used by automation): `extract.py`, `combine.py`,
`analyze_day.py`, `cdd_sync.py`, `generate_report.py`.

Script naming convention (Downloads):
- extraction `extraction_scripts\{tenant}\children_treated_{tenant}.py`
- combine `combine_{tenant}_day{N}.py`
- analyze `analyze_{tenant}_day{N}.py`
- report `generate_kd_azm_day{N}_report.py` (KD), `generate_day{N}_report.py` (FCT+PL combined)
- CDD sync `kd_never_synced.py` (most complete, use as template), `{tenant}_cdd_sync_status.py`

Excel header-row gotcha: SMC Day 3 Excels have a title row at row 1, headers at row 2,
data at row 3+, so read with `min_row=3`. Day 2 Excels and Kaduna Day 2 Excel use
`min_row=2` (no title row). Always check before reading with openpyxl.

Big-file read gotcha: for multi-lakh CHILDREN_TREATED exports (1M-row parts, 180MB+),
openpyxl read_only is far too slow (~100s just to OPEN). Use **python-calamine**
(`CalamineWorkbook.from_path(fn).get_sheet_by_name("Sheet1").to_python()`, ~40s) and
pad each row to the column count (read_only/calamine trim trailing empty cells).

CHILDREN_TREATED household-head JOIN BUG (found ko 2026-07-13, applies to every tenant):
the terms lookups set ES `"size" == batch_size`, so a batch of N ids that matches >N docs
(ES duplicates / multi-membership) is silently truncated -> some individuals lose their
household/head mapping -> blank Household Head Name, nondeterministic. FIX: never let a terms
query truncate -> if `len(hits) >= size`, split the id-list and re-query (helper
`es_terms_fetch` in `Downloads\kogi\children_treated_ko.py`). Same class of bug can blank
Child Name. Interpreting blank HH-head: OIC referral = valid (no household); CDD referral =
unlinked at source; CDD administration = real, recoverable via the member chain.

doseIndex gotcha: CHILDREN_TREATED counts an administered child only on
`ADMINISTRATION_SUCCESS` AND `Data.additionalDetails.doseIndex==1` (one row per child, not per
dose). The raw all-dose ES success count is higher; that difference is the expected report-vs-
dashboard gap, not an error. Dashboard age bands are by DOSE/product, not literal age.

Date-window-clips-early-syncs gotcha (why report < dashboard; found ko 2026-07-13): the report
scopes by a `Data.@timestamp` (ingest-time) window from `get_custom_dates_of_reports()`, NOT by
cycle. If `start_date` is set to the campaign's *distribution* start (e.g. the day the bulk sync
began), it CLIPS records that synced BEFORE that date but belong to the cycle — the dashboard
counts them (it scopes by cycle, cumulative), so the report comes up short. Confirmed TWICE: ko
start_date 2026-07-02 dropped 65 cycle-2 dose-1 (report 1,266,299 vs ES 1,266,375); na
start_date 2026-07-02 dropped 427 (report 1,058,861 vs dashboard 1,059,288), concentrated in
early-sync LGAs (Toto/Nasarawa Eggon/Karu/Doma). To diagnose: count with
`@timestamp < start_date` and `> end_date` — the "before" bucket is the missing records.
Fix: widen `start_date` to the dashboard's range start, OR (robust) scope by
`Data.additionalDetails.cycleIndex.keyword` and drop the @timestamp window so it always equals
the full cycle population like the dashboard. (Distinct from the tiny live-vs-snapshot lag of a
few records between report-generation and dashboard read.)

---

## 4-slot Google-Docs structure (agreed 2026-06-25)
4 separate Google Docs, one per slot; each OVERWRITES itself (clear + rewrite) at its
scheduled time; no appending, no scrolling. Doc IDs are permanent (partner links never
change). Every doc carries `Data as at: DD Month YYYY, HH:MM` at the top.

| Slot | Time | Audience | Content |
|---|---|---|---|
| Morning | 11am | Internal | Overnight snapshot |
| Midday | 2pm | Internal | Midday snapshot |
| Evening | 5pm | Internal | Afternoon snapshot |
| Final | 8pm | Partners (external) | That day's complete final report |

Only the 8pm FINAL goes external; the rest on request. Report times in the scheduler
are UTC.

Overwrite mechanism:
```python
service.documents().batchUpdate(documentId=DOC_ID, body={
    "requests": [{"deleteContentRange": {"range": {"startIndex": 1, "endIndex": end_index}}}]
}).execute()
# then write fresh content
```

---

## Formulas (apply everywhere)

Treatment / coverage rate:
```
treatment_rate = treated / daily_target * 100
daily_target   = individual_target / 4          # 4-day campaign
treated        = records where age-in-band AND Ineligible=no AND Referred=no AND Qty >= 1
```
NOT treated / total_records (old wrong formula). If daily_target = 0, show "NO TARGET";
do not compute a rate.

Duplicate detection key (4 fields; always use all four):
```python
key = (child_name.lower(), ward.lower(), age, household_head_name.lower())
```

Performance bands (4 bands):
| Band | Threshold | Text color |
|---|---|---|
| LOW | < 33% | Red |
| MODERATE | 33-66% | Dark orange |
| ON TRACK | 66-90% | Black |
| HIGH | >= 90% | Green |
(Old 50/70 thresholds and "ACCEPTABLE" label are gone.)

---

## Age bands per drug (they differ; do NOT mix)
- SMC / SPAQ: treated range 3-59 months. SPAQ1 = 12-59m, SPAQ2 = 3-11m. Split is
  qty-based (qty 1 = SPAQ2, qty 2 = SPAQ1 in the old logic).
- AZM (Kaduna, `kd`): single drug, NO SPAQ split. Treated range 1-59 months. Age-based
  split: `azm_1_11` (age <= 11), else `azm_12_59`. Labels `AZM 1-11m` / `AZM 12-59m`.
- Togo MRDN (`tg`): SPAQ 0-3m (0-2m), SPAQ1 3-11m, SPAQ2 12-59m.

---

## DQ audit rules (what each field means)
- Missing Child Name: individual index `name.givenName` + `name.familyName` empty
- Missing HH Head Name: resolved via the 3-step chain (child individualClientReferenceId
  to householdClientReferenceId via household-member index, to head member
  `isHeadOfHousehold=True`, to head individualClientReferenceId, to name via individual index)
- Missing Gender: check BOTH `Data.gender` and `additionalDetails.gender`; location is
  tenant-specific (most tenants `Data.gender`; Kebbi/admin-console `additionalDetails.gender`)
- Missing Lat/Lon: `Data.latitude`/`Data.longitude` (Nigeria) with `additionalDetails.*` fallback (Togo)
- Age=0 / Age>59: `Data.age`
- Duplicates: 4-field key above
- deliveryComments: added between "Duplicates" and "Wards" (index 23 in PERF_COLS);
  analyze scripts count non-empty values (typically all "REDOSE")

---

## Campaign structure (SMC vs AZM)
| Campaign | States | Drug | Notes |
|---|---|---|---|
| SMC | Plateau (`pl`), FCT (`ab`) | SPAQ | combined report PART A Plateau, PART B FCT |
| AZM | Kaduna (`kd`) | Azithromycin | separate report |
| MRDN | Togo (`tg`) | SPAQ | separate ES cluster |

Input data locations: Plateau `children_treated_all (1)\` prefix `pl_`;
FCT `FCT_CHILDREN_TREATED_2026-06-11\` prefix `ab_fc_`, target `fct target.csv`;
Kaduna `children_treated_all kaduna og day2\`, target `kaduna target.csv`;
Togo `extraction_scripts\tg\`, target `togo daily report\togo target.csv`.

---

## Word report structure (SMC combined)
National sections first: National Executive Summary; Combined National Overview Table
(Plateau | FCT | Combined); Critical Issues Comparison Table. Then PART A Plateau, PART B FCT.

Per-state sections KEEP: 1 Executive Summary (purpose only, not a number recap);
2 Operational Overview table; 3 Issues Log (table); 4 Distribution Analysis
(4.1 LGA Summary, 4.2 Top/Least Performing: Table A by rate, Table B low activity <20 rec,
4.3 Below-Average LGAs, 4.5 Non-Administration, 4.6 Duplicates AS TABLE);
5 Sync Status; 6 Anomaly Summary (DATA QUALITY ONLY); 7 Conclusion (max 5 sentences).
REMOVE: 4.4 Age/SPAQ compliance, 4.7 Missing Data standalone, 4.8 Redose standalone.
Anomaly table keeps: qty gap, duplicates above threshold, missing HH, missing child,
age=0, age>59. Removes: LGA treatment rates, unsynced FLW %.

AZM Kaduna report structure: Title "Kaduna State AZM"; Subtitle date range;
Section 1 Day N Operational Overview; 2 System Issues and Resolutions;
3 Distribution Data Analysis (3.1 Performance by LGA, 3.2 Data Quality by LGA,
3.3 Facility Performance, 3.4 DQ Summary); 4 Conclusion; 5 HF Data Sync Status
(5.1 FLW Data Sync Summary, Day N).

---

## Word style rules (see also feedback-doc-style in hcm-context / memory)
- No bookmarks on headings or paragraphs; no emojis; no decorative unicode
- Colors: dark blue headers `#003366`, mid-blue `#1F4E79`, red flags `#C00000`,
  light-blue rows `#D6E4F0`
- Title block: `style="Normal"` + manual Pt(26) bold; do NOT use `style="Title"` (adds border line)
- Subtitle block: `style="Normal"` + manual Pt(14) bold; do NOT use `style="Heading 2"` (adds border line)
- Headings: set `run.italic = False` after adding the run (Heading 4/5 default to italic)

Excel performance-file style: white background on data rows; header row light gray
`#EEEEEE` bold; totals row light gray bold; only Coverage % and Status columns use
colored text (band table); auto-filter on header; freeze panes at column D row 3;
tabs ALL FACILITIES | LOW | MODERATE | ON TRACK | HIGH | NO TARGET | LOW ACTIVITY.

python-docx helpers (reuse verbatim from `generate_fct_report_v2.py`):
`set_bg(cell, hex6)`, `set_borders(cell)`, `hcell(cell, text)`, `dcell(cell, text)`,
`add_page_break(doc)`, `add_section_banner(doc, title, subtitle="")`,
`copy_doc_body(source_doc, target_doc)` (deepcopy all elements except sectPr).

---

## Recorded numbers

### SMC, FCT (`ab`)
Day 1 (11 Jun, corrected via analyze_fct_v2.py): 268 facilities; daily target 251,485;
total records 201,430; treated 200,083; coverage 79.6%; SPAQ1 159,454; SPAQ2 40,629;
duplicates 7,601; missing HH 6,998; missing child 911; HIGH 121, ON TRACK 74, MODERATE
54, LOW 14, NO TARGET 5. FLW total 3,764. (Old wrong formula: 152,328 rec / 151,284
treated / 99.3%; do not use.) 5 NO TARGET facilities with real data: Kogo PHC (567),
Kuduru PHC (349), Phase 3 Quarters PHC (285), Kpache PHC (280), Pipeline PHC (~241).

Day 2 (12 Jun, 23:00): 274 HFs (100% synced); 6 area councils; daily target 253,052;
records 262,567; treated 261,457; coverage 103.3%; SPAQ1 209,658; SPAQ2 51,799;
not admin 13 (Absent 5, Refused 3, Died 2, Migrated 3); duplicates 10,767; missing HH
11,587; missing child 1,078; HIGH 249, ON TRACK 16, MODERATE 3, LOW 1, NO TARGET 5.

Day 3 (13 Jun, 23:00): 274 HFs (100% synced); 6 area councils all HIGH; daily target
253,052; records 255,128; treated 254,346; coverage 100.5%; SPAQ1 205,592; SPAQ2 48,754;
not admin 22 (Absent 3, Refused 2, Died 2, Migrated 15); redose 0; duplicates 10,444;
missing HH 12,189; missing child 1,775; delivery comments 291 (all REDOSE); HIGH 232,
ON TRACK 30, MODERATE 5, LOW 2, NO TARGET 5. FLW total 3,764; synced 3,650 (97.0%).

### SMC, Plateau (`pl`)
Day 1 (11 Jun, OLD formula, recalc when target file arrives): records 57,920; treated
54,365; rate 93.9% (WRONG); SPAQ1 43,417; SPAQ2 10,948; 499 facilities; 17 LGAs;
duplicates 149; missing HH 2,240. Problem LGAs: Pankshin 32.6%, Kanke 54.3%,
Langtang North 67.4%.

Day 2 (12 Jun, 23:00): 775 HFs (879 registered, 88.2% synced); 17 LGAs; daily target
249,351; records 255,779; treated 240,055; coverage 96.3%; SPAQ1 190,595; SPAQ2 49,460;
not admin 27 (Absent 7, Refused 9, Died 2, Migrated 9); redose 727; duplicates 260;
missing HH 11,697; missing child 1,623; HIGH 521, ON TRACK 114, MODERATE 73, LOW 67,
LOW ACTIVITY 7. Problem LGAs: Pankshin 33.0%, Kanke 61.5%, Langtang North 65.8%.

Day 3 (13 Jun, 23:00): 783 HFs (879 registered, 89.1% synced); 17 LGAs; daily target
251,063; records 242,707; treated 229,415; coverage 91.4%; SPAQ1 184,460; SPAQ2 44,955;
not admin 14 (Absent 7, Refused 5, Migrated 2); redose 630; duplicates 228; missing HH
13,042; missing child 1,848; age>59 26, age=0 4; delivery comments 492 (all REDOSE);
HIGH 468, ON TRACK 147, MODERATE 98, LOW 70, LOW ACTIVITY 8. Problem LGAs: Pankshin
34.0%, Kanke 60.0%, Langtang North 56.2% (regression from 65.8%). FLW total 3,977;
synced 3,320 (83.5%).

### AZM, Kaduna (`kd`)
Day 1 (11 Jun, 23:00): 1,065 HFs registered, 926 synced (86.9%), 139 not; 23 LGAs;
daily target 616,203; records 368,480; treated (1-59m) 368,040; coverage 59.7%;
AZM 1-11m 105,259; AZM 12-59m 262,781; not admin 22,034 (Absent 78, Refused 214, Died
80, Migrated 68); redose 2,790; duplicates 592; missing HH 9,345; missing child 642;
HIGH 76, ON TRACK 261, MODERATE 377, LOW 160, NO TARGET 52, LOW ACTIVITY 38. FLW total
7,402; Day 1 sync (facility-rate proxy) 6,435 (86.9%); present as a number, not "estimate".

Day 2 (12 Jun, 23:00): 1,065 HFs, 1,052 synced (98.8%); 23 LGAs; daily target 668,569;
records 557,142; treated 557,002; coverage 83.3%; AZM 1-11m 155,810; AZM 12-59m 401,192;
not admin 139 (Absent 21, Refused 75, Died 31, Migrated 12); redose 3,697; duplicates
850; missing HH 13,265; missing child 1,388; HIGH 319, ON TRACK 406, MODERATE 229, LOW
42, NO TARGET 56, LOW ACTIVITY 7. FLW total 7,402; synced 6,454 (87.2%).

Day 5 (16 Jun, final distribution day): daily target 521,940; treated 170,322; coverage
32.6%; total CDDs 6,090; FLWs synced 3,361; FLW total (report) 7,402; synced (report)
3,101; rate 41.9%. Bands: HIGH 55, ON TRACK 78, MODERATE 187, LOW 307, NO TARGET 1,
LOW ACTIVITY 64. Files: `kd_performance_v2.xlsx`, `AZM_2026_Cycle1_Day5_Kaduna_Report.docx`,
DQ `kd_data_quality_report (2).xlsx`.

CDD sync analysis (Kaduna): `kd_never_synced.py` (ES + SQL) to `kd_cdd_sync_status.xlsx`
(SUMMARY + 23 per-LGA tabs + NEVER SYNCED + LOW SYNCED); `kd_cdd_individual.csv`.
SQL joins `kd.project_staff` to `kd.project` to `kd.project_address` (boundarytype
'Health Facility') to `kd.individual` on `useruuid = staffid`, filtered by ref ID
`CMP-2026-06-09-000345`. `kd.individual.username` is NULL for CDDs, so use ES
`Data.syncedUserName`. Date filter must use `terms` (not `range`) on `Data.taskDates`.
Username agg: `Data.syncedUserId.keyword` size=30000. Numbers: synced during campaign
~8,036; never synced ~451; synced Day 4 (15 Jun) ~7,646.

### MRDN, Togo (`tg`)
Day 1 (2026-06-18): facilities with data 10/86; daily target 38,675 (campaign total
154,705 divided by 4); treated 117 (0.3%); CDDs synced 201/922 (21.8%). Best district ogou
(127/280, 45.4%); worst est mono (6/181, 3.3%). Districts: amou, anie, est mono, ogou
("Ogou" capitalisation variant in staff index; normalise to lowercase).

Togo field paths (separate ES cluster, no tenant prefix): boundary
`Data.boundaryHierarchy.district` (NOT lga); lat/lon `Data.additionalDetails.latitude/longitude`
(NOT `Data.latitude`); child name `Data.additionalDetails.name`; beneficiary ID
`Data.additionalDetails.uniqueBeneficiaryId` (plain text, no decrypt); CDD role
`COMMUNITY_DISTRIBUTOR`; `projectTypeId` in sync index at `Data.additionalDetails.projectTypeId`;
no `campaignNumber` field, filter by `projectTypeId` + `role`. Scripts (tg):
`children_treated_tg.py`, `tg_cdd_sync_status.py`, `tg_performance_report.py`
(`CAMPAIGN_DAYS=4`), `generate_tg_mrdn_report.py`.

---

## Dashboard-vs-Report reconciliation (methodology, proven ab/pl/bo/ba/ke, Jul 2026)

Goal: confirm the custom CHILDREN_TREATED report equals the DIGIT dashboard, explain any gap,
run DQ + drug/age-mismatch. Workflow: read the extract (python-calamine) -> reconcile counts by
PRODUCT -> DQ -> drug↔age mismatch -> explain the delta. Order of likely causes for a delta:
date-window clip, dashboard-flow-scoping, docs-vs-distinct dedup, live sync-timing.

Reconciliation gotchas (each cost real debugging):
- **Band by PRODUCT, not literal age.** Dashboard "Population Covered (3-11)/(12-59)" counts by
  productName (SPAQ 1 / SPAQ 2), NOT the child's literal age. Banding by age looks short by the
  over-age volume (na 51,624 / ab 33,782 / pl 49,608 SPAQ2 given to >59mo). Match product-to-band.
- **Dashboard counts DOCS, report dedups to children.** Dashboard "population covered" = dose-1
  DOC count; the report usually dedups by individualId. Before calling a delta "timing", run
  `value_count` (docs) vs `cardinality` (distinct individualId) on ES: ke dose-1 docs 1,441,600 =
  dashboard but distinct children ~1,390,630 = report -> the 41k "gap" was DUPLICATE dose-1 records
  (~3.7% same-child re-doses), report is the MORE accurate figure. Docs/child > 1.0 = a DQ finding.
- **Date-window clip** (ko 65, na 427, ab 120 ES-proven, pl 191): report scopes by `Data.@timestamp`
  start_date; dashboard is cumulative-by-cycle. Records synced before start_date get clipped. Fix:
  scope by `Data.additionalDetails.cycleIndex.keyword` and drop the @timestamp window. To test:
  count `@timestamp < start_date` for the cycle = the clipped bucket. (Distinct from live snapshot lag.)
- **Live snapshot timing** (bo −2.9%): active campaign, report pulled before dashboard read; ES live
  > dashboard > report, monotonic; date histogram shows a collapsing recent-day tail. Re-pull closes it.
- **The `additionalDetails.flow` field scopes dashboard KPIs** (ba, decisive). Combined SPAQ+VAS
  campaigns tag each task `flow=smcDone` (SMC/SPAQ program) or `flow=vasDone` (VAS program). Dashboard
  "Beneficiary Ineligible" counts flow=smcDone ONLY; "Referred children" counts flow=vasDone ONLY.
  Report counting both flows over-counts (ba ineligible 78,271 both vs 53,321 smcDone; referred 33,447
  both vs 18,917 vasDone). ALWAYS aggregate a mismatching status/KPI `by additionalDetails.flow.keyword`.
- **VAS Red/Blue → age band must be verified EMPIRICALLY, not assumed.** ba: Red VAS=12-59, Blue VAS=6-11
  (REVERSE of the WHO 100k/200k colour convention). Count Red/Blue age distributions before mapping. VAS
  is 6-59mo (not 3-59 like SPAQ).
- **CHILDREN_TREATED generator multi-product flag-RESET bug** (ke Script 2, ba OK). When a script builds
  one row/child with per-product yes/no flags (SPAQ1/SPAQ2/ORS-Zinc or Red/Blue VAS), if the record is
  re-created per product doc instead of created-once-then-accumulated, only the LAST product survives.
  Fingerprint: **ZERO overlap** between co-administered products (SPAQ+ORS overlap 0 = impossible ->
  bug). Fix: create the child record ONCE (`if iid not in child`), then accumulate flags; never reset.
  After fix ke SPAQ+ORS overlap 0 -> 66,135.
- **Ineligible/Referred over-count from no-dedup + status overlap.** Non-success statuses often captured
  as independent task docs (not merged like success), and a child successfully administered one product
  but ineligible/referred for another appears in BOTH buckets. Combined with flow-scoping (above), this
  is why those two KPIs run high. Coverage KPIs (SPAQ/VAS/redose/gender/LGA) usually match.
- **LGA label case mismatches** (Danko/Wasagu vs Danko/wasagu, Koko/Besse vs Koko/besse, Itas/Gadau vs
  Itas/gadau, Tafawa-Balewa vs Tafawa-balewa) — compare case-insensitively before flagging an LGA gap.
- **Re-measure per-LGA on the CURRENT file.** A stale prior extract's per-LGA numbers wrongly localized a
  gap to one LGA (ke "Argungu −44k" was the old buggy file; new file Argungu reconciled at −115).
- **Redose** may be recorded as `VISITED` status with `Redose Quantity Administered > 0` (ba redose
  SPAQ1 4,138 / SPAQ2 15,678 = dashboard exact; VISITED count = redose total).
- **Not in the 25-col extract:** if the report is the buggy 20-col / 13-col schema, Household Head Name
  and lat/lon may be absent -> HH-head & GPS DQ not checkable; use the 25-col productName extract or add
  the columns. Child Beneficiary ID field is tenant-specific: ke uses `additionalDetails.beneficiaryId`
  (NOT `uniqueBeneficiaryId`, which returns blank).
- **Name/HH-head fallback order** = individual index FIRST, then project-task `additionalDetails.childName`
  / `headName` as fallback. Bug to avoid: the household-chain step overwriting HH-head with blank when the
  chain fails to resolve, wiping the additionalDetails.headName fallback (ke 713 "missing" HH-heads).

### HCM API auth for data-fix pushes (task _search/_update)
`_update`/`_search` on project/task need a user with an OPERATIONAL role (SUPERUSER /
SYSTEM_ADMINISTRATOR / CAMPAIGN_ADMIN). A dashboard/console user (roles DASHBOARD_VIEWER /
NATIONAL_SUPERVISOR, e.g. bo `BO_MC_002`) authenticates (200) but the service returns **401
"You are not authorized to access this resource"** — scripts that swallow errors mis-report this as
"task not found" (tell: ~7s/task = 3 failed retries w/ backoff). Use a task-authorized account
(bo `<REDACTED-USER>`). Get the token at runtime from `{tenant}-hcm.digit.org/user/oauth/token` (or
`kebbi-hcm-central.digit.org`), `Authorization: Basic <REDACTED — OAuth client header>`, grant_type
password, tenantId, userType EMPLOYEE. **productVariantId is tenant-specific — never reuse another
tenant's PVAR ids** (a Kogi-token/Kogi-variant script pointed at Borno = wrong host + would corrupt
data). Excel caps 1,048,576 rows/sheet — split across sheets or files for >1M rows.

### Anti-patterns / verification discipline (learned the hard way, ba/ke Jul 2026)
Every one of these was a WRONG call I made and had to retract — verify, don't assume:
- **Don't label a report-vs-dashboard delta "timing" without proof.** Run `value_count` (docs) vs
  `cardinality` (distinct individualId) in ES FIRST. Kebbi −2.7% was called "timing" twice; it was
  actually docs-vs-distinct DEDUP (dashboard counts dose-1 DOCS 1,441,600; report dedups children
  ~1,400k). Dashboard "population covered" = docs, not unique children.
- **Re-measure on the CURRENT file every time; never reuse a prior run's number.** Carried a stale
  duplicate count (73,150 from old pre-fix files) into a report for new files (real 68,384). Also the
  "Argungu missing 44k" was the OLD buggy extract; the new file reconciled (−115).
- **Don't assume a cause you haven't seen** (e.g. "it's a time filter"). The 30 "missing HH-head" were
  NOT a time-filter/join failure — the report literally holds the junk values `"null"`(23)/`"Nan"`(7).
  A DQ "missing" check that treats the strings 'null'/'nan'/'none' as blank will flag junk-but-PRESENT
  values as missing. ALWAYS check the raw cell value (empty vs junk string) before calling it missing,
  and check ES (individual exists? household-member link exists? head resolves?) before "unrecoverable".
- **Pick ONE basis for a total and stay consistent.** VAS total flip-flopped −234 (Red+Blue band) vs
  −252 (gender/recipients). They differ because (a) the DASHBOARD is internally inconsistent — its
  Red+Blue band-sum ≠ its headline/gender total (ba: off by 16), and (b) report dual-product children
  (got both Red+Blue) counted twice in the band sum. Reconcile band-to-band OR recipient-to-recipient,
  never cross; and expect the dashboard's own bands vs total to disagree.
- **VAS colour→age band is empirical, per tenant** (ba: Red=12-59, Blue=6-11, REVERSE of WHO). Count
  Red/Blue age distributions; never assume.
- **Excel: write numeric cells (int/float), not strings.** Blanks = None; don't stringify numbers.
- **A "mismatch/duplicate" flag on one child is often the tip of a duplicate-administration pattern**:
  the same child recorded by multiple CDDs with inconsistent ages (ba: one child logged as 6/18/19 mo
  across 3 CDDs). Pull the individual's FULL record set before concluding it's a single mis-dose.
- **The user repeatedly (correctly) forced verification over assumption** — default to running the
  script/ES query and reading the actual value, then state the finding; flag uncertainty explicitly.

### Matching the dashboard = count what IT counts (DOCS), not distinct children (ba/ke Jul 2026)
The dashboard "population covered" counts administration **DOCS**, NOT distinct children. So:
- **To MOVE the report TOWARD the dashboard, count docs (remove dedup). Adding `distinct` moves AWAY**
  (lower). Dashboard is the with-duplicates figure; distinct-children is lower by the duplicate volume.
- **SPAQ vs VAS asymmetry (critical — don't blanket-remove dedup):**
  * SPAQ HAS `doseIndex` -> dashboard = **dose-1 docs** (ba SPAQ1 409,831 / SPAQ2 1,754,631). A dose-1
    per-child report already MATCHES exactly. Counting ALL SPAQ docs OVERSHOOTS (ba SPAQ2 all-dose
    1,759,569 vs 1,754,631 = +4,938 multi-dose repeats). Keep SPAQ on doseIndex=1.
  * VAS has NO doseIndex -> dashboard = **all VAS docs** (ba Red 1,573,958 / Blue 166,703). The report
    dedups VAS (flag per child+CDD+cycle) so it UNDER-counts (Red 1,573,720). To match, count Red/Blue
    **docs** (no dedup): ES Red docs 1,573,967 ~= dash 1,573,958 (+9), Blue 166,710 ~= 166,703 (+7).
- **Residual is live-timing** (+single digits/tens) because report-extract and dashboard-read happen at
  different instants on an active campaign — not fixable exactly.
- **Duplicate root cause (confirmed ba):** the same child recorded by MULTIPLE CDDs (multi-user), often
  across days, sometimes near-simultaneous (two CDDs 25s apart) and with INCONSISTENT ages (one child
  logged 9/21/22mo by 3 CDDs). Operational/field issue (child moving between CDDs or re-recording), not
  a dosing-logic bug. It's exactly why dashboard(docs) > report(deduped). VAS colour flag can be "outside
  band" purely because a different CDD logged a different age -> pull the individual's full record set.
- Practical rule: **reconcile to the dashboard by counting docs (SPAQ=dose-1, VAS=all)**; ALSO keep a
  separate "distinct unique children" line as the accurate coverage/DQ metric (it will read lower).

### VAS age-band "outside band" = age typos; DOB-verify before alarming (ba Jul 2026)
- Dashboard splits VAS by PRODUCT **and recorded age band** (Red=12-59, Blue=6-11); the report counts
  by PRODUCT only. So report Red +9 / Blue +7 vs dashboard = EXACTLY the N records whose TYPED age is
  outside the dose's band (ba: 9 Red + 7 Blue = 16). Dashboard TOTAL still = report total; the dashboard's
  two bands sum 16 SHORT of its own total = those 16. (So the +9/+7 is the age mismatch, not drift.)
- **DOB-verify them** (`ba_vas_age_mismatch.py`: server-side pull of product-vs-age mismatches -> fetch
  `dateOfBirth` from individual-index -> recompute age at dose date -> verdict). ba result: all 16 =
  "recorded-age error, DOB says dose CORRECT" — right VAS colour for the child's REAL age, only the typed
  age wrong. **0 mis-dose, 0 product mismatch, all valid deliveries.** Fixing the ages (UPDATE_AGE to DOB
  age) would reconcile Red/Blue to 0 — but is blocked by the soft-deleted VAS variant (see context/gotchas);
  cosmetic since doses are correct.
- **Household ID in the report:** task `additionalDetails.householdClientReferenceId` is the CLIENT ref,
  NOT the `id`. If asked for the household `id`, resolve clientRef -> server `id` via `ba-household-index-v1`
  (`Data.clientReferenceId` -> `Data.id`) as a post-step (HH-member index as fallback). Referral rows have
  no household -> blank.
- **DQ per-product, don't blend:** VAS rows carry NO lat/lon (100% blank) -> report GPS on SPAQ rows only.
  When VAS is exported as its own per-doc rows, `Record Type=duplicate` inflates to ~86% (it keys on
  individual+status, and every child now has SPAQ + VAS success rows) -> true dup rate ~1.4% by
  (individual+product+cycle). Missing HH-head/child-name are source-nulls typed as `null`/`nan`/`none`
  by a few CDDs (0 truly-empty, 0 invalid) — count them, don't treat as data loss.

## Facility target files (individual_target per facility)
Target file format (template = `so_target.csv`): `facility_code, facility_name, individual_target, total`
(total usually `NULL`). A target export sometimes ships WITHOUT `facility_name` (e.g. `zf_target.csv` had
only `facility_code, individual_target, total`); the facility name lives in the facility master export
(`ogname*.csv`: `id, tenantid, name, ..., clientreferenceid`). There is NO shared code between the two
files, so **join on the NAME** — the boundary `facility_code` embeds the name uppercased with underscores
(`ADMIN_NI_<L1>_<L2>_<L3>_<L4>_ZA_KAWAYE_PRIMARY_HEALTH_CLINIC` ↔ `za Kawaye Primary Health Clinic`).
GOTCHA: boundary codes are **TRUNCATED** (~60 chars, e.g. `..._PRIMARY_HEALTH_CEN` for `...CENTRE`), so
`endswith`/exact match misses them. Match by **longest-common-prefix** of `norm(name)` vs the code's
name-tail (strip the leading `ADMIN_NI_<numeric levels>_` prefix); `norm(s)=re.sub(r'[^A-Z0-9]+','_',s.upper()).strip('_')`.
On zf this took 544/591 → 591/591 once LCP replaced endswith. Output in the `so_target` column order,
quote the two text columns, leave the number + `NULL` bare.

## SMC Duplicates column — definition changed 2026-08-07 (vashisht.kumar request)

**Problem:** the old duplicate check keyed on head+child+ward+age names only and ran
status-blind, so legitimate revisit patterns inflated it — vashisht supplied the real
per-beneficiary record pattern: 1 ADMINISTRATION_SUCCESS + 2 DELIVERED + 1 VISITED
(re-dose). DELIVERED turned out to be excluded already (analyze.py never queries that
status — only SUCCESS+VISITED and the 7 non-admin statuses), but VISITED and
REFUSED-then-SUCCESS pairs were being counted as duplicates. **Fix (analyze.py
_aggregate_batch):** duplicate key = individual clientReferenceId (resolved via the
task->PB->individual chain the name resolution already walks — NOT raw
Data.individualId; vashisht confirmed CRID is the intended ID) + acting user
(Data.createdBy UUID, one-to-one with username so no join needed) + head + child +
ward + age, checked across ADMINISTRATION_SUCCESS records only. His 4-record
beneficiary now yields 1 treated, 1 redose, 0 duplicates (unit-tested).
**Disclosed consequence:** with the user in the key, the same child treated by two
DIFFERENT CDDs is invisible to this column — the cross-user analysis exists as the
ITN duplicate matrix (see itn-llin-reporting.md) and is portable if ever wanted.
Duplicates counts DROP campaign-wide from the first run with this code — definition
change, not a data change; say so before anyone compares to older reports.
