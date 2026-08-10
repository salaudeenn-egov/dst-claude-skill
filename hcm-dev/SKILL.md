---
name: hcm-dev
description: >-
  Development lane of the HCM knowledge base (part of the `superman` skill). Writing,
  building, or shipping code FIXES to DIGIT HCM Java services: egov-transformer,
  egov-indexer, egov-persister, and the cross-entity validators. Covers the ghost-record
  fixes, transformer error-handling fixes (HDDF-5226), indexer ymls, and build/env facts.
  Triggers: "transformer fix", "indexer yml", "validator fix", "DLQ", "persister",
  "UUID error id", class names like TransformerErrorProducer / ErrorMessage /
  PtProjectBeneficiaryIdValidator, branch names, "validateClientReferenceIdsFromDB".
---

# HCM Dev (lane of `superman`)

HCM Java code-fix work. Content is maintained once in the `superman` skill; this is a
focused entry point.

House rules (always apply): no emojis / decorative unicode; treat file:line/branch/build
facts as last-known and verify against current code before editing.

Read for this lane:
- `~/.claude/skills/superman/references/dev/overview.md` — dev playbook: Problem->Approach->Fix->Status per fix.
- `~/.claude/skills/superman/references/dev/ghost-record-fixes.md` — validator/persister/DLQ fixes (file:line, before/after), architectural options (uForwarder/Outbox/wave-sync), Salama-prod fixes, priority matrix.
- `~/.claude/skills/superman/references/dev/transformer-fixes.md` — UUID error-id fix, null-topic ThreadLocal fix, indexer ymls, /health-* endpoint fixes, JUnit harness, JDK8/maven build + env.

Related lanes: platform architecture -> `hcm-context`; testing a fix -> `hcm-qa`;
reviewing a diff -> `hcm-code-reviewer`. Whole picture -> `/superman`.
