# DST Claude Skill

**The DIGIT HCM knowledge base for the eGov DST team, packaged as [Claude Code](https://docs.claude.com/en/docs/claude-code) skills.**

Months of production experience across ~27 health campaigns (Nigeria, Mozambique, Senegal, Togo, Chad, Burundi, Congo) — distilled into a knowledge base Claude loads automatically when you work on HCM. Instead of relearning why Elasticsearch and PostgreSQL disagree, which index field does not exist, or why a 202 response does not mean the record persisted, you start every session with the team's accumulated answers.

All content is **sanitized**: no credentials, tokens, or usernames (see [Security](#security)).

## Why use this

HCM knowledge has a specific problem: the platform's real behaviour differs from what the
code appears to promise, and the differences only surface in production. A write returns
`202 Accepted` and the record still never persists. A search "finds" a record that exists
only in a 60-second Redis cache. Elasticsearch and PostgreSQL are written by independent
Kafka consumers, so they silently disagree — and every dashboard built on ES inherits the
disagreement. None of this is in the official docs. Until now it lived in one engineer's
head and scattered scripts, and every new person paid the discovery cost again — usually
mid-incident, on a live campaign, with a partner waiting on numbers.

This repo moves that knowledge into Claude's context. When you work on anything HCM,
Claude starts from the team's accumulated answers — including the dead ends, so you do
not repeat them.

### What that looks like in practice

| Without the skill | With the skill |
|---|---|
| Spend an hour discovering that the tracer `requestBody` is not valid JSON (it is a Java toString with unquoted keys) | Claude uses regex extraction from the start — `json.loads` on tracer bodies is a documented dead end |
| Filter the project-task index on `Data.auditDetails.createdTime`, get 0 results, and conclude there is no data | Claude knows that field does not exist on task docs and filters on `Data.taskDates` — verified against the live index |
| Report a record as "persisted" because a search returned it, then watch it vanish a minute later | The Redis cache false-positive rule (`id=None` = cache hit, verify DB after the 60s TTL) is applied to every persistence claim |
| Investigate 18,000 ghost records from scratch — where to look, which index, which tracer query | The 6-phase ES-vs-DB RCA SOP walks the exact investigation, proven on Sokoto, Bauchi, Ondo, and Chad |
| Push recovered child records before their parents and hit `VALIDATION_ERROR` on every one | The dependency order (Individual -> Household -> Member -> Beneficiary -> Task) is a core rule |
| Build a children-treated report and have it disagree with the dashboard, without knowing which is right | Age-band rules (bands follow DOSE/product, not age), date-window-clip artifacts, and reconciliation cases for 10+ tenants are documented with the evidence |
| Trust the dashboard number and report it | House rule: report analysis-script numbers, never silently swap to dashboard figures — with the confirmed discrepancy cases that motivated it |
| Review an HCM diff with a generic checklist | Review runs against the five failure modes this team actually shipped and fixed (cache-before-DB, batch rollback, silent enrichment, ...) |

### What this means for the team

- **Onboarding drops from months to days.** A new engineer's first ghost-record
  investigation follows the same proven SOP as their fiftieth.
- **Fewer production mistakes.** The rules that prevent real incidents (never write test
  data to production tenants, verify past the cache TTL, parents before children) are
  applied automatically, not remembered under pressure.
- **Consistent, defensible numbers.** Reports follow the same formulas, age bands, and
  DQ checks across all campaigns and all team members.
- **Institutional memory survives turnover.** The knowledge is versioned, reviewed, and
  owned by the team — not by whoever happened to debug the incident.
- **It compounds.** Every new RCA, gotcha, or fix folded back in (see
  [Contributing](#contributing)) makes the next investigation faster for everyone.

## What's inside

Three skills. `superman` holds all the content, organized into five lanes; two entry-point skills route into it.

| Skill | Use it for |
|-------|-----------|
| `superman` | The complete knowledge base — all five lanes (context, dev, QA, data support, review). Invoke for anything HCM. |
| `dst` | Data support: ES-vs-DB reconciliation, ghost-record RCA, chain-recovery scripts, per-campaign findings, daily reporting, automation + Airflow |
| `hcm-engineering` | Platform context (entity chain, gotchas), Java code fixes (transformer/indexer/persister/validators), QA with the honest-verification methodology, and HCM-specific code review |

Also bundled: a sanitized **Postman collection** with 183 HCM API endpoints (`superman/references/context/assets/`) and generic **Airflow DAG patterns** (`superman/references/dst/airflow-dag-patterns.md`).

## Quick start

1. Clone and install (requires access to this private repo):

   ```bash
   git clone https://github.com/salaudeenn-egov/dst-claude-skill.git
   cd dst-claude-skill
   cp -r superman dst hcm-engineering ~/.claude/skills/
   ```

   On Windows (PowerShell):

   ```powershell
   git clone https://github.com/salaudeenn-egov/dst-claude-skill.git
   cd dst-claude-skill
   Copy-Item -Recurse superman,dst,hcm-engineering "$env:USERPROFILE\.claude\skills\"
   ```

2. Open Claude Code and try it:

   ```
   /superman
   ```

   Or just ask an HCM question — Claude auto-selects the skill from trigger words (tenant codes, "ghost record", "ES vs DB", "children treated", campaign names, ...).

3. If you use the Postman collection, import it and fill the collection variables (`authToken`, `username`, `password`, `basicAuth`) from your own secret store.

## Example prompts

- "Households exist in ES but not in the DB for tenant `so` — investigate."
- "Why is `householdId` null on these household-member docs?"
- "Build the Day 1 children-treated report for Bauchi."
- "Review this change to the transformer error handling."
- "Write a test plan for the validator-unbundle release."

## Repository layout

```
superman/                     Umbrella skill — the full knowledge base
  SKILL.md                    House rules, core data model, lane index
  references/
    context/                  Architecture, ES/DB data reference, gotchas, API index
    dev/                      Ghost-record and transformer fixes
    qa/                       Test plans, harnesses, verification methodology
    dst/                      RCA SOP, recovery scripts, campaigns, reporting,
                              automation, Airflow migration + DAG patterns
    review/                   Code-review checklist
dst/                          Entry point -> data-support lane
hcm-engineering/              Entry point -> context, dev, QA, and review lanes
```

## Contributing

This is a living knowledge base — it stays valuable only if findings flow back.

1. Branch, edit the relevant reference file, open a PR.
2. Every entry follows **Problem -> Approach -> Root cause -> Fix/verify** — record how the conclusion was reached and what was misleading at first, not just the bare fact.
3. Facts are point-in-time: date-stamp findings, and mark retractions explicitly rather than deleting them (wrong turns are part of the knowledge).
4. **Never commit secrets** — no passwords, tokens, API keys, Basic-auth headers, or usernames. Use `<REDACTED>` / `<REDACTED-USER>` placeholders and keep real values in your local `.env` or secret store. Run a secret sweep before pushing.

## Security

This repository contains **sanitized knowledge only**. All production credentials, auth/session tokens, usernames (`<REDACTED-USER>`), and Basic-auth headers have been replaced with placeholders (full credential + username audit: 2026-08-10). The bundled Postman collection uses `{{authToken}}` / `{{username}}` / `{{password}}` / `{{basicAuth}}` variables — fill them locally after import.

- **No credentials in skills — ever.** Real values live only in your deployment `.env` / secret store and must never be committed (enforced by `.gitignore`, backstopped by `SECURITY.md`). Run a secret sweep before every push.
- **Encrypt saved data at rest.** Campaign extracts and exports contain beneficiary PII — keep them on an encrypted volume or in password-protected archives, delete them when done, and never commit raw campaign data here (aggregate findings and methods only).
- If you find a secret in the files or history, follow `SECURITY.md`: report it, remove it, rotate it.
- This is internal engineering knowledge — the repository must stay **private**.

## Maintainer

Salaudeen (`salaudeen.n@egov.global`) — questions, access requests, and PRs welcome.
