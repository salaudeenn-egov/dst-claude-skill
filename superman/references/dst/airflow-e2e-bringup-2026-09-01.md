# E2E bring-up on dev + MDMS flat schema (2026-09-01)

Continues `airflow-worker-vs-image-2026-09-01.md` (the design settlement). This
records the layer-by-layer production bring-up to the FIRST COMPLETE RUN, the
failure taxonomy demonstrated live, the MDMS config-schema work, and the traps
found. Deployment: **Azure unified-dev** (see cluster facts below).

---

## The bring-up: five layers, each failing loudly, each fixed forward

The packages landed (sri.padma) and the self-contained restructure merged
(PR #2179). Then each scheduler-fired run failed one layer deeper — the correct
shape for a bring-up. In order:

1. **`FileNotFoundError: credential.json not found`** — dst_config had no
   `google_credentials_json`. Fix: embed the service-account JSON in the
   Variable; `dst_config.apply()` materialises it 0600 under /tmp and sets
   `GOOGLE_CREDENTIALS_PATH` itself (`dst_config.py:244-247`). NO file on the
   cluster, NO extra env var — one Variable paste.
2. **`No target book in Drive folder ... Tried: dev_target.csv, ...`** — the
   designed HARD failure ("a report built on zero targets must never be
   published"): `AirflowFailException`, data-error class, zero retries, alert
   "DST report failed". Fix: uploaded `dev_target.csv` (real format from the
   Togo book: `facility_code,facility_name,individual_target,total`).
3. **`Failed to resolve 'elasticsearch-master.es-upgrade'`** — DNS, i.e. the
   URL belongs to a DIFFERENT cluster (see cluster facts). Fix:
   `ES_URL=http://elasticsearch-master.backbone:9200`.
4. **`401 Unauthorized`** on the first backbone attempt, then clean — the
   Variable re-read caught up. Auth errors vs DNS errors: 401 = connected and
   rejected (credential problem); "Name or service not known" = never found the
   server (wrong cluster/namespace). `elastic` + the known dev password work
   against backbone.
5. **FIRST COMPLETE RUN** (slot 07:58, then 08:50): targets 5 facilities loaded,
   task scroll 0 docs (TRUE zero - no dev docs dated today), **staff scroll
   1,020 docs -> 7 unique CDDs** (proves authenticated ES), zero-record guard
   fired, Excels + partner Excel + Word + chart -> Drive, narrative -> Slack,
   **INCOMPLETE warning** posted, `[run-log] appended: Dev Day 4 -> SUCCESS`.
   Every component of the design has now executed in production.

**Pass condition on dev is DEGRADED-and-complete, never green numbers**: dev ES
has no `boundaryHierarchy.healthFacility` and no `age` on any doc, and no task
docs dated today. Green-with-zeros = the guards are broken.

## Failure taxonomy — all three states demonstrated live, same day

| State | What happened | Alert |
|---|---|---|
| FAILURE | missing target book: task red, `AirflowFailException`, no report | "DST report failed - <state> \| <campaign> \| slot ..." |
| DEGRADED | zero records: run green, report PUBLISHED, Run Log SUCCESS | "DST REPORT INCOMPLETE ... is NOT a measurement" + review instruction |
| benign noise | finalize logs `No XCom value found; key="failure"` on SUCCESS | none - the failure key only exists when the pipeline task raised |

## Slot mechanics, operational rules (hard-won today)

- A fired slot's deterministic run id is CONSUMED forever, success or failure.
  To re-run: fresh `report_times` (new id) or clear the DAG run in the UI.
- Failed attempts retry in-place (transient class) and each retry RE-READS the
  dst_config Variable (fresh process under KubernetesExecutor) — a Variable
  paste between attempts rescues the run.
- `campaign_window_contains` rejects any slot outside campaign_start..end —
  a row with last year's dates can NEVER fire regardless of report_times.
- Same time in `report_times` and `partner_report_times` -> ONE run, mode
  `both` (two Slack posts). Different times -> two separate runs. Empty partner
  columns -> the partner *Excel artifact* is still built and uploaded, but no
  partner POST is attempted (explicit skip - audit fix).
- An active row fires DAILY at its times until campaign_end. Deactivate test
  rows or they keep posting.
- Sheets API: write `cycle_index` (and any leading-zero value) with
  `value_input_option="RAW"` - USER_ENTERED coerces "01" -> 1.

## Failure-injection matrix (dev tab rows 3-7, may still exist)

One fault per row, distinct campaign numbers (`CMP-TEST-00T1..T5`) so run ids
cannot collide and the campaign-scoped guard is exercised:
T1 missing target -> hard fail; T2 bogus slack_channel -> report ok, post fails
x3 -> FAILED_POSTS -> alert channel; T3 campaign_days=99 -> CONFIG_CORRECTIONS;
T4 inverted dates -> must NOT fire and must not poison the tick for others;
T5 report_times "25:99" -> never-schedulable warning. The isolation assertion
(one bad row must not kill the tick) is the important one.

## Cluster facts (settle these once)

- **The DST Airflow runs on Azure unified-dev** — kubeconfig
  `D:\DST\scripts\infra\kubeconfig-unified-dev-azure.yaml`, PowerShell profile
  fn `kdev` (context `unified-dev-read`, SA `readonly-shell`). `kdevaws` is the
  LEGACY AWS EKS unified-dev — different cluster, same name; its workloads were
  Pending for days and its configmaps name `es-upgrade`, which does NOT exist
  on Azure. Never copy service URLs across the two.
- Azure cluster services: **ES = `elasticsearch-master.backbone:9200`**;
  airflow ns holds `backbone-airflow-*`; git-sync whole repo at
  `/opt/airflow/dags/repo/`.
- Read-only account CAN `kubectl logs` (live tailing of executing task pods -
  caught runs mid-flight) but NOT secrets/exec/port-forward. Task pods are
  deleted seconds after finish; archived task logs live behind the Airflow
  api-server (UI). Division of labour: live tail = kubectl, post-mortem = UI.
- dev dst_config final shape: env (16 keys incl `ES_INDEX_PREFIX: ""` and the
  backbone ES_URL) + `secrets` {ES_USER, ES_PASS, SLACK_TOKEN, GROQ_API_KEY} +
  `google_credentials_json`. Helper used: build the whole JSON locally from
  .env + credential.json (D:\DST\dst_config.dev.json - DELETE after pasting).

## MDMS on unified-dev.digit.org: gateway + a nasty search gotcha

- Live paths: schema search `/egov-mdms-service/schema/v1/_search` (202), data
  `/egov-mdms-service/v2/_search` (200). `/mdms-v2/*` = 404 on this gateway.
  (Matches dst_config's `MDMS_API_PREFIX=/egov-mdms-service/v2`.)
- **GOTCHA: `MdmsCriteria.schemaCodes` (plural) is silently IGNORED** by this
  MDMS — returns up to `limit` entries of EVERY schema on the tenant (observed:
  roleactions, FormConfig...). Singular `schemaCode` filters correctly.
  `pipeline/mdms.py:242` uses the singular — correct. A wrong criteria field
  here does not error; it returns the whole tenant.
- NO `airflow-configs.*` schemas exist on this env — not ours, and not HCM's
  `campaign-report-config` either. Both need creating before mdms mode.

## The FLAT config schema (replaces the nested-row entry shape)

Problem: the original entry nested the sheet row in a `row` object — opaque in
the MDMS UI, unqueryable per-field. Decision (user): match HCM's style.

Entry `data` is now FLAT: `{**row, "rowIdentity": ..., "deploymentGroup": ...}`
— all 22 sheet columns top-level as strings + 2 envelope keys. `entry_row(data)`
(new helper, mdms.py) strips `_ENVELOPE_KEYS` to recover the row; used by both
the update-detection compare and `get_active_rows_from_mdms`. Round-trip and
change-detection verified. DAGs need zero changes (public function contracts
unchanged). Envelope written LAST in the dict literal so a same-named sheet
column can never shadow it. New sheet columns still flow with no schema change
(additionalProperties not forbidden). Schema artifact:
`dst_data_analysis_report/platform/dst-campaign-config-schema.json`
(24 properties, required minimal: rowIdentity/deploymentGroup/tenant/
campaign_start/active, x-unique rowIdentity).

Framing that survived review: **typed where you query (the Postgres run-log
table), opaque-or-flat where you mirror; validation lives at the human boundary
(plan_sync), not in the store.** HCM's typed schema is right for THEM because
MDMS is their authoring surface; the sheet is ours. NOTE: the plan_sync unit
tests no longer exist (removed in the post-audit test-script cleanup) — the
first real mdms-mode sync run is the create/update/deactivate test.

## Three-way persister alignment (verified mechanically)

producer event keys (15) == persister jsonPaths (15, POSITION-BY-POSITION vs the
INSERT column list) == table columns (15). Positional insert means a swapped
pair would put slot_time into slot_date forever without erroring — check order,
not just names. Idempotency is a 3-part system: uuid4 event_id + UNIQUE + ON
CONFLICT DO NOTHING; removing any one disarms the others. Conditional seam: on
a central instance the producer publishes to `{tenant}-save-dst-report-metadata`
and the yml's bare fromTopic consumes NOTHING, silently (documented in the yml).

## cycle_index now has THREE representations — join hazard

sheet cell / Kafka event / table = "01" (raw); MDMS rowIdentity = "1"
(normalised). Internally consistent per artifact, nothing joins them today, but
"01" != "1" is the bug waiting in any future join or guard comparison. Also the
prime suspect for the still-unexplained 08:45 trigger of an already-succeeded
slot (retime-guard row matching not yet re-verified against a live Run Log row).

## Git errata from this session (self-inflicted, twice)

- `git checkout HEAD -- <package-dir>` to reset caches ALSO reverts every
  uncommitted edit under it, and `git clean -fd <dir>` deletes sibling
  untracked work files. Scope cache cleanup to the `__pycache__` dirs
  themselves, never the parent. (Wiped mdms.py's flat edits + the schema JSON;
  restored from context.)
- Root `.gitignore` had NO pycache rule — added `__pycache__/` + `*.py[cod]`.
  **18 `.pyc` files remain TRACKED under airflow-dags** (committed by earlier
  add -A) — gitignore does not affect tracked files; they need
  `git rm --cached` in some future commit and until then reappear as M noise.

## Open gates (as of 2026-09-01 evening)

- User to commit: .gitignore, mdms.py (flat), schema JSON.
- Padma approval mail (drafted, humanised, no sheet-internals): table +
  persister yml (+ optionally the MDMS schema as item 3).
- MDMS schema _create: ready, awaiting explicit user approval.
- `kafka-python` still missing from the Airflow image — required before
  `DST_MDMS_ENABLED=true` (producer is a silent no-op without it).
- dev tab: control row fires daily; T1-T5 rows present; deactivate when done.
- `D:\DST\dst_config.dev.json` on disk holds the service account - delete.
- Local docker-compose still in bare-image test config (`.bak` backups exist).
