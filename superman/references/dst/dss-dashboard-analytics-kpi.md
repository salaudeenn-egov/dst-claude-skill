# DSS Dashboard-Analytics — KPI validation and design (HDDF-4931, central prod)

How the DIGIT HCM DSS dashboard works end to end (backend, config, frontend), the
validated defect classes in the production chart configs, the verified data-point
catalog per ES index, and the WHO-anchored KPI recommendation set. Built 2026-09-01
to 2026-09-03. Read BEFORE touching any DSS chart config or proposing dashboard KPIs.

## The production stack

- Backend: `egovernments/health-campaign-services` branch `analytics-raw-response-transform`,
  `core-services/dashboard-analytics` (Spring Boot, `com.tarento.analytics`, port 8281).
- Config: `HCM-BAUCHI/health-campaign-config` branch `PROD` (a fork — NOT the egov DEMO
  branch), `egov-dss-dashboards/<tenant>/dashboard-analytics/`. Tenants: ab ba bo ch jg
  kd ke ko na oy oy-bk pl so zf + an unprefixed default (different chart-id generation,
  zero id overlap with tenant configs).
- Frontend: `egovernments/DIGIT-Frontend`, module
  `health/micro-ui/web/micro-ui-internals/packages/modules/health-dss`.
- Local clones under `D:\DST\repos\` (frontend is a sparse checkout of health-dss only).
- Working branch for this task: `HDDF-4931-dq-checks` in the bauchi-PROD clone.

## Architecture facts that gate everything

- Config is loaded from the filesystem at STARTUP only (`ConfigurationLoader`,
  `config.schema.paths` glob) — every config change needs a pod restart. Not MDMS.
- Central-instance mode: `is.central.instance=true`, per-tenant config folders, tenant
  resolved from request `headers.tenantId` via a ThreadLocal. tenantId mandatory.
- Index names come verbatim from `queries[].indexName` — tenant isolation is the `ba-`
  prefix in config, never a `Data.tenantId` filter.
- **Two chartType vocabularies.** MasterDashboardConfig `charts[].chartType` is the
  FRONTEND render type (`table`, `bar`, `donut`, `metric`, `sideBySideBar`, `heatmap`);
  ChartApiConfig `chartType` is the BACKEND handler (`xtable`, `xtableNested`, `line`,
  `pie`, `metric`, `rawResponse`). A "missing xtable case in the frontend" is a
  misreading — xtable renders via master type `table`. Set both correctly for new charts.
- Branch feature `rawResponse`: config-driven transforms over raw ES docs/aggs
  (expression language: ternary, template strings, add/divide/coalesce/formatDate),
  `merges[]` cross-index dataset joins (left/inner/full/concat, chained),
  `cardsList` KPI banner cards. **colorRules are resolved SERVER-side into a flat
  `card.color`** (`RawResponseHandler.java` ~125-127) which the frontend reads
  (`UserActivityMetrics.js` ~102) — RAG threshold cards are config-only, no FE change.
- Response cache (`cache.expiry.time.in.minutes`) — expect staleness while validating.

## Frontend rendering constraints (health-dss module, verified in source)

- vizType dispatch in `components/Layout.js`; unknown vizType silently renders null.
- cards-list: `labelKey` must be a fully-qualified localization code (no prefix added);
  optional `labelKey_DESCRIPTION`; `valuePath` dot-notation only; grid sized for max
  5 cards per row; valueType PERCENTAGE renders toFixed(2)+%.
- CustomTable (all xtable charts): auto-sorts by first column (backend order lost);
  NO footer support (a totals row becomes a sortable data row); fires a paired
  LAST-YEAR request per chart unless `hideInsights: true` (set it on campaign-scoped
  tables); an EMPTY result auto-drills down a level; any filter/date change resets the
  drill stack; drill needs `filter[].column` == plot name with dots replaced by spaces.
- BannerCard: renders `charts[0]` only, drops plots whose `label` is non-null, omits
  moduleLevel from the request, sub-heading hardcodes today's date.
- New localization keys go in the `hcm-dss` module; missing keys render verbatim.
- Icon map is a closed list — unmapped visualizer names show no icon.

## Chart-config gotchas (adding a chart)

- `drillChart` mandatory (`"none"` for leaves) — omitting NPEs in the backend.
- `action: percentage|division` needs exactly 2 aggregationPaths ONLY for metric
  handler; line charts with `computeMultiPaths: true` and xtable use different paths —
  do not "fix" them (validator over-flagged these once).
- aggregationPaths must match agg names in aggrQuery character-for-character
  (findValues lookup).
- `queries[].module` must be `COMMON` or match request moduleLevel, else silently skipped.
- computedFields literal `repsonseToDifferenceOfDates` (typo) is the accepted value.
- Coverage denominators: sum `Data.overallTarget`/`targetPerDay` from project-index with
  `must_not exists <next-lower boundary>` to avoid double-counting target rows.
- Two queries on one index need explicit `key`/`transformKey` (else `_1` collision).

## Validating existing configs — the audit (Problem -> Approach -> Findings)

Problem: asked to "validate existing KPIs"; 347 charts in ba alone, eyeballing
impossible; live pl dashboard showed impossible numbers (329% coverage) that config
reading alone would not have prioritized.
Approach: wrote a static validator (`D:\DST\scripts\dss-dq-kpis\validate_chartapi.py`)
encoding the known failure modes, ran it fleet-wide (per-tenant CSVs in the same dir),
then cross-checked flagged classes against handler source and a live dashboard dump.
Findings (fleet):
- **TERMS_DEFAULT_SIZE — the headline class**: 266-379 live queries per tenant have a
  terms agg with no `size` -> ES caps at 10 buckets -> LGA/ward tables silently
  truncated (visible in prod: DQ WM-entries chart shows ~4 of 20 LGAs). Mechanical fix.
- NO_DATE_FILTER: 115-221 live queries per tenant with empty dateRefField on
  time-series indices — those charts ignore the date picker.
- Campaign-scope DIVERGENCE: jg/kd/ke/ko/na/oy/oy-bk/so have 15-18 live task/stock
  queries with no campaignNumber filter (cross-campaign contamination) while
  ab/ba/pl/zf are clean — a fix was applied to four tenants and never fleet-rolled
  (same divergence class as the automation repo's DUP_MATRIX/CDD_ROLE).
- Hard errors: ch (1 bad requestQueryMap JSON, 1 master ref to missing chart, 6
  aggPath mismatches), bo (1 bad aggrQuery JSON), DEMO default
  (distributionCoverageDistrictSMC both aggPaths unmatched — can never render).
- Dead weight: 13-169 unreachable charts per tenant (not referenced, not in any
  drill chain).
Live-dashboard confirmed defects (pl, 2026-09-03): 329%/345% coverage (multi-cycle
numerator vs single-cycle target — the flagship number is wrong); DQ tab half-dead
(Suspected Frauds and Summary Report render empty); "Beneficiaries not Administered"
donut computed total != displayed total (210,500 vs 211,634); Inventory summary table
bucketed at country ("Nigeria" single row) on a provincial dashboard; absurd insight
deltas ("20300% than yesterday"); User Activity tab contradicts Team Performance
(4,800 vs 3,980 FLWs; "Records Today 10,279,851" > whole campaign); complaints
open-summary columns all zero with non-zero totals; referral completion near zero.

## Data-point catalog (verified live on kdev ES, 2026-09-03; creds session-only)

Entity relations (from recon-recovery chain.py — the canonical join map):
Household.crid = member.householdClientReferenceId; member.individualClientReferenceId
= Individual.crid; Household.crid = PB.beneficiaryClientReferenceId (SMC: beneficiary
= household); PB.crid = task.projectBeneficiaryClientReferenceId. DSS cannot join
across indices except via task-doc denormalized fields, rawResponse merges, or
pre-aggregated indices.

- project-task: administrationStatus/quantity/createdBy/pbCrid/gender/lat/geoPoint
  ~100%; age only on SMC-type campaigns; **doseIndex == cycleIndex 1:1 (cycle 01 ->
  all dose 01) — day-2/3 doses are NOT captured; a 3-daily-doses adherence KPI is
  INFEASIBLE from routine data** (matches the WHO toolkit's own caveat).
- project-index targets: targetType INDIVIDUAL / HOUSEHOLD / PRODUCT /
  INDIVIDUAL_SMC_3_TO_11 / INDIVIDUAL_SMC_12_TO_59 -> households-visited % and
  age-band coverage KPIs are feasible.
- service-task: checklistName rich (ELIGIBILITY, HOUSEHOLD, HF_RF_FEVER/SICK/
  DRUG_SE_CC, TEAM_SUPERVISION, TRAINING_SUPERVISION, IEC, MOBILIZER_FORM,
  PAYMENT_DATA) + supervisorLevel -> supervision-coverage KPI fully supported.
- stock: eventType RECEIVED/DISPATCHED; reason RETURNED/DAMAGED_/LOST_IN_{STORAGE,
  TRANSIT}; blister-return fields (partialBlistersReturned/wastedBlistersReturned)
  exist in prod ba config but NOT in dev data.
- hf-referral symptoms: fever/rash/vomiting/weakness/DRUG_SE_CC; beneficiaryId ~100%.
- **Pre-aggregated KPI indices exist** (custom transformer outputs, chart-ready):
  `stock_reconciliation_summary` (per product|from|to: dispatched/received,
  final_balance, is_reconciled, reconciliation_pct, mismatch_type),
  `stock_allocation_distribution` (per lga/ward: population_covered,
  allocated/distributed stock, distribution_efficiency, stock_per_person),
  `household-coverage-daily-iccd-v2` (date/province/cycle/households_visited);
  prod's `ba-ineligible-summary` is the same pattern. Stock KPIs = plain aggs over
  these; verify which generators run in central prod.

## The KPI recommendation set (WHO-anchored)

Canonical framework: SMC Alliance M&E Toolkit Performance Framework (WHO GMP/TDR,
smc-alliance.org). Its two headline quality indicators — % target children treated in
ALL cycles, and % taking day-2/3 doses properly — are both absent from the dashboard
(the second is infeasible, see above; say so rather than faking it).

Shortlist (feasible + WHO-core + absent/broken today): per-cycle coverage with
distinct beneficiaries (fixes the 329%), full-round adherence (Kibana cohort proves
the data; DSS route = precompute or elevated embed), boundaries-not-reported
(denominator integrity, Chad-118.5% class), courses per CDD team per day (protocol
band), supervision coverage, blister/stock reconciliation (pre-agg index), referral
completion, + the 6 DQ KPIs (over-age, age-zero, missing gender, missing GPS, redose
share, duplicate dose-1 ID-based — definitions validator-passed in
`D:\DST\scripts\dss-dq-kpis\dq_kpis.json`; over-age directly implements WHO's warning
that out-of-range treatments inflate admin coverage).

Analytical/predictive additions (arithmetic, not ML — label them honestly):
projected completion date + LGAs-at-risk (moving_fn rate -> date), required run-rate
ratio (remaining/days-left vs current rate), stockout forecast (balance / trailing
consumption), age-heaping index (digit preference on Data.age — field-forensics for
band reliability), sync-lag percentiles (createdTime -> syncedTime; how live is the
dashboard), daily anomaly z-scores per LGA, CDD risk watchlist (composite), full-round
adherence forecast, end-of-day nowcast (intraday sync curve).

DQ dashboard design (agreed shape): RAG banner cards (rawResponse cardsList +
colorRules) + xtable drill chain District -> Ward -> Health Facility with
hideInsights true; thresholds are program-team sign-off placeholders; partner
visibility of DQ needs explicit program approval (standing policy: DQ never shared
with partners in reports).

## 2026-09-07/08 session — local bring-up, ba formula audit, funder/literature research, KPI catalog v2

### Local dev bring-up (Problem -> Approach -> State)
Problem: run the full stack locally against the ba config. Misleading first reads:
`yarn start` failed in the CORE tree (missing `services/elements/UserPreferences` —
core tree build is broken, health tree is the right one); "mz login screen" was a
stale global-config, not a wrong .env value.
Approach/state: backend `application.properties` -> central mode
(`is.central.instance=true`, tenants=ba, `base.path=file:///D:/DST/repos/Dashboard/
health-campaign-config/egov-dss-dashboards`); ES port-forward is **HTTP** not HTTPS.
`getDashboardConfig` 200 (ba configs load: 347 charts / 3 dashboards); `getChartV2`
400 NPE because **port-forwarded ES rejects the dst cred (401 "unable to authenticate
user" on 9200 AND 9201 = wrong user/password for that cluster, not permissions)** —
the standing blocker. Frontend: health tree lacks cross-env on `start` -> set
`$env:SKIP_PREFLIGHT_CHECK` then `yarn build` + `yarn start:dev`; local backend
routing = dedicated `/dashboard-analytics -> http://localhost:8281` proxy registered
FIRST in `example/src/setupProxy.js`; ba prod global config (verified live) =
`https://central-instance-assets.s3.af-south-1.amazonaws.com/globalConfigDashboard-ba.js`;
restart CRA after any .env edit (REACT_APP_GLOBAL is baked into index.html at start).

### ba PROD formula audit — defects beyond the earlier fleet audit
- Coverage tiles = `value_count` on PB CRID -> DOCS not children; PercentageComputedField
  is UNCAPPED (>100% possible) and silently emits 0.0 on any exception/zero operand
  (misconfig looks like "no activity"). Fix direction: cardinality for "population
  covered", keep value_count only where "administrations" is meant.
- Age-band tiles are OPEN-ENDED: "3-11" = `age lte 11` (includes under-3), "12-59" =
  `gte 12` (includes 60+). Labels claim bounds the filters don't enforce.
- Insight configs on charts whose queries have empty dateRefField ALWAYS read 0%
  (insight pass re-runs the same chartNode with shifted dates; no date field = same
  cumulative number twice). Insights belong only on date-windowed charts. (No separate
  i-config is needed — TarentoServiceImpl reuses the chartNode with "i"+code.)
- `totalHouseholdVisited` counts household-index docs = REGISTERED not visited.
- Unknown visualizationCode / unloaded tenant config = NPE (ClientServiceFactory +
  TarentoServiceImpl deref without null check).

### Transformer downstream field surface (from models, 2026-09-08 — code-read, ES 401)
All app `additionalFields` are copied VERBATIM into `additionalDetails`, and
`cycleNumber` is stamped on EVERY entity (tasks, stock, side-effects, referrals,
checklists, households) -> everything is cycle-scopable. Unused-but-present fields:
task `deliveryComments`, `locationAccuracy` (meters; **unused by any chart; verify
populated in ba prod before building on it**), `dateOfBirth`, `individualId`,
`lastModifiedBy`; stock `waybillNumber` (chain-of-custody), `eventTimeStamp` vs
`dateOfEntry` (backdating detector), `facilityTarget`, `facilityLevel`/
`transactingFacilityLevel`; ServiceIndexV2 `childrenPresentedUS/APE` +
`malariaPositiveUS/APE`/`malariaNegativeUS/APE` (RDT positivity — verify ba checklists
populate these; naming is Mozambique-origin), `ageGroup`; side-effect `symptoms`;
attendance `registerServiceCode/Name/Number`, `signatureData`; project
`campaignDurationInDays`. GPS on tasks: lat/lon/locationAccuracy from task.address;
`geoPoint=[lon,lat]`, null if either coord null. **ba VAS rows carry NO GPS — scope
GPS KPIs to flow=smcDone.**

### Funder expectations (researched 2026-09-08, sources in session/memory)
Gates = data-to-action + subnational tailoring (they fund this platform's lineage);
Global Fund = PU/DR indicators + LFA on-site source verification (numbers must be
re-derivable from record level — our advantage vs paper); PMI = all-4-cycles +
cycle-timeliness (<=28d between cycles) + survey triangulation; AMF (ITN gold
standard) = household-level register w/ member names + supervisor 5% re-registration
("105%") + nets shipped-vs-distributed line-by-line + unannounced PDM ~9/18/27 months
(NOT "8% every 6mo"); GiveWell rewards DISCLOSED limitations and explained
admin-vs-survey deltas (ACCESS-SMC: 86% >=1 cycle but ~55% all four; Zoerhoff: admin
within +/-10pp of survey only ~47%). Partner-view ranking: cycle-completion funnel,
denominator-provenance + >100% auto-flag, commodity reconciliation strip w/ wastage,
ALMA-style RAG scorecard, DQ badge. Three uses of "children reached" (label tiles!):
min across cycles = safe claim, avg = World Malaria Report, max = procurement.

### KPI catalog v2 (adds #16-43 to the WHO-anchored set; full formulas in session)
Field-forensics: age-heaping/Whipple per CDD; **impossible-visit-speed** — gaps
between consecutive records of same CDD but ONLY on householdId CHANGE (siblings =
legit seconds-apart rows; raw-gap version false-flags large families); median gap +
%<60s (flag fast_share>=30% AND >=10 transitions); slow tail = terrain/microplanning
map; needs scripted_metric or (ship-grade) a per-CDD-day pre-agg index. Night-work
hour histogram; same-geohash pile-up; out-of-assigned-boundary work.
Money/effort: present-but-idle (attendance x zero tasks x bills — payment integrity);
payment turnaround (bill statusTransitionTimes); projected completion date
(remaining/run-rate); CDD churn between cycles; attendance integrity (both sessions +
signature %).
Reach/equity: new-vs-repeat reach per cycle; beneficiary gender ratio; HH-visited vs
children-treated divergence; geo-grid coverage vs registration cells.
Care quality: absent->treated (mop-up) conversion; redose clustering per CDD;
registration-to-treatment latency; ineligible-rate z-score per CDD (Taraba class);
caregiver pre-notification reach (scheduled-notification index ~ WHO "knew date in
advance", top adherence predictor per Ibinaiye 2023).
Surveillance/system: fever-referral anomaly per ward; facility referral load vs
staffing; pipeline self-report (transformed-errors by errorCategory ON the dashboard);
cost-per-child proxy (bills / distinct treated — label approximate).
Sync-lag honest form (denominator is unknowable same-day — offline app): trio of
(1) YESTERDAY's lag report card (exact: % synced<24h + median lag via percentiles-50
on `(syncedTimeStamp-createdTime)/3600000`, clamp negatives), (2) today's estimated
completeness from the historical intraday arrival curve, (3) silent-CDDs-today
(staff register as denominator — the never-synced pattern). GPS quality = geoPoint
exists AND locationAccuracy<=50 over smcDone deliveries, bucket_script %; verify
locationAccuracy populated first, else v1 = geoPoint-presence only.
User triage 2026-09-08: Tier-1 items (unique-children coverage, cycle funnel,
timeliness) declared already covered by existing charts; NOT-REPORTED band declined.
Build focus = operational tier (zero-report, CDD activity+sync, workload band,
mop-up), then reconciliation.

## 2026-09-10 — HCM-KEBBI CENTRAL-UAT audit (ITN-Dashboard-Redesign stream)

Problem: a second config fork appeared — `HCM-KEBBI/health-campaign-config` branch
`CENTRAL-UAT` — carrying the ITN dashboard redesign (author Surya Chetty) with new
stock/coverage KPIs; needed the same validation the ba PROD config got. Misleading
first read: the local clone at `D:\DST\repos\health-campaign-config` looked like the
right repo but pointed at egovernments/DEMO — fixed by adding remote `hcm-kebbi` and
checking out `CENTRAL-UAT` (origin/DEMO left intact; three config forks now coexist:
egov DEMO, HCM-BAUCHI PROD, HCM-KEBBI CENTRAL-UAT).
Approach: repo survey + reran `validate_chartapi.py` per tenant
(`findings_UAT_{bednet,ke,si}.csv` in dss-dq-kpis) + read the new KPI chart
definitions and the adding commits (e72a64da Excess/Loss, 9e1a4f39
Beneficiaries-not-Administered, 11754930 Kibana Stock Summary).

Repo shape: 5 tenants (abia, ba, bednet, ke, si), ~1.3-1.4K charts each; bednet+si
carry the new BEDNET national dashboard; also per-tenant egov-indexer (24 YMLs) +
egov-persister trees and the bednet payment-voucher pdf-service configs.

New-KPI verdicts:
- Excess/Loss stock KPIs: stock-index filter
  `Data.additionalDetails.stockEntryType.keyword = EXCESS|LOSS`, sum
  `Data.physicalCount`; percentage vs task-index quantity sum. Formula sound BUT all
  four have empty dateRefField (date picker ignored), the % is the uncapped/silent
  PercentageComputedField, and the denominator uses `match_phrase
  Data.productVariant:"PVAR"` which matches EVERY product variant — fine single-product,
  wrong the day a second product lands.
- beneficiariesNotAdministered*Percent: `computeMultiPaths: true` line charts (the
  8-path ACTION_ARITY flags are the known validator over-reach), BUT the first query
  ("Beneficiary Refused", terms on `lga`) has NO size while every sibling uses
  size 500 -> default-10 truncation inside one chart's own numerator. Also mixes
  pre-agg summary indices (refused/ineligible/registered from
  population-coverage-summary-datewise + ineligible-summary) with live indices
  (died/migrated/absent from project-task, SE/referrals) — different freshness per term.
Hard errors (bednet + si): `totalHouseholdsNotDeliveredDrilldownProvinceBednetsICCD`
has UNPARSEABLE aggrQuery JSON (leftover from the Households-Not-Delivered removal —
NPEs if reached); dangling drill `householdsITNsRedeemedHeatMapProvinceBednetsICCD`;
an empty `{}` chart entry in level-two `DSS_HEALTH_ATTENDANCE_SUMMARY` (viz 10005).
Bulk stats match the PROD fleet pattern (bednet: 1,477 TERMS_DEFAULT_SIZE / 818
NO_DATE_FILTER / 147 dead charts).

Setup how-to (role mappings, MDMS action tests, tabs/rows, Kibana iframeRoutes, date
ranges per role) now lives in `dst/dss-dashboard-setup.md` + the written guide at
`D:\DST\dss-dashboard\DSS_Dashboard_Setup_Guide.md`.

## Tools built this session

- `D:\DST\scripts\dss-dq-kpis\validate_chartapi.py` — static chart-config auditor
  (run: `python validate_chartapi.py <config-dir> --csv out.csv`); per-tenant findings
  CSVs alongside. Known over-flag: ACTION_ARITY on computeMultiPaths line charts and
  xtable division.
- `D:\DST\scripts\dss-dq-kpis\dq_kpis.json` — six DQ KPI definitions in the
  business-intelligence skill's template (passed its metric_validator --strict).
- Local skills installed from `D:\DST\repos\Claude-Skills`: `business-intelligence`
  (KPI template/validators; its SKILL.md now cross-references superman) and
  `data-analyst`.
