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
  + `report.py` (both xlsx + Groq LLM narrative (`openai/gpt-oss-120b` since 2026-08-17;
    Claude/Ollama providers removed) → Word report + progress chart PNG + Slack text)
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
- **[HISTORICAL — narrative is Groq-only since ~Jul 2026; see 2026-08-17 section]
  Claude auth = `ANTHROPIC_API_KEY`, not OAuth.** Claude Code / subscription OAuth
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
  date window isolates the cycle, so blank is safe — BUT stragglers leak: cycle-01 docs with
  device-clock taskDates inside the cycle-2 window count as cycle-2. Measure with
  `cycleIndex="01" AND taskDates>=<cycle2 start>` before trusting blank. Cycle 2 (2026-08-17)
  hit the typed-bare-`2` variant of this class — see "2026-08-17 — Groq migration +
  cycle_index padding" section below.
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

## 2026-08-17 — Groq model migration + cycle_index padding (Chad Cycle 2 zero-doc incident)

### Narrative LLM: llama-3.3-70b-versatile decommissioned → openai/gpt-oss-120b

**Problem.** Groq decommissioned `llama-3.3-70b-versatile` (2026-08-16) — the pipeline's only
narrative model (`report.py _claude()`, Groq-only; Claude/Ollama providers were removed).
It still answered during a grace window, so nothing looked broken yet.

**Approach.** Listed the key's models via `GET /openai/v1/models`, then smoke-tested both
recommended replacements with the pipeline's exact call shape (`max_tokens=300..400`).

**Findings / root causes.**
- `qwen/qwen3.6-27b` is UNUSABLE as-is: dumps its `<think>` reasoning into `message.content`
  and burns the whole token cap (`finish_reason=length`, no answer).
- `openai/gpt-oss-120b` works, but it is a REASONING model: hidden reasoning tokens count
  against `max_tokens`, so small caps truncate or empty the visible output. This is what the
  Taraba Day-6 "hit max_tokens=400" warning was.
- gpt-oss emits narrow no-break spaces (U+202F) before `%` ("88 %") — looks broken in Word.

**Fix (all in `pipeline/report.py`, covers report_itn.py which imports `_claude`).**
- Default + `.env`/`.env.togo`: `GROQ_MODEL=openai/gpt-oss-120b`.
- In `_claude()`: if `"gpt-oss" in model` → add `"reasoning_effort": "low"` (Groq accepts it)
  and pad the request cap by +500; callers keep passing visible-output budgets (400/600/1500).
- Sanitize output: U+202F/U+00A0 → space, `" %"` → `"%"`. (U+2019 curly apostrophe is fine.)
- Verified end-to-end: `finish_reason=stop`, clean prose at the 400-cap Slack call.

**Deployment trap:** every server `.env` (e.g. `/home/jovyan/dst/automation`) sets
`GROQ_MODEL=llama-3.3-70b-versatile`, which OVERRIDES the new code default — servers need the
one-line `.env` edit or reports degrade to `[Narrative not generated — Groq error]`.

**Free-tier limits for gpt-oss-120b on this key** (from `x-ratelimit-*` headers): 1,000
requests/day, 8,000 tokens/minute. ~3 calls per report slot → requests are a non-issue; a very
large issues prompt could 429 on TPM (non-fatal — placeholder text, report still builds).

### cycle_index `2` vs `"02"` — Chad Cycle 2 Day 1 returned 0 task docs

**Problem.** Chad Cycle 2 Day 1 (2026-08-17, 06:03 UTC run): `task-treatment: ~0 docs`,
`task-nonadmin: ~0 docs`, report saved with 0 facilities — while the staff query worked
(3,796 docs). Misleading first read: "06:03 UTC = 07:03 WAT, data hasn't synced yet" —
partially true but NOT the cause; even with data present the query matched nothing.

**Approach.** Agg on `Data.additionalDetails.cycleIndex.keyword` scoped to the campaignNumber:
buckets `"01"` 3.69M / `"02"` 1,276 (growing live) → matching docs EXIST; then per-filter
elimination (dates OK: 433+ docs taskDates=2026-08-17; statuses OK: SUCCESS+VISITED).
Only remaining difference: the filter VALUE.

**Root cause.** The sheet cell holds bare `2`. `numericise_ignore=["all"]` (the earlier
leading-zero fix) only PRESERVES what was typed — it cannot invent the zero. So the query sent
`cycleIndex.keyword="2"`; ES stores zero-padded `"02"`; `term` on keyword is exact → 0.
(Contrast gotcha: doseIndex is "1"/"2"/"3" UNpadded; cycleIndex is "01"/"02" PADDED.)

**Fix / verify.** `config.py _pad_cycle()`: strip, drop trailing `.0`, `zfill(2)` if
digit-only (`2`/`"2"`/`2.0`/`" 2 "` → `"02"`; `"10"`/non-numeric pass through) — tested.
Short-term without deploy: type `'02` (text) in the cell, or clear it (blank = date-window
isolation, see Chad scoping gotchas for the straggler-leak caveat). Scheduler re-reads the
sheet hourly — no restart needed.

**Side finding:** ~880 cycle-02 docs carry taskDates BEFORE the official start
(Jul 30 + Aug 10–16, Aug 16 alone = 232) — either early field starts (then campaign_start in
the sheet is wrong and those never enter any daily window) or client-clock drift. Confirm with
the campaign team before Day 2.

**ES query-authoring gotcha (hit while diagnosing):** `terms` agg with a string
`"missing": "SENTINEL"` on a DATE-mapped field (`Data.taskDates`) throws
`parse_exception ... failed to parse date field [SENTINEL]` and fails ALL shards. Use a
separate `"missing": {"field": ...}` agg + `"format": "yyyy-MM-dd"` on the terms agg instead.


---

## Pipeline modularization (2026-08-14, committed as `2ea7edc` on branch `airflow`)

The pipeline was restructured for the Airflow migration (business logic
UNCHANGED — verified by pyflakes zero-undefined-names, import + execution
smoke tests, and a live E2E run). run.py/scheduler.py's run(cfg) seams are
frozen; both keep working.

**pipeline/core/ — shared plumbing extracted from copy-paste:**
- es.py: scroll_all, scroll_batches, composite_agg (were duplicated across
  analyze/cdd_sync/ITN modules)
- excel.py: style_cell, style_sync_cell, both fill palettes, FLAG_COLOR
  (the two old _HDR_FILLs had DIFFERENT colors — now HDR_FILL vs SYNC_HDR_FILL)
- word.py: hdr, dat, add_heading, add_para, hyperlinks, borders,
  two_col_table, cov_band (report_itn used to import these by _underscore name
  from report.py)
- llm.py: generate_narrative — the old `_claude` was MISNAMED, it is a GROQ
  chat call (GROQ_API_KEY/GROQ_MODEL/GROQ_BASE_URL)
- checkpoint.py: save/load_checkpoint -> {out_dir}/checkpoints/{stage}_dayN.json
- drive.py: upload_raw/download_raw/find_or_create_folder — raw (NO
  conversion), overwrite-by-update (the SA cannot delete), and NO
  anyone-with-link permission (checkpoints carry beneficiary names). notify's
  report uploads still convert+link-share as before.

**Stage structure — collect -> checkpoint -> render:**
analyze and cdd_sync are split: collect(cfg) = ALL ES I/O -> plain data;
save_checkpoint; render(cfg, ...) = pure -> Excel. `rerun_from_checkpoint(cfg)`
rebuilds an Excel OFFLINE (no ES, no creds) — the checkpoint is frozen
evidence of what ES said at run time (a later re-query cannot reproduce a
morning's numbers; cf. the Borno mid-sync-snapshot argument). report.run is
composed of named stages: _load_inputs -> _build_trajectory ->
_publish_excels (Drive uploads finally ISOLATED from doc building) ->
_generate_narratives (all LLM calls in one place) -> _render_docs. Checkpoints
also upload to the campaign's Drive `Day N/temp/` folder at the end of every
run (even failed ones) via notify.upload_checkpoints — so a dead k8s pod
stays debuggable: notify.download_checkpoint(cfg, stage) then
rerun_from_checkpoint.

**Dead code deleted** (the run loop had inlined them long ago):
analyze._fetch_task_docs, analyze._aggregate, analyze._build_hh_head_name_map.

**Self-explanatory renames** (old -> new, the cryptic ones):
_fetch_pb_to_individual -> _map_beneficiary_refs_to_individual_ids;
_fetch_hh_member_map -> _map_individual_ids_to_household_ids;
_fetch_hh_head_map -> _map_household_ids_to_head_ids;
_finalize_fac_data -> _build_facility_rows; _band -> _coverage_band;
_write_tab -> _write_facility_tab; _ts_to_date -> _epoch_ms_to_date;
_load_staff_admin/_project -> _load_staff_by_campaign/_by_project_type;
_load_syncs_admin/_project -> _load_sync_dates_by_username/_by_user_id;
_build_rows -> _build_cdd_rows; _write_df -> _write_dataframe_tab;
_load_perf -> _load_performance_excel; _make_partner_perf_xlsx ->
_build_partner_performance_excel; _load_days_from_es/_load_all_days_perf ->
_load_daily_totals_from_es/_from_files; _build_doc -> _build_report_doc;
_claude -> generate_narrative; _claude_issues -> _generate_issues.
Domain vocabulary (DQ, CDD, LGA) deliberately kept.

**New shared modules:**
- pipeline/schedule_utils.py: compute_trigger_slots(row) — the
  both/internal/partner time dedup, used by BOTH scheduler.py and the Airflow
  scheduler DAG so they cannot drift.
- pipeline/run_log.py: append_run_log (the Run Log tab writer run.py delegates
  to; 10-column format incl. Mode) + fetch_today_runs (read-back for the
  Airflow retime guard). No database anywhere.
- pipeline/mdms.py: the sheet->MDMS mirror client (plan_sync pure diff +
  apply_sync HTTP) — see airflow-migration.md v3 for the full design.
- pipeline/README.md: developer guide (module map, stage structure, offline
  debugging recipe, name-resolution chain, conventions).

**Gotcha for anyone re-reading old notes:** the numericise_ignore=["all"] fix
in config.get_active_rows is NOW COMMITTED (2ea7edc) after having been lost
once as an uncommitted edit.


---

## Report-correctness overhaul (2026-08-18..19, ITN branch) — the Chad C2 "118.5% coverage" incident

Trigger: Chad SMC 2026 Cycle 2 Day 1 Slack post read "coverage 118.5%, 71,477
records vs daily target 59,645"; Ranjani challenged it (how can coverage exceed
100% when identification and administration happen together?). Shaowni asked
for a target relook + regeneration.

### 1. Dynamic-denominator bug (the root cause of 118.5%)

Problem: Day 1 daily target 59,645, Day 2 83,340 — same unchanged CSV.
Misleading first read: stale Cycle-2 targets / wrong CSV. Approach: summed the
CSV (495 facilities, 1,139,200 = exactly the C1 total, unchanged) and read
_load_targets + run(): the banner target = sum of daily targets over rows in
fac_data — and fac_data only contains facilities PRESENT IN THAT DAY'S ES
DATA. Root cause: the denominator silently shrank to the reporting subset
(31% of the book on D1, 44% on D2). True daily = 1,139,200/6 = 189,867; real
coverage D1 37.2%, D2 25.0%, cumulative ~31% — consistent with the 11-21%
CDD sync rate. Fix: _finalize_fac_data (SPAQ, facility grain) and
_finalize_lga_rows (ITN, LGA grain) now build the UNION of reporting rows and
the target book — silent target-book entries become explicit zero rows, so
the Daily Target is STANDARD (full campaign target / days) irrespective of
which facilities synced.

### 2. NOT REPORTED status band (user-specified semantics)

NOT REPORTED = target present but no data synced. New band: records == 0 ->
"NOT REPORTED" in both _band()s (safe: any ES-seen facility has >=1 record;
only zero rows can be 0). BANDS/FLAG_COLOR/STATUS_COLOR/COV_FILL extended;
the Excel gains a dedicated NOT REPORTED tab (the supervisors' chase list).
LOW ACTIVITY reverts to meaning only "reported <10 records". SMC zero rows
show LGA "—" (dash): ch_target.csv has NO lga column, nothing to claim
— a sibling-boundary-code inference was built then REVERTED at user request
(clean alternative if ever wanted: regenerate the CSV with an lga column via
build_ch_target's DB join). ITN zero rows DO carry the real LGA name (targets
are keyed by LGA boundary code; _lga_name_from_code strips the numeric path).

### 3. Display/narrative guard (defect found by user's instinct)

Problem: "I feel NOT REPORTED in the report causes issues." Verified real:
report.py's conclusion prompt picks worst_lga = min(lga_d, by coverage) —
the "—" pseudo-group is permanently 0% and would win EVERY day; the issues
prompt counted ~250 silent facilities as "low activity". Fix: run() builds
lga_display (lga_d minus the "—" group) for the LGA/DQ tables, issues +
conclusion prompts and lgas_total, while g/_grand_totals stays computed from
the FULL set (the standard-target denominator). ITN deliberately NOT filtered:
its silent LGAs have real names, so "worst LGA X at 0%" is true + actionable.

### 4. Cumulative-first presentation (user-specified formula)

Slack message coverage is now CUMULATIVE ONLY: Cumulative Coverage % =
Cumulative Treated (Days 1-N) / Total Campaign Target x 100. Prompt exposes no
daily target/coverage and instructs the LLM never to mention one. Word report
Section 1 reordered: cumulative block first (new "Total Campaign Target" row;
cumulative coverage REBASED to the total campaign target — it previously
divided by target-to-date, which would have contradicted Slack), daily rows
kept second (user: keep, don't remove). ITN ported identically for all three
metrics (households/population/nets). ITN divisor = REAL campaign length
(campaign_end - campaign_start + 1), NOT cfg[campaign_days] — that is how
analyze_itn divides; SPAQ uses cfg[campaign_days] (sheet column, default 4
fallback = a silent trap if the cell is blank on a 6-day campaign).

### 5. Formula notes (self-documenting basis)

Excel banner: "Daily Target (Day N): X (= Campaign Target / N campaign days;
all targets/coverage in this sheet are vs the DAILY target)". Word Section 3
grey note with the live worked formula. Both computed from cfg, nothing
hardcoded; daily runs only (cumulative mode measures the full target).

### 6. Duplicates fix (committed ae087eb) — redose/no-longer-duplicate

Problem: treated+redose (ADMINISTRATION_SUCCESS + VISITED), absent->treated,
refused->treated each counted as 1 duplicate — the 4-field key
(hh_head|child|ward|age) carries no status/date and ran over EVERY record.
Chad D1's "1,193 duplicates" included legitimate redoses. Fix: dedup runs only
inside is_treated (both _aggregate_batch and legacy _aggregate) — metric now
means "same child TREATED twice" (true sync/retry duplicates); genuine
double-treatment still caught. Expect a step-DOWN in reported duplicates
across the change date. Remaining known: dual-product tenants (Kebbi
SPAQ+ORS) still collide (needs the productName-filter fix, open item).

### 7. Operational learnings from the incident

- ES SNAPSHOT GROWTH: Chad D1 treated read 70,669 on the evening run and
  98,758 next morning for the SAME taskDates window — late syncs (CDD sync
  20.8% -> 28.1%). Same class as the Borno mid-sync-snapshot lesson; expect
  early-campaign numbers to keep growing for days.
- CROSS-CYCLE NARRATIVE ARTIFACT: both Chad SMC rows share out_dir
  output/chad, so _read_previous_report compared Cycle 2 Day 1 against CYCLE
  1's last report ("a notable rise from 28.9%..."). Give each cycle its own
  out_dir to keep trend text within-cycle.
- 188,235 vs expected 189,867 daily target (~9.8k missing): suspected
  duplicate facility_name rows in ch_target.csv collapsing in the name-keyed
  tmap (last wins). UNVERIFIED — check df.facility_name.duplicated().
- Day-1 regeneration bypass: bash one-liner
  `TEST_EXTRACT_DATE=2026-08-17 python run.py --state Chad` (env var scoped to
  the command; the box is bash, NOT PowerShell). Row must be active=TRUE.
- The sheet legitimately holds MULTIPLE rows per state (one per campaign/
  cycle; Chad has three). Gotchas: `run.py --state X` matches all same-named
  rows (inactive ones skip harmlessly); scheduler.py's _run_campaign refreshes
  the row by FIRST state_name match — LATENT BUG if two same-named rows are
  ever active simultaneously (fix would match on campaign_number); the Chad
  ITN row's out_dir points at output/oy (copy-paste error, fix before
  reactivating).
- PARTIAL-DEPLOY INCIDENT (again): a box ran new analyze.py with old config.py
  -> every PB batch died on KeyError 'ES_INDEX_PB', silently swallowed by the
  per-batch except -> all names empty, Missing-Name DQ ~100%, duplicates 0.
  Deploy pipeline/ as a whole, never single files. Hardening still open: let
  KeyError propagate out of the batch lookups instead of warn-and-continue.
- PROD DEPLOY COPY: Downloads/testitn/testitn/test is the pulled prod working
  dir (own git repo; .gitignore covers .env/credential.json — verified, and
  their committed REPORTS .env-style files were checked clean). All four
  changed files synced byte-identical. INTENTIONAL divergences: prod
  config.py has DUP_MATRIX_DEFAULT="TRUE" (never overwrite blindly);
  cdd_sync_itn.py in prod is BEHIND repo commit 0f40542 (per-day sync matrix)
  — sync before the next ITN campaign.

State as of 2026-08-19: analyze.py, report.py, analyze_itn.py, report_itn.py
UNCOMMITTED on ITN (+210/-48) and mirrored in the prod copy; dedup fix
committed (ae087eb); regenerate a Chad report and eyeball before committing
the set.


### 8. Fleet rollout of the fix set (2026-08-19..20) — deployment copies + divergence map

The fix set (standard target, NOT REPORTED, dedup-treated-only, cumulative-first,
formula notes, numericise, _pad_cycle) now lives in FOUR working copies, all
byte-identical to the ITN repo except two intentional divergences:

| Copy | Box | Divergence to PRESERVE on any future sync |
|---|---|---|
| D:\DST\automation (ITN branch) | source of truth | — (5 files still UNCOMMITTED as of 2026-08-20) |
| Downloads\testitn\testitn\test | central/Chad prod | config.py DUP_MATRIX_DEFAULT="TRUE"; cdd_sync_itn.py was BEHIND 0f40542 |
| Downloads\automation\automation | Taraba (tab taraba) | none — full wholesale sync was safe (tenant-prefixed, role DISTRIBUTOR) |
| Downloads\togo_automation (1)\automation | Togo (tab togo) | cdd_sync.py role: every "DISTRIBUTOR" swapped to "COMMUNITY_DISTRIBUTOR" (9 sites in current code, was 7 in the old); .env has ES_INDEX_PREFIX= present-and-empty — both load-bearing |

Method that avoids the partial-deploy trap: sync the WHOLE pipeline/ + run.py +
scheduler.py, then re-apply the copy's known divergences — never cherry-pick
single fixed files onto an older base (Taraba's copy pre-dated the PB chain;
copying only analyze.py would have recreated the ES_INDEX_PB KeyError crash).
Wholesale-copy safety check first: diff the copy's cdd_sync.py against repo-
with-role-swapped to prove the role is its ONLY local intent (Togo's was —
the rest of its drift was just an older version). cdd_sync_itn.py's
DISTRIBUTOR_REGISTRAR is Chad-ITN's own constant — never swap that file.

**numericise_ignore LOST A THIRD TIME:** the fix existed only on the airflow
branch; the ITN branch (and hence every deployment copy) never had it. Found
by marker-verification during the Taraba sync (— always grep the markers
after a sync: "NOT REPORTED", "dedup TREATED records only",
"Cumulative coverage (Days 1-", "numericise_ignore", "_pad_cycle"). Re-applied
to ITN config.py + all copies (testitn surgically, preserving DUP_MATRIX).

**Cumulative-mode impact of the fix set** (asked and verified): standard-target
union applies with divisor=1 — Overall Campaign Target becomes the full book
and NOT REPORTED in a cumulative run means "never reported in the whole
campaign" (a useful end-of-campaign list); display filtering + dedup apply the
same way; the cumulative Section-1 layout and banner were already
overall-target-first and are untouched; formula notes correctly absent.
A --cumulative run ending on Day 1 reads ~single-digit coverage vs the FULL
undivided target — mathematically right, easy to misread; for a corrected
daily Day-1 report use TEST_EXTRACT_DATE instead.

**JupyterHub env-repair sequence** (Taraba box, aarch64, Python 3.11): compiled
extensions from a rebuilt environment fail one-by-one in import order while
pure-Python packages work. run.py's bootstrap only installs MISSING packages,
so present-but-broken ones crash forever. Fix in one pass:
`pip install --user --force-reinstall --no-cache-dir numpy pandas lxml pillow matplotlib`
(numpy/pandas = the first crash; lxml = python-docx's etree import; pillow +
matplotlib = the chart, next in line). --no-cache-dir matters — pip re-serves
the same broken cached wheel otherwise. Verify with one import line before
rerunning. Typo trap seen live: `--cumlative` — argparse only surfaces it
after imports succeed.
