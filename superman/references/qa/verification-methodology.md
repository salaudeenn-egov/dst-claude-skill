# Verification Methodology — How to verify honestly (reusable QA playbook)

The user's standing demand: "be honest and reliable on your claim." Report only DB-search-verified facts, never overclaim. This drove the entire method below. Read this before reporting ANY persist or relation result.

## The core loop
claim -> DB-verify past TTL -> if odd, isolate + tiebreak -> if still odd, separate harness-bug from real defect -> report the honest (even messy/contradictory) result and say what is reliable vs noise. Never present a single run or a cache hit as proof.

## Rule 1 — Verify a "persisted" claim AFTER the 60s Redis TTL
A create does `putInCache` in Redis with a 60s TTL. A `_search` seconds after create hits the REDIS CACHE and returns a row with `id=None` — a FALSE-POSITIVE "persisted". During persister lag the DB write never happens and after the TTL the row VANISHES (this is the live-reproduced ghost mechanism).
- ALWAYS re-`_search` by clientReferenceId AFTER 90-100s (past the 60s TTL) to confirm DB commit.
- `id=None` in a search result (no server id) is the tell it is cache-only, not committed. A populated server id = committed.
- Concrete correction this caused: U4/U5 tasks were reported "persists", then re-verified post-TTL as NOT IN DB (id=None was the tell). "Accepted/not-rejected" (202) stayed TRUE; "persists" was FALSE. The task actually failed on a missing-`address` enrichment NPE.

## Rule 2 — Prefer the authoritative embedded back-fill signal over reverse-search
To prove a relation, fetch the CHILD by its own clientRef and check the stored link value and whether the server id BACK-FILLED (e.g. member.individualId non-null proves the individual exists), rather than trusting a reverse-search that may be unfiltered.
- Test back-fill via the synchronous single `_create` RESPONSE (enrichment-time) — it is immediate and lag-independent, no TTL wait needed.
- Corrections this caused:
  - Unfiltered reverse search = false signal: beneficiary `_search` by beneficiaryClientReferenceId returned all 50 (the limit) because that field is not a search filter — NOT "50 matches". Switched to embedded-value check.
  - "Individual didn't persist" was wrong: `/health-individual/v1/_search` returns 0 for a resolvable individual, but member.individualId back-filled to a real id proved it exists. Reframed as a dev search quirk, not a persist failure. (Do not treat "individual _search empty" as non-persist — check the member back-fill instead.)

## Rule 3 — Isolate single runs; parallel runs create false negatives
Running suites in parallel produced many `in_db=false` that were true in clean runs (persister lag under load). For persistence verification, run ONE isolated suite, not parallel. Flag a contended run as noise, not as a regression.
- When a result looks wrong, get a tiebreaker (a clean single isolated run) before concluding — never trust one noisy run.
- Correction this caused: 3 suites in parallel -> many false `in_db=false`; re-ran isolated -> passed.

## Rule 4 — Separate a harness bug from a real defect
A "FAIL" is often a harness bug (wrong wrapper key, wrong endpoint, stale field names, unfiltered filter). Re-query the SAME already-created ids with the corrected harness before claiming a defect.
- Corrections this caused:
  - Wrong search wrapper key: used `HouseholdMemberSearch` -> member search returned 0 (looked like a broken relation). Correct key = `HouseholdMember` (field `clientReferenceId` is a list). Re-queried same ids -> all relations passed.
  - Wrong endpoint: individual create/search must use `/health-individual/v1/...` (the HCM service the app uses), NOT `/individual/v1/...` (dev routes that to the crash-looping egov CORE individual service with a different schema). This false trail produced retracted conclusions like "individuals don't persist in dev" and "dev DB missing hashedmobilenumber column".
  - Stale field names: the "referral NPE bug" was a TEST-DATA mistake — old collection fields (referredToType/referredToId/referredById) left the real required field (recipientType) null, which is what NPE'd. ALWAYS confirm real field names via a `_search` of an existing record before building payloads.

## Rule 5 — Report honest, messy, contradictory results at the right confidence
- "Task null-link always persists" -> refined to "intermittent/non-deterministic" when two identical later runs vanished. Reported the contradiction (a worse but truer finding) rather than hiding it.
- "Side-effect orphan" started as 1 observation; when asked "is this fresh data," admitted it was a single sweep point, then re-ran isolated 3x with captured ids and only then presented it at full confidence (3/3, deterministic).
- Stale prior findings: beneficiary null-link and referral null-link used to persist (2026-07-06 L2/O4); this build drops them -> flagged the doc's D3/D4 lines as STALE rather than leaving them wrong.

## Rule 6 — A down dependency does not mean the thing under test is broken
egov-persister was in CrashLoopBackOff on unified-dev, yet HCM records still landed in the DB. Did NOT conclude "persistence broken" — cross-checked that HCM records commit, concluding HCM uses a different persistence path than egov-persister. Always cross-check the actual path before attributing a failure.
- Related: an oauth 500 "Unknown error occurred in encryption process" = egov-enc-service pod not ready (not a bad credential). A 401 wave = egov-accesscontrol flapping. Individual _create 500 AnnotatedConnectException = a not-ready egov-user replica. Distinguish infra churn from the fix.

## Distinguishing infra noise from real results (dev-specific)
- unified-dev churns: enc-service eviction (oauth 500), accesscontrol flapping (401 on authorized calls), persister Kafka group rebalancing, TLS handshake failures, port-forwards going stale. Green windows are short/random. Wait for all egov/health pods 1/1, persister not rebalancing, enc-service ready before running a verification suite.
- Decrypt-path flows (downsync, individual search with decrypt) fail with UNKNOWN_ERROR "decryption process" because of a MISSING ENCRYPTION KEY in the dev enc-service DB (dev key-seeding gap) — even with enc-service 1/1 READY. That blocks downsync tests; it is a dev data issue, not the fix.

## Rule 7 — A big reported "gap/discrepancy" is a HYPOTHESIS, not a result: disprove the cheap causes before acting
A tool/report count (a reconciliation gap, a "N records missing", a failing metric) is unverified until you falsify the artifact explanations, in cost order, each with a direct query against source. Proven on the ch/Chad "148K project-beneficiaries with no task" — which turned out NOT to be missing treatment. The ladder:
1. **Is the fixed code actually running?** A known false-positive still present, or a missing log line from your fix, means you're on an old/undeployed build — stop and deploy. (Edit tree ≠ runtime tree.)
2. **Sample ~10 flagged items; check the other side UNWINDOWED / unfiltered.** If some are present after all → date-window/lag/filter artifact.
3. **Join-key check** — LEFT JOIN on BOTH candidate keys, count which link column is NULL. Tool joining on the NULL key = false gap. (ch: tasks link only by client-ref, server-id NULL on all → join was correct.)
4. **Duplicate/twin check** — is the relation satisfied under a different record? (ch: 5 of 173,952.)
5. **Quantify at scale** with `NOT EXISTS` — never extrapolate from the 10-sample.
6. **Profile flagged vs known-good** on a discriminating attribute — and keep cutting until the "bad" set has a crisp identity. (ch: first cut = 98% placeholder DOB `1970-01-01`/epoch-0; sharper cut from a colleague's field insight = the task-less PBs are HOUSEHOLD HEADS — cross-tab dosed × `isHeadOfHousehold`: un-dosed 97.8% heads, dosed 95% non-head members. Heads are the adult respondent, not a dosing target → no task by design; placeholder DOB was just the symptom. Real residual ~2,461 non-head members.)
7. **Anchor to the domain fact** (SMC register+dose = same visit) as the plausibility test — the contradiction it exposes is the thread to pull, and resolving it usually clears the domain expert's doubt. **Solicit the field/domain colleague's read** — "they're household heads" was the one-line insight that named the population.
- Corrections this caused: "148K children untreated" → "148K household-head registrations (not dosing targets), only ~2.5K real un-dosed members"; the scary number was benign by-design data, not a coverage failure or a tool defect. Same discipline as Rule 4 (harness bug vs real defect), applied to reconciliation/RCA output. Detail + code fix: [[project_recon_recovery_tool]], dst/es-vs-db-rca.
