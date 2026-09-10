# DSS Dashboard Setup — creating a new dashboard for a campaign

Operational how-to for standing up a new DSS dashboard (new campaign type or new
cycle) in DIGIT HCM, plus chart calculation internals. Source: internal KT sessions
1-4 (Hanu, 2026-09) transcribed and cross-checked against the HCM-KEBBI CENTRAL-UAT
config repo. Full written guide (all 26 sections, complete instructions and guidelines) is
BUNDLED in this skill as `dss-dashboard-setup-guide-full.md` (same folder); the
user-facing copies live at `D:\DST\dss-dashboard\DSS_Dashboard_Setup_Guide.md`
(+ .docx). This file holds the condensed load-bearing facts — read the bundled
full guide when actually executing a dashboard setup or chart change;
`dss-dashboard-analytics-kpi.md` holds the KPI/audit side.

## The four config files (per tenant, egov-dss-dashboards/<tenant>/dashboard-analytics/)

1. `RoleDashboardMappingsConf.json` — role (supervisor level) -> list of dashboard
   IDs the role can access. One dashboard ID per campaign type/cycle (suffixed, e.g.
   `...-smc-nola-26`, IRS variants). Landing-page campaign cards redirect to these
   IDs, so every ID listed must exist in MasterDashboardConfig.
2. `RoleDashboardConfig.json` — the dashboard link definitions per role (URLs).
3. `MasterDashboardConfig.json` — layout (tabs/rows/vizArray) + chart references.
4. `ChartApiConfig.json` — per-chart queries (see the KPI file for gotchas).

New campaign/cycle = clone an existing dashboard block of the same campaign type,
change its `id`, wire it into the role mapping. Per campaign type there are separate
national / provincial / district dashboards.

## MasterDashboardConfig layout rules

- `visualizations[].name` = TAB header (localization code). Entries sharing the same
  `name` are grouped into one tab; each entry = one ROW. Tab order = order of first
  appearance.
- Row numbers must be unique within a tab: a duplicate row number either throws a UI
  error or the duplicate row silently doesn't render.
- `vizArray[].id` must be unique across the ENTIRE file — duplicates break UI rendering.
- vizType dispatch (frontend Layout.js): `chart` (generic bar/pie/line),
  `stacked-collection` / metric collections, banner card (KPI strip; Burundi variant
  shows 6 metrics/card), `heatmap`, `stacked-table`, lat/long, `kibanaComponent`.
  UI constraint: a card-list of length 4 splits into side-by-side bars; **>4 throws a
  UI error** — handled only in frontend code, not config.
- National dashboards use `stacked-table` rows carrying reference URL/logo/type per
  province — these URLs must match the SAME campaign type's provincial dashboards in
  RoleDashboardConfig, else users get redirected to another campaign's dashboard.

## Drill-down filter plumbing (requestQueryMap)

- `drillChart` in ChartApiConfig names the next-level chart; the clicked bucket value
  is passed down as a filter key.
- `requestQueryMap` = {UI filter name -> ES field to apply it on}; matched filters are
  appended to the query's `must` list on top of the hard-coded clauses.
- Field paths differ per index (the classic mistake): `projectTypeId` is
  `Data.projectTypeId.keyword` on project-task / project-staff but
  `Data.additionalDetails.projectTypeId` on household / household-member / stock.
  Cycle filter -> `Data.additionalDetails.cycleIndex`.

## Date-range selection (campaign start/end) is ROLE-dependent

- National supervisor: start/end read from the **project index** (provincial project
  docs' startDate/endDate) because national can't project-search every province. The
  source index is configurable — IRS uses a custom date-target index; SMC/bednet use
  the project index.
- Provincial/district supervisors: UI calls **project search (DB)** directly.
- Wrong date bounds on a national dashboard -> check the configured index first.

## Kibana embedding

- Master config row with chart type `kibanaComponent` + a `pageName`.
- MDMS "UI common constants" master, `iframeRoutes` object: `pageName -> {title
  (localization code), routePath (Kibana share URL with the domain STRIPPED)}`.
- Gotcha (UAT, 2026-09-10): the "Kibana Stock Summary Report" commit added only
  ChartApiConfig/Master entries — no ndjson lives in this repo; the Kibana
  saved-object must exist in the target environment separately or the tab is empty.

## Access setup (MDMS) — without this the user never sees the dashboard

For EVERY new dashboard link create TWO action tests: one for **dashboard-analytics**
(backend API path) and one for **digit-ui** (frontend route). Then create role
actions per role (e.g. national supervisor needs both the national AND provincial
links of the campaign; provincial supervisor the provincial link).

## Localization split

- Tab headers + chart headers = MasterDashboardConfig `name` codes.
- Per-metric labels = ChartApiConfig `chartName`.
- Backend never resolves any of these — upsert in module `hcm-dss` or raw codes render.

## Deploy

dashboard-analytics caches all configs at startup — restart the pod after ANY config
merge. Then verify as each role: landing card, dashboard load, date range,
drill-downs, localized headers.

## Chart calculation internals (KT sessions 2-3)

- National dashboard has NO date picker: today-metrics use `filterForCurrentDay: true`
  + `dateRefField` (day range computed in GMT, not browser tz). Remove both for the
  cumulative variant. Date field is `Data.clientAuditDetails.lastModifiedTime` — NOT
  clientCreatedTime, because master-data reuse keeps original client createdTime.
- Filters ONLY count inside the aggregation's `filter.bool.must`; a top-level
  `query.bool` block is SILENTLY IGNORED by the analytics service (Kibana-drafted
  queries must have their must-list moved into the agg filter).
- requestQueryMap must respect index mapping TYPES: `additionalDetails.cycleIndex`
  is long on project-task — mapping it with `.keyword` breaks/errors the filter.
- Percentage metrics: `action: percentage`, aggregationPaths order = numerator FIRST,
  denominator second, `valueType: PERCENTAGE`. Coverage numerator = dose 1 only
  (dose 2/3 sit in delivered state); denominator = micro-plan target (custom per-cycle
  target index for multi-cycle — no predefined cycle start/end in project index).
- Aggregation-path placement DIFFERS by chart type: bar charts (backend `line`) take
  the path ABOVE the boundary terms split; heat maps (`xtable`) take the NESTED path
  below the split. Wrong placement = empty chart.
- Heat maps: shapefiles -> geojson (geopandas) enriched with center / zoom /
  hasCoordinatesDown / level (1=national 2=province 3=district) / name; registered in
  MDMS as {lowercased_underscored_name -> MinIO/S3 URL}, NO special chars/hyphens in
  keys. Drill works only if hasCoordinatesDown=true AND the key exists in the MDMS
  map. PercentageComputedField newField MUST be literally `percentageValue` or the UI
  won't render the % symbol. Renderer: react-simple-maps.
- Bar charts: `sortValue: descending|ascending` (sorted in service post-ES);
  drill `filterKeys` must exactly match both the drill chart's requestQueryMap keys
  and the index boundaryHierarchy names, else the clicked boundary is silently dropped.
- No subtract computed field exists: multiply the subtrahend query's sum by -1 via
  inline script, then use the ADDITIVE computed field. Same-index branch metrics
  (received vs dispatched) = one query with filtered sub-aggs + bucket_script, not
  two queries. In bar charts computed fields CONSUME their input aggregationPaths
  (removed from final set, replaced by newField).
- Multi-tab cards: several charts in one vizArray `charts[]`, each with its own tab
  name (e.g. "#" vs "%") — works for every chart type, no tab limit.
- Pie (master `donut` / backend `pie`): terms on status/serviceCode; drill needs the
  pie-specific `selectedStack` field on the drill target (NOT plain filterKeys).
- xtable: computed-field inputs MUST be listed in `excludedColumns` or they render as
  duplicate columns (xtable does not auto-consume inputs like bar charts do);
  computedFields evaluate strictly in list order — dependent fields must come earlier.
  Drill filterKeys per level = the boundary COLUMN LABEL of that level.
- Day tabs: no boundary terms at all — all metrics are date_histogram on the device
  timestamp; field varies per index (`Data.@timestamp` vs `Data.syncedTimeStamp`;
  attendance has neither — pick per index).
- Prediction chart (`predictionPath`/`distributionPath`/capped target): planned line =
  overallTarget/campaignDurationInDays cumulated (duration must be >0); actual =
  date_histogram on Data.createdTime (client DEVICE time — wrong device dates fall off
  the chart); keys are GMT day-start millis. Continuous campaigns (bednet) only; IRS
  breaks/holidays are NOT supported in DSS (workaround = holiday-split facility-target
  indices + Kibana).
- Kibana coordinate maps: geo_point mapping MUST be PUT before indexing (dynamic
  mapping infers float; Kibana Maps then offers no field). Existing data: reindex out,
  recreate mapping, reindex back. Choropleths stay in DSS, point maps in Kibana (DSS
  point rendering crashed the browser at volume; Kibana clusters internally). Kibana
  embed routePath must be the PUBLIC share URL (domain stripped) — the normal URL
  fails; kibanaComponent charts have NO ChartApiConfig entry (master row + MDMS only;
  moduleName picks the UI-common-constants module object).

## Lat/long maps, progress bar, maintenance playbook (KT session 4)

- Lat/long (Inventory) maps: point queries live under chart type `points` with
  hardcoded agg names `latitude`/`longitude` (avg aggs); facility points from
  micro-plan facility address, fallback = AVG of stock-transaction coordinates per
  boundary (can fall outside the boundary — accepted). Metric display codes are
  auto-generated (label UPPERCASED, spaces->underscores, e.g. NUMBER_OF_WAREHOUSES) —
  upsert localizations for the generated codes. Stock-status computed newField must
  be literally `status`: UI shows status>=0 as Sufficient/green else
  Insufficient/red; labels hardcoded in UI (wrap in t() + localize = open Mozambique
  task).
- Progress bar is UI logic keyed off the dashboard ID in the URL path: bednet =
  project start/end diff; SMC = master {projectTypeId -> boundaryName -> cycle ->
  start/end epochs} where endDate must be ONE DAY AFTER the real last day (tz slack)
  and an unconfigured cycle silently falls back to project dates (the "cycle 2 shows
  wrong days" symptom); IRS = master of ACTIVE-day epoch lists (country-tz midnights,
  Sundays/holidays/breaks excluded), progress = epochs-before-now / list length.
  Same-named districts under one projectTypeId are indistinguishable (name-based
  lookup; needs boundary-code change).
- Chart removal: remove from MasterDashboardConfig ONLY — never delete from
  ChartApiConfig (IDs are shared across rows/dashboards, e.g. the sync card).
  Row numbers need ascending order + no duplicates, NOT contiguity (1,2,5,7 fine;
  duplicate row = one silently ignored).
- Campaign-specific label rename when a code is shared across campaign masters:
  rename the code in that campaign's master (suffix e.g. _SMC) + upsert new
  localization + restart analytics — never edit the shared message.
- UI components for frequent requests: CustomHorizontalBarChart (bar spacing),
  summary chart (tables), custom pie chart, progress bar logic in pages/index.js.
