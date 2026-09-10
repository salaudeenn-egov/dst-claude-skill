# dashboard-recon — report-vs-dashboard reconciliation utility (notebook)

`D:\DST\scripts\dashboard-recon` (built 2026-08-14, not yet git-committed).
Automates the report-vs-dashboard reconciliation methodology proven manually on
ko/na/ab/pl/bo/ba/ke (see reporting.md "Dashboard-vs-Report reconciliation").
Entry point is `dashboard_recon.ipynb` ONLY — the user explicitly rejected a
CLI (run_recon.py was built and then deleted). `check_gemini.py` remains as a
key/model diagnostic.

## Problem it solves
L2s eyeball agg-script summaries against the DSS dashboard with no standard
for "close enough", no cause library, and no record dumps. Every discrepancy
became an L3 escalation or an unexplained note. The tool encodes the July-2026
recon methodology as deterministic code: normalize -> match -> severity ->
cause hint -> DQ audit -> flagged-record Excel.

## Architecture (AI at the edges only)
- Deterministic Python core: `recon/inputs.py` (loaders + extract counter),
  `recon/compare.py` (severity + cause hints), `recon/output.py` (workbook +
  template recon note), `recon/config.py` (layered config), `recon/ui.py`
  (ipywidgets app, modeled on recon-recovery's RecoveryApp).
- AI is ONLY the optional screenshot transcription (Gemini free tier,
  `google-genai`, GEMINI_API_KEY in the tool's `.env` — free key from
  aistudio.google.com; chosen over OpenRouter (free-roster churn) and Kimi
  (not free)). Model fallback chain in `config/defaults.yaml`:
  gemini-flash-latest -> 3.7-flash -> 3.5-flash -> flash-lite-latest (the
  gemini-2.5-* names are DEAD, 404). 503s fall through the chain live.
  Transcription NEVER enters the comparison without the human confirm gate.

## Config model — CAMPAIGN TYPE, not tenant (user insisted: "all have the
## same dashboard")
Layers: `config/defaults.yaml` -> `config/campaigns/{smc,azm,itn}.yaml` ->
`config/addons/{vas,ors_zinc}.yaml`. KPI lists merge BY KEY, so an addon can
flip fields on a base KPI (vas sets ineligible/referred `flow_scoped: true`)
or append new KPIs. `load_config(campaign_type, addons)`. azm/itn profiles
are UNVERIFIED starting points (no AZM/ITN dashboard recon has ever been
done; itn has no extract mapping — summary input only).

## smc.yaml is the crown jewel — encoded facts (each cost real debugging)
- Band mapping CONFIRMED UNIFORM ko/na/ab/pl/ba: **SPAQ 1 = 3-11m (SMALL
  count), SPAQ 2 = 12-59m (BIG count)**. The tool's first live pl run caught
  a mirror-swap because the aliases were backwards while product_bands were
  right — report SPAQ2 841,514 matched dash "SPAQ1" 841,140. If SPAQ1 vs
  SPAQ2 ever shows a +-300% mirror pair, suspect alias direction first.
- Extract = 25/27-col CHILDREN_TREATED; product col is `productName`
  (camelCase); ba's 27-col variant has Red VAS / Blue VAS as their OWN yes/no
  flag columns (NOT productName values). Column matching is case/space/
  punctuation-insensitive (`_norm_col`).
- Status rows (INELIGIBLE etc.) have NULL productName — the status breakdown
  MUST be counted on all rows BEFORE the product-match gate. count_statuses
  maps by case-insensitive CONTAINS (INELIGIBLE catches
  BENEFICIARY_INELIGIBLE). not_administered = INELIGIBLE+ABSENT+MIGRATED+
  DIED+REFUSED (ab formula; ba matched dash exactly on the four statuses).
- VAS age bands are DQ-only (counting stays by flag — dashboard counts by
  product): Red = 12-59m, Blue = 6-11m, empirically verified on ba and the
  REVERSE of the WHO colour convention; out-of-band = typed-age errors,
  DOB-verify before alarming.
- `ignore_labels` silently skips dashboard tiles with no report counterpart
  (Population targeted, Complaints, per-band redose tiles).

## Dashboard-side inputs (two modes in the UI)
1. Manual form (default): every KPI rendered as a fill-in field the moment
   the mode is selected; 0 = "not on my dashboard" (skipped). No confirm
   gate (user typed them). "Download CSV template" is a single always-live
   data-URI link (a widget Button cannot start a browser download — the link
   IS the button), regenerated on campaign/addon change.
2. Files COMBINED: Summary-Report .xlsx export AND/OR screenshot(s).
   - **The DSS dashboard exports `Summary Report.xlsx`** — 17 cols, one row
     per LGA: Population Administrated, 3-11/12-59 Administered+Target+
     Coverage, Beneficiary refused/died/migrated, Ineligible, Absent,
     Household refused. `dashboard_summary:` mapping in smc.yaml (candidate
     spellings include the export's real typo "Adminstered"). This is the
     PRIMARY source — dashboard's own numbers, zero transcription risk.
   - Screenshots (multi-select, one per dashboard page) are transcribed
     per-image and merged; the FILE is authoritative — screenshot fills gap
     KPIs (Redose tile, cumulative covered) and cross-checks overlaps
     (agree = green box, differ = red conflict, file kept).
   - Review screen is TEXTAREA-based (`name = value` lines) — IntText boxes
     were uneditable in the user's Jupyter frontend. Junk sign: many LGA
     rows with the SAME value (e.g. all 105) = model misread the table;
     delete the lines. A green banner states what the file already covers so
     users don't think the file's LGAs were lost (that confusion happened).

## Report side
- Multi-part extracts are the NORM (Excel 1,048,576-row cap). Paths/glob/
  folder programmatically; UI = drop `input\` folder + Refresh + multi-select
  (browser FileUpload was removed — Jupyter's websocket cap silently drops
  150MB+ uploads; the user hit this live: "already uploaded" but empty).
- Parts counted in ONE pass with shared counters so `distinct_by` dedup holds
  ACROSS part boundaries. Per-part row counts logged. Empty glob/folder
  aborts; mixed xlsx+summary inputs rejected; schema-guard refuses if a
  configured required column is missing in ANY part (old 20-col artifact
  protection); missing FLAG columns = KPI 0 + orange note (non-VAS state
  extracts are normal).
- Progress: throttled (0.3s) IntProgress — reading stage per part (calamine,
  ~30-60s per 165MB part), counting stage every 200k rows. Throttle matters:
  unthrottled updates flood Jupyter IOPub ("message rate exceeded").
- Notebook cell 2 must `importlib.reload` ALL recon modules in dependency
  order — reloading only ui leaves a stale engine (bit the user: VAS flags
  silently not counted on the first real ba run).

## Comparison engine + cause hints (compare.py)
Severity: PASS |delta|<=10 or <=0.05%; MINOR <=0.5%; else MISMATCH; MISSING
when one side lacks the KPI. Hints are the codified cause library, each
naming its proving ES query: date-window clip (all coverage deltas small
negative), docs-vs-distinct (dash uniformly 1-5% higher), flow scoping
(flow_scoped KPI mismatch), live timing (small residual), report-higher
(basis check / stale dash read), else UNEXPLAINED -> escalate. Label
conflicts never overwrite: first value wins + red warning (a junk "Total: 47"
tile once silently clobbered the real 2,165,141 before this rule).

## DQ audit + flagged-record sheets (same row pass, zero extra runtime)
Counts AND captures records: age 0/under/over eligible band (SMC 3-59),
age_missing, child/HH-head EMPTY vs JUNK ('null'/'nan'/'none' typed —
separate counters, junk = CDD typed it, not data loss), duplicates (Record
Type == duplicate), gender missing + split, gps_missing (VAS rows carry no
GPS by design — judge % on SPAQ rows), product_bands swaps (ENABLED for SMC),
VAS band_ checks, test_user_rows (username contains test/uat/demo/dummy/
training — Chad UAT-bleed lesson) with per-account breakdown. Every check
gets a "DQ <key>" workbook sheet with Source File + full original columns,
freeze panes, auto-filter; capped at dq.max_records_per_check (5000) with
LOUD truncation warnings. Extras: full Administration Status breakdown
(verbatim values, no string guessing), gender split, distinct children +
docs_per_child ratio (>1.02 auto-explains docs-vs-distinct), SPAQ+VAS
overlap counter — ZERO overlap on a dual-product campaign = the ke Script-2
flag-reset fingerprint, warned automatically.

## Live validation results
- Mock screenshot end-to-end: transcription digit-exact; 503 fallback chain
  exercised live.
- ba 3-part extract (395MB, 2.35M rows, 139s with calamine): mechanically
  clean; exposed the stale-reload and label-mapping issues now fixed.
- pl 2-part run REPRODUCED the July manual recon: exactly 7 SPAQ1 swaps
  (same number found manually), over-age 30,395 ~ swap_spaq2 30,421 (the
  known pl over-age pattern), duplicates 4.4%, docs/child 1.025. After the
  alias fix the pl Summary export reconciles: total +760 (+0.07%), spaq2
  +374, spaq1 +386, redose exact 0, died/absent/migrated/refused EXACT.
- OPEN FINDING (pl): the Summary export's per-LGA Ineligible column sums to
  202,702 vs extract INELIGIBLE 71,013 vs dashboard tile ~71k — the export
  column uses a different definition (same family as the ba ineligible/flow
  saga). Investigate before publishing pl ineligible from the export.

## Gotchas quick list
- gemini-flash-latest 503s are routine; the chain handles them. AFC warning
  from google-genai is harmless noise.
- Gemini free tier trains on inputs — screenshots are aggregate counts (no
  PII) but confirm acceptability per data policy; typed/manual/export paths
  send nothing anywhere.
- Excel row cap forces part-splitting everywhere; never count parts
  separately when a distinct/dedup basis is in play.
- The dashboard's own bands may not sum to its own headline (ba off by 16);
  compare band-to-band or recipient-to-recipient, never cross.
- Tool is uncommitted — same risk class as the 2026-07-28 recon-recovery
  wipe. Commit it.
