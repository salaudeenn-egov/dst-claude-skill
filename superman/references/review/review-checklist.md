# HCM Code Review Checklist

Per-layer failure modes for DIGIT HCM changes. Each item is the question to ask and
the concrete scenario it guards against. Cite `file:line` in findings; rank data-loss
/ ghost-record findings above everything else.

## Controller / API layer (e.g. ProjectApiController)

- **Cache populated before DB commit?** If `putInCache()` runs before the record is
  durably in DB, a sibling/child request within the 60s Redis TTL can read the cache and
  pass an existence check against a row that never commits -> ghost record. Cache must be
  populated only after confirmed DB write.
- **202 handled as success downstream?** DIGIT create/update returns 202, not 200. Any
  new client/caller that treats non-200 as failure (or 202 as failure) is a bug.
- **Response key plural vs singular?** search returns plural (`Households`), create/update
  singular (`Household`). Mismatched key = silent empty parse.

## Validator / consumer layer

- **Existence check hits DB, not cache?** `validateIds()` -> `findById()` ->
  `findInCache()` can return a Redis-only parent. For a real guarantee use
  `validateClientReferenceIdsFromDB()` (direct SQL). Flag any parent-existence check that
  can be satisfied by cache alone.
- **Parent-not-found error RECOVERABLE?** A NON_RECOVERABLE NON_EXISTENT_RELATED_ENTITY
  gives the parent no chance to arrive; race-condition children are lost. If retry is
  intended, the error type must be RECOVERABLE and a backoff configured.
- **Removed a validator without a null-guard?** When an existence/relationship validator
  is disabled (validator-unbundle), inputs it used to short-circuit now flow downstream
  and can NPE (referral null recipientType, task null projectBeneficiary). Data stays
  safe but the failure is ungraceful. Require matching null-guards where a validator was
  removed.
- **Linking clientReferenceId left nullable/optional?** DB columns for
  individualClientReferenceId / householdClientReferenceId / beneficiaryClientReferenceId
  / projectBeneficiaryClientReferenceId are all nullable in DDL — integrity depends only
  on app validation. Null linking ref -> silent drop (member/beneficiary) or untraceable
  orphan (task/side-effect). A change must not accept a null linking clientReferenceId at
  the service layer.
- **notHavingErrors() / hasErrors filtering?** Confirm error-bearing records are handled
  intentionally, not silently skipped in a way that hides data.

## Repository / persister layer

- **batchUpdate isolation?** One bad record in a `batchUpdate` rolls back the whole
  Kafka-message batch — all N dropped. A persister change must preserve per-record retry /
  single-row fallback so one bad record can't drop valid siblings (poison batch).
- **Failure surfaced?** A DB write failure must publish to egov-tracer AND be routed by the
  error handler (DefaultErrorHandler must have non-retryable types configured, DLQ enabled).
  A `log.error()`-only path (PersistRepository silent skip) is invisible = permanent loss.
- **auto-commit assumption.** `enable.auto.commit=true` is a fixed platform constraint —
  reject any change that relies on flipping it; the fix must work within auto-commit.
- **Empty-rows skip logged?** `if(!rows.isEmpty())` silent skip should at least warn
  (possible JSON-path mismatch) so a mapping error is visible.

## Transformer layer

- **Empty enrichment indexed anyway?** If `searchHousehold` / `searchBeneficiary` returns
  empty and the doc is built with null `householdId` / `memberCount` / linking ref and then
  indexed, that IS the ghost mechanism (TC-08: Bauchi 89/89 tasks with householdId=null,
  absent from DB). Empty enrichment must raise an error or a data-quality flag, not index
  silently.
- **Transient dependency outage retried?** Auto-commit commits the offset even when a
  dependency is down; the record is not reprocessed when the dep recovers (TC-07). Needs
  bounded retry / dead-letter.
- **Error record has a real source topic?** `sendToErrorTopic(req, null, e)` produces an
  error record with a null topic that cannot be replayed (TC-06). Source topic must be
  threaded (or a ThreadLocal fallback used).
- **Error record id unique?** A null/empty error message (uses `getMessage()` on an NPE)
  and a non-unique error doc id (`id: $.topic`) cause overwrite. Use `toString()`/root
  cause for the message and a UUID (deterministic `nameUUIDFromBytes(topic+payload)` so
  Kafka redelivery overwrites rather than duplicates) for the id.
- **Endpoint paths are `/health-*`?** Downstream health services serve under `/health-project`,
  `/health-individual`, `/health-service-request`, `/health-attendance`. Using `/project`,
  `/individual` etc. -> 404 -> null projectId -> NPE (CommonUtils cycle-cache null key).
- **Real projectId source?** Household/member resolve projectId via the record's user ->
  project-staff; a fake staff uuid -> null projectId -> NPE. Task/stock/referral resolve it
  from the record/beneficiary directly.

## Indexer (egov-indexer) yml

- **id JsonPath is String-typed?** `IndexerUtils.buildIndexId` (IndexerUtils.java:365) casts
  each id JsonPath value to String. If `id:` points at a Long field (e.g. a timestamp),
  ClassCastException -> "Indexing will not be done" -> EVERY record for that index dropped
  (TC-09). Point `id:` at a String field (or coerce).
- **id unique per record?** A composite/duplicate id collapses distinct records to one.
- **Index name prefix correct?** New ymls must use the index name the live ES actually has
  (some flows use unprefixed `household-index-v1`, others `{tenant}-...`). Register the yml
  in `egov.indexer.yml.repo.path`.

## Recovery / reconciliation / extraction Python scripts

- **Tracer body parsed with regex, not json.loads?** The tracer `requestBody` is unquoted —
  `json.loads()` always throws and `except: continue` swallows it -> 0 records parsed
  silently. Must use regex UUID extraction.
- **ES index prefix matches the TENANT variable?** `{TENANT}-...-index-v1`; a hardcoded or
  wrong-tenant index returns 0 (misleading dead end).
- **Direction correct?** ES-not-in-DB (ghost) -> CREATE; DB-not-in-ES -> UPDATE (never
  CREATE, would duplicate).
- **Create order respects the chain?** Individual/Household before HouseholdMember/
  ProjectBeneficiary/ProjectTask, else VALIDATION_ERROR.
- **`in (200,202)` check?** Treating 202 as failure will mis-report pushes.
- **Counts saved before `del`?** Referencing a `del`-ed set later = NameError; save
  `len(...)` first.
- **`_count` for real totals?** `_search` caps at 10,000; a hardcoded total may undercount.
- **Idempotent / checkpointed?** Only successful records checkpointed; failures must retry.
- **DRY_RUN default and prod-safety?** Mass-update scripts should default DRY_RUN=True and
  never target production tenants (e.g. `so`) with test data.
