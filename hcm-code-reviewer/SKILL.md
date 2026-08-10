---
name: hcm-code-reviewer
description: >-
  Code-review lane of the HCM knowledge base (part of the `superman` skill). Reviews DIGIT
  HCM code changes against the platform's specific, recurring hazards - not generic linting.
  Invoke when reviewing a diff / PR / patch to any HCM Java service (egov-transformer,
  egov-indexer, egov-persister, health-services validators/consumers/repositories) or to
  data-recovery / reconciliation Python scripts. Knows the ghost-record failure modes,
  cache-before-DB antipattern, batchUpdate whole-batch rollback, silent-enrichment null
  links, and the 202/enum/address requirements. Triggers: "review this HCM change", "review
  the transformer/indexer/validator diff", "will this cause ghost records", "is this safe to
  merge". Use alongside the generic /code-review.
---

# HCM Code Reviewer (lane of `superman`)

Domain-aware review for HCM changes. Content is maintained once in the `superman` skill;
this is a focused entry point. Pair with the generic `/code-review` skill.

The five hazards that matter most: (1) cache populated before DB commit (Redis-bypass ghost
path); (2) existence check reads cache not DB; (3) batchUpdate whole-batch rollback; (4)
transformer indexing a doc with null links on empty enrichment (the ghost mechanism, TC-08);
(5) errors that never surface (fire-and-forget, null-topic error record, non-unique error id).

Read for this lane:
- `~/.claude/skills/superman/references/review/overview.md` — the 5 hazards + how to review + how to report.
- `~/.claude/skills/superman/references/review/review-checklist.md` — per-layer failure modes (controller, validator, persister, transformer, indexer yml, recovery scripts).

Related lanes: the fix itself -> `hcm-dev`; platform mechanisms -> `hcm-context`; defects
found in testing -> `hcm-qa`. Whole picture -> `/superman`.
