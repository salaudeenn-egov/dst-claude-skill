# Chad SMC Beneficiary-ID RCA — the generated-ID defect, fully reproduced (2026-08-07, quantified 2026-08-10)

The deepest single-day investigation in this KB: from "why 304k INVALID_ID tracer errors" to a
live end-to-end reproduction in production, with the defect quantified at ~2.5x the visible
error count. Evidence pack: `Downloads\Chad_SMC_InvalidID_Reproduction_Evidence.md`.
Memory source: `project_ch_smc_idpool_rca.md`.

## Problem (and the misleading first reads)
Chad SMC Cycle 1 (tenant `ch`, 20-29 Jul 2026): 451,301 tracer errors, 304,831 (67.5%)
INVALID_ID, all on `/beneficiary-idgen/id/id_pool/_update` ("ID is blank or does not exist in
the repository: <7-digit>"). Misleading first reads, each overturned in sequence:
1. "Server id_pool persist lag, resolved by itself 26 Jul" (the original RCA) — WRONG.
2. "App-side _update calls failing" — WRONG (app never calls _update).
3. "Client-generation was the known requirement" — WRONG (design said fallback-only; the team
   lead pushed back, which triggered this verification).
4. "Errors stopped = problem gone" — WRONG (errors stopped because misses stopped; generation
   continued silently).

## Approach (the method that worked — reusable)
1. **Get the REAL codebase from the implementing dev** (first RCA cited the wrong repo — always
   confirm repo+branch+APK build with whoever shipped it). Chad SMC = github.com/HCM-NIGERIA-GO-DEEP/
   health-campaign-field-worker-app branch `feat/chad-smc`; APK CHAD-SMC-PROD-2.2.75; cloned to
   `D:\services\health-campaign-field-worker-app-chad-smc`.
2. **Code trace** of the full ID lifecycle (subagent fan-out): where IDs are downloaded, assigned,
   generated, and reported.
3. **Live device test** with prod APK + UAT user (<REDACTED-USER> / <REDACTED>, tenant ch,
   chad-hcm.digit.org) while streaming `adb logcat -s flutter:V` — three rounds: happy flow,
   offline-empty-pool, fresh-install online. Key race: airplane-mode ON in the boundary->home
   transition beats the silent pool auto-download.
4. **DB forensics** on `ch.id_pool` + `ch.id_transaction_log` (the append-only dispatch log is the
   attribution tool — id_pool.lastmodifiedby is overwritten on assign).
5. **Prod pod logs** (beneficiary-idgen, 3 replicas in `egov` ns on ng-central-prd) to catch the
   rejection live.

## Root causes (all in APK 2.2.75 / branch feat/chad-smc)
1. **Pool gate disabled**: `function_registry.dart:3455` `hasMinimumBeneficiaryId` hard-coded
   `return true` ("TEMPORARY OVERRIDE" comment; commit e0671efbb 20 Jun, msg literally says
   "generation of 7 digit beneficiary ID bypassing the id pool"). Download prompts can never
   appear; registration never blocks.
2. **Unconditional fallback**: `screen_builder.dart:391-396`
   `'UNIQUE_BENEFICIARY_ID': latestBeneficiaryId ?? IdGen.instance.generateRandomAlphanumeric(7)`.
   No offline check, no pool-exhausted check, no uniqueness check; format = 7 digits (reads
   uniquenumbercount/uniquenumbertype from the SAME server config as pool IDs -> generated IDs
   are format-identical to real ones, hence collide and are unauditable).
3. **Fragile silent download**: home.dart:154-225 one attempt per app cold-start (online AND
   count==0 only), failures swallowed (debugPrint); screen-entry loader fetches ONE batch of 10;
   no retry anywhere; no operational logging of pool health at all.
4. **Daily dispatch quota per user+device** (captured verbatim): "ID generation limit exceeded:
   Daily limit for user ... Remaining ids: 0". A data-clear/reinstall loses the local 1,500 but
   the server still counts them -> that device gets NOTHING until next day, silently -> every
   registration that day generates.
5. **No server-side guard**: `IdPoolValidatorForUpdate` (beneficiary-idgen) checks ONLY that the
   ID exists in the pool — no dispatched-state check, no ownership check. A generated ID that
   collides with any pool row is silently marked ASSIGNED.

## The mechanism (who calls what)
The APP NEVER calls id_pool/_update (verified across all ~200 fork branches; UniqueIdOpLogManager
is UnimplementedError everywhere; remote repo is dispatch/search-only; the local ASSIGNED mark is
SQLite-only). The **individual service** calls it server-side after each synced registration
(client class `BeneficiaryIdGenService` in health-services-common; wired in individual
application.properties). beneficiary-idgen validates synchronously, pushes valid records to the
TENANT-SPECIFIC topic `ch-update-id-pool-status` (deployed build = branch
`beneficiary-igden-tenant-specific` of health-campaign-services), consumer persists ASSIGNED.
Miss -> ERROR log + (eventually) tracer doc. So: the app is the trigger, the individual service
is the caller, idgen is the rejector — and the app dev saying "we don't call that API" is true.

## Quantification (ch.id_pool / id_transaction_log, verified by query)
- Pool: ASSIGNED 1,466,780 / DISPATCHED 6,406,660 / UNASSIGNED 903,193 = 8,776,633; 8,776,121
  are 7-digit of a 10M space = **87.8% coverage**. First digits 0-7 SATURATED (1M each),
  8=55.7%, 9=22.0% (cross-check: 0.8+0.0557+0.022 = 87.8% ✓). => a generated ID starting 0-7
  (80%) collides with CERTAINTY; only 8/9-starters can still error. ~1.2M numbers left in the
  space (format change imminent anyway).
- Pool growth: Jul1 1.98M ... Jul9 cum 4.96M (49.6% at campaign start), Jul23 +1.73M (66.9%),
  Jul25 +2.09M (87.8%). Errors collapsed after Jul 25 BECAUSE coverage rose — the defect went
  silent, not away.
- **392,829 ASSIGNED-never-dispatched** (join vs id_transaction_log) = hard floor of silent
  collisions. Estimated campaign total generated: **~750k-850k** (misses/day / miss-rate/day);
  ~450k-550k silent. ~1 in 3 ASSIGNED ledger rows may be a collision, retroactively
  unattributable (lastmodifiedby overwritten; rowversion unreliable).
- Duplicate-children proof: ID 3975695 rejected 22 Jul (generated, didn't exist), created by
  batch 23 Jul, dispatched to another CDD, used for a different child 25 Jul. Rejection does
  NOT remove the ID from the first child's saved record => BOTH branches (miss and hit) create
  duplicates.

## The reproduction (the clincher)
Round 2: 28 offline registrations; candidates 9469242/8490579 confirmed ABSENT from id_pool
pre-sync; synced 14:02:30 UTC; 26 siblings ASSIGNED same second; and at **14:02:31.222** pod
beneficiary-idgen-84f4bdd4d4-kghr2 logged
`ERROR IdPoolValidatorForUpdate : ID is blank or does not exist in the repository: 8490579` —
the exact campaign validator+message, triggered by our controlled test. Full chain on one ID:
device log -> DB absence -> prod ERROR log.

## Duplicate cross-check — "how many more like 3975695?" (2026-08-10)

**Problem.** Kaviyarasan P's response to the 3975695 case (in the Slack thread with Sathish P /
Sathish Ramadurai / nikhilesh equidhi): "there is an explicit validator in idgen to block this —
check how many more entries are similar; I will assign someone to verify."
Misleading premise to rebut first: the validator DID fire — the 304,831 INVALID_ID errors ARE
the validator. It blocks only the ledger write, not the registration (child persists with the
generated ID), and rejected numbers are invisible to the pool generator/dispatcher, which
re-issues them. The validator cannot block this duplicate class by design.

**Approach (reusable pattern: tracer-population -> DB cross-check -> classify).**
`Downloads\ch_dup_id_crosscheck.py` — one standalone script the verifier can re-run end to end:
1. Parallel sliced ES scroll (6 slices) of all INVALID_ID `_update` errors for tenant ch;
   regex the rejected ID from errorMessage, the rejecting user from requestBody userInfo;
   aggregate per DISTINCT ID keeping FIRST error ts + the set of rejecting uuids/usernames.
2. LEFT JOIN ch.id_pool (4 parallel read connections, `id = ANY()` chunks of 10k; column
   discovery first so a wrong DB host fails loudly, not wrongly).
3. Classify per ID: absent | pool row createdtime <= first-error (anomaly bucket — expect 0,
   validates the epoch-ms assumption) | created AFTER rejection, split by status and by
   pool-user-vs-rejecting-user.
4. Resolve pool-user uuids -> usernames: ch.individual (username IS populated on chad, unlike
   kd; givenname mirrors username; mobilenumber is enc-service ENCRYPTED — exclude) -> eg_user
   -> ch-user-sync-index-v1 fallback. 100.00% resolved (1 uuid of 155,116 rows).

**Results (full tracer window; campaign-window Jul 20-31 in parens where it differs).**
306,929 distinct rejected IDs -> 92,578 never entered the pool; 214,351 re-created AFTER their
rejection, of which:
- **21,179 (20,119) ASSIGNED by a DIFFERENT CDD = realized duplicates** — two children, one ID.
  Median 1.8d rejection->pool-row, 3.8d rejection->consumption by the other CDD (max 38d).
  The Jul 23 batch re-issued 16,434 of them, Jul 25 another 4,638 — the SAME batches that made
  the error volume collapse manufactured the duplicates; 516 were still being consumed on Aug 1.
- 11 ASSIGNED by the same erroring user (late retry — NOT duplicates, keep separated).
- 133,937 (128,363) DISPATCHED = at-risk (number sits unconsumed on another CDD's device; becomes
  a duplicate on late sync or next-cycle reuse). 59,224 UNASSIGNED = quarantine before dispatch.
- 9,961 IDs rejected from >1 distinct device = phone-vs-phone collisions (duplicates with NO pool
  involvement; overlap with the 21,179 = 529 -> union 30,611 distinct multi-child IDs).
Validation that survived scrutiny: anomaly bucket = 0; zero rows where consuming username equals
a rejecting username (uuid- and username-level checks agree); 3975695 named both sides
(atrone8 rejected 22 Jul 11:24 -> hille-houdjadj9 consumed 25 Jul 13:03; tracer doc 5bbfe8ed-…).

**Actionables handed over:** quarantine the 59,224 UNASSIGNED; block consumption of the 133,937
DISPATCHED; exclude all 306,929 from future pool generation; remediate the 21,179. Deliverable
format that landed well: Excel with humanized sheet names (Summary / "Same ID Used By Two CDDs" /
"ID Sent To Another Device") + per-row usernames AND uuids + all three timestamps, plus the
script for independent re-run. Keep message numbers window-consistent with the attached file —
the verifier's first act is counting rows.

## Remediation design for the 21,179 duplicates (2026-08-12, proposed)

**Problem.** "How do we fix the 21k — and based on what should the new ID be generated?" Two real
children share each ID; a keeper rule and a re-issue source are needed before any script.

**Keeper rule (ledger-consistency principle):** the POOL-CONSUMER (child 2) keeps the ID — the
ledger already records it as legitimately dispatched+ASSIGNED to that CDD; child 1's use was
REJECTED by the validator, so the ID was never validly theirs. Don't fight the ledger with
"first use wins" — that would require rewriting id_pool too. Exception (the 9,961 phone-vs-phone
pairs, neither leg legitimate): if the number was ALSO dispatched to a third CDD (e.g. 6387648 ->
adescu2), retire the number entirely and re-issue BOTH children; else earlier-synced keeps.

**Re-issue source:** NEVER the 7-digit space (87.8% exhausted; the colliding format). A dedicated
server-side remediation series from idgen in a visibly distinct format (prefix / 8+ digits),
dispatched to a system remediation user so the ledger records it — align with the format-change
fix so remediation IDs are collision-proof and auditable as re-issued.

**Finding child 1's individual record (the practical blocker):** identifierId is encrypted, BUT
enc-service encryption is deterministic for searchable fields -> encrypt the plaintext ID via
enc-service and search individuals by ciphertext; disambiguate the two hits with the detail
sheet's users+timestamps (child 1 = rejecting CDD near first_error_ts; child 2 = consuming CDD
near consumption time). PILOT on one ID (3975695) to confirm determinism before scripting.

**Execution:** freeze first (quarantine/exclusion), map records from the detail sheet, update
child 1 via the individual `_update` API ONLY (keeps DB+ES+encryption consistent; never direct
DB edits), checkpointed batch with a 10-ID dry run, log old->new->individual mapping in a
remediation audit table, verify by re-running the cross-check (expect zero) + encrypted-search
spot checks.

**Program decision that gates everything:** were beneficiary IDs written on PHYSICAL CARDS? If
yes, digital re-issue breaks card->record lookup next cycle — cheaper alternative is keep both
records + duplicate-flag/alias on child 1 and resolve at next-cycle contact. If digital-only,
full re-issue before the next cycle. Put this question to the program team, not engineering.

## Reporting artifacts & packaging lessons (2026-08-12)

- Two-document pattern that worked: (1) findings report = polished pre-read (exec summary,
  method table, classification table, recommendations); (2) investigation-narrative RCA =
  submission-grade, chronological story ("why does the tracer have so many errors" -> each
  investigation step) with REAL evidence embedded in shaded monospace blocks (code excerpts with
  commit msgs, device-log fingerprints, verbatim quota response, pod-log smoking gun, tracer-doc
  excerpt, pool-coverage math). Builders: build_chad_smc_beneficiary_id_findings.py +
  build_chad_smc_rca_investigation_report.py (Downloads).
- **Never put shape placeholders ("rising/peak/falling") in an evidence block** — the user caught
  it immediately. Every number block must be real values that SUM to the claimed total
  (daily INVALID_ID: 51,879/66,876/87,226/41,468/29,370/20,011/2,880/2,675/2,416/30 = 304,831
  exactly; source sheet Daily_by_ErrorCode of ch_smc_tracer_rca_20260720_to_date.xlsx).
- Keep the three-number funnel straight when presenting: 304,831 ERROR DOCS -> 306,929 DISTINCT
  rejected IDs (dedup + pre/post-window tails) -> 21,179 CONFIRMED DUPLICATES. Conflating them
  invites "is this real? 300k vs 21k?" pushback.
- Row-level evidence export: ch_smc_invalid_id_errors_rowlevel.xlsx (304,831 rows: docId,
  timestamp, extracted ID, CDD username/uuid, errorMessage) — openpyxl write_only mode handles
  300k+ rows fine. Gotcha: a CSV open in Excel gives PermissionError on read — retry after close.

## Additional durable findings
- **egov-tracer-error-details is NOT real-time**: stalled 5+ hours during testing (newest doc
  frozen at 12:23 UTC while pod errors kept occurring). Kibana error monitoring can show a
  frozen picture; error docs queue in Kafka. Check egov-indexer / consumer lag before declaring
  "no errors".
- **identifierId is ENCRYPTED in ch-individual-index-v1** (`316090|<base64>` enc-service format)
  — plaintext beneficiary-ID search is impossible in the individual index; only id_pool and
  tracer messages hold plaintext. Also verified: the beneficiary ID is NOT denormalized onto
  ch-project-task-index-v1 (query_string probe = 0 hits) — so campaign reporting/dashboards are
  unaffected by the duplicates, but never claim "no data impact" absolutely: the ID lives on
  individual records (encrypted) and its uniqueness guarantee is what's corrupted.
- **idgen `_update` tracer requestBody IS valid quoted JSON** (json.loads works) — the
  "tracer bodies are unquoted pseudo-JSON" gotcha applies to entity-create bodies only.
- **ch.individual has usernames populated** (unlike kd where the column is NULL for CDDs);
  `mobilenumber` in ch.individual is enc-service encrypted — don't put it in deliverables.
- **3 idgen replicas** — a rejection lands on whichever pod consumes that partition; ALWAYS
  search all replicas (`kubectl logs` each, or a label selector).
- 2,074,059 pool IDs DISPATCHED to the single UAT user (load-test residue or attribution bug —
  open question to platform team).
- Interrupted first bootstrap (airplane mid-sync) bricks the app: project saved without
  projectType -> `AppException 'No beneficiary type is selected'` on every home load; only
  Clear Data recovers.
- Detection heuristics for generated IDs: per-record = "assigned but never dispatched to that
  user" (id_transaction_log join); aggregate = legit IDs cluster in each device's contiguous
  dispatched blocks (dispatch is sequential), generated ones scatter uniformly. Value-shape
  alone can't distinguish (same format); leading zero is a generator hint but pool has 0xxxxxxx
  too.

## Fixes recommended (proposed to team, thread with Sathish P / nikhilesh equidhi)
App: restore the gate; retry+surface download failures; gate fallback to offline+exhausted;
DISTINCT format for fallback IDs (prefix/length) so they can't collide and stay auditable;
show pool count. Platform: dispatched-state + ownership validation in _update; rethink daily
quota vs reinstall (auto reFetch of allocated IDs); fix tracer indexing lag. Data: remediation
decision for ~400k+ corrupted ledger rows + duplicate beneficiary IDs before next cycle.

## Ops recipes proven this session
- Live app QA loop: `adb logcat -s flutter:V` + grep `UPDATE_IDENTIFIER_STATUS|No UniqueIdPoolModel`
  (fingerprint of a generated ID) — the HTTP interceptor logs [REQUEST]/[RESPONSE] for every call
  (incl. full dispatch payloads); `pidcat.py org.egov.chad` for human-readable colored view.
- Read-only prod k8s: set KUBECONFIG/AWS env + `kubectl --context <arn> ...` per command (never
  `use-context` — it writes the kubeconfig; and env vars don't persist across tool invocations).
  ng-central-prd holds per-state namespaces incl. `chad`/`chad-prod`; shared services (idgen 3
  replicas, individual) in `egov`/`health` ns.
