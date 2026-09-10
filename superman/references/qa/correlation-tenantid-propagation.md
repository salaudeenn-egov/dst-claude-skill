# Correlation ID / TenantId Propagation Testing (mhbase, 2026-08-25/26)

Verifying the platform claim in `Correlation_TenantId_Propagation_Test_Report.xlsx` (built on
`unified-dev`, gateway build `HCMPRE-2023-corrfix-tenantId-11a40a1`, tracer 2.9.3) against
`mhbase-hcm-dev` (tenant `mhbase`). The report has 4 sheets; the two that matter are
**Test Cases (new gateway)** TC01-TC08 and **Multi-hop & Kafka (verified)** rows 1-5.

**PASS CRITERION (the report's own):** a case passes if the *downstream service logs the SAME
correlationId that was sent*, with `TENANTID=<tenant>`. It is NOT about the response header —
only TC06 (mint) tests the header. Getting this wrong cost a round of false "defects".

## Problem
Report claims PASS on every hop. First runs on mhbase returned NOTHING for any correlation id,
suggesting total failure. Misleading first read #1: "correlation ids appear in zero log lines"
— actually the Grafana window (15 min) had rolled past; the ids were there. Misleading first
read #2: "mdms-v2 and idgen produce no logs" — mdms-v2 does log; it just wasn't being *called*.

## Approach that worked
1. **Always pair a correlation-id search with a control search** on a record id that definitely
   exists (`H-2026-08-26-...`). Control empty => query/time-range wrong, everything else
   uninterpretable. This single discipline resolved most of the early confusion.
2. **Search the GATEWAY-minted id, not the one you sent** (see the split below).
3. **Use the MINT case for clean runs** — send no `x-correlation-id` and no
   `RequestInfo.correlationId`. Then only ONE id exists and "wrong id searched" is eliminated.
4. **Container dropdown + CLEARED search box** to prove a service ships logs at all
   (`{namespace="egov", container="egov-idgen"}`). Distinguishes "id not propagated" from
   "service logs nothing" — otherwise unfalsifiable.
5. Repeat across entity types; a hop missing on one API and present on another is a caching or
   call-path artifact, not a propagation defect.

## Findings on mhbase (7+ runs, individual / facility / household / member / bulk)

**CONFIRMED PASS:** `egov-user` (`/user/_details`) and `egov-accesscontrol`
(`/access/v1/actions/_authorize`) carry the correlation id AND `TENANTID=mhbase` on 100% of runs.
`mdms-v2` PASSES when actually called (receive line + `MdmsDataRepositoryImpl` SQL both carry it).
TC06 mint PASSES — gateway mints a UUID and returns it in the response header.

**DEFECT D1 (High) — one request carries TWO correlation ids.** The gateway mints its own id and
forwards THAT in `RequestInfo.correlationId`; the client's `x-correlation-id` survives only in the
MDC of the FIRST health service. Both appear in the same log line:
```
MdmsClientService -- TENANTID=mhbase, CORRELATION_ID=ROW1-IND :
    requestInfo=RequestInfo(... correlationId=36ea0128-81d6-4fc6-9138-4c1038d3c0f3 ...)
```
Impact: the id a client holds will NOT find the downstream failure; tracing one request end-to-end
requires knowing both ids. Seen 7/7 runs. Suggests mhbase does not run the report's gateway build.

**DEFECT D2 (Open) — `egov-idgen` logs nothing under either id**, on SUCCESSFUL calls, 5/5 runs
that used it (real ids issued: `H-2026-08-26-493914`, `F-2026-08-26-137926/7`). Every neighbouring
service logs correctly. NOT yet scored FAIL: the container-with-cleared-search query was never run,
so "doesn't propagate" vs "ships no INFO logs" is still open. Rows 1-3 of the doc all name idgen.

**D3 (Low)** client id never echoed in the response header; gateway returns its own.

**NOT a defect — the mdms-v2 cache artifact.** Facility/household runs often show
`MdmsClientService` (client side) but no `RestTemplateLoggingInterceptor: Sending request to
mdms-v2` and no mdms-v2 receive line => the service served `IdFormat` from cache and never made the
HTTP call. Proof: the 04:59 household run (cold cache) DID hit mdms-v2 and logged perfectly; the
05:25 run of the same API did not. Never score a missing hop before checking the outbound send line.

**Not applicable:** household-member does not call idgen (member ids are UUIDs, not `H-`/`F-`
format), so an empty idgen result on that path is expected.

## Practical recipe
- Thread names: HTTP `[080-exec-8]`; Kafka consumer `[er#0-0-C-1]`. The thread name is how you
  prove the Kafka hop actually ran.
- Loki needs a stream selector: `{namespace="mhbase"} |= "<id>"`. Select BOTH `mhbase` and `egov`
  namespaces — idgen/mdms/user/accesscontrol live in `egov`.
- mhbase paths are UN-prefixed (`/individual`, `/facility`, `/household`) and MDMS is
  `/mdms-v2/v1/_search`, NOT the report's `/egov-mdms-service/v1/_search` (404 otherwise).
- A 400 is still a valid propagation test — the doc's own TC04 is a 400 marked PASS ("reached
  project with both ids"). What matters is the log line, not the status.
- Chain row 4 (member) to REAL refs from rows 1 and 3 with a ~12s wait, else the
  `individual /v1/_search` hop never fires and looks like a propagation gap.
- Tokens are short-lived single-session: a 401 `InvalidAccessTokenException` mid-suite is normal.

## Artifacts (Downloads/)
`correlation_all_scenarios_curl.txt` (5 multi-hop curls), `sheet1_tc_curls.txt` (TC01-TC08),
`facility_create_curl.sh`, `Correlation-Test-mhbase-2026-08-26.xlsx`
(Summary / Hop Results / Runs / Defects / How to Test).

## Open
Run `{namespace="egov", container="egov-idgen"}` with the search box cleared to close D2.
Row 5 (Kafka bulk) never verified on mhbase — it is the claim the report leans on hardest.
Confirm deployed image versions (`kubectl -n egov get deploy -o ...image | grep -E 'gateway|idgen'`)
before raising D1/D2 as regressions: if mhbase is not on the fix builds, they are expected.
