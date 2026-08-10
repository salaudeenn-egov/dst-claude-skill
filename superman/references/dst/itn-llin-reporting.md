# ITN/LLIN (bed-net) campaign reporting — Chad, built 2026-07/08

New drug-type support added to the `D:\DST\automation` pipeline for household-based
bed-net distribution (ITN/LLIN), alongside the existing SMC/SPAQ/AZM (per-child)
pipeline. Tenant: `chad`.

**Campaign identity (CORRECTED 2026-08-07, verified via live ES agg):** phases 2 AND 3
share ONE campaign number `CMP-2026-06-03-000312` — a terms agg over Aug 1-7 SUCCESS
records returned exactly one projectReferenceId bucket, and phase-3 docs still carry
`projectName: "LLIN_phase2_tchad2026-final-3"` (the projectName is a stale label,
never trust it for phase identity). Phases are distinguished by
`additionalDetails.cycleIndex`, which is an ARRAY (`["03"]`), matching the DSS
dashboard's "Phase" filter. Multiple `projectId` UUIDs exist under the one
projectReferenceId (per-province projects) — a projectReferenceId filter correctly
spans them all. Phase 3 ran 1-8 Aug 2026 (8 days, Day N of 8), not the 4-day cadence
recorded earlier.

## Files and the "never touch SPAQ" rule

Three fully new files, chosen deliberately over branching the existing ones:
`pipeline/analyze_itn.py`, `pipeline/cdd_sync_itn.py`, `pipeline/report_itn.py`.
`analyze.py`/`report.py`/`cdd_sync.py`/`config.py` are **never modified** — confirmed
via `git status` throughout. They reuse ONLY generic, non-branching helpers from the
originals (style constants, docx helpers, `_scroll_batches`, `_fetch_hh_head_map`) via
import. Reason given by the user: SPAQ's core computation (`is_treated` needs
`Data.age`; one child = one dose) fundamentally doesn't apply to ITN
(`beneficiaryType=HOUSEHOLD`, age always null, one household = N nets by household
size) — retrofitting would mean wrapping most of `_build_doc` in conditionals.
`run.py`/`scheduler.py` (the orchestrators, not the "originals") DO get a small
drug-type dispatch: `if cfg.get("drug_type") in {"ITN","LLIN"}: use analyze_itn/
cdd_sync_itn/report_itn else: use the SPAQ modules`.

## Chad ES/DB field-shape facts (chad-project-task-index-v1, chad DB schema)

- Campaign scope: `Data.additionalDetails.projectReferenceId` (NOT top-level
  `Data.campaignNumber` — chad's task docs have no such field).
- Facility tier: `Data.boundaryHierarchy.sppSfd` (name) / `boundaryHierarchyCode.sppSfd`
  (code) — NOT `healthFacility`/`HEALTHFACILITY` like other tenants.
- LGA tier (what ES calls "district"): `Data.boundaryHierarchy.district` (plain name,
  e.g. `ADRE`) / `Data.boundaryHierarchyCode.district` (code, e.g.
  `ADMIN_TC_16_03_ADRE`). **Chad's DB reuses Nigeria-style boundarytype labels**
  (`State`/`LGA`/`Ward`/`Community`/`Health Facility`) even though this is Chad and ES
  calls the same tier "district" — `LGA` in `chad.project_address.boundarytype` IS the
  district tier. Confirmed by cross-checking `pa.boundary` values at `boundarytype='LGA'`
  against real district names seen in ES.
- Delivery record shape: `householdId`, `quantity` (nets), `memberCount` (population),
  `taskDates` (plain date string, e.g. `"2026-07-04"`), `createdTime` (epoch ms),
  `additionalDetails.manualCodes`/`codesScanned` (bednet barcode entry method),
  `deliveredTo=HOUSEHOLD`, `individualId`/`age`/`gender` always null.
- Acting user = TOP-LEVEL `Data.createdBy` (user UUID). `clientAuditDetails`/
  `auditDetails` do NOT exist on chad task docs — a first implementation read only
  the audit paths and silently degraded (verified live 2026-08-06). No `createdTime`
  inside an audit block either, so same-date ordering has no tie-breaker.
  `userName`/`nameOfUser` ARE present top-level on task docs (e.g.
  `katarfacskatarfa2`).
- Task `Data.householdId` = the household **clientReferenceId** (verified: exact match
  on `chad-household-index-v1` `Data.household.clientReferenceId`; the server id is
  idgen-form `H-2026-03-08-…`). Household index wraps fields under `Data.household.*`.
  This is the join key to `chad.household.clientreferenceid` in the DB.
- `additionalDetails.name` is the PRODUCT label (`"Milda"` = MILDA, French acronym for
  LLIN — confirmed: it equals top-level `productName` on every sampled doc), NOT a
  person's given name. The head's name is `additionalDetails.familyNameOfIndividual`
  alone — device-entered, with rampant case/trailing-space variants ("mahamat",
  "Mahamat", "mahamat ") and abbreviations ("mht") that are separate ES keyword values.
- CDD sync index (`chad-user-sync-index-v1`): role field IS present directly on sync
  docs (`Data.role.keyword`) — only two roles ever sync: `DISTRIBUTOR_REGISTRAR` (the
  real CDD role, ~96% of field staff) and `WAREHOUSE_MANAGER`. Plain `DISTRIBUTOR`
  (the role `cdd_sync.py` filters other tenants by) is <4% here — using it would
  undercount by >96%. Sync index ALSO carries `additionalDetails.projectReferenceId`
  and genuinely mixes multiple campaigns' records (proven: a same-tenant sample doc had
  a different campaign's reference) — campaign-scoping is required, not precautionary.

## BUG — target-matching used facility name, never matched anything

**Problem:** every facility/LGA showed `Target = 0`, `Status = NO TARGET` /
`N/A` coverage everywhere, even though real ES activity and a real target file both
existed. **Approach:** compared the target CSV's key column against what ES actually
returns for the same tier. **Root cause:** `chad.project_target` rows are keyed to
`chad.project_address.boundary`, which at `boundarytype='LGA'` is a CODE
(`ADMIN_TC_16_03_ADRE`), not the plain name (`ADRE`) ES exposes at
`boundaryHierarchy.district` — matching on name (or on `facility_name`, at the wrong
grain entirely) silently produced zero matches every time. Confirmed the fix key via
live ES: every task doc with `boundaryHierarchy.district="ADRE"` carries
`boundaryHierarchyCode.district="ADMIN_TC_16_03_ADRE"`, an exact match to the DB code.
**Fix:** match strictly on the CODE field (`boundaryHierarchyCode.district`), never the
name. Verified: ADRE target household/population (83,978 / 404,570) loaded and matched
correctly end-to-end.

DB query (chad tenant) for LGA-level targets, all three beneficiary types confirmed
present (population from INDIVIDUAL, household from HOUSEHOLD, bednet/ITN from
PRODUCT — PRODUCT confirmed live, matches an independent microplan-Excel cross-check
exactly, e.g. ADRE net_target 202,850 both ways):
```sql
SELECT pa.boundary AS lga_code,
       SUM(CASE WHEN pt.beneficiarytype='INDIVIDUAL' THEN pt.targetno ELSE 0 END) AS population_target,
       SUM(CASE WHEN pt.beneficiarytype='HOUSEHOLD'  THEN pt.targetno ELSE 0 END) AS household_target,
       SUM(CASE WHEN pt.beneficiarytype='PRODUCT'    THEN pt.targetno ELSE 0 END) AS net_target
FROM chad.project p
JOIN chad.project_target pt  ON pt.projectid = p.id AND p.isdeleted = false
JOIN chad.project_address pa ON pa.projectid = p.id AND pa.boundarytype = 'LGA'
WHERE p.referenceid = '<campaign reference id>'
  AND pt.beneficiarytype IN ('INDIVIDUAL','HOUSEHOLD','PRODUCT')
GROUP BY pa.boundary;
```
`analyze_itn.py`'s `_load_targets_itn` accepts flexible column-name aliases
(`lga_code`/`district`/`boundary`/`LGA`; `population_target`/`individual_target`/
`TargetPopulation`; `household_target`/`TargetHouseholds`; `net_target`/
`product_target`/`TargetBednets`), case-insensitive, so a differently-named export
still works without a code change.

## Aggregation grain changed to LGA, not health facility (user-directed)

Reason: 1,000+ health facilities but only ~40 LGAs for this campaign, and target data
only ever exists at LGA grain (never facility grain) — matching 1,000+ facility names
1:1 is far more fragile than ~40 LGA codes, and was also never actually the shape of
the real target data. Facility-level detail (records, DQ, low-activity) is still
scrolled and kept in a separate `FACILITY DETAIL` Excel tab, just without a
target/coverage/status column (none exists at that grain — showing one would be
fabricated). "Low Activity Facilities" (records<10) replaces SPAQ's "LOW Coverage
Facilities" section for the same reason.

**BUG — zero-activity LGAs silently vanished from totals.** Problem: `LGAs Covered`
fluctuated day to day (31 of 38, etc.) and the summed Daily/Cumulative Target
ITNs/Households/Population grew or shrank depending on how many LGAs happened to have
a task record that day — making every day's row incomparable to any other. Root
cause: the LGA-rollup (`_aggregate_to_lga`) only ever creates a row for an LGA that
has at least one fac_row that day; an LGA with zero records that day just never
appears, so its target drops out of the sum entirely. Fix:
`_fill_zero_activity_lgas(lga_rows, target_map)` adds a zero-activity placeholder row
(all-zero activity, correct target from `target_map`, LGA name pulled from the target
CSV's own name column) for every LGA present in the target file but absent from that
day's fetch — these correctly band as `LOW ACTIVITY` (records<10 checked first in
`_band()`, matching `analyze.py`'s exact priority order). Verified: a 3-LGA synthetic
test with only 1 active LGA still summed all 3 LGAs' targets correctly after the fix.

**REGRESSION (2026-08-07): this fix was LOST and had to be rebuilt.** The Day 7
phase-3 report was missing ISSEIROM + KOULOUDIA entirely, and its Daily Target
ITNs was short by exactly their 11,700 daily share (target CSV ÷ 8, per-LGA rounded,
matched to the digit on all three measures — that arithmetic proof is the fastest way
to distinguish "targets miscalculated" from "LGAs missing"). `grep _fill_zero_activity`
returned nothing: the documented, verified fix no longer existed in the working tree —
lost as uncommitted work (same failure mode as the recon-recovery branch-reset
incident). Restored as `_fill_zero_activity_lgas(lga_rows, target_map, dup_value)`
(target CSV now also carries `lga_name`; `dup_value` keeps duplicate-matrix columns
uniform: 0 when the matrix measured, None when not). Lesson, twice paid for now:
a fix that isn't committed does not exist.

## BUG — no date filter at all; "Today" and "Cumulative" were the same number

**Problem:** `analyze_itn.py` had no date scoping — every run re-fetched and re-summed
the ENTIRE campaign's records from day one. On Day 1 this is invisible (cumulative-to-
date IS today's total), but from Day 2 on there's no way to isolate "today's activity"
from the one ever-growing total. First patched with a delta (today's cumulative minus
yesterday's saved snapshot) — user correctly rejected this as unreliable ("what if
yesterday's snapshot wasn't generated"). **Root cause / real fix:** `analyze.py` (SPAQ)
already solves this with a real per-day ES range filter
(`{"range": {f"Data.{date_field}": {"gte": cfg["GTE"], "lte": cfg["LTE"]}}}`,
`date_field` defaulting to `"taskDates"`) — `config.py` already generically computes
`cfg["GTE"]`/`["LTE"]` for every row (today's bounds for a daily run, whole-campaign
bounds for `--cumulative`), tenant/drug-agnostic. Confirmed via live ES that
`Data.taskDates` on chad's task docs is the SAME field name AND a genuine ES `date`
type (a GTE/LTE range query returned a correct non-zero count, not a keyword-string
false-negative) — so the identical mechanism just works, zero override needed.
`analyze_itn.py` now applies this filter exactly like SPAQ; `itn_history/
performance_day{N}.xlsx` snapshots are now genuinely per-day (not cumulative-to-date),
and `report_itn.py` sums them for the cumulative view exactly like `report.py`'s
`_load_all_days_perf` — no more delta/subtraction logic anywhere.

Daily-vs-cumulative target: the target CSV holds the FULL campaign-length figure
(not a pre-divided daily allocation). For a daily run, `analyze_itn.py` divides every
target by the real campaign length in days
(`cfg["campaign_start"]`/`["campaign_end"]`, NOT `cfg["campaign_days"]` — `config.py`
defaults that field to 4 when a sheet row doesn't set it, an unrelated trap) before
using it as the Status/Coverage denominator — same "Total Campaign Target = Daily
Target × Campaign Days" relationship `analyze.py`'s own daily mode uses. Cumulative
runs (`--cumulative`) use the full undivided target directly.

## BUG — stale historical snapshot silently corrupts every future day's chart/table

**Problem:** a day's `itn_history/performance_dayN.xlsx` snapshot was written by
OLDER pre-fix code (undivided target, before the zero-activity-LGA fix) and, since
past snapshots are never retroactively regenerated, its wrong stored target value sat
frozen on disk forever. **Approach ruled out:** re-querying ES for that historical
date to regenerate the file (fragile, needs server access, risks re-triggering
Slack/Drive side effects if done via the full pipeline). **Real, permanent fix:**
stop trusting ANY historical day's own stored target value at all, for anything.
`_load_all_days_perf_itn` now only reads/returns each day's own ACTIVITY figures
(nets distributed, households visited, population covered — genuine historical data,
doesn't go stale) and a running cumulative sum of those; it does NOT read the Target
HH/Pop/ITN columns from history at all anymore. The chart and the progress table both
take `current_daily_target` as an explicit parameter, always computed fresh from
**today's** live `g["net_target"]`, never scanned/summed from history. This means a
frozen stale snapshot can sit on disk indefinitely with zero effect on any current or
future report — no regeneration of old files is ever needed.

**Related, subtler bug (same root cause, one level up):** even after fixing the
denominator source, the CHART and the progress TABLE were found to use two genuinely
different formulas for what read as "the same cumulative coverage" number (1.8% vs
7%), confusing the user — turns out `report.py` (SPAQ) itself has this exact
duality: the chart's `_generate_progress_chart` divides by the FULL, fixed campaign
target for every bar (only nears 100% at campaign end), while the Section 4 table's
`cum_target = sum(d["target"] for d in days_data)` (elapsed days' own targets summed —
a "pacing" metric, are we hitting our average daily target so far). Both are real
SPAQ formulas, just for different sections. For ITN, deliberately made BOTH the chart
and the table use the pacing formula (`current_daily_target × elapsed_days`, per-bar
for the chart so each bar's own elapsed-day-count sets its own denominator) — a
disclosed departure from SPAQ's literal chart formula, chosen specifically so chart
and table never show two different percentages for what looks like one metric.
Verified: Day 1 = 7.1% (40,382/568,442), cumulative TOTAL = 6.0%
(68,307/1,136,884), identical in both the table and the rendered chart PNG.

## CDD sync adaptations (`cdd_sync_itn.py`)

- **Roster is sync-derived, not assigned-staff-derived** (disclosed adaptation): SPAQ's
  `cdd_sync.py` roster comes from the STAFF index (assigned CDDs), so it can show a
  true "NEVER SYNCED" CDD (assigned but inactive). That path is blocked for chad — no
  correct campaign-scoping field exists on `chad-project-staff-index-v1`
  (`additionalDetails.cycleIndex` doesn't cover the current cycle at all on staff
  records; `projectTypeId` alone spans every historical LLIN campaign/pilot for the
  tenant). Real fix needs a DB `projecthierarchy` join
  (`SELECT id FROM chad.project WHERE projecthierarchy LIKE '<root>.%'` → filter staff
  by `Data.projectId IN (...)`) — not built, left as a TODO. Instead the roster is
  built FROM the sync records themselves (real, verified), which means a CDD who has
  NEVER synced cannot appear at all — disclosed in the workbook's SUMMARY note row.
- **Status model matches SPAQ exactly (HIGH/MODERATE/LOW/NEVER SYNCED)** — NOT the
  recency-based ACTIVE/DORMANT model tried first. Initial reasoning ("continuous
  months-long campaign, 'synced every day' is meaningless") was correct for phase 2 but
  wrong for phase 3, which is a real short 4-day cycle exactly like SPAQ's own cadence
  — so SPAQ's day-count thresholds (`_sync_status`: HIGH if `n==day`, MODERATE if
  `n>=3`, LOW if `n>=1`) genuinely apply. `NEVER SYNCED` always reads 0 (structural,
  not a bug — see roster note above), column kept for exact table-shape parity.
  User's explicit, repeated instruction on this class of gap: **match SPAQ's literal
  structure/thresholds exactly, don't independently redesign** — even a well-reasoned
  adaptation should be revisited once a fact changes (short vs long campaign).
- **Time-cutoff sync metric works identically to SPAQ**: `Data.createdTime` (epoch ms)
  on chad's sync docs, same `_count_synced_by_cutoff` GTE/LTE-on-epoch-ms mechanism,
  confirmed live (418 CDDs synced by 17:00 UTC on a real day). Gated the same way SPAQ
  gates it: skipped entirely for `--cumulative` runs (meaningless whole-campaign).

## Report structure — mirrors `report.py` section-for-section, LGA terminology throughout

Sections: 1 Campaign Overview (Daily+Cumulative split, matching SPAQ's daily-mode
dual block) → 2 Program Issues (Claude/Groq) → 3 Distribution Data Analysis (3.1
Performance by LGA, 3.2 DQ by LGA, 3.3 Low Activity Facilities, 3.4/3.5 DQ Summary) →
4 Campaign Progress (day-wise table + chart, chart embedded in Section 1 like SPAQ) →
5 CDD Sync Activity → 6 Conclusion. Partner variant strips DQ sections, same trigger
condition as SPAQ (`slack_channel_partners` configured, or always in `--cumulative`).
Hyperlinks (Issues table "Data" column → Performance/CDD-Sync Excel, "Full list"/"Full
CDD sync data" notes) match SPAQ's exact pattern — an early gap found by diffing a real
Chad report against a real Oyo (SPAQ) report side by side.

Terminology: everything renamed `District`→`LGA` (was a genuine confusion point —
Chad's DB happens to call this tier "LGA" even though it's Chad, not Nigeria; "district"
in earlier code was an invented label, not what either ES or the DB actually call it).
"ITN/LLIN Bed-Net Distribution Report" wording simplified — `campaign_name` (e.g.
"ITN Phase 3") is sufficient on its own in the Slack heading and Word subtitle; don't
also append a redundant descriptor.

**Found via direct diff against a real SPAQ report (Oyo Day 2), not by memory** — this
is the most reliable way to catch "matches SPAQ" gaps: dump both `.docx` files'
paragraphs/tables/hyperlinks with `python-docx` and compare side by side, rather than
recalling report.py's structure from context. Caught: missing header 3rd summary line
(Day N — Date | Extract | LGA count | Coverage), missing hyperlinks, wrong CDD-sync
column headers, wrong facility-table columns — all found this way, none from re-reading
code alone.

## Duplicate-distribution matrix (`dup_matrix`, built 2026-08-06)

New feature in `D:\DST\automation` (`analyze_itn.py` + `report_itn.py` + `config.py`),
tenant `chad`, campaign `CMP-2026-01-24-000223`, index `chad-project-task-index-v1`.
**UNCOMMITTED on branch `ITN` as of 2026-08-06 despite repeated prompts — commit first
thing next session** (see `feedback_commit_durable_fixes` memory: uncommitted work has
been silently lost on a branch reset before).

**Aim:** find households treated more than once, especially by DIFFERENT users (same
day = coordination gap; different days = double distribution). 4 buckets only
(unknown-user buckets built then removed on request; user-missing duplicates are
EXCLUDED, with >50%-missing log warnings).

**Matching-key evolution (Problem -> Approach):**
- `householdId`-only matching found 0 different-user dups — misleading first read.
  Root cause: a re-served household gets RE-REGISTERED under a brand-new `householdId`.
- v2 identity = head name + memberCount + LGA -> 1,329 `du_dd` (52.9% of records) —
  false positives. Proven by live agg: "mahamat" alone = 15,169 DISTINCT households
  across 46 LGAs (~330/LGA); top-30 family names all in the thousands; case/spacing
  variants ("mahamat", "Mahamat", "mahamat ") are separate ES keyword buckets.
- v3 (settled): identity = normalised(name+familyNameOfIndividual) + memberCount +
  VILLAGE (`additionalDetails.administrativeArea`, facility-code fallback) -> 266 `du`
  records, reviewable. `householdId` is display-only (removed as grouping key on
  request).
- History = per-run batched terms lookup on `familyNameOfIndividual.keyword` with
  case/trailing-space VARIANTS probed (exact-term prefilter misses variants otherwise),
  `taskDates < today`, client-side identity verification. Cumulative runs skip history
  (window covers all).

**Verified chad ES field facts (live queries 2026-08-06):**
- Acting user = TOP-LEVEL `Data.createdBy` (`clientAuditDetails`/`auditDetails` absent
  on chad; code tries `clientAuditDetails.createdBy` -> `Data.createdBy` ->
  `auditDetails.createdBy`). No `createdTime` on chad -> within-day tie-break falls
  back to scroll order.
- Task `Data.householdId` = household **clientReferenceId** (matched
  `chad-household-index-v1` `Data.household.clientReferenceId`; server id is idgen
  `H-2026-03-08-...`). Household index wraps under `Data.household.*`.
- `additionalDetails.name` was "Milda" on multiple unrelated households — SUSPECTED
  product label (MILDA = French acronym for LLIN), NOT a given name. Agg on
  `Data.additionalDetails.name.keyword` still pending; if confirmed, strip it from
  head-name display (matching unaffected — constant on both sides).

**Design invariants:** opt-in via in-code `DUP_MATRIX_DEFAULT` in `config.py` (sheet
column `dup_matrix` overrides per row; user refused a required sheet column);
fail-soft (any enrichment failure -> None columns, report proceeds); None = "not
measured", never rendered as 0; matrix appended at END of Excel headers (positional
readers unaffected); internal report only — partner doc never renders it, partner
Excel strips the 4 Dup columns AND drops the whole DUPLICATE DETAIL tab (names + CRIDs
+ user UUIDs). Word section merged INSIDE Data Quality Summary as .3 (flat Duplicate
Records row removed when matrix measured); formula line is equation-only; Slack line
removed on request. DUPLICATE DETAIL is HOUSEHOLD-grain: one row per house with all
repeats joined ("; "-separated, position-aligned), original anchored to
campaign-earliest record (repeats never chained); detail evolved: du-only -> du_dd-only
-> all-types-one-sheet-with-Type-column -> FINAL: ONE SHEET PER TYPE ("DUP DIFF USER
DIFF DAY" etc., severity order, sheet only when non-empty; partner strip removes by
"DUP" prefix; mixed household appears on each relevant sheet with only that type's
repeats — no Type column needed); Word note hyperlinks the performance Excel
(`perf_link` threaded through `_dq_summary_table -> _dup_matrix_section`); UNCAPPED
(500-cap removed), CRID-labelled columns (task `householdId` = household
clientReferenceId, verified); user columns show USERNAMES via `_fetch_usernames`
(staff `Data.userId -> userName`, then user-sync `syncedUserId -> syncedUserName`
fallback; unresolved -> raw UUID, own fail-soft; `createdBy==staff.userId` equivalence
UNVERIFIED on chad — check "dup usernames: N/M resolved" log on first real run).
Offline test suite (58 checks) at scratchpad `test_dup_matrix.py` pattern — rebuild if
needed.

**Known limits (disclosed in code):** spelling/abbreviation drift ("mht" vs "mahamat")
and different village spellings -> undercount; same-name+size family in one village ->
rare overcount (adjudicate via Village column + GPS query).

## Cumulative undercount RCA (2026-08-07, Day 7 vs dashboard)

**Problem:** report cumulative 602,770 nets vs dashboard/ES 1,288,022 (~2.1x). Ruled
out: NOT the targets (verified digit-exact vs `chad_target` CSV / 8; only
ISSEIROM+KOULOUDIA missing = zero-activity LGAs dropped from rows/denominator, still
unfixed) and NOT the filter (single `projectReferenceId CMP-2026-06-03-000312` holds
all Aug 1-7 data, matches dashboard; multiple `projectId`s under one referenceId is
normal).

**Root cause:** `itn_history` day files are FROZEN at each day's ~17:15 report time,
but chad syncs roughly half a day's records LATER (`taskDates` unchanged) — no later
daily window ever re-captures those late-arriving records. Proven live: Day-7 ES
103,508 nets vs the report's stored 96,310; window total grew 1,285,727 -> 1,288,022
between two queries taken hours apart on the same day.

**Fix (implemented):** `_load_days_from_es_itn` in `report_itn.py` — per-day ES aggs
computed AT REPORT TIME (cardinality `householdId` ~40k precision, sum `quantity`, sum
`memberCount` = dashboard semantics), now the PRIMARY source for chart/progress/
cumulative; the frozen file-sum is fallback only. Phase 3 = 8 days Aug 1-8;
`cycleIndex` is an ARRAY (`["03"]`) on task docs.

## Edge-case hardening (2026-08-06 audit, self + agent review)

Dateless records excluded (would sort first and poison the original); per-record event
build guarded (malformed doc = 1 skipped event, never a legacy-report crash);
`_user_and_ts` defensive vs non-dict audit blocks; DUP-sheet writing has its own try
(writer failure drops DUP sheets only, keeps legacy tabs); control chars scrubbed +
"="-prefix forced to string (openpyxl formula injection) + joined cells truncated with
visible "(+N more)"; reader pads short rows and uses `int(float())`; mixed
measured/None rows warn; sort tie-break (date, ts, hh); Groq issues prompt gets NO dup
facts (`issues_data` is shared with the PARTNER doc — leak closed); cumulative run no
longer overwrites `itn_history` daily snapshot. All comments rewritten to factual voice
(no session narration). Offline suite ~76 checks.

## BUG — cumulative undercounted ~2x vs dashboard: frozen day snapshots vs late syncs

**Problem:** Day 7 report showed Cumulative Nets 602,770 / coverage ~25%; the DSS
dashboard showed 1,283,644 / 46.9% for the SAME Aug 1-7 window. **Misleading first
reads (all disproven in order):** (1) target calculation wrong — no: report daily
targets matched the target CSV ÷ 8 to the exact digit per LGA; (2) campaign started
before Aug 1 — no: dashboard re-filtered to exactly Aug 1-7 barely moved; (3) campaign
filter too narrow — no: a terms agg showed ONE projectReferenceId holding all the
window's data, matching the dashboard. **Root cause:** each day's numbers are frozen
into `itn_history/performance_day{N}.xlsx` at that day's ~17:15 report time; on chad
roughly HALF of a day's records sync later (evenings/days later, `taskDates`
unchanged), and no later daily run can ever capture them (each run's window is its own
day). Every day's late syncs are therefore lost from the file-summed cumulative,
permanently, compounding to ~2x. **Settling evidence (live):** Day-7 ES nets 103,508
vs the 17:15 report's 96,310; the 7-day window total grew 1,285,727 → 1,288,022
between two identical queries hours apart — data still arriving for past days.
Unique-bednet-code counts (~1.25M) also ruled out status double-counting.
**Fix:** `report_itn._load_days_from_es_itn` — per-day ES aggregations at report time
(cardinality `householdId` with precision_threshold 40000, sum `quantity`, sum
`memberCount` = the dashboard's exact semantics), now the PRIMARY source for the
cumulative overview rows, progress table and chart; the file-sum is fallback only
(logged warning). Mirrors SPAQ's `_load_days_from_es` (report.py), which existed
precisely because Nigeria hit this same wall — the ITN fork simply never ported it.
Doctrine both families now share: **today's figures = honest intraday snapshot;
historical/cumulative series = recomputed from ES on every run.** Corollary found
during the same audit: a `--cumulative` run must never write an itn_history snapshot
(it would overwrite a daily file with campaign-wide figures) — now guarded.

## Duplicate-distribution matrix (dup_matrix) — household re-treatment analysis

Opt-in feature (in-code `DUP_MATRIX_DEFAULT` in config.py; per-row `dup_matrix` sheet
cell overrides) answering: was a household treated more than once, and by the same or
a different CDD, same day or different days?

**Problem:** partners needed double-distribution detection; the naive key
(householdId) found only sync retries — 0 different-user duplicates, because a
re-served household is RE-REGISTERED under a brand-new householdId, which ID-matching
is structurally blind to. **Approach evolution, each step forced by live evidence:**
(1) name+members+LGA identity → 52.9% of records flagged as different-user/different-
day — false positives, proven by agg: "mahamat" alone = 15,169 DISTINCT households
across 46 LGAs (~330/LGA), all case/space variants separate keyword buckets;
(2) settled key = normalised(familyNameOfIndividual) + memberCount + VILLAGE
(`additionalDetails.administrativeArea`, facility-code fallback) → 266 different-user
records, a reviewable list. householdId is display-only.

**Classification:** per identity group, earliest record by (taskDates, createdTime,
householdId) = the original; every LATER record in the run's window lands in exactly
one of 4 buckets (same/diff user × same/diff day, user = `createdBy` UUID — one-to-one
with username, no join needed for matching). History for cross-day evidence = batched
terms lookup on familyNameOfIndividual (case/trailing-space VARIANTS probed — exact
terms miss them otherwise) with `taskDates < today`; cumulative runs skip it (window
already spans campaign). Records missing name, user, or date are EXCLUDED, never
guessed (undercount direction only; loud log warnings when the excluded share is
large). Excel: 4 flagged columns appended at the END of every tab (positional readers
unaffected; None = "not measured" renders blank, never 0) + one UNCAPPED sheet per
type ("DUP DIFF USER DIFF DAY" etc., severity order, sheet only when non-empty),
household-grain (one row per house, repeats joined and position-aligned), usernames
resolved via staff→user-sync join (raw UUID fallback), CRID-labelled household
columns. Word: 4-row table inside Data Quality Summary (internal only), equation-only
formula line, one-sentence method note with Drive link. Partner outputs: every DUP*
sheet dropped whole, 4 columns stripped, NO duplicate facts in any Groq prompt
(issues/conclusion/Slack are shared with partner surfaces — the legacy
"Duplicate records: N" line was removed from all three for the same reason).
Everything fail-soft: enrichment failure → None columns + report proceeds; DUP-sheet
writer failure → legacy tabs survive; per-record event build guarded; device text
sanitised (control chars stripped, "="-prefix forced to string — openpyxl treats it
as a live formula), joined cells truncated visibly. Offline test suite (~84 checks)
pattern: scratchpad test_dup_matrix.py.

**SMC counterpart (analyze.py, 2026-08-07, requested by vashisht.kumar):** SMC's
Duplicates column redefined — key = individual clientReferenceId (resolved via the
task→PB→individual chain, NOT raw `Data.individualId`; vashisht confirmed CRID is the
intended ID) + `createdBy` + head + child + ward + age, ADMINISTRATION_SUCCESS ONLY.
Beneficiary record pattern he supplied: 1 SUCCESS + 2 DELIVERED + 1 VISITED(redose)
per child — DELIVERED is never queried by the pipeline at all; VISITED is counted in
Redose, now excluded from duplicates; REFUSED-then-treated revisits no longer count.
Disclosed consequence of username in the key: the same child treated by two DIFFERENT
CDDs is invisible to this column (the ITN matrix is the portable answer if that's ever
wanted). Numbers drop campaign-wide from the first run — definition change, not data.

## Open / not yet done
- **Commit the dup_matrix feature** — still uncommitted on branch `ITN` as of
  2026-08-06/07; highest-priority open item, do this before anything else next session.
- Non-Administration Analysis section (household-equivalent of Absent/Refused/
  Ineligible/Referred/Died/Migrated) — no ES query has ever confirmed what non-
  `ADMINISTRATION_SUCCESS` statuses exist for chad's household deliveries.
- CDD roster true coverage % (assigned-staff denominator) — needs the DB
  `projecthierarchy` join noted above.
- `LGAs Covered` in the Overview always shows the static target-file count now (e.g.
  38) rather than "X active of Y total" — the zero-activity-LGA fix means every
  target LGA always has a row, so the "how many actually reported today" signal was
  traded away; flagged to the user as an easy follow-up (`X of Y` format) if wanted.
- Head Name display in DUP sheets still shows the composed "name + family" string,
  i.e. "Milda ali" — since `additionalDetails.name` is the product label, the display
  should be `familyNameOfIndividual` alone. Matching is UNAFFECTED (the constant
  "milda" prefix is on both sides of every comparison); display-only cleanup pending.
- **The entire 2026-08-06/07 change set (dup matrix, ES back-fill, zero-activity
  restore, SMC dedup redefinition) was still UNCOMMITTED on branch `ITN` as of
  2026-08-07** — the zero-activity fix has already been lost once this way. Commit
  before anything else moves.
- ISSEIROM+KOULOUDIA (zero-activity LGAs) still missing from rows/denominator in the
  cumulative-undercount context — separate from the itn_history freeze fix, not yet
  resolved.
- `additionalDetails.name` = "Milda" (suspected MILDA/LLIN product label, not a given
  name) — agg confirmation still pending; strip from head-name display once confirmed.
- `createdBy==staff.userId` equivalence on chad — unverified; check the "dup usernames:
  N/M resolved" log on the first real dup_matrix run.
