---
name: hcm-context
description: >-
  Main living context for the DIGIT HCM (Health Campaign Management) platform that
  Salaudeen (L3 SWE, eGov) works on. The shared mental model every other HCM skill
  builds on: platform architecture, offline-sync data flow, the
  Individual->Household->HouseholdMember->ProjectBeneficiary->ProjectTask entity
  chain, why ES and DB diverge (ghost records), ES index + DB table structures, the
  HCM API patterns, production credentials, tenant map, and the hard-won gotchas.
  Read the relevant reference here FIRST for any HCM work, then hand off to the
  work-stream skill (hcm-dev for code fixes, hcm-qa for testing, dst for data
  recovery/reporting, hcm-code-reviewer for reviews). Triggers: "DIGIT HCM", "HCM
  platform", "entity chain", "ES vs DB", "ghost record", "egov-transformer",
  "egov-persister", "clientReferenceId", any tenant code (so/ba/oy/na/ko/ab/od/kd/tg/cg).
---

# HCM Platform — Living Context

The shared, slowly-changing mental model of the DIGIT HCM platform. Other HCM
skills reference this one instead of re-deriving architecture. This is the
"whole application" context that grows over time.

> Living skill. Last synced 2026-07-08. Point-in-time: file:line citations,
> credentials, and tokens may be stale — verify against current code / live
> systems before asserting as fact. See "Keeping this skill current" at the bottom.

## The skill system (how the pieces fit)

All content lives in `superman` as five lanes; two entry-point skills route into it.

| Lane (in `superman/references/`) | Owns | Entry-point skill |
|---|---|---|
| **context** (this) | Platform architecture, entity chain, ES/DB structure, API patterns, credential pointers, tenants, gotchas. The stable core. | `hcm-engineering` |
| **dev** | Writing/shipping code fixes to HCM Java services (transformer, indexer, validators, persister). | `hcm-engineering` |
| **qa** | Test plans, harnesses, execution, DB-verified results, verification methodology. | `hcm-engineering` |
| **dst** | Data support: ES-vs-DB RCA, ghost-record recovery scripting, per-campaign fixes, automation + Airflow, daily reporting. | `dst` |
| **review** | Reviewing HCM diffs against the platform's specific hazards (ghost records, null-safety, cache-before-DB, batch rollback). | `hcm-engineering` |

Raw per-session facts live in the maintainer's local Claude auto-memory. These skills
are the curated, task-oriented layer synced from it.

## House rules (apply in every HCM session)

- **User:** Salaudeen, L3 SWE on the HCM product at eGov. Data investigation + fixes
  across ~27 health campaigns (Nigeria, Mozambique, Senegal, Togo, Chad, Burundi,
  Congo). Writes ES queries, SQL, Python. Work dir `D:\DST`.
- **Doc/response style:** no emojis, no decorative unicode (no arrows, no `────`
  rules). No bookmarks in Word docs. Plain text + standard markdown.
- **Numbers rule:** in reports use the analysis-script numbers, never silently swap
  to dashboard/Metabase figures. Note a discrepancy if asked; keep conclusions on the
  analysis numbers.
- **Honesty rule:** report only verified facts (for persistence, DB-verified past the
  60s Redis TTL). Never present a single run or a cache hit as proof. Say what is
  reliable vs noise.
- **Sensitive data:** Nigeria DB is production, read-only. Never write test data to
  production tenants (e.g. `so`). Use tenant `dev` for testing.

## The one model to hold

```
Individual -> Household -> HouseholdMember -> ProjectBeneficiary -> ProjectTask
```
Every entity references its parent. **DB writes validate parent existence; ES writes
do NOT.** DB (PostgreSQL, schema = tenant code) and ES (Elasticsearch,
`{tenant}-{entity}-index-v1`) are written by INDEPENDENT Kafka consumers:
egov-persister -> DB (can fail silently), egov-transformer -> ES (always succeeds).
Persister fails but transformer succeeds => **ghost record** (in ES, not DB). This
one fact underlies nearly all HCM investigation, recovery, dev, and QA work.

## Reference files

| File | Read it when |
|---|---|
| `references/architecture.md` | You need the platform internals: offline-sync flow, two-topic Kafka pattern, GenericRepository fire-and-forget save, Redis cache structure, the 4 ghost-record root causes, and the critical code locations (file:line). |
| `references/data-reference.md` | You need concrete data facts: ES index structures + field maps, DB table columns, the household->task linkage, HCM API endpoints (202 not 200!), production DB/ES credentials, the tenant/campaign table + API bases + tokens, KD/AZM SQL, tracer search patterns, ES scroll/update snippets. |
| `references/gotchas.md` | Before any ES/tracer/recovery work: the 21 hard-won gotchas + investigation dead ends + common pitfalls. Reading this first saves hours. |
| `references/postman-api-index.md` | You need the real HCM API surface: 183 requests across 23 folders (User/Project/Household/Individual/Task/Beneficiary/Referral/Stock/Attendance/etc.) with method + path. The full importable collection is bundled at `references/assets/HCM_Collection.postman_collection.json` (2.3 MB — import into Postman; do not read raw). |

## Fast orientation facts

- **Tenants** = deployments; DB schema and ES index prefix both use the code:
  `so`=Sokoto, `ba`=Bauchi, `oy`=Oyo, `na`=Nasarawa, `ko`=Kogi, `ab`=FCT/Abuja,
  `od`=Ondo, `kd`=Kaduna, `tg`=Togo, `cg`=Congo, `mz`=Mozambique, `pl`=Plateau,
  `ke`=Kebbi, `si`=Sierra Leone.
- **Drugs/age bands:** SMC = SPAQ (**SPAQ1 = 3-11m, SPAQ2 = 12-59m**; treated 3-59m).
  (Corrected 2026-08-24 — these two labels were swapped here; the CODE is right,
  see dst/reporting.md for the evidence chain.)
  AZM = Azithromycin (1-11m, 12-59m; treated 1-59m). MRDN/SPAQ (Togo) 0-3m/3-11m/12-59m.
- **DIGIT writes return HTTP 202 (Accepted), not 200** — check `in (200, 202)`.
- **egov-tracer `requestBody` is unquoted (not valid JSON)** — `json.loads()` fails
  silently; use regex to extract UUIDs. Recurs in many scripts.
- **`enable.auto.commit=true` is a platform-wide constraint** — never propose changing it.

## Keeping this skill current

- When platform-level understanding changes (new architectural insight, a rotated
  credential, a corrected gotcha, a new tenant), update the matching reference file
  and add a dated Changelog entry. Mark superseded facts STALE rather than deleting
  the history.
- When the user says "update skills", sync new durable platform facts here from the
  auto-memory.
- Source memories: digit_hcm_global_context, digit_hcm_architecture, hcm_gotcha_facts,
  data_discrepancy_rca (mechanism), redis_bypass_proof (mechanism), user_role,
  feedback_doc_style, feedback_analysis_numbers.

## Changelog

- 2026-08-05 — Added mobile app client-side sync/retry/UPLOAD_ERROR_FROM_APP mechanism
  to architecture.md (dumpError() code path, retry-threshold condition, the confirmed
  false-positive reporting defect from live-device QA on Chad ITN). Source: hcm-qa
  chad-upload-error-from-app.md.
- 2026-07-08 — Initial consolidation from the auto-memory (global context, architecture,
  gotchas, ghost-record mechanism, user role + style/number preferences).
