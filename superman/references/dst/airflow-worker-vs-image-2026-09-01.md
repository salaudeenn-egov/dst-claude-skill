# Worker vs image: the gspread outage, the pod detour, and the settled design (2026-08-31 → 09-01)

Companion to `airflow-migration.md` (how the pipeline was built) and
`airflow-audit-2026-08-24.md` (what review found in it). This records how a
production outage was diagnosed, a wrong architecture was built twice and
retired, and where the design finally settled — in the fork
`D:\DST\repos\hcs-new`, branch **`dst-self-contained-dags`** (pushed).

---

## Problem

Every 5-minute tick of the deployed `dst_campaign_scheduler` (AKS, Airflow
3.0.2, KubernetesExecutor) failed:

```
dst_campaign_scheduler.py:89  find_due_campaigns
pipeline/config.py:10         import gspread
ModuleNotFoundError: No module named 'gspread'
```

No campaign report ran at all — and silently, because `dst_config` had no
`SLACK_TOKEN` (`[alerts] SLACK_TOKEN/channel not set`), so the failure alert
itself could not send.

**Misleading first read:** "the fix is our Docker image — put the libraries in
`campaign-data-analysis-report`'s image and the DAGs can use them." That reading
drove two full pod implementations before being retired. It rests on a false
model of what an image is (see Root cause 2).

## Approach

1. Traced the error: NOT a path problem. The error was `gspread`, not
   `pipeline` — proving `dst_common/__init__.py`'s sys.path bootstrap had
   already found the pipeline on the worker's disk. The gap between broken and
   working in-worker was literally four pure-Python wheels.
2. Per-TASK dependency trace (AST walk of imports through `pipeline/` and
   `dst_common/`): 4 of 9 tasks reach `pipeline` → gspread; the other 5 need
   nothing beyond stdlib+airflow+requests. Only `pipeline/config.py` imports
   gspread at module level; every dst_common import of pipeline is lazy.
3. Read the reference architecture properly: `hcm_campaign_scheduler.py` is
   565 lines, **zero** image references, plain PythonOperator — config over
   HTTP (MDMS) via `requests`. Only `hcm_dynamic_campaigns` launches pods.
4. Three-reviewer decision (data eng / senior SWE / this KB): 2× option A
   (packages into the Airflow image), 1× option B (finish pods). MDMS mode as
   an escape hatch was REJECTED by all three (see below).
5. User settled it: packages requested from DevOps (sri.padma, 2026-08-31,
   all 10, "build image", ETA next-day EOD) and the no-pods design chosen.

## Root causes

**1 (the outage).** The migration moved orchestration to Airflow but the
worker runtime never got the pipeline's libraries. `pipeline/config.py:10`
imports gspread at module level, so ANY task touching `pipeline` dies at
import. The scheduler, `finalize_run` and `sync_group_to_mdms` all do.

**2 (the detour).** A category error: *an image is an execution environment,
not an access mechanism.* Code/libraries inside an image are reachable only by
RUNNING a container from it. A DAG in the worker cannot import from an image.
"Use the image so the DAGs can access the libraries" is therefore not a thing —
the only image that can feed the worker is the **Airflow image itself**.
HCM's image exists to (a) ship heavy per-team deps without bloating the shared
runtime, (b) give versioned deploys of a *growing* per-campaign script set
(rollback = paste an old tag), (c) per-report memory limits. None of the three
applies to one stable DST pipeline whose deps are in the Airflow image anyway.
Change-frequency is NOT a driver — dags-folder code updates FASTER (~60s
git-sync) than image code (rebuild + manual tag paste).

**3 (a second latent outage, found while testing).** The sys.path bootstrap
depends on the repo's folder STRUCTURE being on disk: it walks up 3 levels from
`dst_common/__init__.py` and expects `campaign-data-analysis-report` as a
sibling. Works only because git-sync happens to sync the WHOLE repo. The HCM
setup guide specifies `subPath: utilities/airflow-dags/dags` — applied, every
DST DAG dies with "The DST pipeline package could not be located. Looked in:
'/opt/campaign-data-analysis-report'" (the up-3 path math collapsing when
dags/ is mounted bare). Reproduced locally in real Airflow both ways.

## Fix / verify

**The settled design — branch `dst-self-contained-dags`:**

```
utilities/airflow-dags/dags/
    dst_campaign_scheduler.py     @task, in-worker
    dst_campaign_run.py           @task, in-worker (execute_campaign, = run.py)
    dst_config_sync.py            @task, in-worker
    .airflowignore                -> dst_data_analysis_report
    dst_data_analysis_report/     ONE package, self-explanatory name
        common/     Airflow-aware orchestration (was dst_common)
        pipeline/   the report - ZERO airflow imports, still runs standalone
        platform/   dst_report_metadata DDL + persister yml
        requirements.txt
```

- Libraries: from the rebuilt **Airflow image** (sri.padma; precedent:
  `clickhouse_connect` is already in it for two other DAGs).
- No KubernetesPodOperator, no Dockerfile/main.py, no image tag to maintain, no
  sys.path bootstrap (deleted — both packages sit beside the DAGs).
- Layering preserved and verified: pipeline→common imports = 0, airflow
  imports in pipeline/ = 0. Dependency runs one way.
- Verified in real Airflow with ONLY the dags folder mounted, no PYTHONPATH,
  no packages installed: all 3 DST DAGs load (remaining import errors are
  eGov's own DAGs). Survives a subPath git-sync.
- ~52 imports rewritten `pipeline`→package path; rename trick for nested
  rewrites: two-phase with sentinel tokens so the second regex cannot re-match
  the first's output.

**Merge gate:** the rebuilt Airflow image must be live BEFORE this merges, or
the scheduler resumes failing on gspread (loudly, but failing). DAGs travel by
git-sync in ~60s; images by rebuild + human. Image first, always.

**The reviewer answer for "why no image pattern?":** the image's three benefits
each evaluate to not-needed (deps already in Airflow image; one stable pipeline
doesn't need versioned container releases; KubernetesExecutor already pods every
task). And: *we built it, ran it, and decided against it* — pod plumbing was
proven live (pod scheduled, image pulled 17s/152MB, 19 env vars delivered, xcom
sidecar started) before retiring it. Both pod implementations are recoverable:
`git stash list` on `dst-campaign-data-analysis-report` (find_due.py +
command-override dispatch) and commit `e122dc0548` upstream.

---

## Incidents and traps recorded along the way

**e122dc05, the half-migration (severity: worse than the outage it addressed).**
Scheduler converted to a pod but its counterpart (`main.py` dispatch,
`pod_tasks.py`) never landed → pod launches, ignores the mode var, dies on
"DST_CAMPAIGN_ROW is not set", every tick, both retries. It also silently
(a) left `build_retime_guard` with ZERO callers repo-wide — the retimed-slot
guard became dead code, and (b) added `(x or [])` in `collect_due_campaigns`,
making a dead pod indistinguishable from "nothing due" — fail-OPEN on the
decision path where empty is the normal answer (silent-success path #11).
Lesson: a task-relocation commit must ship with its execution-side counterpart
in the SAME change, or land nothing.

**Pod-boundary registry trap (would have shipped blind).** All e134162
observability fixes are module-level registries (`RUN_DEGRADATIONS`,
`FAILED_POSTS`, ...). In a pod they live in the pod's process; DAG-side reads
by module attribute see empty and report green — the `analyze_itn` bug shape
again. Any pod design must pack degradations into the returned marker
(`run_campaign` already does).

**Explicit-marker rule.** A pod whose empty result is a legitimate outcome must
write `{"ok": true, "due": []}` — never a bare list/nothing — and the collector
must raise on a missing marker. Implemented and live-tested against the real
sheet before the design was parked: `{"ok": true, "group": "Dev", "due": []}`.

**If pods ever return, the dispatch options ranked:** (1) command override —
Dockerfile `CMD` (not ENTRYPOINT, matching hcm-custom-reports) + KPO
`cmds=["python", "find_due.py"]`; flag-free, Docker-native. (2) Punjab's `MODE`
env var (`analysis-report/main.py:17` routes on it) — the house pattern for
multi-job images. Never invent a new flag name (`DST_POD_MODE` was invented
before noticing `MODE` existed).

**`git commit` after `git mv` + content edits:** status `RM` = staged rename,
UNstaged modification. Plain `git commit` takes only the rename — the moved
files keep their OLD content. Caught only by reading committed content
(`git show HEAD:file | grep`), not the stat line. Always `git add -A` first or
verify the committed imports.

**`git clean -fd` eats sibling untracked work.** Used to remove untracked
__pycache__, it also deleted the just-created untracked `__init__.py`,
`platform/` and `requirements.txt`. Recovered via `git show HEAD:old/path >`.
Dry-run `-n` first, or pass explicit paths.

**Erasing commits vs reverting:** a merged-upstream commit cannot be traceless —
only a new commit (write it forward-worded, no "Revert"/trailer) or a rewrite
of the shared branch. A fork-only commit CAN vanish: `reset --hard` +
`--force-with-lease`. Resetting a fork branch BEHIND upstream makes PRs show
"nothing to compare" — it cleans your copy but changes nothing deployed. And
never `revert -m` an empty no-op merge — drop it with `reset --hard` (0-line
diff proves it safe).

**MDMS mode rejected as the escape hatch (all three reviewers):** the sheet
fallback still imports gspread; kafka-python still needs an image change; it
disables the retime guard the audit fixed; and v1-only read-only MDMS means
the mirror has never been written in anger.

---

## Config + environment facts (dev deployment)

- **`ES_INDEX_PREFIX: null` in dst_config = key DELETED = tenant-prefixed.**
  Both override layers treat JSON null as delete. The dev cluster's indices are
  UN-prefixed (`project-task-index-v1`, 12.7M docs; no `dev-*` exists), so null
  → queries hit a nonexistent index. Correct value: `""`. The KB's own v4
  example shows `null` (right for tenant-prefixed Bauchi) — copying it to an
  un-prefixed cluster reproduces Defect 1 inverted. STILL OPEN on dev.
- **dst_config on dev has no `secrets` and no `google_credentials_json`** —
  observed pod spec carried no ES_USER/ES_PASS/SLACK_TOKEN/service-account.
  Sheets/Drive/ES auth all fail; alerting silent. STILL OPEN.
- **`REPORT_IMAGE` (eGov's Variable, `hcm_dynamic_campaigns:52`) was observed
  pointing at OUR image** — would launch our entrypoint for HCM reports. Theirs
  to fix; out of DST scope per user instruction.
- **Executor is KubernetesExecutor** (worker pod names:
  `<dag>-<task>-<suffix>`, e.g. `dst-campaign-scheduler-list-deployment-groups-…`)
  → venv caching can never work (fresh pod per task); per-task isolation is
  already free.
- **The deployed Airflow is on AKS** (`aks-mainpool-*` nodes) while the package
  request named `unified-dev` (EKS, where every workload pod had been Pending
  7+ days on `untolerated taint(s)`, 0/2 nodes). Confirm the rebuild targets
  the AKS deployment. Also missing from the request: `kafka-python`.
- Airflow-image contents PROVEN by what other worker DAGs import: requests,
  kubernetes, clickhouse_connect. Airflow 3.0.2 core hard-requires
  cryptography>=41 and pyjwt>=2.10 (a requests+PyJWT Sheets-REST path needs
  zero new packages, if ever wanted).
- dev-tab test row: `CMP-2025-09-18-006990` (16,574 docs, tenant dev, MR-DN)
  — but dev ES has **no `boundaryHierarchy.healthFacility` and no `age`** on
  any doc, so analyze.py skips 100% → the correct outcome of a dev run is
  DEGRADED (zero-record + zero-target guards firing), not numbers. `cycle_index`
  written via sheets API needs `RAW` input option or "01" becomes 1.
- Best-practice audit vs the official checklist: **12.5/15 (~83%)** — design
  10/10 (0 parse-time hits in all three DAGs); deviations documented
  (wall-clock `datetime.now` is load-bearing under CronTriggerTimetable's
  zero-width interval; append-only Run Log mitigated by retries=0); gaps = no
  `ruff --select AIR3` in CI and no committed DagBag test.

## Branch map (fork, as of 2026-09-01)

- `dst-self-contained-dags` — THE deliverable, pushed. No pods, package under
  dags/.
- `dst-campaign-data-analysis-report` — reset to upstream tip `cb5607c8fd`
  (pod state, as deployed); pod-dispatch experiment in `git stash`.
- Upstream `airflow-analysis` — still the broken pod scheduler until the PR
  lands (AFTER the image).
