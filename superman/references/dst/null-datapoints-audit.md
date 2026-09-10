# Null Data Points Audit — D&E CHILDREN_TREATED extracts (SMC 2026)

Source: 2026-09-09/10 sessions; memory `project_user_sync_activity_reports.md` +
`feedback_multisheet_exports.md`. Tooling in `Downloads`: `dump_analysis.py` (analyzer),
`_dq_tools/sheet_watcher_v2.py` (5-min live push), `_dq_tools/sheet_update_v2.py`
(final xlsx-verified push + old->new diff). Output `NULL_DATA_POINTS_AUDIT.xlsx`
(single sheet `missing_data_points`) mirrored to the user's Google Sheet
"Null Data Points Audit - SMC 2026", tab `NULL_DATA_POINTS_AUDIT` (pushed via the DST
service account, which the user granted Editor on that sheet).

## What it measures

Per (state, cycle): null counts for HH Head Name / Child Name / Child Age / Gender /
Lat / Long, plus SPAQ1 (3-11m) / SPAQ2 (12-59m) age-band mismatches, over the extracted
D&E Reports dump (local copy `C:\DE`, Drive tree `D_AND_E Reports/`). 32 state-cycle rows
across 13 tenants (ba bo ch ab jg kd ke ko na oy pl so zf).

## Rules (user-set, verified; v3 2026-09-10)

- **Child Name AND HH Head Name = blank-only missing** — both are mandatory typed input,
  so any typed text ("na"/"nan") counts as filled — EXCEPT a literal `DECRYPT_FAILED`
  value, which is a decryption-pipeline artifact (not a name) and always counts as
  missing PLUS gets its count called out in the Note column
  ("DECRYPT_FAILED values (decryption pipeline failure, counted as missing): ...").
  Non-name data points use the null-string set
  (`"", null, nan, none, n/a, na, decrypt_failed`). Impact of v3 rule change: Bauchi C1
  HH-head 68 -> 26, C2 53 -> 88 (v2 had also changed), C3 -> 6; Chad 1,606 -> 1,604
  (Chad's HH-head nulls are real blanks, and its Referred column has 0 "yes" rows in the
  export, so referral exclusion changes nothing there).
- **Referral/OIC rows excluded** from null counts (structural: no HH link, no GPS —
  100% verified). Matched via `Children Referred == Yes` or
  `Administration Status == Children Referred`, fallback username contains OIC.
- **VAS/ORS rows excluded from GPS check** (secondary-product rows carry no GPS by design;
  Bauchi C1 excludes 1,740,659 VAS rows).
- **INELIGIBLE rows ~96% GPS-missing** (Borno-verified) — app flow doesn't capture GPS on
  the ineligible screen; high-GPS-missing states are partly structural.
- **SPAQ2 "mismatches" in later cycles are cohort aging**, not entry error: children hit
  60-61 months between cycles (Bauchi C3 verified — all 36,400 were 60-61m).
- **Group by the in-file `Cycle Index` column, never folder names** (folders lie: Borno
  "cycle 2" file was a fuller cycle-1 re-export; Sokoto has ONLY cycleIndex 01/02 — the
  tracking-sheet Cycle-3 row is wrong). Drop sliver cycles < 1,000 rows (stray docs inside
  cumulative extracts). Best source per (state, cycle) = file-set with most rows; never
  merge overlapping extracts. "(1)" filename twins = re-downloads, prefer them; UPDATED_*
  sets supersede plain ones.

## Multi-sheet exports gotcha (the v1 undercount — MANDATORY read)

**Problem.** v1 audit numbers looked plausible but many state-cycles sat at exactly
1,000,000 rows, flagged "truncated at export cap". User challenged Sokoto's count.

**Misleading first read.** Exactly-1M row counts pattern-matched to the known 1M export
truncation, so undercounts were explained away as truncation.

**Approach.** Listed `sheet_names` via `CalamineWorkbook.from_path()` across the dump:
15 of ~34 workbooks are MULTI-SHEET.

**Root cause.** The export generator spills overflow rows into `Sheet2`/`Sheet3` inside
the same workbook (and Chad's file is 7 per-district sheets: BATHA..Sila).
`pd.read_excel` without `sheet_name` reads ONLY the first sheet. A Sheet1 at exactly
1,000,000 / 1,048,575 rows usually means "continues on Sheet2", NOT truncated; real
truncation = the LAST sheet hits the cap.

**Fix/verify.** `pd.read_excel(fp, sheet_name=None)` and concat all sheets; truncation
flag only when the final sheet is at the cap. Corrected totals: Chad C1 118,135 ->
1,225,851; Bauchi C2/C3 1.0M -> ~2.17M each; Kebbi C2 -> 1.50M; Sokoto C2 -> 1.25M;
Kogi C3/C4 -> ~1.34M; 14 state-cycles rose. Truncation notes dropped from 20 rows to 12.

## Chad extract format is DIFFERENT (blank vs 0 semantics)

Chad's CHILDREN_TREATED.xlsx is a custom generator: 17 columns
(`Beneficiary Reference ID, Individual ID (if resolved), Province, District,
Health Facility, Locality, Username, Child Name, Age, Gender, Quantity Administered,
Redose Quantity Administered, Child Beneficiary ID, Household Head Name, Cycle Index,
Ineligible, Referred`) across 7 province sheets. Consequences:

- **No latitude/longitude columns at all** (location is Province/District/HF/Locality
  text) and **no productName** -> Lat/Long and SPAQ are BLANK on the audit sheet =
  unmeasurable from this extract, not clean. ROOT CAUSE found 2026-09-10: the Chad
  generator (`children_cohort.xlsx` cohort script) DOES fetch latitude/longitude/
  productName in `_source` but drops them at display-column selection — see
  `dst/cross-cycle-cohort-reports.md`. On the sheet, blank = column absent,
  0 = checked and clean — keep that distinction (the live watcher initially showed 0;
  the xlsx-verified final push corrected it to blank).
- **Referral exclusion did NOT trigger for Chad**: its columns are named
  `Referred`/`Ineligible`, not the Nigeria names the mask looks for — so Chad's 1,606
  HH-head nulls still include structural referral blanks (true DQ count is lower).
  OPEN: add Chad's column names to the referral mask and re-run the Chad file only.
- Chad DQ on the columns it has: Child Name 0, Gender 0, Age 27, HH Head 1,606 of
  1,225,851 rows.

Other per-state extract quirks: Kebbi C3's winning source lacks `productName` (SPAQ taken
from the alternate extract: 0 / 19,584 as an override); Kebbi children reports contain NO
ORS rows (ORS is in separate ORSZ_TREATED files).

## Live-push pattern (log-tail -> Google Sheet)

The analyzer writes an append-only log with one `=> <state> cycle <n>: ...` summary line
per completed source. A watcher regex-parses the log every N minutes and rewrites the
sheet tab with all rows seen so far (best source = max rows wins), notes rows
"live: analysis in progress" until the final table lands, then exits. After it exits, run
the xlsx-verified updater: it diffs sheet-vs-xlsx per cell, prints old->new, rewrites the
tab. Sequence matters — wait for the watcher to exit before the final push or it will
overwrite it on its next tick. Background exit code -1 from the harness does NOT mean the
script failed — check the log for "FINAL TABLE"/"Written:".

## Perf + surviving OS low-memory kills (v3 lesson)

`pd.read_excel(engine="calamine", usecols=<callable>)` ~25-60s per 1M-row sheet (load-
dependent); the full 34-source dump = ~31-60 min. Read only needed columns via the
usecols callable; `del df` between files.

**Problem.** On a RAM-starved machine (<1.5 GB free; Power BI msmdsrv / vmmemWSL eating
memory) the Claude Code harness KILLS background tasks whenever system memory runs low —
even tiny ones like the sheet watcher, and even after per-sheet refactoring. Four
launches died mid-read.

**Fix stack (all three now in dump_analysis.py):**
1. **Per-sheet processing** — never `sheet_name=None` + concat (holds whole workbook
   twice); list `CalamineWorkbook.sheet_names`, read/process/`del`/`gc.collect()` one
   sheet at a time. Truncation flag = LAST sheet at cap.
2. **Checkpoint/resume** — after each source set, serialize its aggregates to
   `dump_analysis_ckpt.json`; on restart skip done sources and RE-EMIT their `=>`
   summary lines to the log (keeps the sheet watcher's regex feed complete). Checkpoint
   auto-deleted after a full run.
3. **Foreground time-budget chunking** — the memory killer targets BACKGROUND commands
   only, so run `python dump_analysis.py <budget_seconds>` as repeated FOREGROUND calls
   (budget ~120-300s; check fires between sources, so budget + worst single source must
   stay under the 600s foreground tool ceiling). Each call resumes from checkpoint.

Also: a background task's exit code -1 in the harness does NOT mean script failure —
check the log for "FINAL TABLE"/"Written:".

## Scope note

The D&E dump contains ONLY SMC-family campaigns (Nigeria SMC 10 states + Chad SMC +
Taraba SMC + Togo retrospectives) — NO ITN campaign extracts; ITN work (Chad dup-matrix,
Ondo recovery) lives in ES/automation, not this Drive tree. Stock report folders in the
dump are a separate deliverable, not covered by this audit.
