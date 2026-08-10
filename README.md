# HCM Skills for Claude Code

A set of [Claude Code](https://docs.claude.com/en/docs/claude-code) skills that
package hard-won knowledge for working on **DIGIT HCM** (Health Campaign Management):
platform architecture, code development, QA/testing, data-support (DST) work, and
code review.

All content is **sanitized** — production credentials, auth tokens, and personal
data have been removed (see [Security](#security)).

## Layout

```
skills/
  superman/                     Umbrella skill — all HCM knowledge
    SKILL.md
    references/
      context/                  Platform mental model: architecture, data model,
                                ES/DB structures, API patterns, gotchas
      dev/                      Code fixes: transformer / indexer / persister / validators
      qa/                       Test plans, harnesses, verification methodology
      dst/                      Data support: ES-vs-DB RCA, recovery, campaigns, reporting
      review/                   HCM-specific code-review checklist
  dst/                          Entry-point skill -> DST lane
  hcm-context/                  Entry-point skill -> platform context lane
  hcm-dev/                      Entry-point skill -> development lane
  hcm-qa/                       Entry-point skill -> QA lane
  hcm-code-reviewer/            Entry-point skill -> code-review lane
```

`superman` holds the full knowledge base; the other folders are thin entry points
that route into the relevant `superman/references/<lane>` content.

## Skills

| Skill | Purpose |
|-------|---------|
| `superman` | Complete DIGIT HCM knowledge base (all lanes). |
| `hcm-context` | Platform mental model — architecture, entity chain, ES vs DB, API patterns. |
| `hcm-dev` | Writing/shipping fixes to HCM Java services. |
| `hcm-qa` | Test plans, harnesses, and DB-verified results. |
| `dst` | Data support — reconciliation, ghost-record RCA, recovery scripts, reporting. |
| `hcm-code-reviewer` | HCM-specific code-review against recurring failure modes. |

## Install

Copy the skill folders into your Claude Code skills directory:

```bash
cp -r superman hcm-context hcm-dev hcm-qa dst hcm-code-reviewer ~/.claude/skills/
```

Then invoke a skill in Claude Code (for example `/superman` or `/dst`), or let
Claude auto-select based on the task.

## Security

This repository contains **sanitized knowledge only**. All production credentials,
auth/session tokens, usernames (`<REDACTED-USER>`), Basic-auth headers, internal
endpoint hosts, and personal data have been replaced with `<REDACTED>` placeholders
(full credential + username audit: 2026-08-10). Real values live **only** in the
deployment `.env` / your local secret store and must never be committed. The bundled
Postman collection uses `{{authToken}}`/`{{username}}`/`{{password}}`/`{{basicAuth}}`
variables — fill them locally after import.

- Do not commit `.env`, credentials, keys, or tokens (see `.gitignore`).
- If you find any secret in the history or files, see `SECURITY.md`.
- This is internal engineering knowledge — keep the repository **private**.
