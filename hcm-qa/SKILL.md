---
name: hcm-qa
description: >-
  QA/testing lane of the HCM knowledge base (part of the `superman` skill). Designing test
  plans, building test harnesses, running tests against dev/staging, and reporting
  DB-verified results for DIGIT HCM. Covers validator-unbundle testing, transformer
  error-handling QA (HDDF-5226), transformer end-to-end scenario testing (happy/sad/exception/edge
  -> ES *-index-v1), persister tests, and the DB-verification methodology.
  Triggers: "test plan", "test harness", "validator unbundle", "unified-dev",
  "null relation", "persister test", "does it persist", "verify DB", "transformer scenario test",
  "happy and sad scenario", "index-v1", "indexer yml", "port-forward check", "silent ghost",
  test-case ids like U1 / S4 / TC-08.
---

# HCM QA (lane of `superman`)

Test design, execution, and honest DB-verified reporting. Content is maintained once in
the `superman` skill; this is a focused entry point.

Core rule up front: verify a "persisted" claim by DB `_search` AFTER 60-100s (past the 60s
Redis TTL). `id=None` in a search result = cache false positive, not a real write. Isolate
single runs (parallel runs create false negatives); separate harness-bug from real defect.
House rules: no emojis / decorative unicode; be honest and reliable on every claim.

Read for this lane:
- `~/.claude/skills/superman/references/qa/overview.md` — QA playbook + the core verify-past-TTL rule.
- `~/.claude/skills/superman/references/qa/verification-methodology.md` — the honest-verification loop + worked corrections.
- `~/.claude/skills/superman/references/qa/validator-unbundle.md` — fix under test, feature flags, U/S/B/D/F/E series, harness + Postman, null-relation findings, persister test plan.
- `~/.claude/skills/superman/references/qa/transformer-error-handling.md` — HDDF-5226 12 test cases, setup, defects (TC-08/09/10), docs.
- `~/.claude/skills/superman/references/qa/transformer-scenario-testing.md` — full end-to-end happy/sad/exception/edge coverage (~42 verified + ~90-case catalog): PASS=error-captured-or-indexed (nulls=service data), the /health-* path + error-id + indexer-yml fixes, dev reference data, evidence-trail triage, gotchas. Artifacts in Downloads/.

Related lanes: the fix being tested -> `hcm-dev`; platform architecture -> `hcm-context`.
Whole picture -> `/superman`.
