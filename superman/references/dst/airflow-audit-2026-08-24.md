# DST automation: three-lens codebase audit and the fixes (2026-08-24)

Companion to `airflow-migration.md`. That file records how the pipeline was
built; this one records what a full adversarial review found in it, what was
fixed, what was deliberately NOT fixed, and why.

Target: `D:\DST\automation`, branch `airflow`, Airflow 3.1.6.

**Commit status.** The fixes landed as **e134162** (2026-08-24 17:07 IST,
"Enhance error handling and reporting in various modules", 16 files, +515/-80).
Verified after the fact that the commit carries the observability/infra fixes and
that NONE of the reverted core-logic/query changes leaked in - `_terms_hits`,
`track_total_hits`, the individualId name leg, the band/cutoff/batch tunables and
the DQ-denominator change are all absent from it. Suites 131/131 against the
committed tree. `DEPLOYMENT.md` remains untracked.

Worth repeating as a habit: after work you authored gets committed, diff the
commit against what you INTENDED to leave behind. A reverted experiment that
silently rides along in a `git add -A` is exactly the kind of change nobody
reviews.

---

## Method (reusable)

Three independent reviewers over the same codebase, each with a different lens
and no sight of the others:

1. **Data engineering** — silent wrong-answer risks, aggregation and join
   correctness, pagination completeness, idempotency.
2. **Senior engineering** — control flow, Airflow-specific hazards, secrets,
   observability, structure.
3. **Domain** — read `superman` SKILL.md + `references/dst/` + the DST memory
   files FIRST, then the code, and report only where the two DISAGREE.

Then every high-severity claim was re-verified in the source before being
reported. That last step mattered: it caught one reviewer overstating, and it
caught a claim of mine that needed narrowing (see "cycle_index" below).

**Nine findings were reached independently by two or more lenses.** Those are
the ones worth trusting first. The domain lens produced findings the other two
structurally could not — it is the only one that can see "documented but never
implemented".

---

## The single biggest theme: silent success

Ten separate paths let the pipeline survive a partial failure WITHOUT telling
anyone. That is the through-line of the whole audit, and it is what most of the
fixes address. In a system whose consumers have no Airflow access, "logged at
WARNING" is the same as "not reported".

Paths found: zero records matched; zero targets loaded; a failed Slack post; a
swallowed per-day ES error; a truncated join; a lost audit row; a discarded Kafka
event; a skipped partner post; an unreachable Variable; and a degradation reason
overwritten by a second one.

---

## Fixed (infrastructure and observability only)

Under a constraint the user set mid-way: **no changes to core logic or ES
queries.** Everything below leaves every number identical.

| Fix | Problem it closes |
|---|---|
| Kafka `send().get(timeout=10)` | `flush()` raises only on TIMEOUT; per-record rejections go to the future. A wrong `DST_RUNS_TOPIC` returned True, so `record_outcome` set `recorded="kafka"` and SKIPPED the Run Log fallback — the audit trail was silently discarded and `dst_report_metadata` stayed empty, defeating the exact guarantee its docstring makes |
| Slack post retry (3 attempts, 429-aware) + `FAILED_POSTS` | the post IS the deliverable, and it had no retry and no registry; a lost post left the run green with an empty Error column |
| Explicit skip paths in `notify.run` | the partner branch had NO else at all — a partner-only slot could do literally nothing and report SUCCESS |
| `_degrade()` accumulates reasons | two direct assignments to `marker["stages"]["report"]` meant the second erased the first: "Drive link missing" hid "the body sent to partners is a placeholder" |
| `analyze.RUN_DEGRADATIONS` + zero-record and zero-target guards | both conditions published a complete, plausible, green report. The code literally logged "the report should not be shared" and then shared it |
| `report.TRAJECTORY_FAILURES` | a swallowed per-day ES error counted that day as 0 and presented the total as measured — and since the Slack post is cumulative-first, that is the headline |
| `config.CONFIG_CORRECTIONS` | `campaign_days` disagreeing with the campaign dates is now reported. The VALUE is unchanged (see "not fixed") |
| Guarded scroll cleanup in `core/es.py` | an unguarded `requests.delete` in `finally` REPLACED the propagating exception, losing the 401/index_not_found message AND the HTTP status the retry logic reads |
| Per-process credentials dir, `O_EXCL` 0600 | fixed path + delete-in-finally: with `max_active_runs=16`, the first task to finish deleted the service-account key out from under a longer-running one |
| Unreadable Variable is fatal UNDER Airflow | a 2-second API hiccup was indistinguishable from "unset", so `sheet_tab` fell back to `Sheet1` and a run could complete successfully against a DIFFERENT deployment. Gated on `AIRFLOW_CTX_DAG_ID`/`AIRFLOW_HOME`/`AIRFLOW__CORE__EXECUTOR` so local runs and JupyterHub are untouched |
| `config.build` / `apply_cumulative` inside `_run_stage` | `_validate_row`'s docstring promised "fails immediately with no retries" — but both were called OUTSIDE the classifier, so a bad sheet cell got 2 retries at 3-minute intervals |
| MDMS `ValueError` re-raised in the scheduler | the bare `except Exception` swallowed the deliberate "DST_MDMS_ENABLED=true but MDMS_URL is not set", leaving a permanent hybrid state that still reported itself as MDMS mode |
| `max_consecutive_failed_dag_runs` removed | ONE DAG serves every tenant and `finalize_run` re-raises, so 3 failures from one bad cell auto-paused reporting for all 27 campaigns — with no alert for the pause itself |
| `dagrun_timeout` 90 -> 200 min | shorter than the task's own worst case (3 x 60m + 2 x 3m = 186m), so a slow campaign was killed mid-retry and `finalize_run` never ran — no Run Log row, no Kafka event |
| `finalize_run(retries=0)` | it ALWAYS raises on a failed run and is not idempotent (append, fresh event_id UUID, Slack warning). A deployment-level `default_task_retries=1` would double every failed campaign's audit rows |
| Trigger operator subclassed with `template_fields = ()` | **`conf` is a Jinja template field** and Airflow renders template fields recursively — so every Google Sheet cell was evaluated as a template. `{{` in a campaign name crashed the tick; `{{ var.value.dst_config }}` in a cell rendered the service-account private key into `dag_run.conf` and the UI |
| Retime guard scoped by campaign+cycle | `append_run_log` wrote Campaign and Cycle; `fetch_today_runs` never read them back, so a Bauchi SMC success suppressed the Bauchi ITN slot |
| `DST_DUP_MATRIX` env key | see below |
| `exc_info=True` on the 6 `log.error` calls inside `except` blocks | — |

### Verifying "no numbers moved"

Mechanical, not by eye:

```bash
git diff -U0 -- pipeline/ | grep -E "^[-+]" | grep -vE "^[-+]{3}" \
  | grep -cEi '"(query|size|terms|aggs|cardinality|exists|_source)"'    # -> 0
git diff -U0 -- pipeline/ | grep -E "^[-+]" | grep -vE "^[-+]{3}" \
  | grep -E "\* *100|/ *(total|treated|denom)|is_treated|age *<= *11"   # -> empty
```

Keep using this. It is the cheapest possible proof that an observability change
did not become a numbers change.

---

## NOT fixed, deliberately — the data-correctness half

These were implemented, then **reverted on instruction** ("don't change core
logic and query"). They remain OPEN and each needs a product decision first.
Listed so nobody re-discovers them from scratch.

1. **Terms-lookup truncation.** Four lookups in `analyze.py` send `"size": _BATCH`
   for a batch of `_BATCH` (5000) ids, no `search_after`, no scroll, no
   `hits.total` check — on one-to-MANY joins. 5000 ids matching 5060 docs returns
   5000; the tail resolves to no household, no head, no name, and is published as
   "Missing HH Name", i.e. read as a FIELD data problem. Nondeterministic between
   runs. `analyze_itn._run_lookup` already scrolls the identical shape.
   Already in the KB as the ko household-head join bug. A working fix (recursive
   split on `total > len(hits)`, with `track_total_hits`) was built and proven to
   recover 15,000/15,000 hits where the old path returned 5,000 — then reverted.
2. **The `task.individualId` name leg is missing.** The rule is "task.individualId
   OR projectBeneficiaryClientReferenceId -> PB"; only the PB leg exists.
   `Data.individualId` IS fetched into `_source` and assigned to a local at
   `analyze.py:372`, then never used. Where tasks carry no PB ref, ZERO names
   resolve, the report shows 100% Missing Child Name, and nothing raises — so
   `NAME_BATCH_FAILURES` does not fire either.
3. **`cycle_index` is a no-op unless `task_campaign_filter=TRUE`.** The early
   return sits above the cycleIndex clause. NARROWED during verification: where
   cycle date windows are disjoint this costs nothing; it bites where they are
   not — the documented case is ch's ~880 cycle-01 stragglers carrying taskDates
   inside the cycle-02 window. Rated MEDIUM, not HIGH. The docstring's "regardless
   of above" means "regardless of which branch", not "regardless of the flag", and
   reads as always-applied.
4. **Two disagreeing definitions of `treated` in one document.**
   `analyze.py:446-452` counts any non-VISITED status with quantity>=1 in band AND
   requires a non-empty healthFacility; `report.py:270-276` requires
   `ADMINISTRATION_SUCCESS` and has no facility test — while its comment claims it
   mirrors analyze exactly. Section 4's Day-N row therefore cannot equal the
   "Children Treated" headline from the same data.
5. **DQ percentages can exceed 100%.** Numerators counted over ALL records,
   denominator is `treated`, column labelled "% of Treated". Age=0 and Age>59 are
   STRUCTURALLY disjoint from the denominator. This is a SECOND, independent cause
   of the Kebbi 103.27% reading — the name-batch fix does not address it.
6. **Daily reports show Daily Target 0 and an all-0% chart.** `_build_trajectory`'s
   daily branch never fills `d["target"]` (only the cumulative branch does), and
   `_load_daily_totals_from_es` sets `target: 0`. The headline coverage tile uses a
   different path, so the document contradicts itself.
7. **Soft-deleted staff inflate NEVER SYNCED.** `_load_staff_by_project_type`
   filters `Data.isDeleted: False`; `_load_staff_by_campaign` (every Nigeria
   admin-console tenant) does not.
8. **`precision_threshold` drift.** Present in `cdd_sync.py`, absent in
   `cdd_sync_itn.py` — so above ~3,000 distinct CDDs the ITN "synced" figure
   silently becomes an approximation while the SMC one stays exact. Bauchi has
   3,764 FLWs. Pure copy-drift, and the clearest argument for shared helpers.

Also open, lower severity: `DELIVERED`/`CLOSED_HOUSEHOLD`/`NOT_ADMINISTERED`
fetched nowhere (Togo: 228k DELIVERED vs 116k ADMINISTRATION_SUCCESS); no
`productName` filter on the primary query (ke); hardcoded 17:00/17:30 UTC cutoffs
applied at every slot, so an 05:30 report publishes "Synced by 17:00" and a
per-name "NOT SYNCED BY 1730" list; `date.today()` instead of `slot_date`; ITN
cumulative nets triple-counting vs the deduped LGA table; band taxonomy mismatch
(see below).

---

## Config reachability: the recurring defect class

`CDD_ROLE` was documented for months and read by nothing. `DUP_MATRIX_DEFAULT`
was an in-code constant with no env key — and the deployment that needs it
(Chad ITN) has no env file, no file mounts and a read-only git-sync checkout, so
`Admin -> Variables` is the ONLY writable surface. The feature was therefore
unreachable by design.

**Fix pattern**, now applied to dup_matrix:

```python
DUP_MATRIX_FALLBACK = "FALSE"
def _dup_matrix_default():
    return (os.getenv("DST_DUP_MATRIX", "").strip() or DUP_MATRIX_FALLBACK)
# resolution: sheet cell  >  DST_DUP_MATRIX  >  fallback
```

Sheet cell (per campaign) beats env (per deployment) beats code (per checkout);
unset reproduces the old value exactly. `DST_DUP_MATRIX=FALSE` is now in the live
`dst_config` Variable so it is visible and editable in the UI.

**Rule going forward:** when adding a per-deployment behaviour flag, add the env
key in the same commit as the feature. A flag only reachable by editing code does
not exist on the hosted deployment.

A wider "make everything configurable" pass (bands 95/70, sync cutoffs, `_BATCH`,
`_WORKERS`, `MAX_DAY_COLS`, `precision_threshold`, the `tenant == "ch"` branch)
was built and then **reverted on instruction** — the ask was for those two keys
only. If it is ever wanted again: every default must equal the literal it
replaces, and the derived TEXT must move with it (the cutoff labels and the
`NOT SYNCED BY 1730` tab name were separately hardcoded, so changing only the
query would have published a row labelled 17:00 describing a different time).

---

## Documentation that describes a system which does not exist

Second theme, and a real debuggability defect:

- `dags/README.md:44-45, 88-93` documents `DST_LOCK_TTL_MINUTES`, a per-tenant
  metadata-DB lock, and a `dst_campaign_runs` audit table. **Grep returns zero
  occurrences of any of them**, and both DAG docstrings say the opposite ("No run
  lock either"). It will send someone hunting a lock table mid-incident. Still
  present — worth deleting.
- The README sheet-schema table lists 15 columns and omits every column that
  changes what ES is queried (`cycle_index`, `task_campaign_filter`,
  `dose_index_filter`, `project_type_id`, `task_date_field`, `dup_matrix`,
  `target_file`), while still listing `hfs_total`/`flws_total`, which nothing
  reads.
- Band taxonomy: code has HIGH>=95 / MODERATE 70-95 / LOW<70; `reporting.md`
  documents four bands (LOW<33, MODERATE 33-66, ON TRACK 66-90, HIGH>=90) and an
  `ON TRACK` tab that no longer exists. So "MODERATE" means different things in
  the automation report and the manual recon deliverables for the same campaign.
  UNRESOLVED — do not silently align one to the other.
- `report.py:1253` tells the reader to "refer to the DIGIT HCM dashboard for
  current data", against the most-repeated rule in this KB.

New: **`DEPLOYMENT.md`** in the repo — mode choice, prerequisites, sheet schema,
the full `dst_config` key reference (33 keys, defaults verified against code not
memory), bring-up, per-tenant differences, troubleshooting, checklist. It
deliberately omits the phantom lock/audit-table sections.

---

## Verified CLEAN (do not "fix" these)

Checked in both code and KB, found correct:

- `ES_INDEX_PREFIX` absent-vs-empty is preserved end to end and derived in
  exactly ONE place (`config.py:381-382`); no literal index strings anywhere in
  `pipeline/`
- both override layers treat JSON `null` as *delete the key*, and both
  snapshot/restore `os.environ` correctly
- names come only from the individual index, never `additionalDetails`
- `_pad_cycle`, `numericise_ignore=["all"]`, the union/zero-row standard target,
  NOT REPORTED band, dedup-treated-only, `lga_display` excluding the pseudo-group
- `CDD_ROLE` and `CDD_ROLE_ITN` both genuinely read
- exception-handler ordering: exactly ONE `try` has multiple handlers
  (`campaign_runner.py:127`) and its TRANSIENT -> DATA -> generic order is correct
  and load-bearing. No bare `except:` anywhere in `dags/` or `pipeline/`
- no top-level DAG-file I/O; `pipeline.config` imported lazily inside task bodies
- run-id determinism, `skip_when_already_exists`, midnight-crossing window logic
- `composite_agg` paginates on `after_key` correctly; the staff/task scrolls are
  true scrolls
- `plan_sync`'s empty-read guard, first-row-wins, keep-last-known-good and
  `protection_key` reasoning

---

## Process lessons

**A fix that is never consumed is not a fix.** The name-batch degradation
registry added earlier the same day was DEAD for every ITN campaign:
`analyze_itn` imports the two name helpers FROM `analyze`, so ITN runs write into
`pipeline.analyze.NAME_BATCH_FAILURES` while the runner's
`hasattr(analyze_mod, ...)` guard inspected `analyze_itn` and always found
nothing. Found by the audit, hours after being written. When a module-level
registry is written by one module and read by another, assert the read side sees
the write side — `_name_registry_modules()` now resolves both.

**Clear-before-populate ordering.** `CONFIG_CORRECTIONS.clear()` was initially
placed with the other registry clears, which run AFTER `config.build()` — so
every correction was wiped before it could be read. Registries must be cleared
before the thing that fills them, not in a tidy block.

**Test doubles drift from the API they stand in for.** Two suite failures after
the Kafka fix were the harness, not the product: the fake producer returned
`None` from `send()`, so it could not model `.get()` on the future. A double that
cannot express the failure mode cannot test the fix.

**Harness traps that cost time again.** Unquoted bash heredocs execute backticks
inside the payload — writing markdown containing code spans through `<<PYEOF`
silently stripped every backticked line. Use `<<'PYEOF'`. And `python -c` with
container paths still needs the PowerShell tool or `MSYS_NO_PATHCONV=1`; a DagBag
parse in the container needs `sys.path.insert(0, '/opt/airflow/dags')` or all
three DAGs report a phantom `ModuleNotFoundError: common`.

**Verify a Variable write by reading it back.** Unchanged advice from the
migration notes, applied again here — a silent `set` failure once left the live
Variable pointing at a dead URL.
