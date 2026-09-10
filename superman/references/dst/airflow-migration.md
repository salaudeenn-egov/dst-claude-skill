# Airflow migration for DST automation (`D:\DST\automation`, branch `airflow-migration`)

Migrating `D:\DST\automation`'s orchestration (currently `scheduler.py`, a homegrown
`schedule`-library polling loop + threading + subprocess watchdog) to Apache Airflow.
`pipeline/*.py` business logic (analyze/analyze_itn/cdd_sync/cdd_sync_itn/report/
report_itn/notify/config) is NEVER modified — only the orchestration layer changes.
As of this writing: fully built and live-tested against the `central`/`localtest`
groups on a local Airflow 3.1.6 deployment; **not yet committed** (working tree on
`airflow-migration`, branched from `ITN` not `main`, since `ITN` has the ITN/LLIN
module support needed for the Chad LLIN row).

## Feasibility verdict (before building)

Given the real scale (~4-6 tenants, 4 report_times/day, ~16-24 runs/day total),
Airflow is technically feasible but genuinely overkill relative to the ops burden
of standing it up (webserver+scheduler+executor+metadata DB) vs. the homegrown
watchdog. User chose to proceed anyway (learning/tooling reasons). See
[[project_dst_automation]] memory for the full write-up. Cheaper alternatives that
would have gotten most of the reliability benefit without new infra: `tenacity`
retries + wiring `run.py`'s existing `_slack_error()` alert into `scheduler.py`'s
failure path too (today only `run.py`'s manual path alerts on failure).

## Local deployment facts

- `D:\DST\airflow` = pure runtime/deployment dir (docker-compose.yaml, config,
  logs, plugins) — **no source code lives here**. Airflow 3.1.6, Docker Compose,
  CeleryExecutor (scheduler/dag-processor/worker/triggerer/apiserver/postgres/
  redis, all separate containers — Airflow 3 splits DAG parsing into its own
  `dag-processor` service). The original `docker-compose.yaml` had been deleted
  on disk but containers were still running from it — recreated from the
  official `apache/airflow:3.1.6` template and modified (see below), then
  `docker compose up -d` cleanly adopted/recreated the existing containers.
- `D:\DST\automation` (the whole repo) is mounted at
  `/home/jovyan/dst/dst/automation` inside every Airflow container — deliberately
  the SAME absolute path the Sheet's `out_dir`/`target_csv` columns already
  assume on production JupyterHub hosts, so `pipeline/config.py`'s path handling
  (absolute `out_dir`, relative `target_csv`) works completely unmodified. Also
  sets `PYTHONPATH` to that path so `from pipeline import ...` resolves.
- `AIRFLOW__CORE__EXECUTE_TASKS_NEW_PYTHON_INTERPRETER=true` — every task
  instance runs in a brand-new interpreter process rather than a reused
  Celery/Local worker process. This is the actual mechanism (not just a nice-to-
  have) preventing one tenant's env vars (ES creds, Slack token, ...) from
  leaking into another tenant's task when many tenants share one Airflow
  deployment. Confirmed this matters for ANY executor with pooled/prefork
  workers (LocalExecutor, CeleryExecutor), not just Celery.
- Local ES test data: the `dev` tenant / `Local Test SMS_CONFIG_2025` campaign
  (`campaignNumber=CMP-2025-05-09-005779`) already existed in the local
  Elasticsearch container from prior HCM testing sessions (see
  [[hcm_working_payloads_dev]]), dated 2023-08-23. Rather than fabricate new
  synthetic data, bulk-updated it via ES `_update_by_query` painless scripts
  (`ctx._source.Data.taskDates = params.d`, etc.) to shift `taskDates`/
  `syncedDate`/`@timestamp` to "today" so the pipeline's date-range query picks
  it up. `individual-index-v1`/`project-beneficiary-index-v1` don't exist
  locally (404) — non-fatal, just inflates DQ "missing name" warnings.

## Tenant grouping — by credential set, not by country (important, non-obvious)

Discovered by actually querying the live Sheet rather than assuming: the
"central" grouping is NOT "Nigeria states" vs "everyone else" — it's **grouped by
which ES credentials a tenant needs**. Confirmed via `gspread`: the shared
spreadsheet (`1MspTor...`) has tabs `Nigeria States` (17 rows: ba/ke/ko/na/oy/ab/
pl/bo/so/zf + BOTH a `ch` SPAQ-non-admin row and a `chad` LLIN-admin row),
`taraba`, `togo`, `Run Log`, `localtest`. Bauchi's own `.env.bauchi` file has a
STALE `GOOGLE_SHEET_TAB=Sheet1` line (the tab was renamed to `Nigeria States`
since) — `.env.bauchi` is labeled "Nigeria central instance" and is in fact the
one deployment covering all 12 tenants on that tab, Chad included, despite Chad
not being a Nigeria state. Diffing `.env.bauchi`/`.env.taraba`/`.env.togo`: all
three share the exact same `ES_URL` host
(`elasticsearch-data.es-cluster-v8:9200`) and the exact same `GOOGLE_SHEET_ID` —
Taraba and Togo were only split into their own tab+`.env` because they need
distinct `ES_USER`/`ES_PASS` (and Togo additionally needs its own `ES_INDEX_PREFIX`
convention + Groq key), not because of any network/country boundary. New states
get added as rows to the existing shared tab; a tenant only gets pulled into its
own tab+env when its ES login genuinely differs.

## Architecture v1 (built, live-tested, then superseded)

One Airflow DAG dynamically generated per `(group, tenant, time, mode)` trigger
slot (e.g. `dst_central_Sokoto_1700_both`) via a `dst_dag_factory.py` reading a
per-group JSON cache (`dst_meta_refresh_dag.py`, a separate 5-min DAG, wrote the
cache — Sheet reads must never happen in DAG-file top-level code, since Airflow
re-executes that ~every 30s x `parsing_processes`, not just every
`dag_dir_list_interval`). Per-tenant secrets via one Airflow Variable per group
(`dst_secrets_<group>`, JSON) + a `tenant_env(group)` context manager (snapshot/
apply/restore `os.environ`) as belt-and-suspenders on top of
`execute_tasks_new_python_interpreter`. Per-tenant concurrency guard via one
Airflow Pool per tenant (`dst_pool_<tenant>`, 1 slot) — replaced scheduler.py's
old per-tenant `threading.Lock`.

**Bug found and fixed in this version:** added `TaskGroup`s (`extract`/`publish`)
for Airflow-UI readability — this silently renamed `check_active` to
`extract.check_active` etc., breaking every hardcoded
`ti.xcom_pull(task_ids="check_active")` string lookup used to gate the
`trigger_rule=ALL_DONE` tasks (`run_report`, `cdd_sync_alert`). Every run started
failing at `run_analyze` with "check_active did not succeed." **Approach:**
recognized that most of those manual pulls were unnecessary in the first place —
for a normal default-`trigger_rule=ALL_SUCCESS` edge, Airflow's own cascade
already guarantees a task's body only runs if its upstream genuinely succeeded,
so those edges can just check the upstream's marker-dict argument directly (no
task_id string needed at all). Only the genuinely `ALL_DONE`-gated edges
(`run_cdd_sync -> run_report`, `run_cdd_sync -> cdd_sync_alert`) truly need a
manual `ti.xcom_pull(task_ids=...)`, and those now use string constants
(`EXTRACT_GROUP`, `PUBLISH_GROUP`) imported by both the task module and the DAG
factory so the group name and the lookup can never drift apart again.

This version is superseded (see below) but the credential-isolation mechanism,
the marker-dict protocol, and the professional-hardening additions
(`execution_timeout` per task, `dagrun_timeout`, `max_consecutive_failed_dag_runs`
auto-pause after 3 consecutive failures, `owner`/`description`/`doc_md`) all
carried forward unchanged into v2.

## Architecture v2 (current) — Scheduler + Processor DAG pattern

Prompted by a real production architecture doc the user shared,
`HCM Reports Automation using Airflow.docx` (a DIFFERENT eGov HCM reporting
system, already live on Kubernetes/KubernetesExecutor, config via DIGIT MDMS,
storage via FileStore, audit via Kafka+Persister->Postgres). **Decision:** adopt
only the DAG-architecture pattern (Scheduler+Processor split, a real Postgres
audit table, their retry-policy distinction) — keep the Google Sheet as config
(zero new infra) rather than migrating to MDMS/Kafka/FileStore, and stay on
CeleryExecutor/Docker-Compose rather than jumping to KubernetesExecutor.

Two DAGs replace the whole per-slot DAG-factory mechanism, eliminating the
cache-JSON indirection entirely:
- **`dags/dst_campaign_scheduler.py`** — runs every 5 min. One task per active
  group (`find_due_<group>`, never a loop over groups inside one task — same
  credential-isolation reasoning as before) reads that group's Sheet tab, and
  for each active row's `compute_trigger_slots(row)` result, checks whether the
  `(time, mode)` falls in the current window. A `flatten` task (pure data
  reshaping, touches no credentials) merges every group's due-list, feeding a
  **dynamically mapped** `TriggerDagRunOperator.partial(trigger_dag_id=
  "dst_dynamic_campaigns", skip_when_already_exists=True).expand_kwargs(all_due)`
  — the number of triggers isn't known until runtime, so this uses Airflow's
  dynamic task mapping (`.expand_kwargs()`, confirmed available on
  `TriggerDagRunOperator.partial()` in 3.1.6) rather than a fixed static
  operator. `all_due` items are `{"conf": {"group":, "row":, "mode":},
  "trigger_run_id": f"{group}_{tenant}_{mode}_{date}_{time_slug}"}` —
  deterministic `trigger_run_id` + `skip_when_already_exists=True` together
  dedupe an overlapping tick re-matching the same slot (their reference doc's
  "dedupe by (identifier, reportName, triggerTime)" rule, adapted).
- **`dags/dst_dynamic_campaigns.py`** — a SINGLE static DAG, `schedule=None`
  (trigger-only). Every task reads `group`/`row`/`mode` straight from
  `dag_run.conf` (via the `dag_run` parameter — a known TaskFlow context key
  Airflow auto-injects, confirmed via `airflow.utils.context.KNOWN_CONTEXT_KEYS`)
  rather than having them threaded through as XCom arguments between tasks.
  Task chain unchanged in substance: `check_active -> run_analyze ->
  run_cdd_sync -> run_report -> run_notify -> finalize`.

**Critical Airflow-3 gotcha found while testing the window-matching (confirmed
empirically, not assumed):** a plain cron-string `schedule="*/5 * * * *"` on
Airflow 3 defaults to `CronTriggerTimetable`, which gives a **zero-width**
`dag_run.data_interval_start == dag_run.data_interval_end == trigger time` — NOT
the `[previous_tick, this_tick)` interval Airflow 2's `CronDataIntervalTimetable`
produced. First `find_due` implementation used `data_interval_start/end` as the
window and always returned `[]` even for a trigger time that should have
matched. **Approach that settled it:** directly queried a real `DagRun` row via
SQLAlchemy (`session.query(DagRun).filter(...).first()`) and printed
`data_interval_start`/`data_interval_end` — both showed the identical timestamp,
proving the zero-width hypothesis rather than a logic bug in the matching code.
**Fix:** switched to wall-clock `datetime.now(timezone.utc)` with a 5-minute
lookback (`window_start = now - timedelta(minutes=5); window_end = now`) — which
is also what the reference doc's own Section 4.2 actually does
("`window_start = now_utc - 1 hour + 1 minute`"), so this was simply following
their pattern more literally instead of over-engineering with Airflow's
timetable semantics.

**Per-tenant concurrency guard — Postgres lock replaces the Airflow Pool.** A
single static Processor DAG serving every tenant via dynamic `conf` cannot use a
static per-task `pool=f"dst_pool_{tenant}"` — `pool=` is fixed at DAG-definition
time, identical for every run of that DAG, but which tenant is running is only
known at trigger time. Confirmed `psycopg2` is already available in the Airflow
image and can reach the same Postgres instance backing Airflow's own metadata DB
directly from task/worker-container code
(`postgresql://airflow:airflow@postgres/airflow`, no new secret — same trusted
internal service already in the compose file). Two new tables, created once via
plain `CREATE TABLE IF NOT EXISTS` (`docker exec airflow-postgres-1 psql ...`):
- `dst_tenant_locks(tenant PK, locked_at, dag_run_id)` — `check_active` does
  `INSERT ... ON CONFLICT (tenant) DO NOTHING`; `cur.rowcount==1` means it won
  the lock, `0` means another run holds it (routine no-op, not a failure — same
  semantics as `scheduler.py`'s old non-blocking `threading.Lock().acquire()`).
  `finalize` (`trigger_rule=ALL_DONE`, wrapped in try/finally) always releases
  via `DELETE WHERE tenant=X AND dag_run_id=Y` — scoping the delete to `Y` means
  a stray/duplicate release from the WRONG run can never clear another run's
  active lock.
- `dst_report_metadata(tenant, state_name, dag_run_id, mode, day, status,
  step_failed, error, drive_link, created_time)` — adapted from the reference
  doc's `hcm_report_metadata`. `finalize` writes one row per REAL report
  attempt, explicitly skipping the routine inactive-day no-op (matches their
  "one record per generated report" semantics) — closes the "no persisted run
  history beyond log files" gap identified during the feasibility check.

**Verified live** (deliberately triggered two concurrent `dst_dynamic_campaigns`
runs for the same tenant via `airflow dags trigger --conf ... --run-id ...`):
first run did the real work end to end (~70s, ES/Groq/Drive/Slack all real);
second run's `check_active` logged "another run already in progress for
tenant=dev — routine no-op" and completed in ~5s with no audit row written,
exactly as designed. First run's audit row and lock-release both correct.

**Marker-dict protocol** (unchanged from v1, the core correctness mechanism
throughout): every task returns `{"ok": True, ...}` (real success), `{"ok":
None, ...}` (routine no-op — tenant inactive, outside campaign window, lock
held elsewhere), or genuinely raises (no XCom pushed at all, raw `None` on
`xcom_pull`). Default-`trigger_rule` edges check the upstream's marker directly
(argument-passed, no XCom-pull-by-name); only the two truly `ALL_DONE` edges use
`ti.xcom_pull(task_ids=...)`. This distinction — and NOT relying on Airflow's
skip/fail state cascade for the ALL_DONE edges — is what avoids misreporting a
routine "tenant inactive today" run as a pipeline failure.

**Retry-policy distinction** (their Section 11.2: "retry only for
infrastructure issues... no retries for script logic errors"): `_run_stage()`
catches `(KeyError, ValueError, TypeError, AttributeError)` from
`mod.run(cfg)` and re-raises as `AirflowFailException` — which marks the task
FAILED immediately with **no retries**, regardless of the task's configured
`retries=2`, since those error types indicate malformed data/config that will
fail identically on retry. Genuine `requests`/HTTP/connection errors propagate
normally and use the default retry policy.

## Folder structure (final)

```
D:\DST\automation\
├── dags/
│   ├── dst_campaign_scheduler.py
│   ├── dst_dynamic_campaigns.py
│   └── common/
│       ├── env.py    (tenant_env, group_secrets, ACTIVE_VALUES)
│       ├── db.py     (acquire/release_tenant_lock, write_audit_row — psycopg2)
│       └── tasks.py  (check_active/run_analyze/run_cdd_sync/run_report/
│                       run_notify/finalize, dst_failure_slack_alert)
├── pipeline/          (UNCHANGED except pipeline/schedule_utils.py — new,
│                        pure extraction of the both/internal/partner dedup
│                        logic from scheduler.py's `_reload_schedule`, so the
│                        old and new orchestrators can't drift on it)
└── scheduler.py, run.py, ...   (old orchestrator, kept during transition;
                                  scheduler.py updated to import
                                  compute_trigger_slots instead of inline logic)
```
The old `airflow_dags/` folder (v1) and its `dst_meta_refresh_dag.py`/
`dst_dag_factory.py` were deleted once v2 was confirmed working; stale DAG
metadata (`airflow dags delete -y <id>`) and the now-unused `dst_pool_<tenant>`
Pools (`airflow pools delete`) were cleaned up too.

## Operational note: secrets and the auto-mode classifier

Commands embedding real credentials inline (e.g. `docker exec ... airflow
variables set dst_secrets_central '{"...ES_PASS":"real-password"...}'`) get
blocked by the auto-mode classifier as a sensitive-write guard. Correct
response is NOT to route around it via a different tool/mechanism — write the
JSON to a local file (Downloads, not the repo) and have the user paste it into
the Airflow UI (Admin -> Variables) themselves. Non-secret CLI actions (Pools,
non-sensitive Variables, DAG unpause/trigger, `CREATE TABLE`) are unaffected and
fine to run directly.

Relates to [[project_dst_automation]], [[hcm_working_payloads_dev]],
`dst/automation-and-structure.md` (the pre-Airflow pipeline this replaces the
orchestration layer of).


---

# Architecture v3 (CURRENT, 2026-08-14..17) — branch `airflow`, no database, universal DST_MODE

Supersedes v2 above. Committed on branch **`airflow`** (`2ea7edc` pipeline
modularization incl. the numericise_ignore fix, `ecc7fc8` DAGs) plus an
uncommitted working tree carrying the no-DB rework, DST_MODE, the MDMS mirror,
Kafka events and the platform/ artifacts. The pipeline modularization
underpinning this (core/ package, collect->checkpoint->render stages,
self-explanatory renames) is documented in automation-and-structure.md.

## The three DAGs (dags/ in D:\DST\automation)

- **dst_campaign_scheduler** (*/5 min) — stateless: every tick re-reads config
  (sheet or MDMS per DST_MODE), computes slots via the shared
  pipeline/schedule_utils.compute_trigger_slots (both/internal/partner dedup,
  also used by legacy scheduler.py so orchestrators cannot drift), matches
  against a WALL-CLOCK window (now-60min, now] (DST_LOOKBACK_MINUTES), and
  triggers one run per due slot with deterministic run-id
  dst_{group}_{tenant}_{mode}_{date}_{hhmm} + skip_when_already_exists=True
  (a slot fires exactly once, ever). Retime guard: sheet mode reads today's
  Run Log rows (common/run_history.build_retime_guard, one sheet read/tick;
  a SUCCESS at-or-after the slot time skips a backwards-retimed slot; FAILED
  rows do not count, so a retimed slot can replace a failed report); mdms mode
  has NO guard (zero sheet/DB on that path — accepted; the platform system
  has none either).
- **dst_campaign_run** (trigger-only, TWO tasks) — execute_campaign_pipeline
  runs the whole chain in ONE task via common/campaign_runner.execute_campaign
  (pod-local files on k8s; data errors -> AirflowFailException no-retry, infra
  errors -> retries=2; cdd_sync failure = "degraded", never fatal; checkpoints
  upload to Drive temp/ in a finally, even on failure). finalize_run
  (ALL_DONE) records the outcome per DST_MODE and RE-RAISES on failure.
- **dst_config_sync** (*/10 min) — the sheet->MDMS listener (below).

dags/common/: slots.py (pure matching), deployment_env.py (groups + secrets +
resolve_dst_mode), run_history.py (sheet retime guard), campaign_runner.py,
dst_kafka_status.py, alerts.py (incl. send_slack_warning). Tests:
tests/test_scheduling.py (13) + tests/test_mdms_sync.py (17), plain python.

## NO DATABASE, ANY MODE (decision 2026-08-17)

Problem: v2 used Airflow's Postgres for a tenant lock + audit/retime-guard
table; platform environments will not grant DB read/write to our code, and the
user ruled sheet mode must be purely Google Sheet too. Approach: audited what
each table actually bought. Root cause of the lock being removable:
deterministic run-ids already make same-slot duplicates impossible; a
cross-slot same-tenant overlap is rare (slots hours apart, runs ~2 min),
worst-case one duplicate Slack post — and the platform's production system
runs with NO lock at all. Fix: tenant_lock.py deleted; audit lives on the Run
Log tab (sheet mode, now with a 10th Mode column; run_log.fetch_today_runs
reads it back for the guard) or as Kafka events (mdms mode). Run DAG went
3 tasks -> 2.

## Universal DST_MODE flag (deployment-wide env var, never per group/DAG)

| | DST_MODE=sheet (default) | DST_MODE=mdms |
|---|---|---|
| Config read | Google Sheet tab | MDMS mirror, sheet fallback on outage |
| Run history | Run Log tab (+Mode col) | Kafka -> persister -> dst_report_metadata |
| Retime guard | Run Log read | none |
| Lock | none | none |
| DB access | zero | zero |

Resolved by deployment_env.resolve_dst_mode() (unknown value -> warn + sheet).

## Sheet -> MDMS mirror (dst_config_sync + pipeline/mdms.py)

Sheet stays the ONLY human surface (tabs = credential groups, rows = tenant
campaigns — Bauchi is just a row; two simultaneous campaigns of one tenant are
two rows with different campaign_numbers). The listener is a POLL (Google
cannot safely push into the eGov gateway): read tab -> plan_sync (PURE diff)
-> apply_sync (HTTP). Identity = tenant::campaign_number (projectTypeId
fallback, then state_name) — mirrors their UUID-detection convention. One MDMS
root tenant per environment; entry data = {rowIdentity, deploymentGroup,
row verbatim}, schema x-unique rowIdentity (artifact:
dags/mdms_schema/dst-campaign-report-config.schema.json).

plan_sync edge cases (all unit-tested): new identity -> create; changed row or
previously-deactivated -> update (reactivates); identity gone -> deactivate
(isActive=false, never hard-delete); INVALID row -> rejected with reasons and
the existing entry KEPT as last-known-good (a typo must not kill a running
campaign); EMPTY sheet read -> deactivation skipped entirely (a transient
read must not wipe the mirror); duplicate identity -> first row wins; other
tabs' entries never touched; changed campaign_number = create + deactivate
(it IS a new campaign); partner-only rows (report_times empty,
partner_report_times set) are VALID — an early validate_row draft rejected
them, caught in the edge-case audit. Rejections nag Slack every sync tick
until fixed (intentional). MDMS writes use a dummy authToken — internal
service-to-service calls bypass the gateway (their own scheduler sends
authToken:"" — hcm_campaign_scheduler.py line 139).

## Kafka lifecycle events (mdms mode) + how THEIR system uses Kafka

Their system, read from code (richer than their docx): ONE topic
save-hcm-report-metadata (tenant-prefixed on central instances), THREE
producers (scheduler DAG: SCHEDULED; processor DAG: TRIGGERED/SKIPPED/
POD_INFRA_FAILED; report pod main.py: POD_STARTED..REPORT_COMPLETED), ONE
schema — an EVENT-SOURCED lifecycle log ranked by status_order
(10/20/30-33/40-terminal), trigger time stamped ONCE and carried through conf
so seconds_since_triggered is consistent across a run's events; lazy singleton
producer that NEVER raises; consumer = egov-persister ->
hcm_report_metadata. Their code never WRITES MDMS — config entries are
created by humans via the API; the sheet-listener is entirely our addition.

Ours: dags/common/dst_kafka_status.py (same conventions), topic
save-dst-report-metadata, events REPORT_COMPLETED/REPORT_FAILED from finalize
(SCHEDULED/RUN_STARTED reserved in STATUS_ORDER). Platform onboarding kit in
platform/: dst-report-metadata-persister.yml (egov-persister config),
dst_report_metadata.sql (event-sourced table + 2 indexes), README.md (env
vars + flow). kafka-python must go into the deployment image requirements.

## Airflow 3 gotchas found by LIVE testing (would have hit k8s too)

1. Problem: find_due failed "GOOGLE_SHEET_ID not set" though Variables were
   set. Misleading first read: env plumbing. Root cause: Airflow 3 FORBIDS
   task code from ORM access — airflow.models.Variable.get raises inside
   tasks; a silent except fell back to a default group with no secrets.
   Fix: try airflow.sdk.Variable first (task context), ORM fallback for
   CLI/parse contexts; empty-secrets fallback warns loudly.
2. Problem: next failure "Could not parse SQLAlchemy URL:
   airflow-db-not-allowed:///". Root cause: the Airflow 3 task runner
   OVERWRITES AIRFLOW__DATABASE__SQL_ALCHEMY_CONN with that sentinel inside
   tasks. (v3 later removed DB use entirely, mooting it — but the sentinel
   fact matters for any future task-side DB idea.)
3. Problem: a brand-new DAG started ticking immediately despite
   dags_are_paused_at_creation. Root cause: the dag_id existed from July's v2
   test — a same-named DAG RESURRECTS with its old unpaused state. Fix: pause
   before configuring; deployment runbook item.
4. Problem: DAG run showed success though execute FAILED. Root cause:
   finalize (ALL_DONE) was the only leaf and succeeded at recording the
   failure — leaf state defines run state, blinding
   max_consecutive_failed_dag_runs. Fix: finalize re-raises after recording.
5. skip_when_already_exists + deterministic run-ids proven live: the next
   tick re-matched the fired slot and the trigger task went "skipped".

Live E2E (2026-08-14, localtest tab, dev-tenant ES fixtures date-shifted via
_update_by_query): full chain green — Drive tree auto-created
Localtest/Dev/<campaign>/Day 1 (+temp/ with checkpoint JSON; log showed
"uploaded:" then "updated:", proving overwrite-in-place), Groq narratives,
both Slack posts, Run Log + audit rows, ~90s. A later run validated the
failure path end to end when local ES was down (retry x2 -> Slack alert ->
FAILED Run Log row naming the step).

## Deploy tree + k8s direction

Reference repo cloned sparse at D:\DST\repos\health-campaign-services
(branch airflow-analysis, utilities/ only). Our deployment artifact lives on
its local branch dst-daily-automation as ADDITIVE-ONLY untracked files:
utilities/dst-automation/ (pipeline code, NO secrets — the repo is PUBLIC)
and utilities/airflow-dags/dags/dst_*.py + dst_common/ (separate package,
imports rewritten, schedule_utils vendored so parsing is self-contained;
their common/ untouched). Their git-sync reads ONE configured branch —
merging into it deploys within ~60s, hence PR-only, never direct commits.
k8s plan: pod-per-run via KubernetesPodOperator + a dst image (their
hcm-custom-reports/Dockerfile is the template; image tag via a
REPORT_IMAGE-style Airflow Variable; GitHub Actions build like
build-hcm-custom-reports.yml); heavy tasks (sheet/MDMS reads, the pipeline)
must run in OUR image — their Airflow image lacks our Python deps. Their
committed REPORTS_GENERATION/.env holds only internal URLs, not credentials
(verified before flagging).

## Ops runbook facts

- dst_groups Variable: on the local box keep it LOCALTEST ONLY —
  dst_secrets_central exists there, and listing central would fire REAL
  campaign runs off the Nigeria States tab.
- DagBag validation without a running Airflow: docker run --rm
  --entrypoint bash -v <repo>:/repo:ro apache/airflow:3.1.6 -c
  "export PYTHONPATH=/repo:/repo/dags && python -c 'import dst_...'".
  The Windows pip Airflow install is broken (airflow.sdk ImportError) —
  always validate in the container.
- Local fixtures: shift the dev campaign's taskDates/createdTime to today via
  _update_by_query before any E2E (8 task + 8 sync docs).
- Local compose has an EMPTY Fernet key -> Airflow Variables (incl. the
  dst_secrets_*) are unencrypted at rest; prod needs a real key or a secrets
  backend.
- mdms-mode go-live still needs: live schema create + first sync against a
  real MDMS_URL; kafka-python in the image; persister yml + table applied.

---

# Architecture v4 (2026-08-21) — ONE Airflow Variable, and 9 defects found by running it

Supersedes the v3 configuration model. Branch `airflow`, local stack
`D:\DST\airflow` (Airflow 3.1.6 + postgres16 + redis + external `elasticsearch`
and `kafka` containers, all on network `airflow_default`).

## Why one Variable: the hosted deployment cannot set env or mount files

The hosted (Bauchi central) Airflow is not ours — no Helm values, no pod spec, no
image. The ONLY writable surface is Admin -> Variables. So every piece of config
must be reachable from a Variable, including the Google service account, which is
normally a file. New module `dags/common/dst_config.py`, Variable `dst_config`:

    {"sheet_tab": "Nigeria States",
     "env":     {"ES_URL": ..., "ES_INDEX_PREFIX": null, "GOOGLE_SHEET_ID": ...,
                 "GROQ_MODEL": ..., "DST_MDMS_ENABLED": "false",
                 "DST_LOOKBACK_MINUTES": "60", "CDD_ROLE": ...},
     "secrets": {"ES_USER": ..., "ES_PASS": ..., "SLACK_TOKEN": ...,
                 "GROQ_API_KEY": ...},
     "google_credentials_json": { ...service account... }}

Precedence: process env < env < secrets. `apply()` is a context manager wrapping
the WHOLE task body — not just `group_environment` — because three consumers read
config OUTSIDE the per-group context and would otherwise never see it:
`mdms_enabled()` (scheduler + finalize), `push_run_event()` (run_history:99) and
`alert_channel()` (the on_failure_callback, which runs AFTER os.environ is
restored — so `alerts.py` calls `dst_config.resolved()` directly).
`google_credentials_json` is materialised 0600 under the temp dir and DELETED in
the finally, so a private key never outlives its task. Absent Variable = legacy
`dst_groups` + `dst_secrets_<name>` path, so local and JupyterHub runs are
unaffected.

Bauchi central is ONE credential set for ~12 tenants on the `Nigeria States` tab
— tenants are ROWS, not groups — so the `dst_groups` array collapses to a single
entry via `dst_config.as_group()`. Groups only earn their keep when Taraba/Togo
(different ES logins) join the same Airflow.

Env audit: 30 distinct config keys, ALL read at task runtime except
`DST_LOOKBACK_MINUTES` (module level in the scheduler — moved inside the task,
because a Variable can never reach a parse-time read).

## Defect 1 (HIGH) — a mounted .env silently changed which ES index was read

**Problem:** the first live run published a complete report — Word narrative,
Excels, Slack post, task SUCCESS — describing ZERO records. **Misleading first
read:** "nothing seeded / wrong campaign window"; two lines later
`[report] cumulative trajectory from ES: 4 days (treated 73)` proved data WAS
reachable. **Approach:** compared `config.build()` inside the container against
the container env — `printenv ES_INDEX_PREFIX` unset (as compose intends) yet
`cfg["ES_INDEX_TASK"]` came out un-prefixed; then printed
`os.environ.get("ES_INDEX_PREFIX")` before and after `import pipeline.config`:
None before, `''` after. **Root cause:** the repo is bind-mounted into the
containers and `config.py`'s `load_dotenv(repo/.env)` supplied the
`ES_INDEX_PREFIX=` line that exists for TOGO. `override=False` protects variables
that already EXIST; it cannot protect a variable the deployment intends to be
ABSENT — and absent-vs-empty is exactly what selects tenant-prefixed vs
un-prefixed indices. It even defeated the documented remedy: a
`{"ES_INDEX_PREFIX": null}` override in `dst_groups` removes the key, but
`group_environment` runs BEFORE `pipeline.config` is imported, so the load put it
straight back. The run queried `project-task-index-v1` (90 stale dev docs)
instead of `so-project-task-index-v1` (84 seeded), matched 0, and reported
success. **This only began biting once the dotenv PATH was fixed** the previous
session; while it pointed at the non-existent `pipeline/.env` the load was a
silent no-op. **Fix:** `config._dotenv_is_local_only()` skips the repo `.env` when
Airflow owns the environment (`AIRFLOW_HOME` / `AIRFLOW_CTX_DAG_ID` /
`AIRFLOW__CORE__EXECUTOR` present), overridable via `DST_LOAD_DOTENV`; declared
explicitly as `DST_LOAD_DOTENV=false` in the stack `.env` and compose.
**Lesson: a `.env` sitting next to the code will eventually override the
deployment that thought it was in charge.**

## Defect 2 (HIGH) — the primary deliverable never reached Drive, on every run

**Problem:** `upload_file failed (non-fatal): <HttpError 500 ... uploadType=resumable>`
on `performance_dayN.xlsx`, three runs out of three, while the CDD workbook, Word
report and chart uploaded fine seconds later. Run SUCCESS, Drive link empty.
**Root cause:** Drive's resumable endpoint 500s unpredictably and in practice hits
the FIRST upload of a run; the performance Excel is always first, and
`upload_file` had no retry and swallowed the failure. **Fix:** 3 attempts with
backoff, permanent 4xx not retried, and `notify.FAILED_UPLOADS` records anything
still lost so `campaign_runner` marks the run
`degraded: not uploaded to Drive (...)`, which alerts.

## Defect 3 (MEDIUM) — CDD_ROLE was documented but never read

**Problem:** standing up Togo required hand-swapping `DISTRIBUTOR` ->
`COMMUNITY_DISTRIBUTOR` at NINE sites in `cdd_sync.py` on every pipeline sync — a
swap the notes record as lost and re-applied more than once. **Root cause:**
`deployment_env.py:12` advertised a `CDD_ROLE` group env key; nothing ever read
it. `cdd_sync_itn.py` likewise hardcoded `DISTRIBUTOR_REGISTRAR` as a module
constant in 5 places. **Fix:** `_cdd_role()` reading `CDD_ROLE` (default
`DISTRIBUTOR`) and `CDD_ROLE_ITN` (default `DISTRIBUTOR_REGISTRAR`). TWO keys
deliberately: one deployment runs both campaign types at once — Bauchi has had SMC
and ITN live in the same window — so a single shared key would silently break one.

## Defect 4 (MEDIUM) — one failure, two Slack alerts

`default_args={"on_failure_callback": ...}` applies to every task, so
`execute_campaign_pipeline` alerted and then `finalize_run` — which re-raises
purely to mark the run failed — alerted again. Verified live: 2 alerts per
failure, the fastest way to get an ops channel muted. **Fix:** `finalize_run`
declares `on_failure_callback=None`; its re-raise is bookkeeping, not news.

## Defect 5 (MEDIUM) — a 404 index_not_found retried for 6 minutes

A tenant with no ES indices failed
`404 Client Error ... /ba-project-task-index-v1/_search` after THREE attempts. A
wrong index name, tenant prefix or credential cannot be fixed by waiting.
**Fix:** `_is_permanent_http()` in `campaign_runner` -> 400/401/403/404 raise
`AirflowFailException`. **Sub-lesson:** the first version of this fix placed
`except Exception` BEFORE `except DATA_ERRORS`, so ValueError/IndexError stopped
failing fast — the existing suite caught it in seconds (cases 67/68). Handler
order is load-bearing: TRANSIENT_ERRORS -> DATA_ERRORS -> generic+HTTP.

## Defect 6 (MEDIUM) — eight kinds of malformed sheet config failed SILENTLY

`config._validate_row()` (active rows only — an inactive row with contradictory
dates is nobody's problem) now rejects these, naming FIELD, VALUE, consequence
and fix, because the message travels into the Slack alert:

- `campaign_end` before `campaign_start` — previously never in window, so no
  report ever ran and nothing warned
- `campaign_days` 0 or negative — it is the DIVISOR for every coverage %: 0 gives
  ZeroDivisionError mid-run after two pointless retries, negative inverts every
  figure
- unparseable or pre-start `mopup_end_date`
- non-numeric `cycle_index` — ES matches `cycleIndex` EXACTLY, so `'O2'` with a
  letter O yields a silently EMPTY report
- unknown `drug_type` — a typo'd ITN silently ran the SPAQ per-child pipeline
- unrecognised `is_admin_console` — silently selects the wrong ES query shape

Non-numeric `campaign_days` still falls back to 4 rather than raising, and
`end == start` is a legitimate single-day campaign.

## Defect 7 (LOW) — an ACTIVE row with no times never reports, silently

`slots._warn_if_never_schedulable()` warns once per tick, naming the campaign.

## Defect 8 (LOW) — the retime guard compared slot times as STRINGS

Sheets strips a leading zero, so a slot written `07:24` reads back `7:24`, and
`"7:24" > "07:24"` lexically — a single-digit hour sorts after every two-digit
one, inverting the guard's `>=`. **Fix:** normalise through `parse_report_times`
before comparing (`run_history.py`).

## Defect 9 (fixture, not product) — "4 CDDs, 4 never synced"

`seed_local_es.py` wrote sync docs with `syncedDate`/`syncedTimeStamp` only, but
`cdd_sync._load_sync_dates_by_username` matches on FOUR fields: `Data.taskDates`
(NOT syncedDate), `syncedUserName`, `role`, `campaignNumber`. The aggregation
matched nothing, so every report said 4 of 4 never synced while the seed intended
3 of 4 — **a fixture bug that looked exactly like a pipeline bug.** Keep the
seed's sync fields in step with `cdd_sync.py`.

## Also corrected

- `drive_token.json` is DEAD. `notify.py:27` and `core/drive.py:24` both call
  `config._resolve_creds_path()` — the SERVICE ACCOUNT. Nothing imports the OAuth
  token. README was stale in 6 places (including a "must NOT be swapped" warning
  and an impossible troubleshooting entry); `utils/gen_drive_token.py` deleted.
  The pipeline needs exactly ONE credential file: `credential.json`.
- Fernet: local compose now reads `AIRFLOW_FERNET_KEY` from the stack `.env` (was
  hardcoded `''`). Verified `is_encrypted=t`. **Do not put secrets in Variables
  before a real key exists** — the process-env approach is safer than an
  unencrypted Variable row in the metadata DB.
- STALE FACT CORRECTED: `dst_secrets_central` no longer exists on the local box
  (the Variables table held only `dst_groups`). Keep config scoped to localtest
  for the original reason — the sheet id decides what a `central` group would read
  — not that one.
- `_PIP_ADDITIONAL_REQUIREMENTS` reinstalls ~10 MB of wheels on every recreate at
  ~77 kB/s, and the apiserver healthcheck fails during that window. Compose's own
  comment says to bake an image; this is the concrete cost of not having done it.

## Test assets (this session)

`dst_mode_tests.py` — 124 assertions over the REAL surfaces (actual sheet tab,
actual Airflow, actual containers): `sched` 20, `config` 8, `e2e` 13,
`concurrent` 14, `change` 9, `cumulative` 21, `infra` 5, `edge` 35. The tab is
snapshotted and restored in a `finally`, and only sheet-using sections touch it,
so an in-process section can run while a live suite holds the sheet. Runs fire
THROUGH the sheet (retime a row to a just-passed slot, let the real 5-minute tick
catch it) because `airflow dags trigger` bypasses the very sheet->scheduler->run
path under test; cumulative is the exception, since its slot is 23:59 on the
mop-up date, so it is triggered with a conf payload.

Proven live: one tick fanning out to 4 concurrent campaigns with distinct run ids
and SEPARATE scratch dirs (`MPLOCAL001c2` vs `MPLOCAL002c3` — the collision that
used to drop the second report); a row retimed AND deleted mid-run does not abort
the in-flight run (conf snapshot); ES stopped mid-run fails loudly and retries
rather than publishing an empty report; a missing target book fails fast in
exactly one attempt with an accepted Slack alert and a FAILED Run Log row.

KNOWN GAP: a `campaign_number` with no ES data yields 0 records and still reports
SUCCESS — a silently empty report. The index-missing case now fails loudly; the
campaign-missing case does not.

## v4 addendum (2026-08-21, later the same day) — MDMS mode VERIFIED end to end

The v3 notes listed as still-needed for mdms go-live: "live schema create + first
sync against a real MDMS_URL; kafka-python in the image; persister yml + table
applied." Three of the four are now PROVEN against real services; the fourth is
blocked by the environment, not by our code.

### PROVEN: Airflow -> Kafka -> egov-persister -> Postgres

Ran against the user's locally running DIGIT stack (egov-persister on 8034
context `/common-persist`, Postgres `pgr` on 5432, Kafka, egov-mdms-service on
8094). Verified, in order:

- `dst-report-metadata-persister.yml` is loaded (it is listed in the persister's
  `egov.persist.yml.repo.path` as `classpath:dst-report-metadata-persister.yml`).
- **Producer/persister contract matches EXACTLY** — all 15 `$.field` json-paths in
  the YAML against the field set `push_run_event` emits, diffed both directions,
  zero drift. Worth re-running as a check after any event-schema change: a
  mismatch would insert silent NULLs, not fail.
- `dst_report_metadata` exists in `pgr` with 16 columns, a UNIQUE index on
  `event_id` (so the YAML's `ON CONFLICT (event_id) DO NOTHING` actually
  de-duplicates) plus campaign and status indexes.
- A real run published `REPORT_COMPLETED` and the persister wrote the row in
  under 5 seconds, every field correct: tenant, state, campaign_identifier,
  cycle_index, deployment_group, mode, day, slot_date, slot_time, status,
  drive_folder_url, timestamp_ms.
- **Exactly one row per run**, and the Run Log tab was NOT written (19 -> 19 rows)
  — mdms mode really does bypass the sheet for history.

Test section: `dst_mode_tests.py mdms`, cases 125-149. Result 23/25; the 2
failures were stale assertions in the test itself (pinned `KAFKA_BROKER` to the
old value, asserted `MDMS_URL` unset after it had deliberately been set), since
corrected. Nothing product-side failed.

### Environment defect (NOT ours): Kafka advertised listener blocked all
### in-container producing

**Problem:** `push_run_event` failed on every run with the audit falling back to
the sheet. **Misleading first read:** "Kafka is unreachable from the container" —
it is not. The worker's own log shows `connected`, `Broker version identified as
4.2`, `Connected` to `kafka:9092`. **Root cause:** one step later in the protocol.
The broker ran with `KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092`, so
its metadata response tells every client "produce to localhost:9092". Inside the
Airflow container `localhost` is the container, where nothing listens. The
persister worked only because it runs on the HOST, where localhost:9092 genuinely
is the broker. **Fix:** a SECOND listener, keeping the first so host clients are
untouched:

    KAFKA_LISTENERS=PLAINTEXT://:9092,INTERNAL://:29092,CONTROLLER://:9093
    KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092,INTERNAL://kafka:29092
    KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,INTERNAL:PLAINTEXT
    KAFKA_INTER_BROKER_LISTENER_NAME=PLAINTEXT

then `KAFKA_BROKER=kafka:29092` for Airflow. Verified by producing from BOTH
sides afterwards. Two traps while doing it: that container keeps its data in
`KAFKA_LOG_DIRS=/tmp/kraft-combined-logs` (NOT the mounted volume), so recreating
it loses every topic — they auto-recreate, but say so before doing it; and
`/opt/kafka/config/broker.properties` inside the image is a STATIC DEFAULT that
does not reflect runtime config, so check `docker inspect` env, not that file.

**Incidentally this proved the Kafka-outage fallback for free**: while broken,
every run logged `failed to push (non-fatal)` -> `falling back to the Run Log tab`
-> `recorded via sheet`. A broker outage never fails a run and never loses the
audit trail. The deliberate broker-stop test is therefore gated behind
`DST_TEST_STOP_KAFKA=1`, since that broker is shared with the DIGIT services.

### BLOCKED: the sheet -> MDMS mirror cannot be tested on a v1-only MDMS

Established from source, not inference:

- `egov-mdms-service` in the user's checkout has ONE controller,
  `@RequestMapping("/v1")` with only `_search` and `_get`. artifactId is
  `egov-mdms-service-**test**` 2.9.0-SNAPSHOT. No `schemaCode` or
  `SchemaDefinition` anywhere in `core-services`; no `src/main/resources/db`
  migrations folder.
- `Digit-Core/build/build-config.yml` shows `mdms-v2` is built FROM
  `core-services/egov-mdms-service` (same work-dir, image name `mdms-v2`, plus an
  `mdms-v2-db` image from `src/main/resources/db`). So v2 lives in that module in
  real DIGIT — this checkout is a trimmed/older variant of it.
- curl evidence: v1 `_search` with a correct v1 body returns a real domain
  response (`{"ResponseInfo":null,"MdmsRes":{}}`); v2 `_create/{schemaCode}` and
  `v2/schema/_create` both return `400 NoResourceFoundException: No static
  resource v2/...`.

**Read "No static resource" as "no controller is mapped there."** Spring only
treats a URL as a filename after route matching fails. If the route existed you
would get a field-level validation error (as the v1 call does), a 401/403, or a
domain error. NOTE egov's exception handler renders this as **400, not 404** — I
initially misread a 400 on `/actuator/health` as "the path exists, bad request".
It does not. Read the source, not the status code.

v1 is also READ-ONLY, so this is a missing capability, not a path or version
string: `plan_sync` produces create/update/deactivate operations and there is
nothing to send them to. `plan_sync`/`apply_sync` therefore remain covered only
by the 13 in-process unit cases. To close it: a v2-capable MDMS (or a stub).

### Defect 11 — MDMS_API_PREFIX: writes ignored the configured path

`search_entries` built its URL from `MDMS_SEARCH_ENDPOINT` (configurable) while
`apply_sync` hardcoded `f"{_base_url()}/mdms-v2/v2/_{action}/{_schema_code()}"`.
A deployment talking DIRECT to the service (context path `/egov-mdms-service`)
instead of through the gateway would have had working searches and every
create/update 404ing — and because apply failures surface as a sync error rather
than a config error, that would have been painful to diagnose. **Fix:**
`_api_prefix()` (env `MDMS_API_PREFIX`, default `/mdms-v2/v2`) now governs BOTH;
`MDMS_SEARCH_ENDPOINT` still wins when set explicitly.

**Production path convention, from HCM's own config** (`health-campaign-services/
health-services/transformer/application.properties`): `egov.mdms.host=...:8099`
and `egov.mdms.search.endpoint=/mdms-v2/v1/_search`. So the deployed service is
named `mdms-v2` on port **8099** and HCM reads it via the v1-compatible endpoint.
Our mirror needs the v2 WRITE API, so keep `MDMS_API_PREFIX=/mdms-v2/v2` there —
the `/egov-mdms-service/v2` form is only for direct-to-service local testing.

### Defect 12 — alert and audit text was not actionable

Three separate problems, all fixed:

1. The degraded alert read `cdd_sync (degraded: produced no sync workbook)` — a
   module name and an internal artifact, with no impact, cause or action. Now:
   `DST REPORT INCOMPLETE — <state> / <campaign> / Day N`, then a bullet per
   problem stating what is MISSING from the report and the likely cause, then an
   explicit Action line.
2. The Run Log Error column read `execute_campaign_pipeline failed — see task
   log`, useless to anyone without Airflow access (on a hosted deployment, that
   is everyone). A raising task pushes no return value, so finalize had nothing
   to write — the execute task now `ti.xcom_push(key="failure", ...)` BEFORE
   re-raising, and finalize records the real exception text.
3. Degradation was hardcoded as `"cdd_sync degraded"` for ANY cause, so a failed
   Drive upload was reported as a sync problem. Naming the wrong culprit is worse
   than naming none. Now derived from the actual stage markers.

### Defect 13 — the Word report upload had no retry at all

Defect 2 (upload retry) only covered `notify.upload_file`, which the EXCELS use.
The Word report — the actual deliverable — and the partner report called
`_upload_to_drive` directly inside a bare `except` that logged a warning: no
retry, no `FAILED_UPLOADS` entry, no degradation. A Drive hiccup meant the Slack
post went out with a missing link and nobody was told. Both now route through
`upload_file`, so `_upload_to_drive` has exactly one caller.

### README restructured for the two flows

`README.md` 272 -> 589 lines. New `## The two flows` (ASCII flow per mode, why
mirror rather than migrate, why Kafka rather than a direct insert, a precise diff
table, and failure behaviour per hop), `## Configuration` (replacing the
env-only section: local `.env` vs the `dst_config` Variable, precedence, why the
repo `.env` is not read under Airflow, multi-credential-set guidance, and the
validation table), `## MDMS mode`, and a full env-var reference. Troubleshooting
gained the three failure modes actually seen: all-zero reports, a green run with
a missing Drive artifact, and the INCOMPLETE alert.

### Tooling gotcha that cost time twice

**Git Bash (MSYS) rewrites anything that looks like a Unix path** when passed to
docker. `docker exec ... /tmp/x` became `C:/Users/.../Temp/x`; `-e
KAFKA_LOG_DIRS=/tmp/kraft-combined-logs` became a Windows path and crash-looped
the broker; `PYTHONPATH=/home/jovyan/...` became `FilesGithomejovyan...`. Use the
PowerShell tool for any docker command carrying container paths, or
`MSYS_NO_PATHCONV=1`. Also: `python -u` for background suites — block buffering
otherwise hides all progress until exit, and piping through `tail` hides it
entirely.

---

## Superseded / extended by the 2026-08-24 audit

Everything above still describes how the pipeline was BUILT. Before changing it,
read `dst/airflow-audit-2026-08-24.md`: three independent reviewers went over
this codebase and found ten distinct silent-success paths, a Jinja template
injection through the Google Sheet into `dag_run.conf`, a fleet-wide auto-pause
kill switch, and eight data-correctness defects that are still OPEN. That file
also lists what was VERIFIED CLEAN, so it is the first thing to check before
"fixing" something here that is already correct.
