---
name: hcm-code-reviewer
description: >-
  Reviews DIGIT HCM code changes against the platform's specific, recurring
  hazards — not generic linting. Invoke when reviewing a diff / PR / patch to any
  HCM Java service (egov-transformer, egov-indexer, egov-persister, health-services
  validators/consumers/repositories, ProjectApiController, GenericRepository) or to
  data-recovery / reconciliation Python scripts. Knows the ghost-record failure
  modes, the cache-before-DB antipattern, batchUpdate whole-batch rollback,
  silent-enrichment null links, fire-and-forget save, tracer/JSON pitfalls, and the
  202/enum/address requirements. Triggers: "review this HCM change", "review the
  transformer/indexer/validator diff", "code review" on HCM code, "will this cause
  ghost records", "is this safe to merge". Use alongside the generic /code-review.
---

# HCM Code Reviewer

Domain-aware review for DIGIT HCM changes. Generic review tools catch style and
common bugs; this skill catches the HCM-specific ways a change silently creates
ghost records, drops data, or breaks the ES/DB contract.

> Living skill. Last synced 2026-07-08. Built on hcm-context (read
> `hcm-context/references/architecture.md` for the mechanisms these checks defend).

## When to use

- Reviewing a diff/PR to HCM Java services or config (ymls, application.properties).
- Reviewing data-recovery / reconciliation / extraction Python scripts.
- Answering "will this cause ghost records / data loss / an ES-DB mismatch?"

Pair with the generic `/code-review` skill: run that for correctness/simplification,
run this for HCM domain hazards. For the fix implementations these checks reference,
see the `hcm-dev` skill.

## How to review

1. Identify what layer the change touches (controller / consumer / validator /
   repository / persister / transformer / indexer / recovery script).
2. Walk `references/review-checklist.md` for that layer — each item is a concrete
   failure mode with the question to ask.
3. For each finding, state the failure scenario (inputs -> wrong outcome), not just a
   rule. Rank by severity: data loss / ghost record > silent-degrade > style.
4. If the host asks for structured output, report via the ReportFindings tool
   (most-severe first); otherwise plain markdown, findings first.

## The five hazards that matter most (HCM-specific)

1. **Cache-before-DB.** Any `putInCache()` (or cache populate) BEFORE the DB write is
   confirmed reopens the Redis-bypass ghost path. Cache must be populated only after DB
   commit. (See ProjectApiController.java:115 antipattern.)
2. **Validator reads cache not DB.** A cross-entity existence check via `validateIds()`
   / `findInCache()` can pass on a parent that is only in Redis, not DB. Direct-DB
   (`validateClientReferenceIdsFromDB`) is required for a real existence guarantee.
3. **Whole-batch rollback.** JDBC `batchUpdate` over a Kafka message runs all N records
   as one transaction — one bad record drops all N (including valid ones). Changes to
   persister batching must preserve per-record isolation / retry.
4. **Silent enrichment -> null links.** Transformer building a doc with null
   `householdId` / linking clientReferenceId when a lookup returns empty, and indexing it
   anyway, IS the ghost mechanism (TC-08). Empty enrichment must be an error/DQ flag, not
   a silent index.
5. **Errors that never surface.** Fire-and-forget `save()`, auto-commit swallowing
   failures, error records with a null source topic or a non-unique/keyless doc id
   (overwrite / index-drop). A failure the operator can't see is worse than a loud one.

## Reference

| File | Contents |
|---|---|
| `references/review-checklist.md` | Per-layer checklist: controller, validator, repository/persister, transformer, indexer yml, and recovery-script checks — each a concrete failure mode. |

## Keeping this skill current

- When a new failure mode is discovered (in dev, qa, or a production incident), add it to
  the checklist as a concrete scenario and add a dated Changelog entry.
- When the user says "update skills", fold new hazards learned this session into the
  checklist.
- Source: derived from hcm-context (architecture, gotchas), hcm-dev (fixes), and hcm-qa
  (defects found). Keep in sync with those.

## Changelog

- 2026-07-08 — Initial authoring from the platform architecture, gotchas, and the
  transformer/validator defects found in QA.
