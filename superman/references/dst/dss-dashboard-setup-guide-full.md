# DSS Dashboard Setup Guide

Setting up a new DSS dashboard for a campaign (SMC / IRS / Bednet-LLIN / ICCD) in DIGIT HCM.

Source: internal KT sessions 1-4 (Hanu, 2026-09) + verified against the health-campaign-config codebase (HCM-KEBBI, CENTRAL-UAT). Part 1 (sections 1-10): configuration and setup. Part 2 (sections 11-15): chart calculation internals. Part 3 (sections 16-21): pie/table charts, the prediction chart, Kibana maps. Part 4 (sections 22-26): lat/long maps, progress bar masters, and common maintenance requests.

---

## 1. Where the configuration lives

All dashboard configuration is stored in the **health-campaign-config** repository — the same repo that holds the indexer and persister configs. The repo is cloned per implementation country.

The dashboard configs sit under:

```
egov-dss-dashboards/<tenant>/dashboard-analytics/
```

Four files drive everything:

| File | Purpose |
|---|---|
| `RoleDashboardMappingsConf.json` | Which dashboard IDs each role (supervisor level) can access |
| `RoleDashboardConfig.json` | Dashboard link definitions per role (URLs used by the landing cards) |
| `MasterDashboardConfig.json` | Dashboard layout: tabs, rows, visualization types, chart references |
| `ChartApiConfig.json` | Per-chart ES queries, indices, aggregations, drill-downs |

The **dashboard-analytics** service reads these files from the repo at startup and caches them. Any change to any of these files requires a **restart of the dashboard-analytics service** on the cluster before it takes effect.

---

## 2. Role-to-dashboard mapping (`RoleDashboardMappingsConf.json`)

For each supervisor level (national, provincial, district), list the dashboard IDs that role should access in its `dashboards` list.

- One entry per campaign type: e.g. an SMC dashboard, an IRS dashboard, a bednet dashboard — each with its own dashboard ID.
- IDs are typically suffixed per campaign/cycle, e.g. `...-smc-nola-26` for the current SMC cycle, `...-irs-mis...` for IRS.
- The campaign cards on the DSS landing page redirect to these dashboard IDs, so every ID referenced here **must exist** in `MasterDashboardConfig.json`.

For a new campaign type or a new cycle: clone an existing mapping entry and point it at the new dashboard ID.

---

## 3. Dashboard layout (`MasterDashboardConfig.json`)

Each dashboard in the `dashboards` list has an `id` (matching the role mapping) and a `visualizations` list. Per campaign type you will typically maintain a separate set of national, provincial, and district dashboards.

To create a dashboard for a new campaign/cycle: **clone an existing dashboard block** of the same campaign type and change its `id`.

### 3.1 Tabs

- Each entry in `visualizations` has a `name` — a localization code that becomes the **tab header** (e.g. DSS Overview, Inventory, Maps, Complaints).
- Multiple entries with the **same** `name` are grouped into one tab; each entry is one **row** in that tab.
- Tab order on screen = order of first appearance in the list.
- To add a new tab: add entries with a new `name`.

### 3.2 Rows and vizArray

- Each row has a `row` number and a `vizArray` — the cards rendered on that row.
- **Row numbers must be unique within a tab.** A duplicate row number will either throw a UI error or silently hide the duplicate row.
- **`vizArray` IDs must be unique across the entire file.** Duplicate IDs cause UI rendering errors.
- Each vizArray item has:
  - `id` — unique numeric ID
  - `name` — localization code used as the **chart header** (upsert the localization for it)
  - `vizType` — which UI component renders it
  - `charts[]` — references (by `id`) into `ChartApiConfig.json`

### 3.3 vizType — how the UI picks a component

The DSS UI (`layout.js` in the DSS module of DIGIT-Frontend) switches on `vizType`:

| vizType | Component / use |
|---|---|
| `chart` | Generic component — bar charts, pie/donut charts, line charts |
| `metric-collection` / `stacked-collection` | Stacked metric cards (recent addition) |
| Banner card | KPI banner strip (recent addition; used e.g. for targets and sync metrics; Burundi variant shows 6 metrics in one card) |
| `heatmap` | Heat maps |
| `stacked-table` | National dashboard summary table with links to provincial dashboards |
| Lat/long type | Geo point rendering (recent addition) |
| `kibanaComponent` (chart type) | Embedded Kibana dashboard (see section 7) |

Two vocabularies exist — do not confuse them: the `vizType`/`chartType` in **MasterDashboardConfig** is the *frontend render type*; the `chartType` in **ChartApiConfig** is the *backend handler* (`metric`, `line`, `pie`, `xtable`, `rawResponse`, ...). A new chart needs both set correctly.

UI-side constraints (handled in the stacked/summary card components, not config):

- A card list response of length 4 splits into side-by-side bars; **more than 4 throws a UI error**. Changes beyond that need frontend work.

### 3.4 Adding a new chart to an existing dashboard

1. Add a new item to the target row's `vizArray` (or clone a row and bump the `row` number).
2. Give it a unique `id` and a meaningful `name` (localization code).
3. Create the corresponding entry in `ChartApiConfig.json` (copy an existing chart of the same type as a starting point; update the queries and `indexName`).
4. Upsert localizations for the new codes.
5. Restart dashboard-analytics.

### 3.5 National dashboard: links to provincial dashboards

The national dashboard uses the `stacked-table` vizType, which carries `reference` URL / logo / type fields per row. These URLs must point at the **matching campaign type's** provincial dashboards as defined in `RoleDashboardConfig.json`. A wrong URL here redirects users to another campaign's (or state's) provincial dashboard and breaks the flow.

---

## 4. Chart queries (`ChartApiConfig.json`)

Each chart ID referenced from the master config has an entry here defining:

- `chartName` — localization code for the **individual metric label** (the master-config row `name` is the header above the chart; `chartName` labels each metric inside it).
- `queries[]` — each with `indexName` (which ES index) and `aggrQuery` (the aggregation to run).
- `drillChart` — the next-level chart ID for drill-down (see section 5).
- `aggregationPaths` / `computedFields` / `action` — post-processing (percentage, division, custom computed logic).

### 4.1 Multi-index charts

One query hits one index. If a chart needs data from several indices, add **separate queries** — they cannot be combined in a single query. Same applies to two different aggregation logics over the same index: two separate queries.

Worked example — Inventory "stock available / used / required" per district:

- **Stock available** → `stock-index`
- **Stock used** → `project-task-index` (sum of `quantity`)
- **Overall target** → `project-index` (for SMC a custom index is maintained instead)
- **Stock required** = target − available → computed via `computedFields` after the three responses come back per district.

(Computed fields internals are covered in the follow-up session.)

---

## 5. Drill-downs

Drill-down is configured per chart in `ChartApiConfig.json`:

- The top-level chart aggregates at one boundary level (e.g. `terms` on district).
- `drillChart` names the next-level chart, whose `aggrQuery` aggregates at the next level (e.g. health facility).
- When a user clicks a bar (say, a district), the UI passes the clicked value to the next-level chart as a **filter key** (e.g. the district keyword). The next chart's query must accept that filter.

### 5.1 `requestQueryMap` — mapping UI filters to index fields

The UI sends filters in the request (`administrativeProvince`, `district`, `cycle`, `projectTypeId`, `province`, ...). `requestQueryMap` maps each incoming filter name (left side) to the ES field it should be applied on (right side). Matched filters are appended to the query's `must` list on top of the chart's hard-coded clauses.

Field paths differ per index — this is the most common mistake:

| Filter | project-task / project-staff index | household / household-member / stock index |
|---|---|---|
| `projectTypeId` | `Data.projectTypeId.keyword` | `Data.additionalDetails.projectTypeId.keyword` |
| `cycle` | `Data.additionalDetails.cycleIndex` (or `Data.taskDates`-adjacent fields depending on chart) | `Data.additionalDetails.cycleIndex` |

Always check where the field actually lives in the target index before writing the mapping.

---

## 6. Date-range selection (campaign start/end)

The provincial dashboard's date picker is bounded by campaign start/end dates, sourced differently per role:

- **National supervisor**: start/end dates are read **from the project index** (the provincial project documents' `startDate` / `endDate` fields), because national users cannot call project search for every province. The index used is configurable per campaign — IRS uses a custom date-target index; bednet/SMC use the project index.
- **Provincial / district supervisors**: the UI calls **project search (DB)** directly and uses the returned start/end dates.

If dates look wrong on a national dashboard, check the configured index and its start/end fields first.

---

## 7. Embedding Kibana dashboards

To render a Kibana dashboard inside DSS:

1. In `MasterDashboardConfig.json`, add a row whose chart type is `kibanaComponent`, with a `pageName` (e.g. a DSS Kibana screen key).
2. In **MDMS**, in the UI **common constants** master, add the `pageName` key to the `iframeRoutes` object:
   - `title` — localization code for the screen header
   - `routePath` — the Kibana **share/public URL with the domain stripped** (relative path only)
3. The UI loads the Kibana link in an iframe for that tab/row.

Note (UAT observation): the "Kibana Stock Summary Report" commit only added ChartApiConfig/MasterDashboardConfig entries — the backing Kibana saved-object (ndjson) lives outside this repo. Confirm the Kibana asset exists in the target environment.

---

## 8. Access setup (MDMS action tests + role actions)

For every newly added dashboard link, create **two action tests** in MDMS:

1. One for **dashboard-analytics** (the backend API path).
2. One for **digit-ui** (the frontend route).

Then create **role actions** granting each role access to the relevant action IDs. Example: the national dashboard link and the provincial dashboard link for a campaign must both be granted to the national supervisor; the provincial link also to the provincial supervisor.

Without both action tests and the role actions, the logged-in user will not see or be able to open the new dashboard.

---

## 9. Localization

- **Tab headers** and **chart headers** come from `MasterDashboardConfig.json` (`visualizations[].name` and `vizArray[].name`).
- **Per-metric labels** come from `ChartApiConfig.json` (`chartName`).
- The backend never resolves these codes — they must be **upserted as localization messages** (module `hcm-dss`, keys conventionally `DSS_HEALTH_*`) or the raw codes display on screen.

---

## 10. Deployment checklist for a new dashboard

1. Clone/adapt dashboard block in `MasterDashboardConfig.json` (new unique `id`, unique vizArray IDs, unique row numbers per tab).
2. Add/adapt chart entries in `ChartApiConfig.json` (correct `indexName`, tenant prefix, `drillChart`, `requestQueryMap`).
3. Add dashboard ID to the role's list in `RoleDashboardMappingsConf.json`; add/verify link in `RoleDashboardConfig.json`.
4. For national dashboards: point `stacked-table` reference URLs at the correct provincial dashboards.
5. Upsert localization messages for all new codes.
6. Create MDMS action tests (dashboard-analytics + digit-ui) and role actions.
7. For Kibana charts: add `iframeRoutes` entry in MDMS UI common constants.
8. Merge config, then **restart the dashboard-analytics pod** — configs are cached at startup.
9. Verify as each role: landing card, dashboard load, date range, drill-downs, localized headers.

---

# Part 2 — Chart calculation internals (KT session 2)

## 11. Simple metric cards

### 11.1 Today vs cumulative (national dashboard)

The national dashboard has **no date-range picker**, so "today" metrics are driven by two chart-config fields:

- `filterForCurrentDay: true` — the backend computes the current day's start/end **in GMT** (not the browser timezone) and applies it as a range filter.
- `dateRefField` — the field the day range is applied on.

Remove both for the cumulative variant of the same metric. Keep any other filters (e.g. `projectTypeId` via `requestQueryMap`) in both variants.

**Which date field:** originally `Data.clientAuditDetails.createdTime` was used, but with master-data reuse a re-registered record keeps its original client `createdTime` — only `lastModifiedTime` updates. Today-metrics therefore use `Data.clientAuditDetails.lastModifiedTime`.

### 11.2 Aggregation path wiring

The **root-level aggregation name** in the `aggrQuery` (e.g. `"Households Registered Today"`) must be listed verbatim in the chart's `aggregationPaths` — otherwise the computed value is never picked up for display.

### 11.3 Where filters must live in the query

The analytics service only reads filters from the aggregation's own `filter.bool.must` block:

```json
{"size":0,"aggs":{"<AggName>":{"filter":{"bool":{"must":[ ...filters... ]}},"aggs":{ ... }}}}
```

A top-level `query.bool.must` object is **silently ignored**. If you draft the query in Kibana with a `query` block, move the final `must` list into the aggregation filter before pasting it into the config.

### 11.4 requestQueryMap and index mapping types

Backend-appended filters (from `requestQueryMap`) must match the **index mapping type** of the target field:

- Text fields: map to `<field>.keyword`.
- `Data.additionalDetails.cycleIndex` on project-task is type **long** — map it **without** `.keyword`, or the query errors / silently fails to filter. (UI sends cycle as `"01"`.)

## 12. Percentage metrics (coverage)

- `action: "percentage"` on the chart; `aggregationPaths` order is significant: **numerator first, denominator second**. Set `valueType: "PERCENTAGE"`.
- Standard coverage: numerator = children administered from **project-task index** (`administrationStatus = ADMINISTRATION_SUCCESS`, `deliveredTo` INDIVIDUAL/HOUSEHOLD as appropriate, **dose 1 only** — dose 2/3 records sit in delivered state and are indirect; age-band filters like `age lte 11` for 3-11).
- Denominator = micro-plan target. For multi-cycle campaigns there are no predefined per-cycle start/end dates, so a **custom target index** is maintained (per cycle, per day) instead of the project index; the cycle/province filters arrive via `requestQueryMap`.
- Deprecated pattern: per-date-range targets computed as days-in-range × `targetPerDay` via an additive computed field — no longer used; coverage is now shown against the entire target.

## 13. Heat maps

### 13.1 GeoJSON pipeline

1. Country team provides **shapefiles** (.shp + companions). Nigeria's came from public sources; other countries from their country teams.
2. Convert to GeoJSON with scripts (geopandas), one file per drillable boundary.
3. Enrich each GeoJSON with six properties (raw GeoJSON is not used directly):
   - `center` — map auto-centering (computable via geopandas)
   - default `zoom` levels — found by manual testing
   - `hasCoordinatesDown` — `true` enables drill-down on click (requires a matching entry in the MDMS mapping, see below)
   - `hasCoordinatesUp` — legacy, no longer used
   - `level` — 1 = national, 2 = province, 3 = district; drives drill-up/down logic
   - `name` — the actual boundary name from ingestion
4. Upload each GeoJSON to MinIO (previously S3) and register it in **MDMS** as a key-value map: key = boundary name **lowercased, spaces replaced with underscores, no special characters** (hyphens have caused breakage); value = the MinIO/S3 URL.
5. Rendering uses `react-simple-maps`. A click on a boundary drills down only if `hasCoordinatesDown` is true **and** the boundary's key exists in the MDMS map.

### 13.2 Heat map coverage calculation

- Chart type is `xtable`. Both numerator and denominator queries do `terms` on the boundary field (province/district/village).
- **Always set `size`** on the terms agg — village/locality levels routinely exceed ES's default 10 buckets.
- Percentage per boundary is computed via `computedFields` with action **`PercentageComputedField`**: `fields[0]` = numerator aggregation path, `fields[1]` = denominator path. For heat maps the aggregation paths given are the **nested** paths (below the terms split).
- **`newField` must be exactly `percentageValue`** — the UI only renders the % symbol for that literal name (lat/long maps use plain-number fields instead).

## 14. Bar charts

- Backend chart type is `line`. The `terms` agg splits on the boundary level to show (district on a provincial dashboard); set `size`.
- Sub-aggregation per KPI type: `sum` of `Data.quantity` for doses given, `value_count` for children administered, `cardinality` for distinct users/synced.
- **Aggregation path difference vs heat maps:** bar charts take the path **above** the boundary split (the outer filter agg name); heat maps take the nested path below the split. Get this wrong and the chart renders empty.
- `sortValue: "descending" | "ascending"` orders the bars — the sort is applied in the analytics service after the ES response, since ES bucket order is not guaranteed.
- Drill-down: `filterKeys` on the chart names the filter passed to the `drillChart` when a bar is clicked. The key names must exactly match (a) the drill chart's `requestQueryMap` keys and (b) the index's `boundaryHierarchy` field names — a mismatch means the clicked district is silently not applied to the drill chart.

## 15. Multi-index and computed-field bar charts (inventory example)

- **Same-index branches — use nested aggregations, not extra queries:** stock available = received − dispatched, both from the stock index. One query with two filtered sub-aggregations (`eventType: RECEIVED` / `DISPATCHED`) plus a **`bucket_script`** whose `buckets_path` subtracts `Outgoing.sum` from `Incoming.sum`.
- **Cross-index arithmetic — computed fields:** "stock required for rest of campaign" = facility target (target/SMC custom index) − distributions done (project-task, sum of `Data.quantity` with `administrationStatus = ADMINISTRATION_SUCCESS`, dose 1).
  - There is **no subtract computed field** — the convention is to multiply the to-be-subtracted query's sum by −1 in an inline `script` and then use the **additive** computed field.
  - Computed fields **consume their inputs**: any aggregation path listed in a computed field's `fields` is removed from the final result set and replaced by the `newField`. So 3 aggregationPaths + 1 computed field over 2 of them = 2 bars in the final chart.

---

# Part 3 — Pie charts, table charts, prediction chart, Kibana maps (KT session 3)

## 16. Multiple tabs inside one chart card

Any vizArray entry's `charts[]` list can hold several charts, each with its own tab name (a localization code, e.g. "#" for numbers and "%" for percentage). The UI renders one tab per entry — this is how "number vs percentage" toggles on bar charts work. It applies to every chart type, and there is no limit on tab count.

## 17. Pie charts

- Master config chartType `donut`; ChartApiConfig chartType `pie`.
- Simple case: one query with a `terms` agg on the split field (e.g. PGR `applicationStatus`, or `serviceCode` for complaint types). If the slice categories are backed by different filter conditions rather than one field, write separate queries per slice.
- `size` on the terms agg: safe to omit only when the vocabulary is provably under 10 values (e.g. 6-7 service codes); otherwise set it explicitly.
- `aggregationPaths` lists the statuses/service codes to display.
- **Drill-down is pie-specific:** in addition to `drillChart`, the drill target must carry the `selectedStack` field — the UI passes the clicked slice via `selectedStack` (not the plain bar-chart filterKeys mechanism), because pie responses are handled differently in the frontend. Hover "drillable" affordance appears automatically when drill is enabled.

## 18. Table charts (xtable) — excludedColumns and ordered computed fields

- Master chartType `table`; ChartApiConfig chartType `xtable`.
- Tables typically pull from many indices (household, project-task, project/micro-plan, referral...), one query per source, then combine with `computedFields`.
- **`excludedColumns` is mandatory hygiene:** any aggregation path consumed by a computed field must be listed in `excludedColumns`, otherwise both the input columns AND the computed column render (duplicated metrics). Unlike bar charts, xtable does not auto-remove computed-field inputs.
- **Computed fields evaluate in list order, top to bottom.** If a computed field's input is itself computed (e.g. numerator = stock-index metric + project-index metric), that producing computed field must appear EARLIER in the list. Nothing is resolved out of order.
- Drill-down: the `filterKeys` must name the **first (boundary) column's label** at each level — that column header is what gets passed as the filter to the next level. SMC uses two drill levels; IRS drills to village, so each level's boundary column must be mapped.

## 19. Day tab (date-wise tables)

The "Day" tab variants of table charts use **no boundary terms at all** — every metric (households registered, children administered, refusals...) is a `date_histogram`. The histogram field is usually `Data.@timestamp` (device timestamp) but **varies per index** — some indices only have `Data.syncedTimeStamp`, attendance has neither; pick a field that actually exists in each queried index.

## 20. Actual-vs-planned progress chart (prediction line)

Three chart-config fields drive it: `predictionPath`, `distributionPath`, and a capped-target field.

- **Planned line (blue):** cumulative `targetPerDay` from campaign start to end. `targetPerDay` = `overallTarget / campaignDurationInDays` (computed only when duration > 0 — projects with `overallTarget` 0 plot nothing). Start/end come from the project-index `startDate`/`endDate` aggregations (with multiple districts starting at different times, prefer an average).
- **Actual line (yellow):** `date_histogram` on project-task `Data.createdTime` — the **client device time**. Records with wrong device dates silently fall outside the plot (fix = reindex/update, not config).
- **Prediction line:** appended from the current actual point forward at the planned slope (assumes planned daily coverage from today), answering "when do we finish at plan rate".
- Histogram keys are **GMT day-start milliseconds**; the UI converts them to dates — timezone mismatches show up as day-shift bugs.
- **Scope limits:** works cleanly for continuous campaigns (bednet). SMC needs start/end adjustments per cycle; IRS (with breaks/holidays) does NOT work in DSS — the workaround is custom facility-target indices that split targets across working days, charted in Kibana. There is no way to exclude Sundays/holidays from the planned line in DSS itself.

## 21. Kibana coordinate maps

- **The index mapping must declare `geo_point` BEFORE any data is indexed.** ES dynamic mapping infers lat/lon numbers as `float` (and never `geo_point`), and Kibana Maps then offers no geometry field for the data view. For a fresh index: `PUT` the mapping with `"geoPoint": {"type": "geo_point"}` first, then index.
- Existing data with a float mapping: reindex to a temp index, delete the original, recreate it with the `geo_point` mapping, reindex back.
- Heat maps (choropleths) are not practical in Kibana — only coordinate/point maps. Conversely, coordinate maps were tried in DSS and crashed the browser at real data volumes (too many points), so coordinate maps live in Kibana (which clusters points internally) and choropleth heat maps live in DSS. GPS accuracy problems (points at a common wrong spot, out-of-boundary) are app-side capture errors — not fixable at the map layer.
- Embedding recap (see also section 7): a `kibanaComponent` chart has **no ChartApiConfig entry** — only the master config row (chart type `kibanaComponent`, `pageName`, `moduleName` = which UI-common-constants module object, e.g. the IRS block) plus the MDMS `iframeRoutes` entry. The `routePath` must come from Kibana's **public share URL** (enable sharing, export the public URL, strip the domain, keep the `/kibana...` path) — the normal browser URL does not work.

---

# Part 4 — Lat/long maps, progress bar, maintenance requests (KT session 4)

## 22. Lat/long (coordinate + stock status) maps in DSS

Master config vizType is the lat/long map type; this is the Inventory map showing per-district stock metrics plus facility points.

- **Point sources:** (a) facility coordinates from the micro-plan, stored in the facility's address lat/long at facility creation; or (b) fallback when not provided — the **average** of latitudes/longitudes captured during stock transactions per district/AP (older records carry lat/long in `additionalDetails`, transformed from app `additionalFields`). Average can land outside the boundary when accuracy is poor — accepted limitation (a single random point would be worse; boundary polygons aren't available index-side).
- **Config contract for points:** queries whose results are map points must sit under chart type `points`, and the coordinate aggregation names are effectively hardcoded — `latitude` / `longitude` (avg aggs). Facility buckets follow the `f_<facilityName>` convention used app-side. Rendering = react-simple-maps markers.
- **Per-district metrics:** facility count = `cardinality` on `facilityId` under a district terms agg; stock received/dispatched = sums split by `eventType`; stock in hand = received − dispatched (nested agg + bucket_script, section 15). The metric's display code is generated from the aggregation label **uppercased with spaces → underscores** (e.g. "Number of Warehouses" → `NUMBER_OF_WAREHOUSES`) — upsert a localization for that generated code.
- **Stock status (Sufficient/Insufficient):** computed field whose `newField` must be named exactly **`status`**. UI logic: `status >= 0` → "Sufficient" (green boundary), else "Insufficient" (red). The labels are currently hardcoded strings in the UI — to localize them, wrap in the translation hook (`t(...)`) and upsert localization codes (an open Mozambique request).

## 23. Progress bar (campaign days elapsed)

The dashboard header progress bar is UI-side logic (`pages/index.js`), keyed off the **dashboard ID in the URL path** (e.g. path contains `smc-...-26` → SMC handling). Three behaviors:

- **Bednet (continuous):** plain difference between the logged-in user's project start/end dates vs today. Not-yet-started campaigns show a negative value.
- **SMC (cycles):** a progress-bar **master** keyed `projectTypeId → boundaryName → cycle → {startDate, endDate}` (epoch millis). Rules: startDate = actual campaign start in GMT; **endDate must be set one day AFTER the actual last day** (timezone slack between GMT/Africa/India). An unconfigured cycle silently falls back to the project's overall start/end — the "wrong days on cycle 2" symptom means the master lacks that cycle's dates.
- **IRS (breaks/holidays):** a separate master keyed `projectTypeId + boundary → list of active-day epochs` (country-timezone midnight millis), listing ONLY working days — Sundays, holidays, stock-break days excluded. Progress = (count of epochs earlier than the current request time) / (list length). Configure each district's own active-day list when districts run different periods.
- **Known limitation:** two same-named districts under one projectTypeId cannot be distinguished — the lookup is name-based; fixing it requires switching to boundary codes (code change).

## 24. Removing / reordering charts (the most frequent country request)

- Remove a chart **only from MasterDashboardConfig** — never delete from ChartApiConfig (the same chart ID is often referenced by other rows/dashboards, e.g. the sync card appears in Overview, Team Performance, and Data Quality).
- After removal, row numbers do NOT need renumbering — they only need to stay **in ascending order with no duplicates** (1,2,5,7 is fine; a duplicate row ID gets one row ignored).
- Reordering = change the row numbers to the desired order.

## 25. Campaign-specific localization renames

If a chart header code is shared across campaign dashboards (e.g. the same code in IRS and SMC masters) and only one campaign wants a new label: **rename the code in that campaign's master entry** (append a suffix, e.g. `..._SMC`), upsert the localization for the new code, then restart the analytics service. Editing the shared code's message would change every campaign that uses it.

## 26. Frontend components (where UI-level requests land)

- `CustomHorizontalBarChart` — all bar charts (bar spacing/width requests go here).
- Summary chart component — table-type charts.
- Custom pie chart component — donut/pie.
- Progress bar component + `pages/index.js` — progress-bar data logic (section 23).
- Translation: wrap hardcoded strings with the translation hook and upsert localization codes.

---

## Appendix: verified gotchas (from the HDDF-4931 config audits)

These recur across tenants and cause "numbers don't match" tickets:

- **Empty `dateRefField`** on a query means the dashboard date filter is silently ignored for that chart.
- **`terms` aggregations without `size`** truncate to ES's default 10 buckets (e.g. a 20-LGA state shows 10 rows).
- `action: percentage|division` needs exactly 2 `aggregationPaths` — unless the chart uses `computeMultiPaths: true` (line charts), which is a different handler.
- `drillChart` is mandatory for non-rawResponse chart types — use `"none"` for leaf charts; omitting it throws NPEs.
- Two queries against the same `indexName` need explicit `key`/`transformKey` or their results collide.
- Percentage computed fields are **uncapped** (can exceed 100%) and silently return 0.0 on any error or zero operand.
- Chart IDs/labels say District/AP/Locality/Village but Nigerian configs aggregate on `state/lga/ward/healthFacility/community` — trust the field, not the label.
- Boundary/index names are tenant-prefixed in config, not in code — the tenant prefix must be written into every `indexName`.
