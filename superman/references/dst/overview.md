---
name: dst
description: >-
  DST is Salaudeen's DIGIT HCM DATA SUPPORT work-stream at eGov: ES-vs-DB
  (Elasticsearch vs PostgreSQL) reconciliation and ghost-record RCA, data-recovery
  scripting, per-campaign data fixes, the automation pipeline, and campaign daily
  reporting. Invoke for any campaign data investigation, reconciliation, recovery
  script, or report build. Concrete triggers: tenant codes so/ba/oy/na/ko/ab/od/kd/tg/cg,
  "ghost records", "ES vs DB", "reconciliation", "chain recovery", "children treated",
  "CDD sync", "campaign report", "householdId null", "egov-tracer", and campaign names
  (Sokoto/Bauchi/Oyo/Ondo/Kaduna/Togo/FCT/Plateau/Nasarawa/Kogi).
---

# DST — HCM Data Support, Recovery & Reporting

Salaudeen's living knowledge base for the DIGIT HCM data-support work-stream: finding
and explaining ES/DB mismatches, recovering ghost and orphaned records, fixing
per-campaign data, running the automated reporting pipeline, and building the daily
campaign reports partners see.

> Living skill. Last synced 2026-07-08. Point-in-time: tokens rotate, counts and file
> paths may be stale — re-verify before use. See "Keeping this skill current".

## When to use / not use
Use for: ES-vs-DB reconciliation, ghost-record RCA, chain-recovery and push scripting,
per-campaign data fixes (Sokoto/Bauchi/Oyo/Ondo/Kaduna/Togo/Nasarawa/Kogi/FCT),
the automation pipeline, and daily campaign reporting (dashboard vs report, DQ audits).

Do NOT use for:
- HCM platform Java code fixes (validators, persister, consumers) → use the `hcm-dev` skill
- Test execution / QA harnesses → use the `hcm-qa` skill
- Platform architecture (offline sync flow, Kafka two-topic pattern, ES/DB divergence
  point, Redis structure), production credentials, and the full HCM gotcha list →
  reference the `hcm-context` skill. Do NOT duplicate those here.

## Reference files
| File | Read it when |
|---|---|
| `references/es-vs-db-rca.md` | Investigating any DB/ES count mismatch or ghost records; need the 6-phase SOP, tracer queries, the 90,143-ghost RCA, causes A/B/C/D, or what "proven" means |
| `references/recovery-scripts.md` | Writing/running reconciliation (db↔ES) or chain-recovery scripts; need the 5-link check, 12M-record perf optimizations, checkpoint pattern, tracer regex, or create/update dependency order |
| `references/campaigns.md` | Working a specific tenant; need its config, API base, last-known token, findings, recovery status, or script inventory |
| `references/reporting.md` | Building a daily campaign report; need the pipeline, slot structure, treatment-rate/duplicate/band formulas, per-drug age bands, per-day numbers, python-docx helpers, or DQ rules |
| `references/automation-and-structure.md` | Touching `D:\DST\automation`; need the scheduler→config→analyze/cdd_sync/report/notify flow, .env/creds, known bugs, or the D:\DST folder reorg + naming conventions |
| `references/null-datapoints-audit.md` | Auditing D&E CHILDREN_TREATED extracts for missing data points / SPAQ band mismatches; need the counting rules (blank-only child name, referral/OIC + VAS/ORS structural exclusions, cohort-aging SPAQ2), the MULTI-SHEET export gotcha (pd.read_excel reads only Sheet1 — v1 undercounted 14 state-cycles; Chad = 7 district sheets), Chad's different extract format (no GPS/productName columns, `Referred`/`Ineligible` names), the live log-to-Google-Sheet push pattern, or the audit sheet/tooling locations |
| `references/cross-cycle-cohort-reports.md` | Building/reviewing any cohort or "reuse across cycles" report (was the child administered in C1 AND C2): the PBC-stable-across-cycles pivot model, the all-cycles date-window cardinal rule, the Chad `children_cohort.xlsx` generator walkthrough (enrichment chain, bulk decrypt, SUCCESS+VISITED merge, per-province sheets), and the defect checklist (per-event flags treated static, redose drops, GPS/productName fetched-but-dropped, size-vs-hits, age-band double counting, timezone) |
| `references/dst-team-and-hcm-modules.md` | You need the wider picture: what the DIGIT Success Team (DST) is (L1/L2/Manager support org) and the full HCM module surface (HRMS, attendance, localization, microplan ingestion, inventory, dashboards, custom reports, payments, helpdesk, admin console, infra monitoring) that data work sits inside |

## Recurring gotchas (data work)
Reference `hcm-context` gotchas for the full platform list. High-value data-work ones:
- DIGIT create/update APIs return HTTP 202 (Accepted), not 200 — do not treat 202 as failure.
- Tracer `Data.apiDetails.requestBody` is NOT valid JSON (Java toString, unquoted keys)
  → extract fields with regex, never `json.loads()` (silent 0-results bug otherwise).
- Match the ES index prefix to the tenant (`so-`, `ba-`, `kd-`…). Togo (`tg`) has NO
  prefix. Querying the wrong index returns a misleading 0, not an error.
- `_search` caps at 10k hits — use `_count` for totals and scroll for full extraction.
- ES-not-in-DB = record never persisted = CREATE it; DB-not-in-ES = never indexed = UPDATE/re-index it.
- Create children only after the parent is in DB (Household → Member → Individual → PB → Task).
- On large scripts, save counts/references BEFORE `del` + `gc.collect()`, or the summary NameErrors.
- Match key for DB/ES comparison is `clientReferenceId` (in both), never the server ID (PTB-/PT-).
- Task index (`*-project-task-index-v1`): `Data.taskDates` (YYYY-MM-DD) works for date
  filtering; `Data.auditDetails.createdTime` and `Data.isDeleted` DO NOT EXIST on task docs.
- Field locations are tenant-specific — verify with an ES sample before filtering: gender
  (`Data.gender` vs `additionalDetails.gender`), lat/lon (`Data.latitude` vs `additionalDetails.latitude`).
- Report numbers = analysis-script numbers, NOT dashboard numbers (dashboard confirmed wrong).
- Tokens in campaigns.md are LAST-KNOWN and rotate; prod DB/ES creds live in `hcm-context`.

## Problem → Approach → Solution playbook

### Sokoto SMC ghost RCA (`so`)
Problem: `so-project-task-index-v1` had 4,664,185 tasks vs 4,645,970 in DB — ES higher
by 18,526 ghosts (plus 311 DB-only). Approach: comparison script on clientReferenceId
(doseIndex=01, ADMINISTRATION_SUCCESS), then tracer classification and beneficiary
timestamp checks. Root cause: beneficiary race condition (~8,244; 12,189 unique
beneficiary UUIDs in "Related entity does not exist" tracer errors) + idgen outage Sep 21
(~282) + Jul 28 indexer failure (311 DB-only, all rowversion=1). Remediation: verify the
12,189 beneficiaries exist in `so.project_beneficiary`; push beneficiaries first via
`so-save-project-beneficiary-topic`, then the 18,526 tasks; re-index the 311. Status: push
built + tested locally; production push PENDING beneficiary dependency check (as of 28 May 2026).

### 90,143-ghost RCA (Bauchi `ba` + Sokoto `so`)
Problem: 90,143 ghost records (ES not in DB) across 5 entity types; 747 DB-not-in-ES.
Approach: comparison scripts → slim ID files → `prove_all_ghost_tracer.py` classifies
each record by tracer error; `prove_race_condition.py` proves timing; Redis bypass proven
live on localhost. Root causes: A UPLOAD_ERROR_FROM_APP 62,958 (69.8%, a mobile retry
indicator not a cause), B silent persister failure 26,533 (29.4%, batchUpdate rollback +
misconfigured KafkaConsumerErrorHandler + auto-commit=true), C race condition 424, D idgen
Sep 21 34. Remediation: short-term Java fixes listed (belong to hcm-dev); long-term =
mobile wave-based sync. Full detail + "what proven means" in es-vs-db-rca.md.

### Bauchi chain recovery (`ba`)
Problem: find all data gaps for the 2026-06-22→07-01 campaign. Approach: `ba_chain_recovery.py`
5-link integrity check over 8.19M task docs (2,173,263 unique PB refs). Findings: Gap A
22,093 (0% tracer, unrecoverable), Gap B 2,126 (305 in tracer), Gap C 8, Gap D 2,828 (449),
Gap E 808,291 (likely campaign-incomplete). Remediation: `ba_chain_push.py` pushes Gap B
(305) + Gap D (449) = 754 records (set USER_UUID first); `ba_recon.py` handles ES↔DB recon.
Status: 754 pushable; PT create needs `resources[0].tenantId`; HHM head-of-household updates
still rejected by DIGIT (unresolved).

### Ondo ITN null-householdId recovery (`od`)
Problem: -1,858 bednet gap (dashboard vs report); 3,718 ADMINISTRATION_SUCCESS tasks with
`Data.householdId = null` (Sep 20-21 indexer wrote null before household DB write). Approach:
dump the 3,718, map householdId via `project_beneficiary → household clientReferenceId`
(project_task has no householdid column in DB). Recovered 3,356 (36 HH in DB, 3,320 in ES
only from Sep 21 outage); 362 unrecoverable (= 1,081 bednets). Remediation pipeline: update
36 DB households → create ES-only households in DB → bulk-create 3,356 tasks → bulk-create
20,637 members → ES-patch householdId. Status (2026-06-16): tasks bulk-create DONE
(3,356/3,356); households create ~640/3,267; members + ES patch NOT yet run.

### Oyo chain recovery + 12M-record perf optimization (`oy`, template `kd`)
Problem: run the 5-link chain check on campaigns with 12M+ records without OOM. Approach:
`oy_chain_recovery.py`; the `kd` `data_Recovery_Script (1)` is the optimized template —
two-pass member scroll (sets not maps), inline gap computation, `del`+`gc.collect()` between
phases, `TRACER_BATCH=1000` separate from `BATCH=5000`, payload capped at 2000 chars, counts
saved before `del`. Together ~6 GB RAM saved. Known bugs: tracer callbacks use `json.loads`
on unquoted requestBody (always 0 found) — fix with regex; Gap E search term should be
`projectBeneficiaryClientReferenceId` not `Tasks`.

### Nigeria SMC individual recovery (`oy`/`na`/`ko`/`ab`)
Problem: null HH-head names + missing Beneficiary IDs in the CHILDREN_TREATED report,
caused by missing Individual records. Approach: extract individual-create errors from tracer
(`/individual/v1/bulk/_create`), search HCM for existing, push the missing (30-50 workers,
DUPLICATE_ENTITY = OK). Counts: oy 4,182 missing; na 29,049; ko 30,151 (mostly duplicates —
exist under different clientReferenceId); ab 25,480 missing of 28,406. Confirmed dead ends:
lat/lon unrecoverable if null at submission (task GPS is the only source); missing
Beneficiary ID can't be recovered by re-creating individuals; Step 2 (HouseholdMember
re-push) still needed to fix remaining head-name nulls.

### Reconciliation-scripts approach (`D:\DST\scripts\script-reconciliation`, tested on `cg`)
Problem: generic, per-entity db↔ES reconciliation by re-pushing missing records through
Kafka. Approach: two scripts (`db not in es .py`, `reconcile_es_not_in_db.py`), JOBS config
per tenant/entity, `config.json` creds, `DRY_RUN` flag. Learnings: `api_response_key` must
match (`Households` not `HouseholdList`); project-beneficiary ES doc is flat
(`es_data_path=""`); read-only creds mean you can only test detection artificially — use a
real ghost campaign to test the push path. Congo household reached DB=ES=9,598 after 45 pushes.

### Reporting pipeline method
Problem: produce accurate daily campaign reports (dashboard is unreliable). Approach:
extract (ES scroll) → combine → analyze (metrics + DQ) → cdd_sync → generate_report (Word).
Coverage = treated / (individual_target/4); duplicate key = 4 fields
(child, ward, age, hh_head); bands LOW<33 / MODERATE 33-66 / ON TRACK 66-90 / HIGH>=90.
Age bands differ by drug (SMC/SPAQ 3-59; AZM 1-59, no SPAQ split; Togo MRDN). 4 report
slots (11am/2pm/5pm/8pm) each OVERWRITE their own Google Doc; only 8pm goes to partners.
The automated version lives at `D:\DST\automation`. Per-day numbers, formulas, style rules,
and python-docx helpers in reporting.md; automation bugs in automation-and-structure.md.

## Keeping this skill current
- After a DST session (new campaign day, new investigation, script change), add a dated
  Changelog entry and update the relevant reference file. Keep campaign status current.
- When the user says "update skills", sync from auto-memory.
- Source memories: sop_db_es_rca, data_discrepancy_rca, hcm_investigation_findings,
  hcm_investigation_process, project_reconciliation_scripts, project_oy_chain_recovery,
  project_ba_campaign, project_sokoto_smc, project_ondo_itn,
  project_nigeria_smc_individual_recovery, project_smc_day1_reporting, project_azm_kaduna,
  project_togo_mrdn, project_reporting_structure, project_dst_automation, project_dst_structure,
  project_chad_itn_dup_matrix.

## Changelog
- 2026-08-07 — itn-llin-reporting.md: added the dup_matrix duplicate-distribution
  feature (matching-key evolution, verified chad ES field facts, design invariants —
  UNCOMMITTED on branch ITN, commit next session), the Day-7 cumulative-undercount RCA
  (itn_history frozen-snapshot root cause + ES-at-report-time fix), and the 2026-08-06
  edge-case hardening pass. Source: project_chad_itn_dup_matrix.
- 2026-07-08 — Initial consolidation from 16 memory files.
- 2026-07-08 — Added `references/dst-team-and-hcm-modules.md` from the DIGIT Success Team
  training schedule (DST team structure + full HCM module surface).
