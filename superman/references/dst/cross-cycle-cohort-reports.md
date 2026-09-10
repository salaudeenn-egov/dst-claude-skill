# Cross-Cycle Cohort / Reuse Reports (CHILDREN_TREATED cohort pivot)

Source: 2026-09-10 session — review of the Chad cohort generator (`children_cohort.xlsx`,
tenant `ch`, campaign CMP-2026-06-27-000416) and a Sokoto (`so`) cohort/new-registration
draft. This is the report family that answers "was the child administered in Cycle 1 AND
again in Cycle 2?" (master-data reuse, the Borno reference case).

## The model that makes cross-cycle reporting possible

A projectBeneficiary is created ONCE at registration and reused every cycle — so
`projectBeneficiaryClientReferenceId` (PBC) is the stable cross-cycle child key. Pivot
tasks by PBC into one row per child with a column group per cycle
(`Cycle-01 Date/Age/Status/Quantity`, `Cycle-02 ...`): a returning child has both groups
filled; a new Cycle-2 registration has Cycle-01 empty. `Date of Registration` = earliest
task date across all cycles; keep BOTH a static registration-time Age and per-cycle Age
(children age across cycles — the SPAQ2 cohort-aging effect).

**THE cardinal rule: the date window must span ALL cycles.** Every cohort/reuse
computation only sees what the task query fetched. If `get_custom_dates_of_reports()`
returns only the current cycle's window, Cycle-1 events never enter the frame — every
child looks newly registered, Cycle-01 columns stay empty, and "earliest date" silently
means "earliest this window". Scope by `campaignNumber`/projectId term + full-campaign
window (same lesson as recon-recovery's confirm-existence-all-dates).

## Chad generator (`children_cohort.xlsx`) — how it works

1. **Fetch**: scroll `ch-project-task-index-v1`, statuses
   ADMINISTRATION_SUCCESS / VISITED / NOT_ADMINISTERED, `campaignNumber` term filter,
   `syncedTimeStamp` range. Rows with no PBC are skipped but COUNTED + sampled (visible
   loss, never silent).
2. **Enrich** via the canonical chain task -> project-beneficiary -> individual
   (never additionalDetails names): PBC batches -> `beneficiaryClientReferenceId` ->
   individual name + encrypted UNIQUE_BENEFICIARY_ID. Index records by PBC in a dict
   FIRST — the O(N*M) rescan-per-individual was the real cause of a "kernel busy, no
   output" hang.
3. **Decrypt in bulk**: enc-service `_decrypt` accepts a LIST — 2,000 ids per call,
   3 retries with backoff, persistent failures stamped `DECRYPT_FAILED` (visible, not
   blank). Never one HTTP call per id (Sokoto draft did; won't finish at 1M+ scale).
4. **HH head**: household-member index filtered `isHeadOfHousehold: true` -> head
   individual id -> reuse already-fetched names, fetch only the rest.
5. **Status reshaping**: each distinct NOT_ADMINISTERED reason
   (`additionalDetails.taskStatus`) becomes a dynamic yes/no column; Administration
   Status column keeps only ADMINISTRATION_SUCCESS (VISITED / NOT_ADMINISTERED live in
   their own yes/no columns).
6. **Merge SUCCESS+VISITED** pairs on (PBC, Cycle Index) — key MUST include cycle or a
   C1 VISITED merges into a C2 SUCCESS.
7. **Pivot by PBC** (the reuse view), then export ONE SHEET PER PROVINCE
   (write_only openpyxl) — this generator is why Chad's workbook has 7 district sheets
   (see dst/null-datapoints-audit.md multi-sheet gotcha).

## Review findings (defects to check in any cohort script)

- **Window** (above) — the #1 way the reuse view silently collapses.
- **Per-event flags treated as static**: Ineligible/Referred taken from `group.iloc[0]`
  lose per-cycle info (child referred only in C2 shows "no"). Make them per-cycle or
  any-yes aggregate.
- **Redose rows dropped**: keep-first per (PBC, cycle) treats a second
  ADMINISTRATION_SUCCESS in the same cycle (a redose — Chad SPAQ has them) as a dupe.
  Aggregate quantity or emit a redose column.
- **GPS/productName fetched but never output**: the Chad generator pulls latitude /
  longitude / productName in `_source` and drops them at display-column selection —
  the ROOT CAUSE of Chad being unmeasurable for GPS/SPAQ in the null-datapoints audit.
  One-line fix: add them to display columns.
- **size-vs-hits fetch risk**: enrichment queries send 5,000 ids with `size: 5000`, no
  scroll — duplicate docs (individual updates, multiple head records) can push hits
  past size and silently lose mappings. Scroll or size >> batch.
- **Age-band double counting** (Sokoto draft): banding rows per event and intersecting
  across cycles counts a band-crosser (3-11m in C1 -> 12-59m in C2) in BOTH bands.
  Band each child once, at a fixed reference point (registration or latest cycle).
- **Missing projectId/campaignNumber filter** (Sokoto draft): so-/bo- indices can hold
  multiple campaigns — unfiltered match_all mixes them.
- **Timezone**: Nigeria = Africa/Lagos (UTC+1), NOT Africa/Harare; and never format
  local time then append "Z" — the window edges shift by the offset.
- Scroll perf: `"sort": ["_doc"]` beats sorting by syncedTimeStamp for full extraction.
- Display-column rename trap: records carried `_individualId` but display list named
  "Individual ID (if resolved)" — the col-not-in-df loop silently created it all-blank.
  Any `for col not in df.columns: df[col]=""` backfill can mask a missing rename.
