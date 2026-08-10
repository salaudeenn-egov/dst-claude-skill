---
name: hcm-context
description: >-
  Platform-context lane of the HCM knowledge base (part of the `superman` skill).
  The DIGIT HCM mental model: architecture, offline-sync data flow, the
  Individual->Household->HouseholdMember->ProjectBeneficiary->ProjectTask entity chain,
  why ES and DB diverge (ghost records), ES index + DB table structures, HCM API patterns
  (202 not 200), production credentials, tenant map, and hard-won gotchas. Invoke for any
  platform fact or "how does HCM work" question. Triggers: "DIGIT HCM", "entity chain",
  "ES vs DB", "clientReferenceId", "egov-transformer", "egov-persister", "ES index",
  "DB table", "HCM API", "credentials", tenant codes (so/ba/oy/na/ko/ab/od/kd/tg/cg).
---

# HCM Context (lane of `superman`)

The stable platform mental model. Content is maintained once in the `superman` skill;
this is a focused entry point into it.

House rules (always apply): no emojis / decorative unicode; report analysis-script
numbers not dashboard numbers; verify persistence past the 60s Redis TTL (id=None =
cache false positive); never write test data to production tenants (use `dev`).

Read for this lane:
- `~/.claude/skills/superman/references/context/overview.md` — orientation, skill map, fast facts.
- `~/.claude/skills/superman/references/context/architecture.md` — data flow, Kafka two-topic pattern, fire-and-forget save, Redis, the 4 ghost-record root causes, critical code file:line.
- `~/.claude/skills/superman/references/context/data-reference.md` — ES index + DB table structures, household->task linkage, API patterns, credentials, tenant/token map, KD SQL, tracer patterns, ES scroll/update.
- `~/.claude/skills/superman/references/context/gotchas.md` — 21 gotchas + dead ends. Read before any ES/tracer work.
- `~/.claude/skills/superman/references/context/postman-api-index.md` — 183 HCM API endpoints (full collection bundled in the same folder's `assets/`).

For the whole picture across all work-streams, use `/superman`.
