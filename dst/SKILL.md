---
name: dst
description: >-
  Data-support lane of the HCM knowledge base (part of the `superman` skill). DST is
  Salaudeen's DIGIT HCM data work at eGov: ES-vs-DB (Elasticsearch vs PostgreSQL)
  reconciliation and ghost-record RCA, data-recovery scripting, per-campaign data fixes,
  the automation pipeline (including its Airflow migration), and campaign daily reporting.
  Invoke for any campaign data investigation, reconciliation, recovery script, report build,
  or Airflow DAG work on the automation pipeline. Triggers: tenant codes
  so/ba/oy/na/ko/ab/od/kd/tg/cg, "ghost records", "ES vs DB", "reconciliation",
  "chain recovery", "children treated", "CDD sync", "campaign report", "householdId null",
  "egov-tracer", "Airflow", "DAG", "scheduler.py", and campaign names
  (Sokoto/Bauchi/Oyo/Ondo/Kaduna/Togo/FCT/Plateau/Nasarawa/Kogi).
---

# DST (lane of `superman`)

HCM data investigation, recovery, and reporting. Content is maintained once in the
`superman` skill; this is a focused entry point.

House rules (always apply): DIGIT writes return 202 not 200; tracer requestBody is unquoted
(regex, not json.loads); match ES index prefix to tenant; report analysis-script numbers not
dashboard; ES-not-in-DB=CREATE / DB-not-in-ES=UPDATE; save counts before del; no emojis /
decorative unicode; never write test data to production tenants.

Read for this lane:
- `~/.claude/skills/superman/references/dst/overview.md` — DST playbook: per-campaign Problem->Approach->Root cause->Remediation->Status.
- `~/.claude/skills/superman/references/dst/es-vs-db-rca.md` — 6-phase SOP, tracer queries, the 90,143-ghost RCA, "what proven means".
- `~/.claude/skills/superman/references/dst/recovery-scripts.md` — db<->ES reconciliation, relational chain check (5-link legacy scripts; 6-link A-F in the workbench), 12M-record perf optimizations, checkpoint pattern, tracer regex, CREATE vs UPDATE, dependency order.
- `~/.claude/skills/superman/references/dst/campaigns.md` — per-tenant config/API/token/findings/status/scripts (so/ba/oy/na/ko/ab/od/kd/tg/cg/ke).
- `~/.claude/skills/superman/references/dst/reporting.md` — daily pipeline, 4-slot structure, formulas, per-drug age bands, per-day numbers, python-docx helpers, DQ rules.
- `~/.claude/skills/superman/references/dst/automation-and-structure.md` — `D:\DST\automation` pipeline + `D:\DST` folder conventions.
- `~/.claude/skills/superman/references/dst/airflow-migration.md` — migrating the automation pipeline's orchestration to Apache Airflow: Scheduler+Processor DAG pattern, Postgres-lock per-tenant concurrency, the Airflow-3 CronTriggerTimetable gotcha, marker-dict success/no-op/failure protocol.
- `~/.claude/skills/superman/references/dst/dst-team-and-hcm-modules.md` — what DST (DIGIT Success Team) is + the full HCM module surface.

Related lanes: HCM Java fixes -> `hcm-dev`; test execution -> `hcm-qa`; platform facts ->
`hcm-context`. Whole picture -> `/superman`.
