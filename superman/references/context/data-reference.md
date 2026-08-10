# DIGIT HCM — Data Reference

Concrete data facts: ES indexes, DB tables, linkage, API patterns, credentials,
tenants, SQL, tracer patterns. Point-in-time — tokens rotate, verify before use.

## ES index structure (prefix varies by tenant)

| Index | Contains | Key fields |
|---|---|---|
| `{tenant}-household-index-v1` | Household registrations | `Data.household.clientReferenceId`, `Data.household.memberCount`, `Data.additionalDetails`, `Data.geoPoint` |
| `{tenant}-household-member-index-v1` | HH member links | `Data.householdMember.clientReferenceId`, `.householdClientReferenceId`, `.individualClientReferenceId`, `.isHeadOfHousehold` |
| `{tenant}-individual-index-v1` | Individuals | `Data.individual.name.givenName/familyName/otherNames` |
| `{tenant}-project-task-index-v1` | Deliveries | `Data.householdId`, `Data.projectBeneficiaryClientReferenceId`, `Data.administrationStatus`, `Data.quantity`, `Data.taskDates` |
| `{tenant}-project-beneficiary-index-v1` | Enrollments | links household to project |
| `{tenant}-user-sync-index-v1` | CDD sync | `Data.syncedUserId`, `Data.syncedUserName`, `Data.taskDates`, `Data.boundaryHierarchy.*`, `Data.role` |

Two document formats coexist:

| Entities | ID field | Date field | Format |
|---|---|---|---|
| Household, HouseholdMember, ProjectTask | `Data.{entity}.clientReferenceId` | `Data.@timestamp` (ISO) | sync-wrapped under `Data.{entity}.*` |
| Individual, ProjectBeneficiary | `clientReferenceId` (flat) | `auditDetails.createdTime` / `clientAuditDetails.createdTime` (epoch ms) | flat doc, no Data wrapper |

Project-task ES doc id = `taskClientReferenceId + projectBeneficiaryClientReferenceId + tenantId` (concatenated, no separators).

Per-entity date filter fields (confirmed):

| Entity | Field | Format |
|---|---|---|
| Household | `Data.household.auditDetails.createdTime` | epoch ms |
| HH Member | `Data.householdMember.auditDetails.createdTime` | epoch ms |
| Individual | `@timestamp` | ISO string |
| ProjectBeneficiary | `@timestamp` | ISO string |
| ProjectTask | `Data.taskDates` | `YYYY-MM-DD` string |

Project-task index gotchas (confirmed ba-project-task-index-v1, 2026-07-01): `Data.taskDates`
range works; `Data.auditDetails.createdTime` and `Data.isDeleted` DO NOT EXIST on task docs
(return 0); `Data.createdTime` (epoch ms) and `Data.@timestamp` (ISO) do exist. Always verify
a new field with a sample query before filtering on it.

## DB tables (PostgreSQL, schema = tenant)

| Table | Key columns |
|---|---|
| `{t}.household` | `id`, `clientreferenceid`, `numberofmembers`, `additionaldetails` (JSON, head name), `tenantid` |
| `{t}.household_member` | `id`, `clientreferenceid`, `householdid`, `individualid`, `isheadofhousehold` |
| `{t}.individual` | `id`, `clientreferenceid`, `name` (JSONB) |
| `{t}.project_beneficiary` | `id`, `clientreferenceid`, `beneficiaryclientreferenceid` (= household clientReferenceId), `beneficiaryid` (often NULL by design) |
| `{t}.project_task` | `id`, `clientreferenceid`, `projectbeneficiaryclientreferenceid` — **NO householdid column** |

Household head name lives in `household.additionaldetails` JSON:
`{"fields": [{"key": "newField1_<uuid>", "value": "Head Name"}]}` (key contains `newField1`).

### Household -> ProjectTask linkage (critical)

```
project_task.projectBeneficiaryClientReferenceId
  -> project_beneficiary.clientreferenceid
  -> project_beneficiary.beneficiaryclientreferenceid
  = household.clientreferenceid
```
`project_task` has NO householdid column. `Data.householdId` in the ES project-task index is
a denormalized field computed at index time by walking this chain. If the chain is unresolved
at index time -> `Data.householdId = null` -> permanently null (never backfilled).

## HCM API patterns

Base URL: `https://{tenant}-hcm.digit.org` (e.g. `https://ondo-hcm.digit.org`).

| Endpoint | Method | Body key | Response key | Status |
|---|---|---|---|---|
| `/household/v1/_search` | POST | `Household: {clientReferenceId: [...]}` | `Households` (plural) | 200 |
| `/household/v1/_create` | POST | `Household: {...}` | `Household` | 202 |
| `/household/v1/_update` | POST | `Household: {...}` | `Household` | 202 |
| `/household/member/v1/bulk/_create` | POST | `HouseholdMembers: [...]` | `HouseholdMembers` | 202 |
| `/project/task/v1/bulk/_create` | POST | `Tasks: [...]` | `Tasks` | 202 |
| `/project/beneficiary/v1/_search` | POST | `ProjectBeneficiary: {...}` | `ProjectBeneficiaries` | 200 |

DIGIT returns 202 (Accepted) for create/update — check `in (200, 202)`. RequestInfo:
`{"RequestInfo": {"authToken": "<token>"}}`. `RequestInfo.apiId` is REQUIRED for individual
creates (downstream user call fails without it). Individual create/search must use the HCM
service path `/health-individual/v1/...` NOT the core `/individual/v1/...` (different service,
different schema). Household create needs: tenantId, clientReferenceId, memberCount,
householdType, address{tenantId, type, locality{code}}. Household member create requires the
parent household to exist in DB first (VALIDATION_ERROR otherwise). `isHeadOfHousehold=True`
members CANNOT be updated via API (duplicate-head constraint). Task resources[] REQUIRE a
`tenantId` field.

## Production credentials (last-known — rotate/verify)

Nigeria production DB:
```
host: <REDACTED-DB-HOST>  port: 5432
database: ngcentralprd  user: <REDACTED-USER>  password: <REDACTED>  sslmode: require
```
Nigeria production ES (internal — only from server/Jupyter, not local):
```
URL: <REDACTED-ES-URL>
user: <REDACTED-USER>  password: <REDACTED>
```
Togo ES (separate cluster, Basic auth, no tenant prefix on indices):
```
Authorization: Basic <REDACTED>
```
Local dev: project svc localhost:8080, Kafka 9092, Redis 6379, DB localhost:5432
(postgres/postgres/postgres), schema public.

## Tenant / campaign map (tokens are last-known — verify)

| Tenant | State | Campaign | API base | Token (last-known) |
|---|---|---|---|---|
| `so` | Sokoto | SMC | | — (production, read-only) |
| `ba` | Bauchi | SMC / MR-DN | `https://bauchi-hcm.digit.org` | `<REDACTED-TOKEN>` (2026-07-01) |
| `oy` | Oyo | SMC 2026 | `https://oyo-hcm.digit.org` | `<REDACTED-TOKEN>` |
| `na` | Nasarawa | SMC 2026 | `https://nasarawa-hcm.digit.org` | `<REDACTED-TOKEN>` |
| `ko` | Kogi | SMC | `https://kogi-hcm.digit.org` | `<REDACTED-TOKEN>` |
| `ab` | FCT/Abuja | SMC | `https://fct-abuja-hcm.digit.org` | `<REDACTED-TOKEN>` |
| `od` | Ondo | ITN | `https://ondo-hcm.digit.org` | `<REDACTED-TOKEN>` |
| `kd` | Kaduna | AZM Cycle 1 2026 | | ref `CMP-2026-06-09-000345` |
| `tg` | Togo | MRDN/SPAQ | (Togo cluster) | projectTypeId `644c4356-5214-11ee-be56-0242ac120003` |
| `pl` | Plateau | SMC | | |
| `ke` | Kebbi | SPAQ (+ORS-Zinc) | | |
| `cg` | Congo | (UAT) | | |

## KD / AZM SQL (proven, campaign ref CMP-2026-06-09-000345)

Table chain: `kd.project` -> `kd.project_staff` (CDDs, `staffid`=`kd.individual.useruuid`) ,
`kd.project_address` (boundary), `kd.project_target`. `kd.individual.username` is NULL for
CDDs — get username from ES `Data.syncedUserName`. Boundary code
`ADMIN_NI_01_01_<lga>_<ward>_FACILITY_NAME` — readable name = `" ".join(code.split("_")[6:]).title()`.
`kd.project` uses `projecthierarchy` (not `path`); always filter `isdeleted=false`.

```sql
-- total CDDs for campaign
SELECT COUNT(DISTINCT ps.staffid)
FROM kd.project_staff ps
JOIN kd.project p ON p.id = ps.projectid AND p.isdeleted = false
JOIN kd.project_address pa ON pa.projectid = p.id AND pa.boundarytype = 'Health Facility'
WHERE p.referenceid = 'CMP-2026-06-09-000345' AND ps.isdeleted = false;   -- 8,487
```
```sql
-- find which table holds a column
SELECT table_schema, table_name FROM information_schema.columns
WHERE column_name = 'useruuid' ORDER BY 1,2;
```
ES username agg: `terms Data.syncedUserId.keyword size 30000` + sub-agg `Data.syncedUserName.keyword`.
Date filter on `Data.taskDates` must use `terms` (list of `YYYY-MM-DD`), NOT `range`.

## Tracer search patterns (egov-tracer-error-details)

`requestBody` is unquoted (keys have no quotes) — `json.loads()` always fails; use regex.
Tenant filter uses the unquoted style `tenantId: oy,` (NOT `"tenantId":"oy"`) in the task-body
match; but the SOP Kibana query uses `"tenantId":"so"` WITH quotes in `Data.apiDetails.requestBody`
(both styles appear across indices — test which matches). Timestamp field `Data.@timestamp`
(not auditDetails, can be null). Error agg field `Data.errors.errorMessage.keyword`.

| Find | Query |
|---|---|
| All task errors (all tenants) | match_phrase `projectBeneficiaryClientReferenceId` (~206,208 docs) |
| Task errors for one tenant | + match_phrase `tenantId: oy,` (Oyo -> 8,393) |
| Household errors | match_phrase `householdClientReferenceId` |

```python
UUID_PAT = r'[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}'
PB_RE    = re.compile(r'projectBeneficiaryClientReferenceId:\s*(' + UUID_PAT + r')')
```
Tracer window: data purged before ~Aug 1, 2025; Jun-Jul 2025 records not verifiable in tracer.

## ES scroll / update snippets

```python
# scroll extraction
resp = requests.post(f"{ES}/{index}/_search?scroll=5m",
                     json={"size":1000, "_source":True, "query":{...}})
sid = resp.json()["_scroll_id"]
# then POST /_search/scroll {"scroll":"5m","scroll_id":sid} until hits empty
```
```
POST /{index}/_update/{doc_id}    {"doc": {"Data": {"householdId": "<value>"}}}   # 200 updated|noop
POST /_bulk  (application/x-ndjson)  {"update":{"_index":..,"_id":..}}\n{"doc":{...}}
```
Use `"_source": True` — a `_source` list with dotted nested fields breaks extraction.
`_search` caps total display at 10,000 (`"relation":"gte"`) — use the `_count` API for the
real total.
