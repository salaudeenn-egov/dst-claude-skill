# Chad ITN Upload-Error-From-App Investigation (2026-08-05)

QA of the mobile app's client-side error-report mechanism for Chad ITN. Ties to the
missing-household-member (Gap B, ~3,673) / missing-project-task (Gap E, ~15,878) RCA.
Live-device reproduction on the actual PROD APK, not code-reading alone.

## What it does & why we test it
When a sync-up retry is exhausted, the app is supposed to report the permanent failure
to the tracer via `dumpError()` -> `POST error-handler/handle-error`, landing as an
`UPLOAD_ERROR_FROM_APP` doc in `egov-tracer-error-details`. The QA goal: prove whether
this pipe actually works end-to-end on a real device, and find out why household-member
`UPLOAD_ERROR_FROM_APP` volume (267) is so much lower than household's (3,526) even
though Gap B has 3,673 affected households.

## Setup
- Repo: `HCM-NIGERIA-MC/health-campaign-field-worker-app`, branch `master-chad`. Env:
  `BASE_URL=https://chad-hcm-prod.digit.org/`, `TENANT_ID=chad`,
  `SYNC_DOWN_RETRY_COUNT=3`, `ENV_NAME=PROD` (confirmed against the actual shared env,
  matches what was reverse-engineered from code beforehand).
- Test APK + creds supplied by the implementation team (Venkat): user `<REDACTED-USER>` /
  `<REDACTED — ask implementation team>`, tenant PROD, transactions deleted after activity.
- Device: real Android phone via `adb` (USB; wireless adb blocked by AP/client
  isolation on the office WiFi — had to use USB). Log capture: `adb logcat flutter:V
  *:S | grep ...` scoped to the app's own `flutter` tag (unscoped logcat is 99% noise —
  facility-master JSON dumps, unrelated system daemons like `mnld`/`agps`).
- ES cluster: `elasticsearch-data.es-cluster` (NOT the shared Nigeria `es-cluster-v8`) —
  same "try both, don't assume" pattern as the tracer-dump scripts.

## Code path (confirmed by reading, then matched against live logs)
- `DataRepository.dumpError()` (`data_repository.dart:279-329`) POSTs to
  `error-handler/handle-error` with `errorDetail.errors[].errorMessage:
  "UPLOAD_ERROR_FROM_APP"`.
- Retry-count increment (`registration_delivery_imports.dart:390-402`,
  `oplog.dart:313-341` `updateSyncDownRetry`) only fires when the server responds
  successfully but THIS record's `serverGeneratedId` comes back null — NOT on a
  transport-level failure. A pure "no network at all" outage never reaches this
  increment path.
- `sync_service.dart:63-72` wraps the whole `PerformSyncUp.syncUp()` (incl. the
  `dumpError()` call itself) in one try/catch that just stores the error and calls
  `service?.stopSelf()` — no logging unless MDMS `FIREBASE_CONFIG.enableCrashlytics`
  is on for the tenant (not confirmed for `chad`).

## Result matrix (live device, 2026-08-05)
- **Total outage from the very first attempt**: retries loop (`SocketException: Failed
  host lookup`) indefinitely, `nonRecoverableError` stays `false` — consistent with the
  code (this path needs a server response, which never arrives). Once reconnected, the
  original create DOES eventually succeed (proven for two test households at
  17:34:43-46 IST) — the record was never lost locally, just stuck retrying.
- **Mid-transfer cutoff** (network dropped right after a prior call succeeded, not
  before): produces a DIFFERENT family of signatures — `SocketException: Connection
  reset by peer`, `HttpException: Software caused connection abort`, `HttpException:
  Write failed`, `SocketException: Network is unreachable`. These indicate the
  connection died mid-request/response, not "no network at all."
- **Reproducing a genuine, confirmed `UPLOAD_ERROR_FROM_APP`**: confirmed 3 times across
  household, household-member, individual, and project-beneficiary entities, each with
  a `RESPONSE - 200` on the `error-handler/handle-error` call AND (for two of the three)
  the actual tracer document pulled back via ES `_search` matching the exact
  `clientReferenceId`s sent.
- **Reproducibility is inconsistent.** A short interruption self-recovers cleanly with
  no error at all (e.g. household -> individual -> project-beneficiary ->
  household-member cascade completing in ~14s after a brief "connection abort" blip).
  Only an interruption that both (a) lasts long enough / hits the right timing to cross
  the retry threshold, and (b) later gets a working connection for the report call
  itself, produces a confirmed tracer entry.

## THE finding: confirmed false-positive `UPLOAD_ERROR_FROM_APP`
Two independent, fully-timestamped cases where the app reported a record as permanently
failed AFTER it had already synced successfully:

1. **Case 1** — household `26a5ffd0-90cb-11f1-b2e6-81327043a211` (+ member
   `26a73851`/individual `26a6ea30`, PBs `26a73850`/`3a1ee4a1`/`7a321760`). Server
   persist: household 12:52:08 UTC, member+PBs 13:22:35-36 UTC. `dumpError()` succeeded
   (200) at 13:29:31-34 UTC — 7-27 minutes AFTER these records had already synced.
   Tracer docs confirmed by direct ES fetch: `_id: 037733c2-6b31-4c95-869c-2559caccdea8`
   (`/household/member/v1/bulk/_create`) and `_id:
   26098f7e-ea76-4426-89f0-e9b5e46e103d` (`/project/beneficiary/v1/bulk/_create`).
2. **Case 2** — household `c0511ff0-90d2-11f1-b2e6-81327043a211`. On-device creation
   13:37:06 UTC, server persist 13:37:57 UTC (51s later). `dumpError()` reported it as
   `UPLOAD_ERROR_FROM_APP` at 13:40:22 UTC — 2m25s after it had ALREADY succeeded
   server-side.

**Working root-cause theory (not yet traced to an exact code line):** the create request
reaches the server and is persisted successfully, but the connection drops while the app
is reading the response (matching the "connection abort" / "reset by peer" / "write
failed" signatures observed right around these moments). The app never registers the
success, so it counts as an apparent failure client-side; once enough of these
accumulate, `dumpError()` fires — reporting a record that was never actually lost.

## Gating implication
The tracer's `UPLOAD_ERROR_FROM_APP` counts (38,635 total in the Aug 1-5 production
dump; 3,526 household / 267 household-member — see `dst`/campaign notes for the full
entity breakdown) are NOT a clean proxy for genuine permanent data loss. Some unknown
fraction are false positives like the two cases above. Conversely, the low
household-member count relative to Gap B's 3,673 affected households suggests most Gap
B member-creation failures never even reach the report-attempt stage (stuck in the
"total outage, never re-tried the report" state, or the pure-outage path that never
increments the retry counter at all).

## Open / not yet verified
- Exact code path for the "response lost after success" theory — inferred from error
  signatures + timing correlation, not confirmed against a specific line.
- Whether Crashlytics (`firebaseConfig.enableCrashlytics`) is actually enabled for
  `chad`'s MDMS config — would confirm whether the swallowed exception in
  `sync_service.dart` has ANY visibility today.
- Whether the false-positive rate generalizes across the full 38,635 production count,
  or whether the two live-tested cases are unusually fast round-trips (51s, few minutes)
  compared to typical field conditions.
- Sahil Gupta + Naveen J's independent DB/log analysis (same week) found 1,269
  household-members reaching the server >6h late, plus ~27,000 HTTP 401
  auth-token-expiry events — a separate root-cause candidate not tested here. Their
  1,269 is far short of the 3,673 Gap B population — the fate of the remainder is
  still unexplained by either analysis.
- Genuine permanent loss (device never reconnects, storage cleared, app reinstalled
  before syncing) was discussed as a test scenario but not executed this session —
  everything confirmed "recovers" assumed the device eventually reconnects.

## Artifacts
- Full write-up with all evidence: `Downloads\chad_itn_upload_error_investigation_report.docx`
  (generator: `Downloads\chad_itn_upload_error_investigation_report.py`).
- Tracer dump + analysis scripts: `Downloads\chad_itn_tracer_dump_aug_to_date.py`,
  `Downloads\chad_crossref_missing_vs_tracer_dump.py`,
  `Downloads\chad_export_found_in_tracer.py` (note: raw UUID-substring cross-referencing
  produces false positives — must filter by apiUrl matching the actual missing
  child-entity type, e.g. Gap B needs `household/member`, not just any doc containing
  the household's id — see the "relevance filter" comment in
  `chad_export_found_in_tracer.py`).
- Creator/user lookup script (best-effort, ES-based, not yet run):
  `Downloads\chad_missing_hhm_task_creator_lookup.py`.
