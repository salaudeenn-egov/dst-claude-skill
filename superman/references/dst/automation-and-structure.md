# Automation pipeline and D:\DST folder structure

Two topics: (1) the automated daily reporting pipeline at `D:\DST\automation`, and
(2) the `D:\DST` folder reorganization and naming conventions.

---

## DST Campaign Automation — `D:\DST\automation` (own git repo)

Fully automated daily performance reporting for DIGIT HCM SMC/SPAQ campaigns.
Google-Sheet driven, one row per campaign.

Pipeline flow:
`Google Sheet` (one row per campaign)
→ `scheduler.py` (watchdog; re-reads the sheet hourly; fires at `report_times`, which are UTC)
→ `config.py` (computes DAY / GTE / LTE from `campaign_start`)
→ `analyze.py` (ES task scroll + individual/HH name lookup → `performance.xlsx`, 6 tabs)
  + `cdd_sync.py` (staff + sync agg → `cdd_sync.xlsx`)
  + `report.py` (both xlsx + Claude API → Word report + progress chart PNG + Slack text)
→ `notify.py` (Drive upload with ownership transfer to <REDACTED-EMAIL> + Slack post)

`run.py` runs the whole thing once, manually.

### Config / credentials
- `.env`: `ANTHROPIC_API_KEY`, `SLACK_TOKEN`, `ES_URL`/`ES_USER`/`ES_PASS`,
  `GOOGLE_SHEET_ID`, `GOOGLE_DRIVE_FOLDER_ID`
- `credential.json` = Google service account (Sheets only)
- `drive_token.json` = Gmail OAuth (Drive uploads)
- Do NOT swap `credential.json` and `drive_token.json` — different auth, different use
- Output goes to `out_dir` per sheet row (e.g. `D:\DST\output\ab`)

### Multi-deployment / multi-tenant (added 2026-07 — Togo + Taraba)

One codebase now runs as several independent deployments (one box per ES cluster).
Three env vars make it per-environment; the rest of a campaign's config is its Google
Sheet row. (All ES/API secrets live in `.env` only — never in the sheet or this doc.)

- `GOOGLE_SHEET_TAB` (config.py `get_active_rows`, default `Sheet1`) — the worksheet
  this deployment reads. One tab per environment on the same shared sheet (e.g. `togo`,
  `taraba`, `Sheet1`=Bauchi central). **Also names the top Drive folder.**
- `ES_INDEX_PREFIX` (config.py `build`) — controls the ES index names:
  unset → `{tenant}-` (Nigeria central, tenant-prefixed e.g. `ba-`, `taraba-`);
  set EMPTY → no prefix (Togo dedicated cluster: `project-task-index-v1`).
- `GOOGLE_RUNLOG_TAB` (run.py, default `Run Log`) — optional per-env run-log tab.
- Per-env `.env` files kept as `.env.<name>` (bauchi/togo/taraba), gitignored via
  `.env.*` + `!.env.example`; copy the right one to `.env` on each box.

Per-campaign Drive organization (notify.py `campaign_folder_id`, all uploads routed
through it incl. `run_cumulative`'s direct `upload_file` calls): files auto-file under
`<Instance>/<State>/<Campaign>/<Day N | Cumulative (Days 1-N)>` where Instance =
`GOOGLE_SHEET_TAB` (title-cased), State = `state_name`, Campaign = `campaign_name`
(falls back to `drug_type`). `""`/unset root → old flat behaviour (safe fallback).

**CDD role is tenant-specific** (cdd_sync.py, all 7 role filters): Nigeria admin console
= `DISTRIBUTOR`; Togo/Chad non-admin = `COMMUNITY_DISTRIBUTOR`. Hardcoded per-deployment
(user chose the plain string over a config-derived value); the Taraba deploy zip is
built with the word swapped to `DISTRIBUTOR` **inside the archive only** so the repo
stays `COMMUNITY_DISTRIBUTOR` for Togo. Wrong role → `cdd_sync` silently returns 0 CDDs
(verify against `{tenant}-project-staff-index-v1` `Data.role.keyword` agg).

Deploy-zip patterns (built via python zipfile): exclude `.git`/`__pycache__`/`logs`/
`output`/`.pyc`; the lean code-only zip also excludes `.env`+`credential.json` (~54KB,
avoids the 413 upload limit). A **turnkey** zip can bundle `credential.json` + a ready
`.env` + `targets/<t>_target.csv` (secrets inside — handle carefully).

Gotchas hit this cycle:
- **`cycle_index` must be TEXT in the sheet** — Sheets coerces `02`→number `2`, and the
  ES filter `Data.additionalDetails.cycleIndex.keyword = "2"` matches NOTHING (empty
  report). Force the cell to plain-text format and write RAW.
- **`target_csv` facility_name must equal ES `boundaryHierarchy.healthFacility`** (lower-
  cased match in `_load_targets`). Togo DB gives readable names via the admin self-join;
  Taraba's self-join returns blank AND ES casing is quirky (`Sunkani MCHC`, `Sabon Gari
  PHC hea 01`) — derive from the boundary code then override mismatches from ES
  (`boundaryHierarchyCode.healthFacility`→`boundaryHierarchy.healthFacility` agg).
- **Claude auth = `ANTHROPIC_API_KEY`, not OAuth.** Claude Code / subscription OAuth
  tokens 429 immediately (plan-capped) and expire (subscription refresh ~days; a
  `claude setup-token` OAuth is ~1yr but still rate-limited). API key is the reliable
  path for the unattended scheduler. `report.py` `_claude` uses `Anthropic(api_key=...)`.
- **Box hazards:** broken numpy (`No module named numpy._core._multiarray_umath`) →
  `pip install --user --force-reinstall --no-cache-dir numpy pandas`; Jupyter file-UI
  won't create dot-files (`.env`) — use a terminal heredoc/echo; Jupyter injects `-f`
  into `sys.argv` so argv-parsing scripts see `-f` (fall back to `os.getcwd()`/defaults);
  Linux case-sensitive paths (`~/DST` vs `~/dst`) — `_resolve_creds_path` fallback saves it.

### Togo deployment (tg — SMC 2026 Cycle 2)
tab `togo`; `ES_INDEX_PREFIX=` (empty, un-prefixed `project-task-index-v1`);
ES user `elastic` @ `es-cluster-v8` (pass in `.env`); `is_admin_console=FALSE`,
`task_campaign_filter=TRUE`, `project_type_id=644c4356-5214-11ee-be56-0242ac120003`
(MR-DN), `cycle_index=02` (text), `drug_type=SPAQ`, CDD role `COMMUNITY_DISTRIBUTOR`;
16-19 Jul; report_times `05:30`, partner `12:00,05:30`; channels internal `<REDACTED-SLACK-CHANNEL>`
/ partner `<REDACTED-SLACK-CHANNEL>`.

### Taraba deployment (taraba — SMC 2026 Cycle 1)
tab `taraba`; tenant-prefixed (`taraba-project-task-index-v1`, `ES_INDEX_PREFIX` unset);
ES user `<REDACTED-USER>` @ `es-cluster-v8` (Basic-auth user + pass in `.env`);
`is_admin_console=TRUE`, `task_campaign_filter=FALSE`,
`campaign_number=CMP-2026-07-08-000045`, `drug_type=SPAQ`, CDD role `DISTRIBUTOR`;
15-18 Jul; 384 facilities, target 558,913, 1,582 DISTRIBUTOR CDDs; report_times `05:30`,
partner `12:00,05:30`; same channels as Togo.

### Chad deployment (ch — SMC 2026 Cycle 1, added 2026-07-20)
Runs on **Sheet1** (same central box as the Nigeria states), tenant `ch`, tenant-prefixed
indices (`ch-project-task-index-v1`, `ES_INDEX_PREFIX` unset), es-cluster-v8, ES user `<REDACTED-USER>`.
Campaign `CMP-2026-06-27-000416` (projectName `MR-DN_27june_2026`), projectTypeId
`ea1bb2e7-06d8-4fe4-ba1e-f4a6363a21be` (code MR-DN, SPAQ, French product names). CDD role
**`DISTRIBUTOR`**. Slack `<REDACTED-SLACK-CHANNEL>`. report_times **`16:30` UTC = 5:30 PM WAT** (Chad is
UTC+1). Target **486 HFs / 1,139,200** from DB `ch.project_target` at **HEALTHFACILITY**
level (LOCALITY-level targets sum to an inflated ~3.3M — do NOT use); `build_ch_target.py`
joins DB target→ES by the `ADMIN2_TC…` boundary code.

**Scoping gotchas:**
- projectTypeId `ea1bb2e7` is **SHARED** with 4 UAT/test campaigns (`TEST_UAT`, `UAT_TEST_CHAD`,
  `TESTING_CHAD`, `CHAD_TESTING`, all `ADMIN_NI`/`ADMIN2_CH` boundaries). Scope the live report
  by **campaignNumber** (+ today's date window) — projectTypeId alone bleeds test data.
- `cycle_index` left **blank** in the sheet: gspread `get_all_records()` numericises `"01"→1`,
  and `Data.additionalDetails.cycleIndex.keyword="1"` matches nothing (empty report). The daily
  date window isolates the cycle, so blank is safe.
- Schema/tenant is `ch` (DB schema `ch.`, indices `ch-`) even though boundary codes read `ADMIN2_TC`.

**Chad ES data shape (differs from Nigeria — drove the analyze.py generalisation):**
- `task.individualId` is **null**. `boundaryHierarchy` uses UPPERCASE `HEALTHFACILITY` +
  `country/province/district/locality` (no `lga`/`ward`/`healthFacility`).
  `boundaryHierarchyCode.HEALTHFACILITY` = `ADMIN2_TC…` code (matches DB `project_address.boundary`).
- `productName` French (`SPAQ (3 à 11 mois)` / `SPAQ (12 à 59 mois)`); gender in
  `additionalDetails`; lat/lon top-level; age in months.
- `ch-user-sync-index-v1` carries `campaignNumber` + role `DISTRIBUTOR` + `taskDates` +
  `createdTime` but **NO `projectTypeId`** → the `cdd_sync` "synced-by-cutoff" must scope by
  **campaignNumber**, not projectTypeId.

**Name resolution — generic core logic in `analyze.py` (all tenants, no tenant gate).** Child &
household-head names come ONLY from the **individual index** via joins, never from
`task.additionalDetails.childName/headName`. Deterministic chain:
- child: `task.projectBeneficiaryClientReferenceId` → `project-beneficiary.beneficiaryClientReferenceId`
  → **individual index** → `name.givenName + familyName`
- head: that beneficiary → `household-member` (by `individualClientReferenceId`) → `householdClientReferenceId`
  → `household-member` (`householdClientReferenceId` + `isHeadOfHousehold=true`) → head
  `individualClientReferenceId` → **individual index** → name

Proven equivalent for Nigeria: `PB.beneficiaryClientReferenceId == task.individualId` (Bauchi),
so one chain serves all. **Index field shapes:** project-task nested under `Data.*`;
**project-beneficiary + individual are FLAT** (root `clientReferenceId`,
`beneficiaryClientReferenceId`, `name`); household-member nested under `Data.householdMember.*`.
Requires `config.py` `ES_INDEX_PB = {prefix}project-beneficiary-index-v1`, name maps keyed by
task `clientReferenceId`, and the task query `_source` MUST include
`Data.clientReferenceId` + `Data.projectBeneficiaryClientReferenceId` (+ `Data.boundaryHierarchyCode`)
or the whole chain silently no-ops (empty `pb_refs` → all names "missing"). See
[[feedback_name_resolution_individual_index]].

### State as of 2026-07-02
Working tree clean at start. Last logged run: 2026-06-25 FCT (tenant `ab`) Day 1 with
`TEST_EXTRACT_DATE=2026-06-13` override active, Bauchi row `active=FALSE`. Known
non-fatal warning: the first Drive upload attempt per run can fail with
`conversionUnsupportedConversionPath` (Excel→Sheet conversion) before succeeding.

### Bugs found and fixed (2026-07-02)

BUG (analyze.py) — Missing HH Name always flagged. `run()` passed `{}` as
`hh_name_map` to `_aggregate_batch`, so `hh_head` was always "" → every record flagged
Missing HH Name (Kebbi Day 1 showed 86,196 = 103.27%; >100% because the count spans all
records but `report.py:823` `_dq_summary_table` denominator is `treated` only). The
HH-head resolution functions existed but were never chained. FIX: added
`_build_hh_head_name_map(cfg, child_ind_ids)` = the 3-step DIGIT chain (child
`individualClientReferenceId` → `householdClientReferenceId` via household-member index
→ head member `isHeadOfHousehold=True` → head `individualClientReferenceId` → name via
individual index), wired into `run()`'s batch loop. Reference method:
`extraction_scripts\tg\children_treated_tg.py` steps 4→5→6 (6 of 8 children_treated
scripts use it: tg/oy/na/kd/jg/ko). FCT (`fct`) and `pl` are exceptions — they resolve
Household ID only (ID-based output), no head-name hop. Task `individualId` ==
household-member `individualClientReferenceId`. `isHeadOfHousehold` exists on the
household-member index across all tenants. Not yet committed/re-run as of fix time.

BUG #2 (analyze.py) — Missing Gender, corrected TWICE. Original read
`additionalDetails.gender`. First "fix" to `Data.gender` BROKE Kebbi (Kebbi `ke`,
admin-console, stores gender in `additionalDetails.gender`: 14:40 old-code Missing
Gender=0; after Data.gender change it went to 99,467/103% = field-not-found). FINAL
FIX: check BOTH — `if not (doc.get("gender") or add.get("gender"))` in both
`_aggregate_batch` and `_aggregate`. `Data.gender` kept in both `_source` lists.
Lesson: gender location is tenant-specific (Data.gender for most, additionalDetails.gender
for Kebbi/admin-console) — always check both, like lat/lon.

DQ audit verdict (2026-07-02): Missing Child Name (individual index
name.givenName+familyName), Age=0/Age>59 (Data.age), Duplicates (4-field key
hh_head/child/ward/age) all CORRECT vs references. FIXED: `missing_lat_lon` now reads
top-level `Data.latitude`/`Data.longitude` (Nigeria) with `additionalDetails` fallback
(Togo), both added to `_source` and both read sites. Latent, NOT fixed: `cdd_sync`
`synced_cdds` = sum(HIGH column) undercounts from Day 2 on (HIGH = synced every day;
should be Total − NEVER = HIGH+MODERATE+LOW). Day 1 unaffected.

ORS-Zinc / secondary product (2026-07-02): Kebbi (`ke`, state row 3 in config Sheet1)
delivers ORS-Zinc alongside SPAQ. ES `Data.productName.keyword` values for `ke`:
`SPAQ 2`, `SPAQ 1`, `null`, `ORS-Zinc` (Kebbi is SPAQ drug_type despite the
`nigeria-kebbi-azm` folder name). ORS column was missing because the config sheet had
NO `secondary_product` column (config.py defaults missing col to "" → ORS disabled for
all). FIXED: added `secondary_product` header at column 26 and set Kebbi row =
`ORS-Zinc`. ORS Day-1 (2 Jul) = 1,114 successful deliveries (age 3-59,
ADMINISTRATION_SUCCESS) across 32 facilities. STILL OPEN: the main SPAQ treatment query
in analyze.py `run()` filters only on administrationStatus+date, NOT productName, so
ORS-Zinc (1,114) + null (1,428) leak into treated/records and inflate duplicates (child
with SPAQ+ORS same day = dedup collision). Reference extraction scripts avoid this by
keying one row per individualId. Proposed fix: add a productName filter (SPAQ 1/SPAQ 2
for SPAQ, AZM for AZM) to the main queries.

Partner report DQ-strip (2026-07-02, report.py): partner Word doc already omitted DQ
sections 3.2/3.5 but still linked the full performance Excel via "Full list in
performance Excel". Added `_make_partner_perf_xlsx(perf_path)` which writes a
`{perf}_partner.xlsx` copy with the 6 DQ columns (Missing HH/Child/Gender/Duplicates/Age)
stripped from every tab (re-merges row-1 banner). Partner build uploads the stripped
copy and overrides perf_link/perf_path; internal report keeps the full Excel. Tested
stripping locally on Kebbi Excel (26→20 cols).

Relates to reporting.md, campaigns.md (Kaduna `kd`, Bauchi `ba`), and the doc-style
feedback in memory.

## Multi-deployment hardening — Nigeria States / Togo / Bauchi VAS / Sokoto (2026-07-23)

**Config loading & the gspread numericise trap**
- `run.py` loads the ROOT `.env` (`dirname(run.py)/.env`); `config.py` separately `load_dotenv`s `pipeline/.env`. On the boxes the real file is the ROOT `.env` (next to run.py). A bare `python -c "from pipeline import config; config.get_active_rows()"` loads only `pipeline/.env` and errors "GOOGLE_SHEET_ID not set" — do `load_dotenv('.env')` first to test standalone.
- FIXED (`config.get_active_rows`): now `sheet.get_all_records(numericise_ignore=["all"])`. Without it gspread numericises leading-zero values — `cycle_index "02"→2→"2"` broke the Togo ES `cycleIndex.keyword` filter → 0 docs (had also bitten Chad); can mangle `campaign_number`/IDs. All config is str()/int(float()) coerced, so all-string is safe. This is the generic fix for the whole leading-zero class.

**ES_INDEX_PREFIX rule** — `idx_prefix = f"{tenant}-" if os.getenv("ES_INDEX_PREFIX") is None else <value>`
- Line ABSENT → tenant-prefixed (`ba-`/`ch-`/`so-`… Nigeria central + Chad).
- Line PRESENT but EMPTY (`ES_INDEX_PREFIX=`) → un-prefixed (Togo dedicated cluster: `project-task-index-v1`). Must be present-and-empty for Togo, NOT omitted, else it queries `tg-…`(empty) → 0 docs.

**Generic secondary products** (config/analyze/report) — supersedes the single ORS-Zinc path above
- The existing `secondary_product` column (no new column) is parsed by `config._parse_secondary_products` into a LIST. Format inside that one cell: `productName|label|ageMin|ageMax` entries, `;`-separated. Age band OPTIONAL — omit (`Red VAS|Red VAS`) to count the product across ALL ages by productName (the name encodes the band; age-filtering undercounts vs dashboard). A bare value with no `|`/`;` (e.g. `ORS-Zinc`) → legacy single product, age 3-59, tab `ORS-ZINC` (Kebbi/Sokoto unchanged).
- analyze `_fetch_secondary_counts(cfg, spec)` loops each spec (+`_build_campaign_filters`, no-op when task_campaign_filter FALSE); `_write_secondary_tab` = one tab per spec. report: one overview line + one facility column per product; per-facility count stamped `f["secondary"][label]`. No tenant/VAS hardcoding.
- Bauchi (`ba`) SPAQ+VAS: exact productNames `Red VAS` (older, 12-59m, ~3.1M all-time) + `Blue VAS` (younger, 6-11m, ~353K). Cell = `Red VAS|Red VAS; Blue VAS|Blue VAS`. Kebbi/Sokoto = `ORS-Zinc`. Names are case-sensitive `.keyword` — get exact strings from a `Data.productName.keyword` agg.

**CDD role now config-driven** (`cdd_role`) — was globally hardcoded DISTRIBUTOR
- `cfg["cdd_role"]` = sheet col `cdd_role` OR env `CDD_ROLE` OR default `DISTRIBUTOR`. Togo box sets `CDD_ROLE=COMMUNITY_DISTRIBUTOR`. All role filters in cdd_sync now use it.
- **user-sync index gotcha:** `projectTypeId` is nested at `Data.additionalDetails.projectTypeId.keyword` (NOT top-level). The STAFF index has it top-level (`Data.projectTypeId`); the SYNC index does not. New `_cutoff_scope(cfg,today,cutoff_ms)` unifies both cutoff fns: scope = campaignNumber if set else `additionalDetails.projectTypeId`; role = `cfg["cdd_role"]`; agg key syncedUserName(admin)/syncedUserId(else). Removed the old `tenant=="ch"` branch.

**Cumulative report** (`run.py --cumulative <END>`, Drive-only, never Slack)
- Progress chart built day-by-day from ES (`report._load_days_from_es`) so it covers EVERY campaign day (start..DAY) regardless of which `performance_dayN.xlsx` exist; per-day treated mirrors `is_treated` (ADMINISTRATION_SUCCESS + age band + qty≥1, +doseIndex if flag). `run_cumulative` sets DAY/campaign_days = actual (end−start)+1 (not capped by sheet).
- CDD sync in cumulative shows OVERALL totals: "CDDs Synced (≥1 day) = Total − Never" (HIGH+MOD+LOW) + "Fully Synced (all N days) = HIGH"; the today-cutoff 17:00/17:30 rows are skipped (meaningless cumulatively). This RESOLVES the "Latent, NOT fixed" `synced_cdds`=HIGH note above — cumulative now uses Total−Never (Section-1 overview, 5.1 table, and all 3 narrative prompts); daily still uses HIGH by design.

**DELIVERED status gotcha (OPEN, critical)** — DIGIT task `administrationStatus` includes `DELIVERED` (and `CLOSED_HOUSEHOLD`, `NOT_ADMINISTERED`) that the DST pipeline does NOT fetch/count. analyze counts only ADMINISTRATION_SUCCESS(+VISITED) + the 7 non-admin statuses; the secondary fetch requires ADMINISTRATION_SUCCESS. On fresh/early data `DELIVERED` can dominate (Togo all-time 228k DELIVERED vs 116k ADMINISTRATION_SUCCESS; Bauchi/Sokoto ~2:1 early). Symptom: 0 treated / 0 VAS / 0 ORS while data is clearly arriving. UNRESOLVED whether DELIVERED is transient (→ADMINISTRATION_SUCCESS on sync) or terminal — test a COMPLETED cycle's status agg: mostly ADMINISTRATION_SUCCESS ⇒ transient (wait); large DELIVERED remains ⇒ terminal ⇒ add DELIVERED to counted statuses (primary treated + secondary fetch).

**Google Drive facts** (service account `dst-report@digit-health-75ed0`)
- SA has NO My Drive quota → can only create files inside a SHARED DRIVE. `GOOGLE_DRIVE_FOLDER_ID` may be a Shared Drive root (`0A…`) or a folder within one (`1…`); both fine if inside a shared drive. notify.py uses `supportsAllDrives=True` everywhere and treats the id as a parent.
- SA role on the drives is Contributor: can create + list + rename, but CANNOT delete/move/trash (delete → 404). Cross-shared-drive migration = `files.copy` (not move); originals stay as backup; cleanup needs a human/Content-Manager.
- Nigeria-States central: sheet tab renamed `Sheet1` → `Nigeria States`; Drive root renamed → `Nigeria States` (id `143az7HpAVe3xLWMe5H3JExyJMpSiLHgp` in shared drive `0AMYr_ABn_ArOUk9PVA`). Central `.env`: `GOOGLE_SHEET_TAB=Nigeria States`, `GOOGLE_DRIVE_FOLDER_ID=143az7…`, NO `ES_INDEX_PREFIX` line, NO `CDD_ROLE`. Renaming a folder keeps its id (so `GOOGLE_DRIVE_FOLDER_ID` unchanged).

---

## D:\DST folder structure and naming conventions

`D:\DST` is the main work folder, being reorganized into:
- `D:\DST\campaigns\` — one subfolder per campaign (e.g. `nigeria-bauchi-smc\`, `moz-nampula-smc\`)
- `D:\DST\scripts\` — reusable/tool scripts and git repos (`dst-kit` already here; also `script-reconciliation`)
- `D:\DST\repos\` — large reference repos (digit-core)

Each campaign folder has: `queries\`, `scripts\`, `data\`, `notes\`.

Naming conventions:
- Campaign folder: `[country]-[state/region]-[type]` e.g. `nigeria-bauchi-smc`, `moz-cabo-delgado-smc`
- File: `[schema-code]-[type]-[purpose].[ext]` e.g. `ba-sql-dose-delivery-count.sql`, `ko-es-duplicate-detection.kibana`

Why: the old `DISPENCARY\` folder (misspelled) had inconsistent naming (spaces,
ALL_CAPS, abbreviations), scripts saved as `.txt`, and files in the wrong campaign folder.

Key cleanup issues found:
- `extaction scription.txt` in bauchi folder was actually the OYO script → moved to
  `nigeria-oyo-smc\scripts\oy-extract-task-ids.py`
- `project heichacy.txt` in Oyo folder contained Gombe/Ondo/Adamawa SQL → split into 3 files
- `DISPENCARY\cabo\` and `cabo data\` were duplicate Cabo Delgado folders → merged
- `DISPENCARY\salma nampula\` and `nampula\` were duplicate Nampula folders → merged
- `Untitled.ipynb` in nampula → `moz-nampula-task-ingestion.ipynb` (bulk ingestion, ran 9808 tasks)
- `ingest_tasks_single.py` at root → MOZ Cabo Delgado single-task retry script

Note: many active scripts still live under `C:\Users\Salaudeen N\Downloads\` (extraction,
recovery, and report scripts referenced throughout this skill) — the reorg into
`D:\DST\campaigns\` is ongoing, not complete.

## Progress-chart coverage formula (report.py `_generate_progress_chart`, 2026-07-27)

One coverage formula for the chart: **Cumulative Coverage % = Cumulative Treated (Days 1..N) / Total Campaign
Target x 100.** The DENOMINATOR has two forms:
- **Daily report:** `total_target = max_daily_target * campaign_days`, where `max_daily_target = max(d["target"]
  for d in days_data)` — the FULLEST day's target sum (NOT literally each day's), deliberately, so an early
  partial-extract day with a tiny target sum doesn't explode the bar (the old ">100% / 916%" bug). Assumes the
  daily target is ~uniform across days.
- **Cumulative report** (`--cumulative`): `total_target = overall_target` (the full campaign target from the
  target CSV, passed in directly), NOT reconstructed from daily x days.

Caption line (`Total Campaign Target = Daily Target x N days`) is the DAILY form; it's slightly off for
cumulative runs (there the denominator is the CSV overall target). Daily and cumulative charts agree only when
`max_daily_target * campaign_days == overall_target` (per-day target files sum to the CSV total). Offer to make
the caption mode-aware if exactness matters.
