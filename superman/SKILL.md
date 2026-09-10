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

> Living skill. Last synced 2026-08-27. Point-in-time: file:line citations, tokens,
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
- `qa/correlation-tenantid-propagation.md` — correlation-id + tenantId propagation verification on
  mhbase against the platform report: the two-id gateway split, the idgen log gap, the mdms cache
  artifact, control-search discipline, and the mint-case method.
- `qa/mhbase-console-perf-testing.md` — mhbase env + Admin Console perf benchmarking: the
  householdType silent-drop root cause (202+cache then permanent loss; back-fill comes from Redis
  not DB), unified-template hard rules (immutable rows, mandatory targets, Warehouse-only, numeric
  phones, Boundary-Management-only ingestion), benchmark tier method + mapping semantics, device-log
  QA additions, deliverable locations.
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
- `dst/dss-dashboard-analytics-kpi.md` — the DSS dashboard end to end (HDDF-4931):
  dashboard-analytics backend on `analytics-raw-response-transform` (rawResponse
  charts, server-resolved colorRules), HCM-BAUCHI PROD config layout, the TWO
  chartType vocabularies (master = frontend render type, ChartApiConfig = backend
  handler), frontend health-dss rendering constraints, the fleet chart-config audit
  (terms-size 10-bucket truncation class, ignored date filters, 8-tenant
  campaign-scope divergence) with `validate_chartapi.py`, the live-verified
  data-point catalog per index (doseIndex==cycleIndex — 3-daily-doses KPI
  infeasible; supervision checklists rich; SMC band targets exist; pre-aggregated
  KPI indices `stock_reconciliation_summary` / `stock_allocation_distribution`),
  and the WHO/SMC-Alliance-anchored KPI recommendation set + DQ card/table design.
  2026-09-08 additions: local dev bring-up (health tree + central mode + ES-401
  blocker), ba PROD formula defects (uncapped %, docs-vs-children, open age bands,
  insights-always-0%), transformer field surface (locationAccuracy, waybillNumber,
  RDT checklist counts), funder expectations (Gates/GF/PMI/AMF/GiveWell), and KPI
  catalog v2 (#16-43: forensics, present-but-idle, projected completion, sync-lag
  honest form). Read BEFORE any DSS chart config change or dashboard KPI proposal.
- `dst/dss-dashboard-setup.md` — how to stand up a NEW DSS dashboard (new campaign
  type or cycle): the four config files incl. RoleDashboardMappingsConf /
  RoleDashboardConfig (role -> dashboard-ID lists, per-cycle ID suffixes),
  MasterDashboardConfig tab/row/vizArray rules (same name = same tab, unique row
  numbers per tab, file-wide unique viz IDs, >4 cards throws), requestQueryMap filter
  plumbing (projectTypeId path differs task vs household/stock indices), role-dependent
  date-range sourcing (national = project index, provincial/district = project-search
  DB), Kibana embedding via MDMS iframeRoutes, and the MDMS access setup (two action
  tests + role actions per link), PLUS chart calculation internals from KT sessions
  2-4: filterForCurrentDay/GMT today-metrics, agg-filter-only query parsing (top-level
  query.bool ignored), cycleIndex long-vs-keyword, numerator-first percentage rules,
  aggregation-path placement per chart type (above split for bars, nested for
  heatmaps), heatmap geojson/MDMS pipeline + literal `percentageValue`, xtable
  excludedColumns + ordered computedFields, pie selectedStack drill, prediction-chart
  mechanics, Kibana geo_point pre-mapping + public-share-URL embeds, lat/long `points`
  charts + literal `status` field, SMC/IRS progress-bar masters (endDate +1 day; IRS
  active-day epoch lists), and the remove-from-master-only chart-removal rule. The
  COMPLETE 26-section guide is bundled as `dst/dss-dashboard-setup-guide-full.md`
  (user copies at `D:\DST\dss-dashboard\DSS_Dashboard_Setup_Guide.md` + .docx). Read
  the setup file before creating, modifying, or wiring any dashboard chart; open the
  full bundled guide when executing the work.
- `dst/dashboard-recon-tool.md` — the `D:\DST\scripts\dashboard-recon` notebook utility that
  automates report-vs-dashboard reconciliation (campaign-type config smc/azm/itn + vas/ors_zinc
  addons, Summary-Report .xlsx export as primary dashboard source, Gemini-screenshot fallback
  with confirm gate, deterministic severity + cause hints, DQ audit with flagged-record Excel
  sheets, test-user detection). Read for any "report vs dashboard" recon run, and for the
  encoded facts: SPAQ 1 = 3-11m / SPAQ 2 = 12-59m band mapping, status rows have NULL
  productName, the pl Summary-export Ineligible-column discrepancy (202,702 vs 71k).
- `dst/dst-team-and-hcm-modules.md` — what DST (DIGIT Success Team) is + the full HCM module surface.
- `dst/partner-feedback-form.md` — the partner feedback form for the daily
  campaign report (English + French, built via the Forms API). Read for: which
  credentials can create a Google Form (API keys CANNOT; a service account
  would own the form and cannot hand it over; OAuth Desktop or Apps Script are
  the working routes), Forms API v1 limits, the francophone terminology map
  (district / canton / CPS / MILDA / DC — never 'LGA' in a French deliverable),
  and three PARTNER-FACING facts: data quality is never shared with partners,
  partners do not receive the short Slack summary, and the report's proper name
  is the Campaign Data Analysis Report.
- `dst/partner-french-deliverables.md` — how to ship the French version of a
  partner report + its two data workbooks, and the message that carries them.
  Read BEFORE translating anything for a francophone tenant (tg/ch/Burundi/
  Congo/Senegal) for: the terminology rule (use the pipeline's own vocabulary —
  DC not DBC, canton not villages — and check what prior cycles shipped), fr-FR
  localisation (U+202F thousands separator, `84,8 %`, `22/08/2026`, `06h40`,
  never touch numeric cells), the python-docx hyperlink trap (`p.text` includes
  hyperlink text but `p.runs` excludes it -> doubled strings), the four-item
  openpyxl pre-check before renaming sheet tabs, and the partner-message
  conventions (no figures, ships the FR feedback form, tag named readers not
  `@channel`).
- `dst/airflow-mdms-mode-and-test-matrix-2026-09-03.md` — MDMS mode fully live
  (flat schema created + 5-point pre-create check, sync 4 verbs, Kafka->persister->
  Azure-Postgres table closed), the cluster-mismatch credential saga (es-upgrade vs
  backbone, all AWS-vs-Azure), a 15-case live edge/validation matrix, the ONE fix
  (validate_row start<=end turns swapped-dates from silent no-op to visible reject),
  verified-deliberate behaviours (orphan-on-date-edit, mdms mode-change re-fire,
  tick batching), and the alert-copy restyle.
- `dst/airflow-e2e-bringup-2026-09-01.md` — the layer-by-layer production
  bring-up to the FIRST COMPLETE RUN (creds -> target -> ES DNS -> 401 -> done),
  the live failure taxonomy (FAILURE vs DEGRADED vs benign), slot mechanics
  (consumed run ids, both-mode, daily refire), Azure-vs-AWS unified-dev cluster
  facts (ES = elasticsearch-master.backbone), the MDMS schemaCodes-plural
  silently-ignored gotcha, the FLAT config entry shape + entry_row(), the
  3-way persister alignment method, and the cycle_index "01"/"1" join hazard.
- `dst/airflow-worker-vs-image-2026-09-01.md` — the gspread outage RCA, why
  "use an image for library access" is a category error (image = execution
  environment; only the AIRFLOW image can feed the worker), the settled no-pods
  design on branch `dst-self-contained-dags`, the e122dc05 half-migration
  incident, pod-boundary registry trap, git RM/clean/erase-commit traps, and
  dev-deployment config facts (ES_INDEX_PREFIX null==delete, missing secrets).
  Read WITH airflow-migration.md before any DST Airflow deployment work.
- `dst/airflow-audit-2026-08-24.md` — the three-lens adversarial audit of
  `D:\DST\automation` (data-engineering / senior-engineering / domain lenses, then
  re-verification in source). READ BEFORE changing that codebase: it lists what was
  fixed (observability and infrastructure only), the eight data-correctness defects
  that are still OPEN and why they were deliberately not fixed, the config-reachability
  defect class (documented-but-unreachable flags), the docs that describe subsystems
  which do not exist, and the list of things VERIFIED CLEAN that must not be "fixed".
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
- **Drugs/age bands:** SMC=SPAQ (**SPAQ1 = 3-11m, SPAQ2 = 12-59m**; treated 3-59m). AZM=Azithromycin
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

- 2026-09-10 (later) — `dst/null-datapoints-audit.md` updated for the v3 audit run.
  (1) Rule change: HH Head Name now blank-only like Child Name (typed "na" = filled),
  with DECRYPT_FAILED still missing + surfaced in the Note column; Chad Referred=yes
  exclusion added (turned out 0 rows in the export — Chad's 1,604 HH-head nulls are real
  blanks). (2) NEW "surviving OS low-memory kills" section: the harness kills BACKGROUND
  tasks under system memory pressure regardless of the task's own footprint (4 launches
  died at <1.5 GB free); fix stack = per-sheet reads (never sheet_name=None+concat) +
  per-source checkpoint/resume (re-emit summary lines for the watcher) + foreground
  time-budget chunking (`dump_analysis.py <seconds>`, killer only targets background).
- 2026-09-10 — NEW `dst/cross-cycle-cohort-reports.md` (cohort-script reviews, same-day
  session). Problem: "does this report capture reuse from Cycle 1 to Cycle 2?" — two
  scripts reviewed (Chad generator + Sokoto draft). Approach: traced how the pivot key
  survives cycles (projectBeneficiary created once, PBC stable) and what each query
  actually fetches. Key findings: the all-cycles date-window cardinal rule (windowed
  fetch silently collapses the reuse view — every child looks new), per-event flags
  wrongly treated static, redose rows dropped by keep-first dedupe, and the Chad GPS/SPAQ
  audit blanks ROOT-CAUSED to the generator fetching lat/long/productName but dropping
  them at display-column selection (cross-linked into dst/null-datapoints-audit.md).
  Plus the defect checklist: bulk decrypt (2K/call, DECRYPT_FAILED sentinel), size-vs-hits
  enrichment risk, age-band double counting, Africa/Lagos-not-Harare tz, O(1) PBC record
  index vs the O(N*M) hang.
- 2026-09-10 — NEW `dst/null-datapoints-audit.md` (Null-DQ audit sessions 2026-09-09/10).
  Problem: audit totals sat at exactly 1M rows, explained away as export-cap truncation;
  user challenged Sokoto. Approach: listed workbook sheet_names across the dump — 15 of 34
  files are multi-sheet. Root cause: exports spill overflow into Sheet2/Sheet3 (Chad = 7
  district sheets) and pandas default reads only Sheet1; v1 undercounted 14 state-cycles
  (Chad 118K -> 1.23M, Bauchi C2/C3 -> ~2.17M). Fix: sheet_name=None + concat; truncation
  = LAST sheet at cap only. Also captures the audit rules, Chad's divergent extract format
  (no GPS/productName; Referred/Ineligible column names -> referral exclusion didn't
  trigger, OPEN item), blank-vs-0 semantics on the audit sheet, and the live
  log->Google-Sheet watcher + xlsx-verified final-push pattern.
- 2026-09-10 — NEW `dst/dss-dashboard-setup.md` + `dst/dss-dashboard-analytics-kpi.md`
  extended (HCM-KEBBI CENTRAL-UAT session). Problem: a third config fork
  (HCM-KEBBI/CENTRAL-UAT, ITN-Dashboard-Redesign) carried new Excess/Loss stock and
  Beneficiaries-not-Administered KPIs needing the ba-PROD-style validation; misleading
  first read: the local clone pointed at egovernments/DEMO (fixed via added remote,
  not re-clone). Approach: validate_chartapi.py rerun on bednet/ke/si
  (findings_UAT_*.csv) + commit-level read of the new KPI definitions. Findings: new
  KPIs formula-sound but empty dateRefField + uncapped % + match_phrase "PVAR"
  matches all variants; a real default-10 terms truncation INSIDE
  beneficiariesNotAdministered's own refused-numerator; leftover chart with
  unparseable aggrQuery JSON; dangling drill; empty {} master chart entry; Kibana
  Stock Summary has no ndjson in the repo. Also transcribed FOUR dashboard KT
  sessions (Hanu) into `D:\DST\dss-dashboard\DSS_Dashboard_Setup_Guide.md` (+ .docx,
  26 sections) and encoded them as the new setup reference: setup/config (role
  mappings, MDMS action tests, tab/row/vizArray rules, requestQueryMap,
  role-dependent date ranges) + calculation internals (agg-filter-only parsing,
  chart-type-dependent aggregation-path placement, literal `percentageValue`/`status`
  newFields, xtable excludedColumns, pie selectedStack, prediction chart, Kibana
  geo_point pre-mapping, SMC/IRS progress-bar masters, remove-from-master-only rule).
- 2026-09-10 — NEW `qa/stock-supply-chain-flow.md` (curated from stock-flow test memory,
  2026-08-24 session). Problem: stock transfers silently offer no receiver when warehouse
  users are stacked at one level — looks like missing master data. Approach: reasoned from
  transfer resolution (receive leg needs a WAREHOUSE_MANAGER scoped to the receiving
  boundary). Rule: exactly one WM per boundary level (country->state->LGA->ward->HF) +
  DISTRIBUTOR at leaf; verified 6-user mhbase setup with boundary codes recorded (creds
  redacted, file pointer only). Also folds in stock-side reporting facts (2-slice stock
  scroll, warehousemanager activity sheet, Bauchi WM 1,073, D&E stock-report folders out of
  null-audit scope).
- 2026-09-08 — `dst/dss-dashboard-analytics-kpi.md` extended (HDDF-4931 session
  2026-09-07/08). Problem: run the DSS stack locally + widen the KPI set with research;
  misleading first reads: core-tree `yarn start` failure looked env-related (core tree
  build is simply broken — use the health tree), the "mz login screen" looked like a
  wrong .env (stale CRA start / global-config bake-in), and the ES 401 looked like
  permissions (it's a wrong user/password for that cluster — rejected on 9200 AND
  9201). Approach: live curl/endpoint probes (getDashboardConfig 200 in central mode,
  getChartV2 NPEs on the ES 401), formula-level audit of ba PROD ChartApiConfig
  (uncapped/silent PercentageComputedField, docs-vs-children value_count, open-ended
  age bands, insights-always-0% on empty dateRefField), transformer downstream-model
  code read for the full ES field surface (locationAccuracy, waybillNumber, RDT
  counts in ServiceIndexV2, cycleNumber on every entity), and two web-research sweeps
  (WHO/ACCESS-SMC/AMP literature + funder expectations: Gates/Global Fund PU-DR/PMI/
  AMF-105%-PDM/GiveWell). Output: KPI catalog v2 (#16-43 — field-forensics incl.
  impossible-visit-speed with the sibling/household-transition trap, present-but-idle,
  projected completion, mop-up conversion, pipeline self-report), the honest 3-part
  sync-lag design (offline app = same-day denominator unknowable), GPS-quality
  formula, and the user's Tier-1 triage (coverage/funnel/timeliness declared covered;
  build focus = operational tier). ES cred for the local port-forward still open.

- 2026-09-03 — New `dst/dss-dashboard-analytics-kpi.md` (HDDF-4931 DSS KPI work,
  2026-09-01..03). Problem: "validate existing KPIs and add meaningful ones" on the
  central-prod DSS dashboard — misleading first reads everywhere: the frontend
  appeared to lack an xtable renderer (actually two chartType vocabularies: master =
  render type, ChartApiConfig = handler); colorRules looked unconsumed by the frontend
  (actually resolved server-side into `card.color`); the validator's ACTION_ARITY
  errors were its own over-reach (computeMultiPaths/xtable semantics); and dev
  doseIndex 02 volume looked like day-2 dose capture (actually doseIndex==cycleIndex
  1:1, so the WHO 3-daily-doses adherence KPI is INFEASIBLE from routine data).
  Approach: three-layer verification — static fleet audit of all 14 tenant configs
  with a purpose-built validator (`D:\DST\scripts\dss-dq-kpis\validate_chartapi.py`),
  source reads of backend handlers + the health-dss frontend module, live kdev ES
  field-coverage probes, and a live pl dashboard dump; KPI set anchored to the SMC
  Alliance M&E Toolkit (WHO GMP/TDR). Findings: terms-aggs-without-size truncate
  LGA/ward tables to ES's 10-bucket default fleet-wide (visible in prod); 115-221
  queries/tenant ignore the date picker; campaign scoping fixed in ab/ba/pl/zf but
  never fleet-rolled to 8 other tenants; live pl shows 329% coverage (multi-cycle
  numerator vs single-cycle target), a half-dead DQ tab, and cross-tab FLW-count
  contradictions. Also discovered pre-aggregated KPI indices
  (stock_reconciliation_summary, stock_allocation_distribution) that make stock KPIs
  plain aggs. Deliverables: fleet findings CSVs, six validator-passed DQ KPI
  definitions (dq_kpis.json), WHO-anchored KPI shortlist + analytical (run-rate /
  forecast / age-heaping / sync-lag) additions; recommendation doc pending the
  user's own KPI list. Role note: DST recommends, another team implements.

- 2026-08-26 — Correlation-id/tenantId propagation verified on mhbase (new
  `qa/correlation-tenantid-propagation.md`) + unified-template and benchmark findings folded into
  `qa/mhbase-console-perf-testing.md`. Problem: the platform report claims PASS on every hop, but the
  first mhbase runs returned nothing for any correlation id and later runs showed mdms-v2 and idgen
  silent — read initially as total propagation failure. Approach: paired every id search with a
  control search on a known-existing record id (which exposed the first readings as a rolled-past
  15-min window), switched to the MINT case so only one id exists, and used the container dropdown
  with a CLEARED search box to separate "id not propagated" from "service ships no logs". Root
  causes: (1) the gateway mints its OWN correlationId and forwards it in RequestInfo while the
  client's x-correlation-id survives only in the first service's MDC — one request, two ids, seen
  7/7 runs; (2) missing mdms-v2 hops were cache hits, proven by a cold-cache run that logged
  perfectly and a warm-cache run of the same API that did not; (3) egov-idgen alone logs nothing on
  5/5 successful calls — still OPEN pending the cleared-search container query. Also corrected two
  of my own wrong readings: "mdms-v2 produces no logs" (it does) and treating the response header as
  the pass criterion (the report only tests it in the mint case). Template side: Facility USAGE
  col M is editable but Facility TYPE col J is immutable — "fixing" an invalid type triggers
  IMMUTABLE_CELL_TAMPERED and fails the whole sheet, strictly worse than the 2 row-level errors it
  replaced. New defect: facility create with no facilityType stores the `usage` value into
  facilityType, 202, and the record is then un-correctable via the console (householdType family).
  Benchmark claim-vs-reality captured: validation 113-180x inside target, 20K campaign +4 min over,
  boundary ingestion 97 min vs 5 min, and 1,102 users lost with failures clustering in batch 400-499
  at every tier.

- 2026-08-24 — Transformer error-handling ROUND 2 on mhbase folded into
  `qa/transformer-error-handling.md` + `context/gotchas.md`. Problem: the error index now holds 646
  docs, which read as "TC-09 fixed, error handling works" — misleading, because doc-count > 0 says
  nothing about whether the docs are usable or whether the silent-loss path was touched. Approach:
  new ES-ONLY verifier (`Downloads\transformer_error_handling_test.py` v4 — Kafka only injects; all
  verdicts from a pre-injection ES baseline vs a post-settle range query, plus a per-doc D1-D7
  grader and an always-green control), run twice for reproducibility. Root causes/findings: TC-09
  IS fixed; but **batch isolation is broken — `[valid, bad, valid]` loses 0/2 healthy siblings**
  (NEW, HIGH, same class as persister EDGE-1); doc `_id` is a UUIDv3 content hash so identical
  failures collapse to one doc and later runs OVERWRITE earlier evidence (proven on doc
  `401501dc`); `errorMessage` is null on 19% of docs (NPE class) and embeds the payload on the rest,
  making it unaggregatable; `payload/stackTrace.keyword` are `_ignored`; error docs carry no entity
  identity so they cannot be correlated or filtered per tenant; TC-08 ghost mechanism and TC-12
  tracer unchanged. **KB CORRECTION: TC-02 empty `{}` is NOT a silent no-op** — it produces an error
  doc (`[]` is the genuine no-op). Also recorded two of our own harness bugs (v3 silently skipped
  markerless cases; Kafka offset windows are unattributable on a shared global topic) and the ES
  query recipes/gotchas for this index (missing-vs-exists over-count, `time_zone` on `now/d`,
  `match_phrase` on dotted FQNs returns zero). Fix recommendation: emit exception class + root cause
  as their own field, keep the payload out of the message, and give the doc id an identity component
  — one change addresses D1, the cardinality problem and the overwrite together.


- 2026-08-17 — New `dst/dashboard-recon-tool.md`: the `D:\DST\scripts\dashboard-recon`
  notebook utility (built 2026-08-14 over one long session, uncommitted) that automates
  the July report-vs-dashboard recon methodology. Problem: L2s eyeballed agg summaries
  vs the DSS dashboard with no tolerance standard, no cause library, no record dumps.
  Approach: deterministic core (normalize -> severity -> cause hints -> DQ + record
  capture) with AI only at the screenshot-transcription edge (Gemini free tier, fallback
  model chain, mandatory confirm gate). Root causes fixed along the way, each verified
  live: browser FileUpload silently drops 150MB+ extracts at Jupyter's websocket cap
  (replaced with an input\ drop-folder + Refresh); notebook reload of only recon.ui left
  a stale engine (cell 2 now reloads all modules); SPAQ band ALIASES were mirror-swapped
  vs the (correct) product_bands — the live pl run exposed it as a +-300% mirror pair;
  a junk "Total: 47" tile silently overwrote a mapped total (conflicts now never
  overwrite); IntText widgets uneditable in the user's frontend (review is textarea
  `name = value` lines). Verified: pl run reproduced the July manual findings exactly
  (7 SPAQ1 swaps, over-age ~ swap_spaq2, dupes 4.4%); ba statuses match dash exactly.
  Discovered: the DSS dashboard exports `Summary Report.xlsx` (per-LGA, 17 cols) — now
  the primary dashboard source; its Ineligible column sums 202,702 vs extract/tile ~71k
  (open definitional question, ba-flow-saga family). User-driven design decisions:
  campaign-TYPE config (smc/azm/itn + vas/ors_zinc addons), notebook-only (no CLI),
  drop-folder inputs, manual fill-in form as default dashboard source, combined
  file+screenshot mode (file authoritative, screenshot fills gaps + cross-checks),
  test-user (test/uat/demo) record detection, DQ record sheets per check.
- 2026-08-14 — `dst/chad-smc-beneficiary-id-rca.md`: two new sections from the 08-11/12 sessions.
  (1) Remediation design for the 21,179 duplicates. Problem: which child keeps the ID and where
  do replacements come from. Approach/decision: ledger-consistency keeper rule (pool-consumer
  keeps; the rejected phone-generated leg is re-issued), phone-vs-phone exception (retire the
  number if also dispatched to a third CDD), re-issue from a distinct-format server-side
  remediation series (never the 87.8%-exhausted 7-digit space), child-1 record located via
  deterministic enc-service ciphertext search (pilot 3975695 first), updates via individual
  _update API only, and the gating PROGRAM question: physical cards vs digital-only (flag+alias
  vs full re-issue). (2) Reporting artifacts & packaging lessons: the two-document pattern
  (pre-read findings report + investigation-narrative RCA with real embedded evidence), the
  no-placeholders-in-evidence-blocks rule (every number block must sum to the claimed total —
  real daily breakdown 51,879...30 = 304,831), the three-number funnel (304,831 error docs ->
  306,929 distinct IDs -> 21,179 duplicates) to preempt "300k vs 21k" pushback, and the
  row-level Excel export recipe (openpyxl write_only, 300k+ rows; CSV-open-in-Excel =
  PermissionError gotcha). Also corrected on 08-11: task-index beneficiaryId field EXISTS but is
  ~1%-coverage sparse (3 of 21,179 pairs visible; ID 2911861 = first directly observed silent
  collision) — task index unusable for duplicate detection; gotchas updated accordingly.
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
- 2026-08-27 — New `dst/partner-feedback-form.md`: built and shipped a partner
  feedback form for the daily campaign report, in English and French (two separate
  live forms, 13 questions each, 11x5 usage grid). AUTH findings, all verified
  against live API responses: an API key CANNOT create a form ('API keys are not
  supported by this API' — a key identifies a project, not a principal); the
  service account is the wrong tool even with the API enabled, because it would
  OWN the form and ownership cannot be transferred to a person; OAuth Desktop
  client + InstalledAppFlow works, and Apps Script (`FormApp.create()`, runs AS
  the user) is simpler for one-offs. Note the SA and the OAuth client live in
  DIFFERENT GCP projects, so the 'enable the API' link in the SA error names the
  wrong project. `forms.body` carries no identity, so you cannot tell which
  account owns a form you just created. Forms API v1 cannot set the confirmation
  message or attach a response spreadsheet; a grid is a `questionGroupItem` with
  required set PER ROW; delete items in DESCENDING index order or you remove the
  wrong questions. PARTNER POLICY captured: data quality is NEVER shared with
  partners (a draft question offering it was wrong), and partners do not get the
  short Slack summary today. Francophone map recorded: district / canton / CPS /
  MILDA / DC / formation sanitaire — 'LGA' in a French deliverable means it was
  translated from the Nigeria template without review. REVISED same session to
  3 parts / 12 questions / 8-row grid: the report's real name is the **Campaign
  Data Analysis Report**; 'Programme Issues and Resolutions' is MISLEADING to
  partners (it actually surfaces data issues - low-performing facilities, low CDD
  sync); campaign list is SMC (not 'SMC / SPAQ' - SPAQ is the drug) + Polio +
  **Co-delivery** (more than one medicine or product); LGA+facility and the three
  CDD-sync rows each merged to one; exact-time question must ask the TIMEZONE;
  Part 4 removed (losing the only priority-ranking question) with Name and
  Organisation added as required. Rebuild-vs-patch rule: wholesale rebuild is
  safe only BEFORE a form is shared, since it replaces every item id and breaks
  existing responses' column mapping.
- 2026-08-27 — New `dst/partner-french-deliverables.md`: translated the full Togo
  CPS 2026 Cycle 3 cumulative Days 1-7 partner set into French (Word report + the
  performance and CDD-sync workbooks), and recorded the method. TERMINOLOGY: the
  first pass used DBC for CDD and Villages for Wards — both correct textbook
  French, both WRONG here, because the francophone map in
  `dst/partner-feedback-form.md` reflects what the pipeline emits (DC, canton) and
  partners read an unfamiliar synonym as a DIFFERENT metric. Load that map BEFORE
  translating, and check what prior cycles actually shipped. python-docx TRAP,
  root-caused: `paragraph.text` INCLUDES hyperlink text but `paragraph.runs`
  EXCLUDES it (hyperlink runs sit under `w:hyperlink`), so the usual "match
  `p.text`, write into `runs[0]`, blank the rest" idiom produced doubled strings
  ('CDD Sync Data ↗Données de synchronisation des DC ↗'); fix is to translate
  `p.runs` and each `p.hyperlinks[i].runs` as SEPARATE groups, with the plain part
  and the link text as separate dictionary keys. openpyxl: pre-check defined
  names, CF rules, hyperlinks and cross-sheet formulas before renaming tabs (all
  four empty for these two workbooks, so renaming was safe; freeze panes and
  autofilter refs are sheet-local and survive). Only STRING cells need fr-FR
  localisation — numeric cells carry a number format and render per the reader's
  locale. Dictionary-driven translation chosen over machine translation because it
  fails loudly: the untranslated-strings log should end up containing nothing but
  district and facility proper nouns. Two SOURCE defects carried through verbatim
  and reported rather than silently patched: report section 5.1b is
  column-misaligned (values shift one column right under an 8-column header), and
  an `ogou`/`Ogou` case collision makes the generator emit a sheet named `ogou1`.
  PARTNER MESSAGE conventions confirmed: French, ships the FR feedback form
  alongside the report, no figures and no DQ, and tag the named accountable
  readers — never `@channel` (widening the tag on a routine cumulative on an
  external channel reads as escalation; widen by NAME only).
- 2026-09-03 — New `dst/airflow-mdms-mode-and-test-matrix-2026-09-03.md`: MDMS
  mode proven end to end on Azure unified-dev; every credential/URL was an AWS
  leftover wrong for Azure (backbone ns, not es-upgrade); 15-case matrix run live;
  ONE genuine gap fixed (validate_row start<=end); target-fallback + mode-change
  re-fire + orphan-on-date-edit documented as by-design; alerts restyled.
  Uncommitted fixes under dags/ (git-sync, no rebuild). CONT (same doc): cumulative
  mop-up double-fired (two run ids = campaign_key churn under MDMS mode, no retime
  guard backstop) -> added slots._warn_on_duplicate_cumulative (observability only,
  data-hygiene is the real fix); internal@11/partner@12 routing verified correct
  end to end (no change); setup guide shipped as .docx not .md.
- 2026-09-04 — Same doc, CONT: alert restyle to ONE builder (alerts.build_alert_blocks)
  + severity colour bar via attachment (red COLOR_FAILED / amber COLOR_INCOMPLETE /
  slate COLOR_INFO) across failure+incomplete+config-sync (Slack has no css; blocks
  flatten on copy; live runs old committed code); live 20-case happy/sad/edge matrix on
  the dev tab (+5min stagger, timing CSV); full HCM-structured docs set
  (DST_Reports_Automation_Documentation.docx: 12-sec User Guide + 12-sec Tech Doc, deep
  persister-yml / dst_report_metadata DDL+queries / add-a-deployment-group sections);
  Google Docs Tabs are NOT API-creatable (write-into-existing only). All doc edits
  uncommitted; bars invisible live until committed.
- 2026-09-01 (later) — New `dst/airflow-e2e-bringup-2026-09-01.md`: first
  complete production run achieved on Azure unified-dev (DEGRADED-and-complete
  = pass); all three alert states demonstrated live; dst_config finalised
  (backbone ES, embedded service account, secrets); MDMS gateway paths + the
  plural-schemaCodes-ignored trap; mdms entry shape flattened (HCM-style) with
  schema artifact in platform/; persister/producer/table verified 15==15==15
  positionally. Gates: Padma table approval, MDMS schema create, kafka-python.
- 2026-09-01 — New `dst/airflow-worker-vs-image-2026-09-01.md`: production
  gspread outage traced to worker-runtime deps; two pod implementations built,
  live-proven and retired; final design = all tasks in-worker + libraries baked
  into the Airflow image (DevOps approved) + code moved under dags/ as
  `dst_data_analysis_report/` (survives subPath git-sync — the old sys.path
  bootstrap did not); e122dc05 half-migration killed the retime guard silently;
  best-practice audit 12.5/15. dst_config on dev STILL needs ES_INDEX_PREFIX ""
  (not null) + secrets + google_credentials_json.
- 2026-08-24 — New `dst/airflow-audit-2026-08-24.md`: three independent reviewers
  audited `D:\DST\automation` (branch `airflow`); 9 findings reached by 2+ lenses
  independently. Dominant theme SILENT SUCCESS — ten paths survived partial failure
  without telling anyone (zero records, zero targets, failed Slack post, swallowed
  per-day ES error, truncated join, lost audit row, discarded Kafka event, skipped
  partner post, unreachable Variable, overwritten degradation reason). FIXED
  (observability/infra only, no number moves): Kafka delivery now confirmed via
  `send().get()` — `flush()` raises only on timeout, so a rejected event returned True
  and SKIPPED the Run Log fallback; Slack post retry + `FAILED_POSTS`; accumulating
  `_degrade()`; zero-record/zero-target guards; guarded scroll cleanup; per-process
  0600 credentials dir; unreadable Variable fatal under Airflow only; config errors
  classified fail-fast; fleet-wide `max_consecutive_failed_dag_runs` kill switch
  removed; `dagrun_timeout` raised above the retry budget; `finalize_run(retries=0)`;
  and **sheet cells were being Jinja-rendered into `dag_run.conf`** — `{{ var.value.
  dst_config }}` in a cell would render the service-account private key into the UI.
  REVERTED on instruction (no core-logic/query changes) and still OPEN: terms-lookup
  truncation, the missing `task.individualId` name leg, `cycle_index` no-op unless
  `task_campaign_filter=TRUE`, two disagreeing `treated` definitions, DQ percentages
  over 100%, daily-report targets of 0, soft-deleted staff, `precision_threshold`
  drift. Also: `DST_DUP_MATRIX` env key (the CDD_ROLE defect class repeating), new
  `DEPLOYMENT.md`, and SKILL.md + `reporting.md` had the SPAQ1/SPAQ2 labels SWAPPED
  against the code and three recon memory files — corrected.
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

- 2026-08-19 — mhbase-hcm-dev round folded in. NEW `qa/mhbase-console-perf-testing.md` (env facts,
  householdType root cause with full Problem->Approach->Fix, template rules, console benchmark
  method, device-log QA). Appended: `qa/persister-regression.md` (mhbase 46-TC run 41P/2F; EDGE-2
  confirmed + scope pinned to task-level loss w/ batch isolation intact; household delete no-op now
  3 envs), `qa/validator-unbundle.md` (mhbase: unbundled AND orphans persist incl. ghost projectId -
  ghost-record manufacturing; always-on validators hold; head-rule artifact), `qa/transformer-
  error-handling.md` (standalone transformer_test_suite.py, marker-based, env-var creds),
  `context/gotchas.md` #22-27 (cache false positive, cache-sourced back-fill, template rules,
  Excel-open overwrite trap, identifiers[] extraction, single-session tokens).

- 2026-08-20 — Transformer error-handling LIVE RUN on mhbase folded into `qa/transformer-error-handling.md`:
  suite v2 (two-stage Kafka/ES verification + TC-14 control), env facts (error topic is GLOBAL
  unprefixed `transformer-error-topic`; tracer is `-enriched`; consumers need List<Entity> array
  payloads), and results — TC-08 ghost-record mechanism REPRODUCED ON DEMAND (valid task + ghost
  beneficiary -> indexed with null links, no error emitted), TC-09 437 error records in Kafka but no
  ES error index (indexer subscription gap; the July id fix IS present, 8/8 records have ids),
  TC-04 blank errorMessage, TC-12 no tracer forwarding. `context/gotchas.md` #28-31 (asymmetric topic
  prefixing, array payloads, always-inject-a-control, 404-vs-403 index semantics).
