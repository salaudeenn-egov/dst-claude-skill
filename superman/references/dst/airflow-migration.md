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
