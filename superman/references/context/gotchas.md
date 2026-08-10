# DIGIT HCM — Gotchas, Dead Ends, Pitfalls

Hard-won facts that took hours to discover. Read before any ES/tracer/recovery work.
These correct wrong assumptions — do not re-learn them.

## Tracer / ES query gotchas

1. **egov-tracer `requestBody` is NOT valid JSON** — keys are unquoted
   (`{projectBeneficiaryClientReferenceId: 607f..., tenantId: oy, ...}`). `json.loads()`
   always throws, `except: continue` swallows it, zero records parsed. Always use regex.
2. **Tracer search term for task errors is NOT `"Tasks"`** (only ~6,941 docs). Use
   `match_phrase: "projectBeneficiaryClientReferenceId"` (~206,208 docs), then filter
   `tenantId: oy,` (unquoted style). `"Tasks"` as a key is not tokenized reliably.
3. **Tracer stores the PARENT id in the error message** ("Related entity does not exist in
   db <beneficiaryId>"), not the child/task's own id. Search by parent beneficiary id.
4. **Match ES index prefix to the TENANT variable.** Each tenant has its own
   `{TENANT}-household-index-v1`. Querying `ab-...` while running on `oy` returns 0 hits — a
   misleading dead end. Construct the index name from the script's TENANT config.
5. **`_search` caps total at 10,000** with `"relation":"gte"`. Use the `_count` API for the
   real total.
6. **`"_source":[dotted.nested.field]` breaks extraction** — use `"_source": True`.
7. **Verify a new task-index field before filtering.** `Data.auditDetails.createdTime` and
   `Data.isDeleted` DO NOT EXIST on task docs (return 0). Use `Data.taskDates` for dates.
8. **`json.loads("null")` returns Python `None` without raising** — guard with
   `isinstance(body, dict)` or the script silently skips all tracer records. Likewise
   `"userInfo": null` -> `.get("userInfo", {})` returns None; use `(body.get("userInfo") or {})`.

## API / recovery gotchas

9. **DIGIT returns 202, not 200** for create/update — check `in (200, 202)`.
10. **DB-not-in-ES fix is UPDATE, not CREATE.** The record is in DB; create would duplicate.
    UPDATE (with the full record from search) triggers Kafka -> transformer re-index.
    ES-not-in-DB (ghost) is the CREATE case.
11. **Create children only after the parent is in DB** — dependency order:
    Individual, Household, then HouseholdMember, ProjectBeneficiary, ProjectTask. Otherwise
    VALIDATION_ERROR.
12. **`isHeadOfHousehold=True` members cannot be updated via API** (duplicate-head constraint).
13. **Task resources[] require `tenantId`** (`NotNull.taskRequest.task.resources[0].tenantId`).
14. **doseIndex values are `"1"/"2"/"3"`**, not `"01"/"02"/"03"`.
15. **`project_task` has NO householdid column**; `project_beneficiary.beneficiaryid` is NULL
    by design (not a bug); `Data.householdId` in ES is denormalized ES-only.
16. **Individual create/search uses `/health-individual/v1/...`** (HCM service), NOT
    `/individual/v1/...` (core service, crash-looping in dev, different schema with
    `hashedmobilenumber`). Do not treat `/health-individual` `_search` returning 0 as
    non-persist in dev — it is a dev search quirk; check member back-fill instead.
17. **Project-task create requires an `address` object** — without it ProjectTaskEnrichmentService
    NPEs (single 400, bulk silently 0).

## Python / memory gotchas

18. **Save counts BEFORE `del`.** `del ind_ids` removes the binding immediately; any later
    `len(ind_ids)` raises NameError. Save `ind_count = len(ind_ids)` first, then del + gc.collect().

## Platform-constraint gotchas (do not propose changing)

19. **`enable.auto.commit=true`** (PersisterConsumerConfig.java:68) is platform-wide across all
    DIGIT services. Off the table.
20. **UPLOAD_ERROR_FROM_APP is a mobile RETRY indicator**, not a server error. It comes from
    Flutter `data_repository.dart` `dumpError()`, NOT the server consumer. 69.8% of the 90k
    ghosts carried it — they are retries of an already-created ghost, not the original cause.
    Do NOT search the Java codebase for this string.
21. **DLQ and validateClientReferenceIdsFromDB already exist.** DeadLetterQueueConfiguration.java
    is fully implemented, just commented out (`/* */`). validateClientReferenceIdsFromDB()
    exists at GenericRepository.java:392. The fixes are enabling/calling existing code, not
    writing new code.

## Corrected numbers (superseded — do not reuse the old ones)

- Sokoto PT race condition = **279 confirmed** (not 12,189 — that counted all tracer errors).
- Bauchi idgen = **16 confirmed** (not 558). idgen total across all entities = **34**.
- 90,143 total ghosts covers only certain entities (Sokoto PT + Bauchi PT/HH/PB/HHM); Sokoto
  HH/PB/HHM were NOT analyzed.

## Investigation dead ends (do not repeat)

- Searching tracer by the task's own clientReferenceId (it stores the parent id).
- Searching Java for UPLOAD_ERROR_FROM_APP (it is in Flutter).
- Trying to prove transformer reads Topic A to explain ES bypass — transformer config says
  Topic B (`save-project-task-topic`, application.properties:61). Production ConfigMap was never
  confirmed (kubectl blocked). Leave as "unconfirmed" rather than resolving.
- Querying tracer for Jun-Jul 2025 data — purged before ~Aug 1, 2025; returns empty (expected).

## Reliability note

For any "persisted" claim, verify by DB `_search` AFTER 60-100s (past the 60s Redis TTL). A
search seconds after create hits the Redis cache and returns a row with `id=None` = FALSE
positive. See the hcm-qa skill's verification-methodology for the full protocol.

## HCM write-API gotchas (task / product-variant `_update`) — ba VAS, Jul 2026

- **Soft-deleted product variant blocks ALL task `_update`.** project/task `_update` validates
  `resources[].productVariantId` against a variant search done WITHOUT deleted. If the variant is
  `isDeleted=true`, the validator can't see it -> `NON_EXISTENT_RELATED_ENTITY [PVAR-...]`,
  `type=NON_RECOVERABLE`. This blocks EVERY task for that product (all VAS in ba), not just target rows.
  Confirm: `product/variant/v1/_search?includeDeleted=false` (empty) vs `includeDeleted=true` (found,
  isDeleted:true). Fix = restore the variant (`isDeleted=false`) FIRST, then the task updates pass.
- **The error message names an entity that may EXIST globally.** "Related entity does not exist [PVAR-...]"
  means "not found under the validator's scope (non-deleted)", not literally missing — always check
  `includeDeleted=true` before concluding it's absent.
- **Update error text is truncated/misleading** — the real reason is at the TAIL of `Errors[0].message`
  (after a huge requestBody echo). Print `message[-2000:]`, not `[:200]`.
- **product-variant `_update` payload:** send `ProductVariant` as a **LIST** and round-trip the FULL
  object from `_search` (only flip the changed field). A single object / hand-built payload ->
  `JsonMappingException: Failed to deserialize certain JSON fields`.
- **userInfo is enriched from the token** — a fake `userInfo.uuid` is fine; server replaces it from the
  authToken. If `_search` succeeds but `_update` 400s, it is payload/validation, NEVER auth.
- **Report "Client Reference ID" column = RESOURCE-level ref, not the task ref** (differ by one digit).
  To search a task use `taskClientReferenceId` (`Data.taskClientReferenceId`), not the report column —
  the report column returns 0 tasks.

## Chad SMC beneficiary-ID gotchas (2026-08-07/10 investigation; full RCA in dst/chad-smc-beneficiary-id-rca.md)

- **identifierId is ENCRYPTED in {tenant}-individual-index-v1** (enc-service format `316090|<base64>`).
  Plaintext beneficiary-ID search there is impossible; plaintext lives only in id_pool and tracer
  error messages. Also NOT denormalized onto {tenant}-project-task-index-v1 (query_string probe
  on ch = 0 hits) — reporting/dashboards unaffected by beneficiary-ID duplicates.
- **idgen `_update` tracer requestBody IS valid quoted JSON** (json.loads works) — the global
  "tracer bodies are unquoted pseudo-JSON" gotcha applies to entity-create bodies only.
- **ch.individual usernames ARE populated** (unlike kd, where the column is NULL for CDDs and you
  must fall back to the user-sync index); but `ch.individual.mobilenumber` is enc-service
  ENCRYPTED — exclude it from deliverables.
- **egov-tracer-error-details is NOT real-time.** Observed a 5+ hour indexing stall (newest doc
  frozen while pod errors kept occurring; docs queue in the Kafka error topic). Never conclude
  "no errors" from Kibana without checking the index's newest @timestamp vs now, and idgen/indexer
  consumer lag.
- **Multi-replica services: search ALL pods for a log line.** beneficiary-idgen runs 3 replicas;
  the rejection for our test ID was on replica #2 only. A single-pod `logs -f` monitor misses it.
- **kubectl read-only discipline:** use `--context <arn>` per command; `kubectl config use-context`
  WRITES the kubeconfig. In this harness, env vars ($env:KUBECONFIG etc.) do NOT persist across
  tool calls — set them in the same command as the kubectl call.
- **id_pool attribution:** `id_pool.lastmodifiedby` is OVERWRITTEN on assign; the append-only
  `id_transaction_log` (id_reference/user_uuid/status) is the only dispatch-history source.
  rowversion does NOT reliably track dispatch->assign transitions.
- **Beneficiary-ID dispatch quota is DAILY per user+device** ("ID generation limit exceeded ...
  Remaining ids: 0"). A data-clear/reinstall strands the device pool-less until the next day; the
  app swallows this silently.
- **App-side generated-ID fingerprint in device logs:** `UPDATE_IDENTIFIER_STATUS: No
  UniqueIdPoolModel found for id: X` (adb logcat -s flutter:V). Happy path logs `-> ASSIGNED via
  local repository` — which is LOCAL SQLite only; the app never calls id_pool/_update (the
  individual service does, server-side).
- **Interrupted first bootstrap bricks the Chad app**: airplane mode mid-first-sync saves a
  project without projectType -> `AppException 'No beneficiary type is selected'` on every home
  load; only Clear Data recovers.
