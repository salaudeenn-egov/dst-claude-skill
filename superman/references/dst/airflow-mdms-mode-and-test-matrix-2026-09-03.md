# MDMS mode live + the full edge-case test matrix (2026-09-03)

Continues `airflow-e2e-bringup-2026-09-01.md`. That got the FIRST complete run;
this records MDMS mode going fully live (schema created, sync working, Kafka →
persister → table closed), a deliberate 15-case edge/validation test run against
live infra, the findings, the one code fix that resulted, and an alert-copy
refresh. Deployment: Azure unified-dev (see prior doc for cluster facts).

---

## MDMS mode: every piece now proven live

- **Config schema created** on dev: `airflow-configs.dst-campaign-report-config`,
  FLAT (24 props = 22 sheet cols + rowIdentity + deploymentGroup), x-unique
  rowIdentity, required minimal. Created via
  `POST /egov-mdms-service/schema/v1/_create`, read back byte-identical. The
  five-point pre-create alignment check (schema code == mdms.py DEFAULT_SCHEMA_CODE;
  envelope keys == _ENVELOPE_KEYS; every field the code reads exists; no dead
  fields; x-unique == row_identity's key) is the reusable gate — run it before any
  schema create.
- **dst_config_sync working** — all four verbs observed live: CREATE (new rows),
  UPDATE (edited), DEACTIVATE (rows removed from sheet -> isActive:false in MDMS),
  REJECT (bad rows, with a per-row reason alert). The sync pod mirrors sheet ->
  MDMS; the scheduler then reads MDMS via get_active_rows_from_mdms -> entry_row().
- **Kafka -> persister -> table closed.** First event landed once KAFKA_BROKER was
  corrected (see below); `[finalize] outcome recorded via kafka`; topic offset
  moved 0 -> 1 -> N. Persister (`egov` ns, NOT backbone) is subscribed to
  `save-dst-report-metadata`. Table `dst_report_metadata` is on the MANAGED Azure
  Postgres `unified-dev-server.postgres.database.azure.com/unified-dev-db` (creds
  in k8s secret `db`, unreadable by the read-only kdev account) — NOT the
  in-cluster `backbone-airflow-postgresql` (that's Airflow's own metadata only).

## The cluster-mismatch credential saga (all the same root cause)

Every "it worked yesterday" failure this session was ONE thing: dst_config held
values from the AWS-EKS unified-dev, wrong for the Azure deployment the Airflow
actually runs on. Found and fixed in sequence, each surfacing only after the
previous cleared:

| Symptom | Wrong value | Correct (Azure) |
|---|---|---|
| `ModuleNotFound gspread` | (packages absent) | sri.padma added 11 to the Airflow image |
| ES DNS fail | `elasticsearch-master.es-upgrade` | `elasticsearch-master.backbone:9200` |
| ES 401 | stale ES_PASS | read from `elasticsearch-master-credentials` secret |
| Slack invalid_auth | stale SLACK_TOKEN | current bot token |
| Kafka DNS fail then silent | `release-name-kafka.kafka-kraft.svc...` | `kafka-kraft.backbone:9092` |
| MDMS InvalidURL | Helm's ambient `MDMS_SEARCH_ENDPOINT` (no leading slash) concatenated raw | code lstrip fix + `/egov-mdms-service/v2/_search` in dst_config |

**Lesson: on a NEW deployment, do not assume ANY service URL/secret carries over.
Read each from the cluster the Airflow actually runs in.** The pattern was
invisible because each fix only revealed the next layer.

**Also: re-pasting the dst_config file to fix ONE key silently reverted the others
to stale values** (fixed Kafka -> brought back stale ES_PASS/SLACK_TOKEN). Build
the WHOLE correct config once, paste once. And "mode-dependent" bugs were a red
herring — ES/Slack are resolved the same way in both modes; the correlation was
with WHICH config version was live, not with sheet-vs-mdms.

## The 15-case test matrix (live, against real infra)

Staggered rows on the dev tab, +2 min apart, distinct campaign numbers.

**BEHAVIOUR (verified correct)**
- internal-only (no partner) -> mode `both` — DELIBERATE (compute_trigger_slots:62
  `if not partner: return [(t,"both")]`, backward-compat). A TRUE internal-only slot
  needs a partner time that DIFFERS.
- partner-only -> `partner`, DQ-stripped.
- both same time -> ONE run, mode `both`, TWO docs.
- both diff times -> TWO runs, `internal` + `partner`.
- CDD-missing guard fires on FAKE campaign numbers (no staff match) but NOT the real
  one (7 CDDs) — a real, useful behavioural distinction.

**VALIDATION**
- empty tenant, bad time `25:99` -> REJECTED at sync (visible Slack alert).
- campaign_days=0 -> HARD FAIL at run (divisor validation).
- missing target -> did NOT hard-fail: `download_target_book` tries CONVENTION
  fallback names, found `dev_target.csv`, ran degraded. So a wrong `target_file`
  does NOT fail if a convention-named book exists — the folder must be truly empty.

**THE ONE GENUINE GAP (fixed):** start > end (swapped dates) -> SILENT no-op.
Verified across three layers: validate_row (sync) had no ordering check -> entered
MDMS; scheduler `campaign_window_contains` (inverted range contains no dates) ->
slot silently dropped; run `_validate_row` would catch it -> never reached. Net:
syncs clean, shows active, never runs, no report/error/alert. THE only silent
failure in the whole matrix.

**SCHEDULING**
- staggered slots carry correct slot-time in the run id, but FIRE batched at the
  5-min `*/5` tick (+2 min spacing does not stagger wall-clock). Genuine wall-clock
  staggering needs times across DIFFERENT ticks AND enough lead time to survive the
  MDMS sync delay — a row added with times close to "now" has both times PAST by the
  time the ~10-min sync mirrors it, so they all fire on one tick. Rule: in MDMS mode
  give edits ~15-min lead time or slots batch.
- rejected rows correctly never fire.

## Verified-deliberate (looked like bugs, are not)

- **Orphan MDMS entry after a date edit.** `protection_key` (mdms.py:136)
  deliberately matches tenant+campaign+cycle, DROPPING campaign_start, so a start
  typo does NOT deactivate a running campaign (reject-and-keep). Cost: changing
  campaign_start creates a NEW identity and leaves the old entry ACTIVE — a
  duplicate a human must clean. Safe side of the trade, by design.
- **Mode-change re-fire in MDMS mode.** Clearing a partner time flips a slot
  internal->both -> new run id -> run-id dedup can't catch it -> re-fires. MDMS mode
  has NO retime guard by design (zero sheet access on the scheduling path); sheet
  mode's guard covers `{mode,"both"}` and would suppress it. Documented trade-off,
  confirmed live. In practice only bites when config changes intra-day (i.e. our
  testing) — real ops don't retime mid-day.

## The one code fix this earned

`pipeline/mdms.py::validate_row` — added `campaign_start <= campaign_end`:
turns the swapped-date SILENT no-op into a VISIBLE sync rejection (same Slack
warning path as empty-tenant / bad-time). Verified: swapped -> rejected with
reason, valid -> accepted. One-line-class fix; the only genuine hole the matrix
found. Everything else was correct-by-design.

## Alert copy refresh (this session)

Three alert types restyled to a consistent, scannable Slack shape (bold headers,
sectioned, dash bullets, NO emojis/decorative unicode per the doc-style rule):
- `run_history.py` degraded -> "*Report incomplete* — {state} · {campaign} · Day N"
  with *What is missing* / *Before you share it* sections (diagnostic content kept
  verbatim, only the wrapper restructured).
- `alerts.py` hard-fail -> header "Report failed — {state}"; ALSO fixed the bare
  "Failure" detail (when no exception threaded) to point at the run id/task log.
  The failure alert already used Slack BLOCKS (header + field grid + code block +
  context) — it only LOOKED bad pasted as plain text; blocks flatten on copy.
- `dst_config_sync.py` -> "*Config sync — N row(s) rejected* · {group}" with dash
  bullets per rejected row.

## Ambient-Helm-env audit (do this on any new deployment)

Every env the code reads, crossed with what the platform chart injects. Two more
found beyond ES/Kafka/MDMS: `TENANT_ID` (chart default flows in if not in
dst_config) and `IS_CENTRAL_INSTANCE_ENABLED` (if a chart sets it true, the
producer prefixes the topic to `{tenant}-save-dst-report-metadata` and the bare
persister subscription consumes NOTHING, silently). Both pinned in dst_config env
now. MDMS gotcha confirmed live: criteria key `schemaCodes` (plural) is SILENTLY
IGNORED and returns the whole tenant; singular `schemaCode` filters — the code
uses the singular (correct).

## Uncommitted at session end (fork, branch dst-self-contained-dags lineage)

.gitignore (pycache), pipeline/mdms.py (flat entry + URL lstrip + start<=end),
common/dst_kafka_status.py (broker-state logging), common/run_history.py +
common/alerts.py + dst_config_sync.py (alert restyle),
platform/dst-campaign-config-schema.json, common/slots.py (duplicate-cumulative
warning — see next section). All under dags/ -> git-sync, no rebuild.
NOT committed (user commits). Dev test rows still active on the sheet — deactivate
so they stop firing daily.

## Cumulative report double-fired — the dedup model, end to end (2026-09-03 cont.)

**Problem.** TC14 mop-up posted TWICE the same day — two DIFFERENT Google Doc
URLs (so two genuine pipeline runs, not one run to two channels), both
`cumulative` ("Day 4 of 8 / Campaign Target"). User: "i see both execute ...
how it check codebase".

**The two-layer dedup (traced in code, no live access needed).**
- Layer 1, the ONLY layer the cumulative slot gets: the deterministic
  `trigger_run_id` `dst_{group}_{tenant}_{campaign_key}_{mode}_{date}_{time}`
  (slots.py:54) + `TriggerDagRunOperator(skip_when_already_exists=True)`. Same
  slot -> same id -> never fires twice.
- Layer 2, the backstop when the id CHANGES: the retime guard
  (`run_history.has_report_since`). **Skipped entirely in MDMS mode** by design
  (zero sheet/DB on the scheduling path).
- The cumulative slot is always `mode=cumulative`, always `23:59`, one per row
  (slots.py:96-97). Its run id varies on exactly ONE thing: `campaign_key` =
  campaign_number -> else project_type_id -> else campaign_name, + cycle_index
  (schedule_utils.py:12-24).

**Root cause.** Two DIFFERENT run ids were generated for the same tenant+date,
which (date/time fixed at mopup_date/2359) means `campaign_key` differed. Two
ways, BOTH MDMS-mode/test-only: (a) two ACTIVE MDMS entries for TC14 (the
documented orphan-on-identity-edit hazard — old entry stays isActive:true), or
(b) one entry whose campaign_name/number/cycle was edited between two 5-min
ticks -> new key -> re-fire, no retime guard in MDMS mode to catch it. In real
ops neither happens: a row is created once with a stable campaign_number and not
edited mid-campaign, so the deterministic id blocks the repeat. Confirmed the
double-post is NOT the internal-vs-partner "different times" case — those are
DAILY slots, never cumulative.

**Fix (observability only, no behaviour change).** Added
`slots._warn_on_duplicate_cumulative(due)` — after building the due list, if two
active rows yield a cumulative slot for the same (tenant, slot_date) with
different campaign, log a loud warning naming both and telling the human to
deactivate the stale row. Did NOT suppress the second fire: a tenant may
legitimately run two campaigns sharing a mop-up date, and suppression is a design
reversal. The real remedy is data hygiene (deactivate the duplicate/orphan
entry). A HARD guarantee would need the retime/history guard enabled for the
cumulative slot in MDMS mode (reverses the no-DB-on-scheduling-path design) —
left as a user decision, not applied.

## internal@11 / partner@12 routing — verified end to end (correct, no change)

**Problem/question.** "internal target 11, partner 12 -> internal posts 11,
partner posts 12, NOT both". **Already correct.** Full chain checked:
- `compute_trigger_slots` (schedule_utils.py:59-68): internal=[11:00],
  partner=[12:00] -> `[(11:00,"internal"),(12:00,"partner")]`. `both` results
  ONLY when a time is in BOTH cells, or partner_report_times is EMPTY
  (backward-compat -> every internal slot posts both).
- scheduler `expand_kwargs(all_due)` -> each due dict (trigger_run_id + conf) is
  its OWN dst_campaign_run; `skip_when_already_exists` dedups on the id.
- run DAG (dst_campaign_run.py:90) `execute_campaign(conf["row"],
  conf.get("mode","both"))` -> passes mode through.
- runner (campaign_runner.py:340) `mode="both" if is_cumulative else mode` — the
  `both` override applies ONLY to the mop-up; daily slots keep internal/partner.
- notify.py:341-342 `do_internal = mode in ("both","internal")`,
  `do_partner = mode in ("both","partner")` -> internal->slack_channel only,
  partner->slack_channel_partners only.
If both channels appear to post at one time, check the ROW: partner cell blank,
same time in both cells, or `slack_channel`==`slack_channel_partners`.
Each slot is its own dag_run -> its own Word doc, Drive upload, and run event
(internal report keeps DQ, partner report strips it).

## Setup guide delivered as .docx (not .md)

User wanted the operator guide as a shareable Word doc, "i dont need the md".
Built `D:\DST\DST_Campaign_Report_Setup_Guide.docx` from the 9-section SETUP.md
via python-docx (120 paras, 2 tables, Consolas code blocks, navy headings, no
emojis/decorative unicode/bookmarks per doc-style rule). SETUP.md kept in repo
for developers unless user says delete.

## Alert restyle: one builder + severity colour bar (2026-09-04)

**Problem.** User pasted a failure alert that read as a flat run-on ("DST report
failed - Dev Campaign TC10 Days=0 ... Failure ...") and asked for "style and
css". Two facts: Slack has NO css — the nearest thing is an ATTACHMENT COLOUR
(a coloured left bar); and Block Kit FLATTENS when copied out of Slack, so it
only looked bad pasted. The live cluster was also running the OLD committed
alerts.py (uncommitted restyle not deployed).

**Approach/Fix.** One shared builder `alerts.build_alert_blocks(header, lead,
fields, detail_label, detail, detail_code, context)` -> header + lead line +
2-col field grid + divider + labelled detail + muted context. `_post` gained a
`color` arg; when set, blocks go INSIDE `attachments:[{color, blocks}]` (blocks
at top level would render a second bar-less copy). Three severity colours:
`COLOR_FAILED #D64541` (red, hard fail), `COLOR_INCOMPLETE #E5A50A` (amber,
degraded / config-sync), `COLOR_INFO #5B6B7B` (slate). All three alerts now use
it consistently: failure (alerts.py, red), incomplete (run_history.py, amber),
config-sync nothing-deactivated / rows-rejected (dst_config_sync.py, amber).
`send_slack_warning` gained `blocks=`/`color=`. Side fixes: the bare "Failure"
detail now shows the real reason (`ValueError: campaign_days is '0'...`); the
config-sync rejected alert now passes `group_name` so its token resolves from
`dst_secrets_<group>`. Files uncommitted; WON'T show live until committed +
git-synced (bars are invisible on the old committed code).

## Live 20-case test matrix on the dev tab (2026-09-04)

Wrote 19 rows / 20 slots to sheet `1nykkqDdFWWe1kjwrNBVEf9lkn3rwoga6xOSXWWMnC0I`
tab `dev` (22 cols; the dev tab has TWO extra cols vs the older list:
`dose_index_filter` @14, `secondary_product` @20; no `project_type` col). Staggered
+5 min 06:15-07:50 UTC on the */5 ticks, channel `C0ALY7EQVSR`, unique
campaign_number CMP-DEV-QA-NN per row. Coverage: 8 happy (both-same-time, int/par
split, SPAQ/AZM/ITN, task_campaign_filter, cycle02), 1 amber (missing target), 5
red (days=0, bad drug, cycle=abc, mopup<start, days!=span), 6 fire-nothing edge
(empty tenant, bad time 25:99, swapped dates, past window, active=FALSE, no
times). Sheet STRIPS leading zeros ("06:15"->"6:15") but parse_report_times
re-normalises, so slots still match. Timing note saved
`D:\DST\DST_Test_Matrix_Timing.csv`. Connected via the embedded service account
(`dst-report@digit-health-75ed0.iam.gserviceaccount.com`) from
`D:\DST\dst_config.dev.json` google_credentials_json (spreadsheets scope). NOTE:
these fire daily until deactivated; dst_config.dev.json still holds the LIVE SA
key (rotate + delete when done).

## Full technical+user documentation set (2026-09-04)

Mirrored the user's reference `Downloads/HCM Reports Automation using Airflow
(2).docx` structure (ONE file, two Title-separated docs: a 12-section User Guide
+ a 12-section Tech Doc, H1-H4, tables, code). Built
`D:\DST\DST_Reports_Automation_Documentation.docx` (python-docx, TITLE:->Word
Title+page-break, #### support). CRITICAL adaptation: our DST design is NOT the
HCM pod+image+FileStore model — documented git-sync self-contained code (no
per-report image), sheet-vs-MDMS modes, Google Sheet+Drive+Slack+Groq. Deep
sections added from code: 8.3 full `dst_report_metadata` DDL + 2 indexes, 8.4
four ops SQL queries, 9.6 the persister yml (serviceName dst-report-automation,
INSERT ... ON CONFLICT (event_id) DO NOTHING idempotent, 15 jsonMaps, central
`fromTopic` comma-list), 13. add-a-deployment-group (dst_groups env override:
null REMOVES a key -> tenant-prefixed, "" -> un-prefixed; dst_secrets_<name>
name must match group name). Also produced the split pair
`DST_Campaign_Report_User_Guide.docx` + `..._Technical_Guide.docx`. Rendering
delegated to background general-purpose agents (fast, keeps context clean) with
a reusable `build_docs.py` converter under the job tmp.

## Google Docs "Tabs" cannot be created via API (2026-09-04)

User asked for one Google Doc with two tabs. Verified against
developers.google.com/workspace/docs/api/how-tos/tabs: the Docs API can WRITE
INTO existing tabs (target by `tabId`) but has NO request to CREATE a tab —
tabs are a UI-only creation. So no programmatic two-tab doc from scratch; the
empty tabs must be made once in the Docs UI, then content inserted by tabId.
Fallbacks offered: user creates 2 empty tabs then API fills them; or one Doc /
one .docx with two sections. User chose "one Doc, two sections".
