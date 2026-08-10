# Recovery scripts — reconciliation, chain recovery, and push patterns

Two families of scripts do the data-recovery work: generic **reconciliation**
(db↔ES, one entity at a time) and **chain recovery** (the full 5-link entity chain
in one pass). Both find gaps and push missing records back through Kafka / the API.

Core rule (memorise):
- ES record not in DB → the record was never persisted → CREATE it (push through save topic / bulk _create API)
- DB record not in ES → the record was never indexed → UPDATE / re-index it
- Always create parents before children (dependency order below)

---

## Reconciliation scripts — `D:\DST\scripts\script-reconciliation\`

Detect and fix DB↔ES mismatches per entity by pushing missing records through Kafka.

| Script | Direction | What it does |
|---|---|---|
| `db not in es .py` | DB → ES | Finds DB records missing from ES; fetches via API; pushes to Kafka |
| `reconcile_es_not_in_db.py` | ES → DB | Finds ES records missing from DB; reads full ES doc; pushes to Kafka |

Config:
- Credentials from `config.json` in the same dir — NOT env vars
- `DRY_RUN = True` in both — set `False` to push live
- Was pointed at Congo UAT (`azure-congo-uat-db`, tenant `cg`)

JOBS configured (the shape you copy for a new tenant/entity):
- `db not in es .py` → `cg / household`: index `household-index-v1`,
  topic `save-household-topic`, API `http://household.egov:8080/household/v1/_search`,
  `api_response_key=Households`
- `reconcile_es_not_in_db.py`:
  - `cg / household`: `household-index-v1`, `es_unique_id_field=Data.household.clientReferenceId`, `es_data_path=Data.household`
  - `cg / project_beneficiary`: `project-beneficiary-index-v1`, `es_unique_id_field=clientReferenceId`, `es_data_path=""` (flat doc, no nesting), topic `save-project-beneficiary`

Findings from testing (2026-06-10/11), Congo household:
- DB=9,598 / ES=9,598 fully in sync after pushing 45 missing records
- The 45 were `not_in_api` initially because `api_response_key` was wrong
  (`HouseholdList` → fixed to `Households`)
- Project-beneficiary ES doc is FLAT (no `Data.projectBeneficiary` wrapper) →
  `es_unique_id_field=clientReferenceId`, `es_data_path=""`
- config.json ES creds are read-only (`<REDACTED-USER — read-only ES account>`) → cannot write ES directly
- DB access read-only → cannot insert/delete test data

Infrastructure (for logging pushed runs):
- `migrations/V1__create_reconciliation_tables.sql` — `reconciliation_log` + `reconciliation_failed_record` tables
- `persister/reconciliation-persister.yml` — maps topics `save-reconciliation-log`
  and `save-reconciliation-failed-record` to those tables
- Run logs push to `save-reconciliation-log`; failed IDs to `save-reconciliation-failed-record` (only when `DRY_RUN=False`)

Testing limit: read-only DB + read-only ES = can only test detection, not the full
push path artificially. Best real test is a campaign with genuine ghosts (e.g. `so`
had 18,526 confirmed).

---

## Chain recovery scripts — the 5-link entity integrity check

| File | Tenant | Config |
|---|---|---|
| `C:\Users\Salaudeen N\Downloads\extraction_scripts\oy\oy_chain_recovery.py` | oy | Jun 4–8 2026 |
| `C:\Users\Salaudeen N\Downloads\data_Recovery_Script (1)` | kd | Jun 11–12 2026 — the perf-optimised template |
| `C:\Users\Salaudeen N\Downloads\ba_chain_recovery.py` | ba | Jun 22 – Jul 1 2026 |

`data_Recovery_Script (1)` is the 12M+ record template. Output filename prefix is
hardcoded per script — rename for a new tenant.

The 5 links checked:
| Gap | Check | Meaning |
|---|---|---|
| A | HHMember.householdClientReferenceId → Household | Orphan member (HH missing in ES) |
| B | Household → HHMember | Household has no members |
| C | HHMember.individualClientReferenceId → Individual | Individual missing in ES |
| D | Individual → ProjectBeneficiary | No PB for individual |
| E | ProjectBeneficiary → ProjectTask | No task for PB (= missed treatment) |

For each gap: searches `egov-tracer-error-details` and dumps the payload for re-push.

### Date-filter differences per entity (they are NOT uniform)
| Entity | Date field | Format |
|---|---|---|
| Household | `Data.household.auditDetails.createdTime` | epoch ms |
| HH Member | `Data.householdMember.auditDetails.createdTime` | epoch ms |
| Individual | `@timestamp` | ISO string |
| ProjectBeneficiary | `@timestamp` | ISO string |
| ProjectTask | `Data.taskDates` | date string `YYYY-MM-DD` |

Project Task index field gotchas (confirmed on `ba-project-task-index-v1`, 2026-07-01):
- `Data.taskDates` range filter WORKS — use it for date filtering
- `Data.auditDetails.createdTime` — DOES NOT EXIST on task docs (returns 0 results)
- `Data.isDeleted` — DOES NOT EXIST on task docs (returns 0 results)
- `Data.createdTime` exists (epoch ms), `Data.@timestamp` exists (ISO string)
- Always verify a new field with an ES sample query before adding it as a filter.

### Performance optimizations for 12M+ records (applied 2026-06-25)
Original script OOM-crashed the kernel at 12M records. Fixes applied to `data_Recovery_Script (1)`:
| Change | RAM saved |
|---|---|
| Two-pass member scroll: pass 1 builds `mbr_hh_set` + `mbr_ind_set` only; pass 2 builds gap rows | ~2.5 GB |
| Removed `mbr_ind_map` dict (12M entries) → `mbr_ind_set` | ~1.2 GB |
| Removed `pb_ben_ids` set — computed inline: `gap_d_ids = ind_ids - set(pb_data.values())` | ~1.2 GB |
| `del` freed sets + `gc.collect()` between phases | frees completed phases |
| `TRACER_BATCH = 1000` separate from `BATCH = 5000` | ~80% less peak RAM per tracer batch |
| `payload_json` capped at 2000 chars | reduces `found` dict size |
| Saved `ind_count = len(ind_ids)` and `task_count = len(task_pb_ids)` BEFORE `del` | fixes NameError in summary |

Critical pattern: always save counts/references needed later BEFORE calling `del`.
Accessing a deleted variable in the summary block causes NameError. Pass 2 for
members only runs `if gap_a_ids or gap_c_ids` — skipped entirely if no gaps.

### Known bugs in the chain-recovery scripts (as of 2026-06-24)
BUG 1 — Tracer callbacks always parse 0 records (ALL gaps). Both `cb` and `cb_broad`
use `json.loads(body_str)`, which always throws on the unquoted tracer requestBody;
the `except Exception: return` silently swallows it → every gap shows 0 found.
Fix needed: replace `json.loads` with regex extraction in both callbacks. Was
reverted at user request 2026-06-24 — do NOT re-add without asking.

BUG 2 — `TRACER_CFG` E key=`"Tasks"` matches only ~6,941 docs across all tenants.
Correct search term is `"projectBeneficiaryClientReferenceId"` (~206,208 docs).
Combined with BUG 1, GAP E tracer always shows 0.

### Tracer regex extraction (the fix for BUG 1)
The tracer `Data.apiDetails.requestBody` is NOT valid JSON (it is a Java toString,
unquoted keys) — `json.loads()` throws. Extract fields with regex instead of parsing.
See `hcm-context` gotchas (tracer body not JSON) for the pattern.

### GAP A caveat
Households in GAP A may be legitimately `isDeleted=True` in ES, or have a sub-tenant
`tenantId` (not exactly `"oy"`), or were found via search API (= DB) not ES. Query
the household index WITHOUT the isDeleted filter for those IDs before assuming loss.

---

## Push scripts (create/update the recovered records)

`ba_chain_push.py` (Downloads):
- Auto-finds latest `ba_chain_recovery_*.xlsx` via glob
- `USER_UUID` starts as `"FILL_IN_FROM_LOGIN"` — MUST update before running
- Pushes Gap B + Gap D only (gaps with tracer coverage); A/C/E near-zero coverage

`ba_recon.py` (Downloads) — reconciliation push (ES_NOT_IN_DB + DB_NOT_IN_ES):
- Input folder `ba_recon (1)`, checkpoint `ba_reconciliation_push_checkpoint.json`
- Captures full error response (no truncation)
- Fix learned: PT create needs `tenantId` inside `resources[0]` (else
  `NotNull.taskRequest.task.resources[0].tenantId`)
- Open issue: HHM updates where `isHeadOfHousehold=True` are rejected by DIGIT
  (can't update existing head); two-step workaround was reverted at user request

### Checkpoint pattern (large pushes)
Push scripts write a checkpoint JSON so a crashed/paused run resumes without
re-pushing. Delete the checkpoint file for a clean fresh run.

### Push worker config (individual/task recovery push scripts)
- `WORKERS = 30` (create), `50` (search + FCT create)
- `BATCH_SIZE = 500`, search `limit=500`
- `timeout = 15s` (create), `30s` (search)
- DUPLICATE_ENTITY treated as OK (not a failure) → goes to Duplicate sheet
- Output sheets: Summary, Created_OK, Duplicate, Failed, Skipped

### Dependency order for CREATEs (parents before children)
1. Household (must exist in DB before its members — else VALIDATION_ERROR)
2. Household Member
3. Individual
4. Project Beneficiary
5. Project Task (needs its beneficiary present; `resources[0].tenantId` required)

DIGIT create/update APIs return HTTP 202 (Accepted), not 200 — see campaigns.md and
hcm-context for the per-API response-key list (`Households`, `ProjectBeneficiaries`, etc.).

---

## Unified workbench — `D:\DST\scripts\recon-recovery` (the "combine all" tool)

Notebook app (`recon_recovery.ipynb`, 2 cells) that merges reconciliation + relational
chain error-tracer recovery + push into ONE UI. Supersedes running the standalone
scripts separately. Structure: `core/{config,db_client,es_client,api_client,entities,
chain}.py`, `core/utils/{processor,excel,dates}.py`, `core/ui/{app,common}.py`. NO
config.json — secrets live in `.env` (ES + DB, plus in-cluster `HCM_SVC_*` service
hosts); per-run UI inputs are just Tenant + Start/End date (DATE PICKERS) + the
"Include error-tracer recovery" checkbox. In-cluster mode: push hits service DNS
directly (`http://{service}.egov:8080`), no domain URL / no auth token (userInfo-based).
`core/utils/dates.py` holds the shared `epoch_ms/iso/noop` (was duplicated in
chain+processor); `es_client.tracer_search` was DEAD and is removed — the only tracer
scan is `chain._tracer_scan`.

Two INDEPENDENT analyses (do NOT cross-relate):
- **Reconciliation** (`run_check`) — per-entity DB vs ES id diff → ghosts (ES-not-in-DB,
  CREATE) + index-failed (DB-not-in-ES, UPDATE). Always runs.
- **Error-tracer chain recovery** (`core/chain.py`, `run_chain`) — relational 6-link
  A–F check + tracer dump (F added 2026-08-10, see its entry below). Runs only when
  the "Include error-tracer recovery" checkbox is on; else reconciliation-only.

Push (`run_push`): reconciliation ghosts rebuilt from ES (no tracer) + index-failed
UPDATE; chain-recovered payloads pushed **search-DB-first, CREATE only if absent**.
NO checkpoint (removed by design — re-run re-checks every id; search-then-create makes
it safe). Single `_create`/`_update` per record (NOT bulk) and **single-threaded** —
the known perf gap; `ba_chain_push` used `ThreadPoolExecutor(10-30)`. Push result
workbook has one sheet per outcome: CREATED / RE_INDEXED / DUPLICATE_EXISTS / FAILED /
ALL_RESULTS, each row carrying Source + Operation + plain-English Outcome.

Chain workbook per gap: `GAP_*` (all missing) | `TRACER_*` (recovered, pushable) |
`PROCESSED_*` (already flagged isProcessed=true, skipped) | `UNREC_*` (not found in
tracer) + `SUMMARY`. Tracer has 3 fallback levels (primary exact-url+body_key → fallback
same-key/other-url → direct tenant-scoped scroll matched in-memory).

### isProcessed skip + flag + output/UX (2026-07-15, VERIFIED on bo)
- **Skip already-processed (MANDATORY, no toggle).** Each tracer scan reads
  `Data.isProcessed` and classifies matched ids: `found` (new → pushable) vs `already`
  (isProcessed=true → listed in `PROCESSED_*`, EXCLUDED from push). Absent field →
  read as False (safe). `_is_processed` accepts bool `true` / `"true"` / `1`.
- **Flag on push (AUTOMATIC, no checkbox).** After the chain push, `run_push` sets
  `Data.isProcessed=true` on fully-recovered tracer docs — flags a doc ONLY when EVERY
  recovered crid it carried pushed OK (bulk doc holds many crids). Matched by ES `_id`
  OR `Data.uuid.keyword` (match_phrase fallback if that mapping is absent). Uses the
  SAME ES cred (reads AND updates; `dst` HAS update rights — confirmed). `conflicts=proceed`,
  no forced refresh. `es_client.flag_tracer_processed(doc_ids)`.
- **Read-only-cred failure is non-fatal + surfaced.** 401/403 → clear RuntimeError →
  caught → `flag_note`/`flag_ok=False` shown as an amber note + amber banner (not buried
  in the log). Push already succeeded; a missed flag only costs a redundant search next
  run (search-DB-first prevents any re-create).
- **Verified:** push flagged 153 bo docs → `_count` `Data.isProcessed=true` + tenant `bo`
  phrase = 153. tracer index `egov-tracer-error-details` is SHARED (no `bo-` prefix);
  scope by `match_phrase '"tenantId":"bo"'` on requestBody.
- **Output is now self-explanatory:** ONE shared analyze timestamp across recon dir,
  chain dir, and the zip; descriptive folders `<t>_reconciliation_<ts>` /
  `<t>_error_tracer_recovery_<ts>` / `<t>_push_<ts>`; a `README.txt` at the zip root
  explains every file; the redundant reconciliation summary `.csv` was dropped (xlsx only).
- **UI/UX:** Start/End are `DatePicker`s; large downloads (>25 MB) stream via the Jupyter
  `/files/` endpoint (JupyterHub-prefix aware) instead of failing to inline.
- **Service names verified in-cluster (bo):** all 6 `_search` endpoints answer HTTP 200
  with the right resp_key; `Individual` resp_key is singular `Individual` (CORRECT — a
  prior review doubt, refuted live). HouseholdMember search needs timeout ≥30s on big tenants.
  Standalone read-only checker: `Downloads/verify_services.py`.
- **OPEN ISSUE — push "user is invalid".** Create/update send a FABRICATED `RequestInfo.userInfo`
  (uuid = `DEFAULT_USER_UUID` placeholder `f47ac10b…`, id=0, userName `recon-recovery`) — DIGIT
  can reject it. There is no UI field for a real user. Fix pending: surface + validate a real
  user identity before push. (Separate from the isProcessed flag, which uses the ES cred.)
- **Docs + deploy (2026-07-16):** repo has `SOP.md` (end-user) + `DEPLOYMENT.md` (new-env
  setup) + `verify_services.py` (read-only connectivity checker, now IN the repo — run
  `python verify_services.py <tenant>` from the folder; auto-finds local `.env`). Shareable
  Word set in `Downloads/`: `Reconciliation_Recovery_User_Guide.docx` (complete, with 8 inline
  captioned screenshot PLACEHOLDER boxes to replace), `..._Screenshots_Guide.docx` (capture
  checklist), `..._Deployment_Guide.docx`. Doc style: no bookmarks, no emojis/unicode
  (see [[feedback_doc_style]]). Deploy needs: DB read cred, ES cred with read + `_update_by_query`
  on `egov-tracer-error-details`, real DIGIT push user (the open user-is-invalid item), and
  `HCM_SVC_*`/`HCM_SERVICE_TEMPLATE` DNS confirmed via `kubectl get svc -A`.

### Implementation-status caveat (2026-07-15) — RESOLVED 2026-08-10
A 2026-07-15 read of `core/chain.py` found no confirm step (gaps were a pure date-windowed
set-diff). A full-code read on 2026-08-10 confirms the pass-2 IS implemented and committed:
`EXIST_CHECK` (per-gap index/id-field/tenant-field/isDeleted tuples) + `_existing_ids()`
(batched terms filter + terms-agg, no date window) run after the pass-1 diff for every gap.
Chain gap counts on multi-cycle tenants are trustworthy on current `main`.

### Chain-recovery correctness principles (hard-won on bo, Jul 2026)
These are the rules that make the relational check CORRECT for multi-cycle data:

1. **Master data is reused across cycles; per-cycle data is fresh.**
   - Master (stable clientReferenceId, reused every round): Household, Individual,
     HouseholdMember. Have NO `projectId`.
   - Per-cycle (new id + new `projectId` each campaign): ProjectBeneficiary, ProjectTask.
   - So a 2026 member legitimately points at a 2025 household — that is NOT an orphan.
     Proven: member `d4f054b0…11f1` (created 2026-07-09) → household `bdd9e0e0…11f0`
     (created 2025-07-05, isDeleted=false, present). `11f0`/`11f1` UUID-v1 stamps encode
     the year.

2. **Never date-window the EXISTENCE side of a relational check.** Windowing the parent
   set makes any child referencing an older parent a phantom gap (this produced 4,180
   fake Gap-A orphans and a 131k fake Gap-E on bo). Fix implemented (chain.py): `EXIST_CHECK` +
   `_existing_ids()` — the windowed set-diff yields *suspects* ("pass 1"), then each suspect is
   confirmed against ALL data ("pass 2": no date filter, batched `terms`+`terms` aggregation on
   the related index). Only ids with genuinely no match anywhere survive → tracer. Log shows
   `Gap X — after all-dates existence check: N truly missing; M dismissed`.
   `EXIST_CHECK` per gap: A→household-index/Data.household.clientReferenceId.keyword,
   B→member-index/…householdClientReferenceId, C→individual-index/clientReferenceId,
   D→pb-index/beneficiaryClientReferenceId, E→task-index/Data.projectBeneficiaryClientReferenceId.
   NOTE: reconciliation (`run_check`) does NOT get this pass-2 (by design) — it stays a plain
   windowed diff; push's `check_exists` already skips any record that exists, so no duplicates.

3. **Distribution is inherently current-cycle via id-uniqueness.** A ProjectTask links to
   its PB by `projectBeneficiaryClientReferenceId` = the PB's cycle-unique clientReferenceId.
   A previous-cycle task CANNOT reference a 2026 PB id, so Gap E's "exists elsewhere" can
   only match a CURRENT-cycle task — a prior round can't falsely satisfy "distributed today".
   No projectId filter needed for E's correctness (the id already scopes it).

4. **Gap E large + ~0 tracer coverage = NOT recoverable — and it is mostly HOUSEHOLD HEADS.**
   RCA on ch (2026-07-22): a 148K "PB with no task" Gap E was NOT missing treatment. Ruled out
   with data: join is correct (tasks link only by `projectBeneficiaryClientReferenceId`;
   `projectbeneficiaryid` is NULL on all 830k tasks), not duplicates (5 of 173,952), not window
   (pass-2 handles that). ROOT CAUSE: **97.8% of task-less PBs are HOUSEHOLD HEADS** — the adult
   respondent registered per household, never a dosing target, so no task BY DESIGN (cross-tab:
   un-dosed 182,180/186,170 heads; dosed 333,862/351,859 non-head members = the children).
   FIX in chain.py: Gap E drops PBs whose beneficiary is a head (`head_ind_ids`, built from the
   member load's `isHeadOfHousehold`), leaving only the real ~2,461 un-dosed eligible members.
   **DOB was REJECTED as the criterion** — a missing/`1970-01-01` birth date is only a SYMPTOM
   (heads are untracked adults); keying on it would wrongly drop a real child with an unentered
   DOB. Lesson: filter on the DIRECT semantic flag (isHeadOfHousehold), never a correlated symptom.
   Still do NOT push Gap E. The genuinely recoverable data is in the RECONCILIATION ghosts, not Gap E.

5. **`projectId` is the true campaign key** (not the date window). 2026 bo project
   `529ac0ad…`; 2025 SMC `ec6efaa8…`. Households/individuals have no projectId; scope the
   per-cycle side by projectId if a clean single-campaign figure is needed.

### Gotchas specific to this tool
- **Tracer phase-3 hangs on large gaps.** The direct id-string scan re-scrolls the whole
  tracer index once PER 50-id batch → ~2,100 full scans for a 100k gap → freeze. Cap it
  (`PHASE3_MAX_IDS`, skip phase 3 above ~1000 ids; phases 1–2 catch the real matches).
- **Reconciliation date-field alignment (FIXED 2026-07-21, confirmed on ch/Chad).** The DB
  side (`db_client.load_ids`) windows on a per-entity date column; the ES side windows on the
  entity's `es_date_field`. If the two describe different events, you get false ghosts. The
  worst case was ProjectTask: ES `Data.taskDates` = DEVICE/administration date, DB was on
  `createdtime` = SERVER persist/sync date. A delivery administered Jul-17 but synced Jul-20/21
  lands inside a 20–21 DB window but outside the ES window → false "DB not in ES". ch: 9,738
  task "ghosts", ES `_count taskDates<start` = 11,109 → whole gap was false; the ES docs existed
  the whole time (proven with `Downloads/ch_task_reindex_test.py`: search+`_update` bumped DB
  rowVersion 2→3 / HTTP 202 but the ES doc was already present and its `Data.createdTime` stayed
  the original — task `_update` also does NOT re-index, the task index is built by the transformer
  off create/sync, not the update REST path). **Fix:** per-entity `db_date_col` (default
  `createdtime`) — ProjectTask → `clientcreatedtime` (device time; == ES `Data.createdTime`, its
  date == `Data.taskDates`). Edits: `entities.py` (task base key + `get_entities` default),
  `db_client.py:load_ids`. Null `clientcreatedtime` drops the row (ES taskDates usually absent too
  → balanced); COALESCE if a new small gap appears.
- **DB date column alignment — CORRECTED 2026-07-22 (supersedes a wrong 07-21 note).** A 07-21
  change had set recon DB `db_date_col` to the DEVICE clock (`clientcreatedtime`) for ALL entities
  on the theory that ES windows on device time. VERIFIED WRONG on ch: ES `auditDetails.createdTime`
  == DB **`createdtime` (SERVER)** for household/member/individual/PB (sampled 12 divergent records,
  all `es_eq_server=true`; the one device-looking case, `e720b930`, was a late-synced exception).
  So the recon DB side stays on **`createdtime`** for all audit entities; ONLY ProjectTask uses the
  device clock (`COALESCE(clientcreatedtime, createdtime)`, matching ES `Data.taskDates` = device
  admin date). The chain (chain.py) also windows Individual/PB on `auditDetails.createdTime` (not
  `@timestamp`, which drifts on re-index). **KEY LESSON: NO single DB column aligns with ES 100%** —
  a late-synced minority is device-clocked while the bulk is server-clocked — so column choice can't
  fully fix it; that is exactly why chain relies on the unwindowed pass-2 (`_existing_ids`) and recon
  relies on push `check_exists`. General rule: don't reconcile a single live day — use the FULL
  campaign window on a SETTLED period (≥1 day old) so sync-lag doesn't inflate index-failed.
- **egov-persister DB `createdtime` = server persist time, not device time** (`Data.createdTime`
  in ES is the device time). They differ by the sync lag.
- The tracer requestBody parse uses `json.loads` (frozen per house rule) — recovery=0 for a
  gap could be genuine (nothing failed) OR the unquoted-body issue; B/D recovering >0 proves
  the parser works on that run.
- **A narrow date window inflates the suspect counts.** The entity loads are date-windowed, so a
  1-day window (bo was run on a single day 2026-07-09) makes tons of legitimately-related records
  fall just outside the slice → huge suspect counts and lots of 'exist elsewhere' churn (bo Gap E:
  109,158 suspects → only 106,403 real after confirm). Run the FULL campaign window for clean
  numbers, not a single day.
- **'exists elsewhere' defined:** the loads are date-windowed but `_confirm_present` has NO date
  filter, so 'exists elsewhere' = the id IS in the index but on a date OUTSIDE the loaded window —
  a reused prior-cycle parent (Gap A/C), or an adjacent-day / other-round child (Gap B/D/E). Only
  ids found nowhere (0 hits, any date) are real gaps → tracer.
- **Verified end-to-end (bo, Jul 2026):** A/C dropped to 0 (confirmed households/individuals exist
  from 2025 via live lookup); B/D/E 'real' survivors confirmed genuinely missing via 0-hit `terms`
  queries on the child index (household-member / project-beneficiary / project-task). The confirm
  logic is correct in both directions.

### Gap D relinked for SMC: beneficiary = household, not individual (2026-07-28)
- **STATUS (2026-08-10, verified by full-code read + `git log`): COMMITTED on `main`.** The
  2026-07-28 "NOT on main" correction is now itself outdated — the household-based relink was
  subsequently committed (commit `074d478` "Update README and chain recovery logic for clarity on
  household beneficiary relationships") and the working tree is clean. `chain.py`'s live line is
  `"D": mbr_hh_set - pb_ben_ids`; the README relation table shows the household-based D. The
  history below is kept for the how/why.
- **Problem.** Gap D was coded generically as `Individual.clientReferenceId ->
  ProjectBeneficiary.beneficiaryClientReferenceId` (an individual with no PB record) — this was
  the assumption validated on ch/Chad and baked into `chain.py` since the multi-cycle work. It
  was never wrong on ch, so nothing flagged it as SMC-specific.
- **Approach.** Not re-derived empirically this session — the user (domain owner) told the tool
  directly: for SMC, `ProjectBeneficiary.beneficiaryClientReferenceId` actually holds the
  HOUSEHOLD's clientReferenceId (the same value as `HouseholdMember.householdClientReferenceId`),
  not the individual's. Recorded here as a domain correction, not an independently-proven RCA —
  verify against a live SMC tenant's PB docs before relying on it elsewhere.
- **Root cause.** The SMC data model registers the PB against the household, despite the field's
  name (`beneficiaryClientReferenceId`) suggesting an individual.
- **Fix.** `chain.py`: Gap D changed from `ind_ids - pb_ben_ids` to `mbr_hh_set - pb_ben_ids`
  (`mbr_hh_set` = the set of `householdClientReferenceId` values collected off HouseholdMember —
  already computed for Gap A/B, just reused). Updated in lockstep: `GAP_LABELS["D"]`,
  `GAP_COLS["D"]` (now `HouseholdMember_householdClientReferenceId`), the gap-row builder, the
  module docstring's link table, and `README.md`'s relation table. `TRACER_CFG["D"]` /
  `EXIST_CHECK["D"]` did NOT need to change — both already keyed on the field name
  `beneficiaryClientReferenceId`, only the SET being diffed against it changed. Gap C (HHMember ->
  Individual) is untouched; `ind_ids` still feeds it.
- **OPEN — Gap E's head-exclusion filter is now broken (flagged to user, not yet fixed).** The
  Jul-22 ch fix (principle 4 above) excludes Gap E survivors via
  `pb_data.get(pb, "") in head_ind_ids` — comparing PB's `beneficiaryClientReferenceId` against a
  set of INDIVIDUAL clientReferenceIds of household heads. Now that `beneficiaryClientReferenceId`
  is a HOUSEHOLD id for SMC, that comparison spans two different ID namespaces and will silently
  match ~nothing → Gap E quietly reverts to the raw "PB with no task" count, un-excluding heads.
  Not fixed because the correct fix depends on a domain answer not yet given: is
  ProjectBeneficiary still one-record-per-eligible-individual (just mis-stamped with the household
  id), or genuinely one-record-per-household now? The two answers need different filters (a
  different lookup field vs. abandoning the head-exclusion concept entirely). Ask before touching
  Gap E again.

### Gap F added — direct Household -> ProjectBeneficiary link (2026-08-10)
- **STATUS: UNCOMMITTED working-tree edit on `main`** (user was advised to commit after a live
  run; this repo has lost uncommitted work to a branch reset before — verify `git status`).
  Not yet run against a live tenant.
- **Problem.** Gap D (household-based since the SMC relink) only sees a household THROUGH an
  existing HouseholdMember record — its candidate set is `mbr_hh_set`, built by scrolling
  `household-member-index-v1`. A household whose member records never synced at all is
  structurally invisible to D, even if its PB is also missing. (Same blind-spot family as the
  chad Gap B/C finding: detection scoped to records that already exist in ES.) User asked for
  "one more chain in relationship": `Household -> Data.household.clientReferenceId` vs
  `ProjectBeneficiary -> beneficiaryClientReferenceId`.
- **Approach.** A naive `set(hh_ids) - pb_ben_ids` would re-report nearly every Gap D id
  (member-referenced households dominate the household index) and the push would attempt the
  same PB create twice in one run. So F is scoped to exactly what D cannot see.
- **Fix (`core/chain.py` + `core/utils/processor.py` + README + app.py zip-readme).**
  `gap_ids["F"] = set(hh_ids.keys()) - pb_ben_ids - mbr_hh_set` — households with NO member
  record and NO PB; each household lands in exactly one gap (D or F), no double-push.
  `GAP_LABELS["F"]` = "Household (direct, no member record) not registered as a Project
  Beneficiary"; `TRACER_CFG["F"]` and `EXIST_CHECK["F"]` are identical to D's (the missing
  record type is a ProjectBeneficiary either way — body key `ProjectBeneficiaries`, id field
  `beneficiaryClientReferenceId`, confirm against `project-beneficiary-index-v1`);
  `GAP_COLS["F"]` = `Household_clientReferenceId` + `locality` (reuses the `hh_ids` crid->locality
  map, like Gap B). Every `for g in "ABCDE"` loop widened to `"ABCDEF"` (pass-1 log, pass-2
  confirm, tracer scan, per-gap CSV, summary, workbook — F inherits the 4-phase tracer scan,
  isProcessed skip/flag, and PROCESSED_F/UNREC_F sheets automatically). Push:
  `CHAIN_GAP_ENTITY["F"] = "ProjectBeneficiary"`, `CHAIN_PUSH_ORDER = ["A","C","B","D","F","E"]`
  (F's PBs created right after D's, before tasks). Verified: py_compile clean + config-dict
  consistency check (all six gaps in every dict).
- **Design note.** If the full raw set (including D overlap) is ever wanted, drop the
  `- mbr_hh_set` term — one-line change — but then the same PB can be attempted under both D
  and F in one run (second attempt lands as DUPLICATE via `check_exists`/API, noisy not fatal).

### HFReferral commented out — SMC-only entity (2026-07-28)
- **STATUS (2026-08-10, verified by full-code read): COMMITTED on `main`.** The 2026-07-28
  "NOT on main" correction is outdated — `entities.py`'s `_BASE` on current `main` HAS the
  HFReferral block commented out (with the re-enable note in place), so ITN tenants no longer
  404 on `hf-referral-index-v1`. Remember to re-enable (uncomment) when reconciling an SMC
  tenant that needs HFReferral coverage.
- **Problem.** Reconciliation on tenant `chad` (an ITN/bednet campaign) errored every run:
  `index_not_found_exception: no such index [chad-hf-referral-index-v1]` — the whole entity loop
  wastes 3 retries before `run_check`'s per-entity try/except skips it (the tool never crashed,
  just noisy + slow).
- **Approach.** User confirmed HFReferral (Health-Facility Referral) is an SMC-only concept — ITN
  campaigns have no referral workflow, hence no index.
- **Root cause.** `entities.py` hardcoded HFReferral into `_BASE` unconditionally for every tenant.
- **Fix.** Commented out the HFReferral block in `core/entities.py`'s `_BASE` list (kept in place,
  not deleted, with a note on why + how to re-enable). `build_hf_referral()` is left defined but
  unused. Re-enable (uncomment the block) when running against an SMC tenant. A per-tenant/
  campaign-type toggle was considered (`include_hf_referral` param threaded through
  `get_entities`/`build_campaign_context`) but the user asked for the simpler comment-out instead —
  don't re-introduce the toggle unless asked.

### Still-open, unfixed bug: `build_household()` locality field mismatch (re-confirmed 2026-07-28)
- **Problem.** Every reconciliation-ghost Household pushed via CREATE silently sends
  `address.locality.code = null`, regardless of the real value.
- **Approach.** `entities.py:build_household()` reads `Data.household.address.locality.code` off
  the push-time row. `es_client.fetch_details()` builds that row via `pd.json_normalize` on the
  raw ES `_source`, so the column name exactly mirrors the real ES field path. `chain.py`'s own
  household loader (`cb_hh`) proves the real path is `.address.localityCode.code` (note
  "localityCode", not "locality") — confirmed live, not just a doc mismatch.
  called out again this session; last flagged in a 2026-07-22 review pass and never fixed.
- **Root cause.** One-word field-name typo (`locality` vs `localityCode`) in the payload builder,
  never caught because `_safe()` returns `None` on a miss instead of raising.
- **Fix.** NOT YET APPLIED (pending user go-ahead) — one-line change:
  `_safe(row, "Data.household.address.locality.code")` -> `_safe(row, "Data.household.address.localityCode.code")`
  in `build_household()`. Only affects the reconciliation push path (ES-rebuilt CREATEs), not the
  chain-recovery push (which replays raw tracer payloads verbatim, unaffected).

### Tracer recovery false-negatives on chad — parser, bare-array body, and a 4th fallback (2026-07-28)
- **STATUS (2026-08-10): COMMITTED on `main`.** This fix and the error-response truncation fix
  were committed after the 07-28 session (commit `bd30f99` "Increase response message length
  limits..."); the working tree is clean. All four chad-era changes (this, truncation, Gap D
  relink, HFReferral comment-out) are now on `main` — the earlier "only 2 of 4 survived the
  branch reset" state no longer applies.
- **Problem.** A confirmed-missing Gap C individual (`958946c0-664e-11f1-9b90-c30ea38fbd92`,
  tenant `chad`) was trivially findable by hand — a plain Kibana `match_phrase` on the literal id
  string against `egov-tracer-error-details` returned it instantly (doc `3a532b4a-...`,
  `errorMessage: UPLOAD_ERROR_FROM_APP`, `url: /individual/v1/bulk/_create`) — but `chain.py`'s
  tracer scan reported it as not-in-tracer (would have landed in `UNREC_C`).
- **Approach.** Traced the doc through all 3 existing fallback phases by hand. Found the requestBody
  was `[{dateOfBirth: 12/06/2006, tenantId: chad, clientReferenceId: 958946c0-..., ...}]` — a
  Java-toString-style object literal (unquoted keys AND unquoted scalar values) AND a bare array
  with no `{"Individuals": [...]}` wrapper. Confirmed the analysis date window (12 Jun–20 Jul 2026)
  did NOT exclude it (doc timestamp = 12 Jun, inside the window) — ruling out the window as the
  cause and isolating it to parsing/search-shape assumptions.
- **Root cause — a layered set of unstated shape assumptions, ANY one of which silently dropped
  the record with zero logging:**
  1. Phases 1/2 (primary/fallback) require the literal `body_key` phrase (e.g. `"Individuals"`) in
     the requestBody — a bare array has no such wrapper, so ES never returns the doc for these
     phases at all.
  2. Phase 3 (direct) DOES retrieve it (no body_key/url filter) — but the callback's
     `json.loads(raw_body)` always throws on the unquoted-key format (`except Exception: continue`
     silently dropped it), and even a hypothetical successful parse would still hit
     `if not isinstance(body, dict): continue` since the parsed value is a list, not a dict.
  3. No fallback existed with ZERO assumptions (no tenant phrase, no date window, no url/body_key) —
     the exact shape of a manual `match_phrase`-on-the-id-string Kibana query.
- **Fix (`core/chain.py`):**
  1. Added `_loose_json(s)` — NOT a hand-rolled recursive-descent parser (a ~120-line first draft
     was rejected as too much new logic for the size of the fix) but a ~20-line regex preprocessor
     (`_LOOSE_KEY_RE`/`_LOOSE_VAL_RE`/`_LOOSE_NUM_RE`) that quotes bare keys/values then hands the
     result to `json.loads()` — all real nesting/escaping stays in the stdlib parser. `_tracer_scan`'s
     `cb()` now falls back to it only when `json.loads()` fails. Verified against the exact chad
     record plus edge cases (negative numbers, ISO timestamps with embedded colons, empty `{}`/`[]`,
     nesting). Known accepted limitation: a value containing a literal comma would be mis-split —
     not a real HCM payload field, so acceptable.
  2. `cb()` now also accepts a **bare-array body** as the record list directly, instead of assuming
     every body is `{"<body_key>": [...]}`.
  3. Added **phase 4** (`PHASE4_MAX_IDS = 1000`): one `match_phrase` query per id still unfound after
     phase 3, with NO tenant/date/url/body_key constraint at all — the literal Kibana-style query
     that proved the record was findable. Capped so a large residual gap (>1000 ids) can't hang the
     run on a per-id ES round-trip storm (same hang risk as the historical phase-3-cap gotcha on the
     old oy script) — skip is logged, not silent.
  4. This fix is inside the SHARED `_tracer_scan()`, called once per gap in `recover_chain`'s
     `for g in "ABCDE"` loop — applies identically to all 5 gaps (Household/HHMember/Individual/
     ProjectBeneficiary/ProjectTask), not just the Individual case that surfaced it.
  5. Push (`processor.py`) needed NO changes for this — chain.py always re-serializes the recovered
     record via `json.dumps(rec)` before writing `payload_json` to the CSV, so push's
     `json.loads(payload_by_id[rid])` only ever sees clean, guaranteed-valid JSON regardless of the
     original tracer body's format. Verified the CSV round-trip (pandas `to_csv`/`read_csv`)
     preserves a payload with nested objects/arrays and a value containing an embedded comma+quote.
  6. **Honest limits, not fully closed:** phase 4's 1000-id cap means a gap with a larger residual
     count skips it entirely; `_loose_json`'s comma-splitting limit remains; a record that was never
     logged to the tracer at all (dropped before any error-logging step) is unrecoverable regardless.
     Confirm actual coverage per run by checking the `UNREC_<gap>` sheet, not by assuming "all fixed."
- **Full error-response truncation was also hiding push failures.** Push summary showed
  HouseholdMember (Gap B, 430/430) and ProjectBeneficiary (Gap D, 139/139) failing at 100%, but the
  actual DIGIT error text was unreadable — two independent hard truncations: `api_client.py:
  classify_error()`'s raw-response fallback was `resp_text[:400]`, and `processor.py`'s `record()`
  capped every `Response` cell at `str(resp)[:500]`. Both raised to `[:4000]` so the real validation
  error survives into the `FAILED`/`ALL_RESULTS` sheets. Nothing had been pushed yet for those two
  gaps, so re-running push is safe (search-then-create skips anything already created) — re-run to
  get the real error text before diagnosing the 100%-failure root cause.

### RCA of the 100%-failure push gaps, once the truncation fix revealed the real errors (chad, 2026-07-28)
Analyzed `chad_push_FAILED_20260728_101633.csv` after the truncation fix — both failure sets are
100% one single cause each, not noise:
- **HouseholdMember (Gap B), 430/430 = `INDIVIDUAL_NOT_FOUND`, and 430/430 are household HEADS**
  (`isHeadOfHousehold: true` — checked across all 430, no exceptions). Root cause: a real detection-
  scope gap in `chain.py`, not a data-quality fluke. Gap C ("individual missing") only builds its
  candidate list (`mbr_ind_map`) from household members that ALREADY EXIST in ES (`cb_mbr` scrolls
  `household-member-index-v1`). A Gap-B-recovered household member never existed in ES in the first
  place, so its `individualClientReferenceId` was never fed into `mbr_ind_map` — Gap C structurally
  cannot see or recover it, even if that individual is also genuinely missing. Since these are heads
  (known from the ch/Chad RCA — principle 4 above — to be the record class most likely to have an
  incomplete/never-synced registration), it's very plausible the head's own Individual record failed
  in the same original sync batch as their HouseholdMember. **Not yet fixed** — would need Gap C's
  scope extended to also cover the individual references inside Gap-B-recovered payloads, which is a
  design change, not a one-liner. Verification step given to user (not yet run): pick 2-3 of the
  "not found" individual ids and query DB directly to confirm genuine absence.
- **ProjectBeneficiary (Gap D), 139/139 = `DUPLICATE_ENTITY`** — every one already exists per DIGIT's
  own uniqueness check at CREATE time, yet `check_exists()`'s pre-flight search (via the search API)
  didn't find any of them beforehand. Leading hypothesis (not yet DB-confirmed): the same ES/DB-
  divergence (ghost-record) pattern this whole toolkit exists to catch, just surfacing in `check_exists`
  itself — the search API is commonly ES-backed while DIGIT's duplicate rejection is DB-enforced, so a
  record that exists in DB but hasn't been ES-indexed yet reads as "not found" pre-push and "duplicate"
  at create. Since `chain.py`'s own Gap D detection (`pb_ben_ids` + the pass-2 `_existing_ids` confirm)
  reads from that SAME `project-beneficiary-index-v1`, if it's lagging for these 139 households, Gap D
  would have wrongly flagged them as missing a PB in the first place — these may not be real gaps at
  all. Verification step given to user (not yet run): pick 2-3 of the PB clientReferenceIds and query
  DB directly to confirm they already exist.
- Neither RCA is proven past the hypothesis stage — both need a direct DB query the user hasn't run
  yet (no DB/ES access from this environment). Don't present either as settled fact in a future
  session; check whether the user has since run the verification queries.
