# Stock / Supply-Chain Flow — model + test setup (mhbase-hcm-dev)

Source memory: `project_hcm_stock_flow_test.md` (2026-08-24) + user-sync reporting sessions.

## The model: role is tied to boundary level

**Problem.** Testing the stock flow in the SMC APK (tenant `mhbase`, app `com.digit.hcm`,
`https://mhbase-hcm-dev.digit.org`) — naive user setups (several warehouse managers parked at
one level) leave transfer legs with no eligible receiver, and the failure is quiet: the
dispatch screen simply has no valid target, which reads like missing master data.

**Approach.** Reasoned from how a transfer resolves its parties: each hop is
dispatch-at-boundary-N -> receive-at-boundary-N+1, and the receive leg is only offered to a
WAREHOUSE_MANAGER whose assigned boundary IS the receiving boundary. So the setup must mirror
the boundary tree, not headcount.

**Root cause / rule.** Stock can only move down the chain if **every level of the boundary
tree has exactly one WAREHOUSE_MANAGER scoped to that boundary**; the leaf (health facility)
additionally needs a DISTRIBUTOR for last-mile issue.

**Working setup (verified, 6 users all CREATED/Active/Permanent):**

| Level | Role | Boundary code |
|---|---|---|
| Country | WAREHOUSE_MANAGER | `ADMIN_NI` (Nigeria) |
| State | WAREHOUSE_MANAGER | `ADMIN_NI_01_BAUCHI` |
| LGA | WAREHOUSE_MANAGER | `ADMIN_NI_01_14_SHIRA` |
| Ward | WAREHOUSE_MANAGER | `ADMIN_NI_01_14_18_DANGO` |
| Health facility | WAREHOUSE_MANAGER | `ADMIN_NI_01_14_18_04_BA_RIM_KATANGA_HEALTH_POST` |
| Health facility (leaf) | DISTRIBUTOR | same code as HF above |

Credentials: console-download xlsx in Downloads (`userCredential - 2026-08-24T191300.765 (1).xlsx`,
sheet `Create List of Users`); values are `<REDACTED>` by policy — read from the file/secret
store, never from this KB.

## Console/unified-sheet conventions that interact with stock

- Facility Type: only `Warehouse` accepted on the mhbase build ("Health Facility" rejected).
- Role-to-level mapping convention (unified user sheet): CDD/DISTRIBUTOR at health-facility
  level, TEAM_SUPERVISOR/ward, DISTRICT_SUPERVISOR/LGA, HELPDESK+PROVINCIAL/state,
  NATIONAL/country, **warehouses at every stock-chain hop** — see
  `qa/mhbase-console-perf-testing.md` for the full sheet rules.
- Single-session tokens: re-login as the same user (e.g. portal login mid-test) invalidates the
  app session (`context/gotchas.md` #27) — use distinct users per surface while testing stock.
- Device timing: facility downsync is client-bound (~9 rec/s); on a tenant carrying perf-test
  facility batches a full sync is 30+ min — schedule stock-flow app tests accordingly.

## Stock on the reporting/DST side

- Stock transactions live in the stock ES index alongside tasks; the user-sync/activity
  pipeline scrolls it with 2 parallel slices (vs 8 for tasks) and reports WAREHOUSE_MANAGER
  activity as the `warehousemanagerNactivity` sheet (staff x taskDate pivot); parse
  `recordedTime` once after fetch, not per-row.
- Campaign staff-role facts: Bauchi prod = DISTRIBUTOR 17,103 + WAREHOUSE_MANAGER 1,073 (no
  COMMUNITY_DISTRIBUTOR).
- The D&E Reports Drive tree carries per-state `stock report` / `Stock Report` folders (seen
  for Bauchi C1, Borno C2, Kogi C2, Nasarawa C2, Oyo C2, Plateau C2) — stock extracts are a
  separate deliverable from CHILDREN_TREATED and are NOT covered by the null-datapoints audit.
