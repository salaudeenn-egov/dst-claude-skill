# DST Team Context + HCM Module Surface

Background context for the DST work-stream, distilled from the DIGIT Success Team
training curriculum. Explains what DST is and the full HCM operational surface the
team supports — so data work is understood in its wider context. (Session-recording
passwords from the source sheet are intentionally omitted; they are not durable facts.)

## What DST is

**DST = DIGIT Success Team** — the support/operations org for HCM (Health Campaign
Management) implementations. Tiered: **L1, L2, and Manager** roles, with a defined
governance mechanism and published metrics. DST supports live campaigns end to end:
readiness, monitoring, incident management, data integrity, dashboards, and reporting.
Salaudeen is an L3 SWE doing the data-investigation/fix work within this context.

DST responsibilities seen in the curriculum:
- HCM product + Salama application overview (Bednet, SMC).
- Campaign readiness tracker; sync with the Health Team on upcoming campaigns.
- Governance mechanism + metrics to publish.
- Incident management and monitoring tools.
- Country/campaign familiarity: Mozambique Bednet/SMC, Liberia (Margibi + national),
  Senegal SMC, Nigeria LF, Kebbi SMC, plus the Nigeria SMC/AZM/ITN campaigns tracked in
  the `campaigns` reference.

## HCM module surface (the whole product the data work sits inside)

The HCM training curriculum covers these modules — useful when a data task touches a
part of the product beyond the entity chain:

| Module | What it covers |
|---|---|
| HCM app features | Registration + Distribution flows (the offline CDD app). |
| HRMS / User management | User creation, bulk user creation, bulk deactivation/reactivation. |
| Supervision, Monitoring & Attendance | Supervision monitoring; bulk attendance creation. |
| Localization | Language + localization config; loading localization data; removing a language. |
| Microplan ingestion (backend) | Master-data templates; understanding a micro plan; multi-part ingestion process. |
| Inventory management | Stock / product / product-variant / project-resource. |
| Dashboards & M&E | Kibana dashboards (Bednet + SMC versions); configuring a new KPI in Kibana; custom dashboards. |
| Custom Reports | Executing custom reports (distinct from the Python report pipeline). |
| HCM Payments | Payment flows. |
| Complaints Management / Helpdesk | PGR / helpdesk. |
| Admin Console (no-code) | Setting up a campaign, delivery rules, uploading user data. |
| Infra / Monitoring | Monitoring-tool usage; infra monitoring. |
| Data | DB queries and views; ingestion using Jupyter. |

## Why this matters for DST data work

- The entity chain (Individual -> Household -> HouseholdMember -> ProjectBeneficiary ->
  ProjectTask) is the data spine, but campaigns also depend on HRMS users, project-staff
  (CDD) assignment, microplan/master data, boundaries/localization, and inventory —
  a data anomaly can originate in any of those (e.g. CDD sync issues trace to
  project-staff + user-sync, not the delivery chain).
- Dashboards/Kibana are the operational view; the Python analysis pipeline is the
  authoritative reporting view. Per the numbers rule, reports use analysis numbers, not
  dashboard numbers (see hcm-context house rules).
- Admin Console (no-code) is where campaigns, delivery rules, and user data are set up —
  relevant when a tenant stores fields differently (e.g. Kebbi/admin-console stores gender
  in `additionalDetails.gender`, unlike most tenants' `Data.gender`).

Source: "Training Schedule for DIGIT Success Team.xlsx" (DST L1/L2, DST Manager,
Prerna/Suresh, and HCM training sheets).
