---
name: hcm-engineering
description: >-
  Engineering lanes of the HCM knowledge base (part of the `superman` skill): platform
  context, code development, QA/testing, and code review for DIGIT HCM. Invoke for any
  platform fact or "how does HCM work" question (entity chain, ES-vs-DB divergence,
  ghost records, ES index / DB table structures, API patterns, gotchas); for writing or
  shipping code fixes to HCM Java services (egov-transformer, egov-indexer,
  egov-persister, cross-entity validators, DLQ); for test plans, harnesses, and
  DB-verified QA (validator unbundle, transformer error-handling HDDF-5226, persister
  regression, verification methodology); and for reviewing HCM diffs against the
  platform's recurring hazards. Triggers: "DIGIT HCM", "entity chain", "ES vs DB",
  "clientReferenceId", "ES index", "DB table", "HCM API", "transformer fix",
  "indexer yml", "validator fix", "DLQ", "persister", "test plan", "test harness",
  "validator unbundle", "unified-dev", "does it persist", "verify DB", "index-v1",
  "review this HCM change", "will this cause ghost records", "is this safe to merge",
  test-case ids like U1 / S4 / TC-08, class names like TransformerErrorProducer.
---

# HCM Engineering (lanes of `superman`)

Platform context, development, QA, and code review for DIGIT HCM. Content is maintained
once in the `superman` skill; this is a focused entry point into its four engineering
lanes. For data-support work (RCA, recovery, campaigns, reporting) use `/dst`; for the
whole picture use `/superman`.

House rules (always apply): no emojis / decorative unicode; report analysis-script
numbers not dashboard numbers; verify persistence by DB `_search` AFTER 60-100s (past
the 60s Redis TTL — `id=None` in a search result = cache false positive); isolate
single runs; separate harness-bug from real defect; treat file:line/branch/build facts
as last-known and verify against current code; never write test data to production
tenants (use `dev`); no credentials in any file — placeholders only.

## Lane 1 — Platform context (read first for any HCM task)

The stable mental model everything else builds on.

- `~/.claude/skills/superman/references/context/overview.md` — orientation, skill map, fast facts.
- `~/.claude/skills/superman/references/context/architecture.md` — offline-sync data flow, Kafka two-topic pattern, fire-and-forget save, Redis, the 4 ghost-record root causes, critical code file:line.
- `~/.claude/skills/superman/references/context/data-reference.md` — ES index + DB table structures, household->task linkage, API patterns (202 not 200), credential pointers (values redacted), tenant map, KD SQL, tracer patterns, ES scroll/update.
- `~/.claude/skills/superman/references/context/gotchas.md` — 21 hard-won gotchas + dead ends. Read before any ES/tracer work.
- `~/.claude/skills/superman/references/context/postman-api-index.md` — 183 HCM API endpoints (sanitized collection bundled in the same folder's `assets/`).

## Lane 2 — Development (HCM Java code fixes)

- `~/.claude/skills/superman/references/dev/overview.md` — dev playbook: Problem->Approach->Fix->Status per fix.
- `~/.claude/skills/superman/references/dev/ghost-record-fixes.md` — validator/persister/DLQ fixes (file:line, before/after), architectural options (uForwarder/Outbox/wave-sync), priority matrix.
- `~/.claude/skills/superman/references/dev/transformer-fixes.md` — UUID error-id fix, null-topic ThreadLocal fix, indexer ymls, /health-* endpoint fixes, JUnit harness, JDK8/maven build + env.

## Lane 3 — QA / testing

Core rule up front: verify a "persisted" claim by DB `_search` after the cache TTL;
never present a single run or a cache hit as proof.

- `~/.claude/skills/superman/references/qa/overview.md` — QA playbook + the verify-past-TTL rule.
- `~/.claude/skills/superman/references/qa/verification-methodology.md` — the honest-verification loop + worked corrections.
- `~/.claude/skills/superman/references/qa/validator-unbundle.md` — fix under test, feature flags, U/S/B/D/F/E series, harness + Postman, null-relation findings.
- `~/.claude/skills/superman/references/qa/transformer-error-handling.md` — HDDF-5226 12 test cases, setup, defects (TC-08/09/10).
- `~/.claude/skills/superman/references/qa/transformer-scenario-testing.md` — end-to-end happy/sad/exception/edge coverage, evidence-trail triage.
- `~/.claude/skills/superman/references/qa/persister-regression.md` — persister/bulk regression: method, verified behaviours, evidence-backed unhandled edge cases.
- `~/.claude/skills/superman/references/qa/mhbase-console-perf-testing.md` — mhbase env + console perf benchmarking: householdType silent-drop root cause, template rules, benchmark tiers/mapping, device-log QA.
- `~/.claude/skills/superman/references/qa/chad-upload-error-from-app.md` — live-device app-upload QA method (adb + flutter logcat).

## Lane 4 — Code review

The five hazards that matter most: (1) cache populated before DB commit (Redis-bypass
ghost path); (2) existence check reads cache not DB; (3) batchUpdate whole-batch
rollback; (4) transformer indexing a doc with null links on empty enrichment (the ghost
mechanism, TC-08); (5) errors that never surface (fire-and-forget, null-topic error
record, non-unique error id). Pair with the generic `/code-review` skill.

- `~/.claude/skills/superman/references/review/overview.md` — the 5 hazards + how to review + how to report.
- `~/.claude/skills/superman/references/review/review-checklist.md` — per-layer failure modes (controller, validator, persister, transformer, indexer yml, recovery scripts).
