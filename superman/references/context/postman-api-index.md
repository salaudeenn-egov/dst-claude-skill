# HCM Postman Collection — Endpoint Index

Auto-generated index of the bundled collection `HCM Collection`.
Contains 183 requests across 23 folders.

Full collection: `references/assets/HCM_Collection.postman_collection.json` (2.3 MB — import into Postman; do not read raw).


## User Services
- `POST  ` {{URL}}/user/users/_createnovalidate  — User CreateNoValidate QA
- `POST  ` {{URL}}/user/oauth/token  — OAuth Token
- `POST  ` /user/users/_updatenovalidate  — User Password Reset
- `POST  ` {{URL}}/user/_search  — User Search
- `POST  ` /egov-hrms/employees/_create?tenantId=mz&_=1681391576320  — User Creation HRMS DEMO
- `POST  ` {{URL}}/health-hrms/employees/_create?tenantId=mz  — User Creation HRMS QA
- `POST  ` {{URL}}/health-hrms/employees/_create?tenantId=mz  — User update HRMS QA
- `POST  ` {{URL}}/health-hrms/employees/_update?tenantId=mz  — User update HRMS QA
- `POST  ` /user/users/_updatenovalidate  — User Update No Validate
- `POST  ` {{URL}}/user/users/_updatenovalidate  — User Update No Validate Copy
- `POST  ` {{URL}}/egov-hrms/employees/_search?limit=10&offset=0&tenantId=mz&codes=amp-attendance-user  — search employee
- `POST  ` {{URL}}/health-hrms/employees/_update?tenantId=mz&_=1696486392041  — update HRMS
- `POST  ` {{URL}}/egov-hrms/employees/_create?tenantId=mz  — HRMS create Supervisor Copy

## Project
- `POST  ` {{URL}}/health-project/v1/_create  — Project Create INDI
- `POST  ` {{URL}}/health-project/v1/_create  — Project Create HOUSE
- `POST  ` /project/v1/_create  — Project Create bulk
- `POST  ` /project/v1/_create  — Project Create Multiround
- `POST  ` {{URL}}/project/data/errordump  — Project Data error dump
- `POST  ` /health-project/v1/_update  — Project Update local
- `POST  ` {{URL}}/health-project/v1/_update  — Project Update
- `POST  ` {{URL}}/project/v1/_update  — Project Create Copy
- `POST  ` /project/v1/_search?offset=0&limit=1000&tenantId=si  — Project Search

## Project Staff
- `POST  ` {{URL}}/project/staff/v1/_create  — Project Staff Create
- `POST  ` {{URL}}/health-project/staff/v1/bulk/_create  — Project Staff Bulk Create
- `POST  ` {{URL}}/project/staff/v1/_search?limit=100&offset=0&tenantId=ba  — Project Staff Search
- `POST  ` /project/staff/v1/_update  — Project Staff Update
- `POST  ` {{URL}}/health-project/staff/v1/bulk/_update  — Project Staff Bulk Update
- `POST  ` {{URL}}/health-project/staff/v1/_delete  — Project Staff Delete
- `POST  ` {{URL}}/health-project/staff/v1/bulk/_delete  — Project Staff Bulk Delete

## Product
- `POST  ` {{URL}}/product/v1/_create  — Product Create
- `POST  ` {{URL}}/product/v1/_search?limit=10&offset=0&tenantId=mz&includeDeleted=true  — Product Search
- `POST  ` {{URL}}/product/v1/_update  — Product Update

## Product Variant
- `POST  ` {{URL}}/product/variant/v1/_create  — Product Variant Create
- `POST  ` {{URL}}/product/variant/v1/_search?limit=100&offset=0&tenantId=bi  — Product Variant Search
- `POST  ` {{URL}}/product/variant/v1/_update  — Product Variant Update

## Project Resource
- `POST  ` /project/resource/v1/_create  — Project Resource Create
- `POST  ` /project/resource/v1/bulk/_create  — Project Resource Bulk Create
- `POST  ` {{URL}}/project/resource/v1/bulk/_update  — Project Resource Bulk Update
- `POST  ` {{URL}}/health-project/resource/v1/bulk/_delete  — Project Resource Bulk Delete
- `POST  ` {{URL}}/project/resource/v1/_update  — Project Resource Update
- `POST  ` {{URL}}/health-project/resource/v1/_delete  — Project Resource Delete
- `POST  ` /project/resource/v1/_search?limit=1000&offset=0&tenantId=si  — Project Resource Search

## Facility
- `POST  ` {{URL}}/facility/v1/_create  — Facility Create
- `POST  ` {{URL}}/facility/v1/_update  — Facility Update
- `POST  ` {{URL}}/facility/v1/bulk/_update  — Facility Bulk Update
- `POST  ` {{URL}}/facility/v1/bulk/_create  — Facility Bulk Create
- `POST  ` {{URL}}/facility/v1/bulk/_delete  — Facility Bulk Delete
- `POST  ` {{URL}}/facility/v1/_delete  — Facility Delete
- `POST  ` {{URL}}/facility/v1/_search?limit=1000&offset=0&tenantId=cg  — Facility Search

## Project Facility
- `POST  ` {{URL}}/health-project/facility/v1/_create  — Project Facility Create
- `POST  ` {{URL}}/project/facility/v1/_search?limit=100&offset=0&tenantId=cg  — Project Facility Search
- `POST  ` {{URL}}/health-project/facility/v1/_update  — Project Facility Update
- `POST  ` {{URL}}/project/facility/v1/_delete  — Project Facility Delete
- `POST  ` {{URL}}/project/facility/v1/bulk/_create  — Project Facility Bulk Create
- `POST  ` {{URL}}/health-project/facility/v1/bulk/_update  — Project Facility Bulk Update
- `POST  ` {{URL}}/health-project/facility/v1/bulk/_delete  — Project Facility Bulk Delete

## Household
- `POST  ` /household/v1/_create  — Household Create
- `POST  ` {{URL}}/household/v1/bulk/_create  — Household Bulk Create
- `POST  ` /household/v1/_search?limit=200&offset=0&tenantId=si  — Household Search
- `POST  ` /household/v1/_update  — Household Update
- `POST  ` {{URL}}/household/v1/bulk/_update  — Household Bulk Update
- `POST  ` {{URL}}/household/v1/_delete  — Household Delete
- `POST  ` {{URL}}/household/v1/bulk/_delete  — Household Bulk Delete
- `POST  ` {{URL}}/household/v1/_search?limit=100&offset=0&tenantId=default&includeDeleted=true  — Household Search - Proximity Based

## Individual
- `POST  ` /individual/v1/_create  — Individual Create
- `POST  ` /individual/v1/_create  — Individual Bulk Create
- `POST  ` /individual/v1/bulk/_create  — Individual Bulk Create Copy
- `POST  ` /individual/v1/_search?limit=1000&offset=0&tenantId=si  — Individual Search
- `POST  ` /individual/v1/_update  — Individual Update
- `POST  ` {{URL}}/health-individual/v1/bulk/_update  — Individual Bulk Update
- `POST  ` {{URL}}/health-individual/v1/_delete  — Individual Delete
- `POST  ` {{URL}}/health-individual/v1/bulk/_delete  — Individual Bulk Delete
- `POST  ` /individual/v1/_search?limit=1000&offset=0&tenantId=default&includeDeleted=true  — Individual Search - Proximity based search with identifier
- `POST  ` /individual/v1/_search?limit=1000&offset=0&tenantId=default&includeDeleted=true  — Individual Search - Proximity Based Search

## Project Beneficiary
- `POST  ` /project/beneficiary/v1/_create  — Project Beneficiary Create
- `POST  ` {{URL}}/health-project/beneficiary/v1/bulk/_create  — Project Beneficiary Bulk Create
- `POST  ` /project/beneficiary/v1/_search?limit=1000&offset=00&tenantId=si  — Project Beneficiary Search
- `POST  ` {{URL}}/project/beneficiary/v1/_update  — Project Beneficiary Update
- `POST  ` {{URL}}/health-project/beneficiary/v1/bulk/_update  — Project Beneficiary Bulk Update
- `POST  ` {{URL}}/project/beneficiary/v1/_delete  — Project Beneficiary Delete
- `POST  ` {{URL}}/health-project/beneficiary/v1/bulk/_delete  — Project Beneficiary Bulk Delete

## Project Task
- `POST  ` {{URL}}/health-project/task/v1/_create  — Project Task Create
- `POST  ` {{URL}}/project/task/v1/_create  — Project Task Create MultiRound
- `POST  ` {{URL}}/health-project/task/v1/bulk/_create  — Project Task Bulk Create
- `POST  ` {{URL}}/project/task/v1/bulk/_create  — Project Task Bulk Create MultiRound
- `POST  ` /project/task/v1/_search?limit=200&offset=1600&tenantId=bi&includeDeleted=true  — Project Task Search
- `POST  ` /project/task/v1/_update  — Project Task Update
- `POST  ` {{URL}}/health-project/task/v1/_search?limit=1000&offset=0&tenantId=mz&includeDeleted=true  — Project Task Search Copy
- `POST  ` {{URL}}/health-project/task/v1/bulk/_update  — Project Task Bulk Update
- `POST  ` {{URL}}/health-project/task/v1/_delete  — Project Task Delete

## Side Effect
- `POST  ` {{URL}}/referralmanagement/side-effect/v1/_create  — Side Effect Create
- `POST  ` {{URL}}/referralmanagement/side-effect/v1/bulk/_create  — Side Effect Bulk Create
- `POST  ` {{URL}}/referralmanagement/side-effect/v1/_search?limit=1000&offset=0&tenantId=mz  — Side Effect Search
- `POST  ` {{URL}}/referralmanagement/side-effect/v1/_update  — Side Effect Update
- `POST  ` {{URL}}/referralmanagement/side-effect/v1/bulk/_update  — Side Effect Bulk Update
- `POST  ` {{URL}}/referralmanagement/side-effect/v1/_delete  — Side Effect Delete
- `POST  ` {{URL}}/referralmanagement/side-effect/v1/bulk/_delete  — Side Effect Bulk Delete

## Adverse Event ADRM
- `POST  ` /adrm/adverse_event/v1/_create  — Adverse Event Create
- `POST  ` /adrm/adverse_event/v1/bulk/_create  — Adverse Event Bulk Create
- `POST  ` {{URL}}/project/task/adverse_event/v1/_search?limit=100&offset=0&tenantId=default&includeDeleted=true  — Adverse Event Search
- `POST  ` {{URL}}/project/task/adverse_event/v1/_update  — Adverse Event Update
- `POST  ` {{URL}}/project/task/adverse_event/v1/bulk/_update  — Adverse Event Bulk Update
- `POST  ` {{URL}}/project/task/adverse_event/v1/_delete  — Adverse Event Delete
- `POST  ` {{URL}}/project/task/adverse_event/v1/bulk/_delete  — Adverse Event Bulk Delete

## Referral Management
- `POST  ` {{URL}}/referralmanagement/v1/_create  — Referral Create
- `POST  ` {{URL}}/referralmanagement/v1/bulk/_create  — Referral Bulk Create
- `POST  ` {{URL}}/referralmanagement/v1/bulk/_create  — Referral Bulk Create Copy
- `POST  ` {{URL}}/referralmanagement/v1/_search?limit=100&offset=0&tenantId=mz  — Referral Search
- `POST  ` {{URL}}/referralmanagement/v1/_update  — Referral Update
- `POST  ` {{URL}}/referralmanagement/v1/bulk/_update  — Referral Bulk Update
- `POST  ` {{URL}}/referralmanagement/v1/_delete  — Referral Delete
- `POST  ` {{URL}}/referralmanagement/v1/bulk/_delete  — Referral Bulk Delete

## Household Member
- `POST  ` /household/member/v1/_create  — Household Member Create
- `POST  ` {{URL}}/household/member/v1/bulk/_create  — Household Member Bulk Create
- `POST  ` /household/member/v1/_search?limit=100&offset=0&tenantId=bi  — Household member Search
- `POST  ` {{URL}}/household/member/v1/_update  — Household Member Update
- `POST  ` {{URL}}/household/member/v1/bulk/_update  — Household Member Bulk Update
- `POST  ` {{URL}}/household/member/v1/_delete  — Household Member Delete
- `POST  ` {{URL}}/household/member/v1/bulk/_delete  — Household Member Bulk Delete

## Stock
- `POST  ` {{URL}}/stock/v1/_create  — Stock Create
- `POST  ` {{URL}}/stock/v1/bulk/_create  — Stock Bulk Create
- `POST  ` {{URL}}/stock/v1/bulk/_create  — Stock Bulk Create Copy
- `POST  ` /stock/v1/_update  — Stock Update
- `POST  ` {{URL}}/stock/v1/bulk/_update  — Stock Bulk Update
- `POST  ` {{URL}}/stock/v1/_delete  — Stock Delete
- `POST  ` {{URL}}/stock/v1/bulk/_delete  — Stock Bulk Delete
- `POST  ` /stock/v1/_search?offset=0&limit=100&tenantId=mz  — Stock Search

## Stock Reconciliation
- `POST  ` {{URL}}/stock/reconciliation/v1/_create  — Stock Reconciliation Create
- `POST  ` {{URL}}/stock/reconciliation/v1/bulk/_create  — Stock Reconciliation Bulk Create
- `POST  ` {{URL}}/stock/reconciliation/v1/_update  — Stock Reconciliation Update
- `POST  ` {{URL}}/stock/reconciliation/v1/bulk/_update  — Stock Reconciliation Bulk Update
- `POST  ` {{URL}}/stock/reconciliation/v1/_delete  — Stock Reconciliation Delete
- `POST  ` {{URL}}/stock/reconciliation/v1/bulk/_delete  — Stock Reconciliation  Bulk Delete
- `POST  ` {{URL}}/stock/reconciliation/v1/_search?limit=100&offset=0&tenantId=mz  — Stock Reconciliation Search

## Survey
- `POST  ` {{URL}}/egov-survey-services/egov-ss/survey/_create  — Survey Create
- `POST  ` {{URL}}/egov-survey-services/egov-ss/survey/_search?limit=100&tenantIds=default  — Survey Search
- `POST  ` {{URL}}/egov-survey-services/egov-ss/survey/_update  — Survey Update
- `POST  ` {{URL}}/egov-survey-services/egov-ss/survey/_delete  — Survey Delete
- `POST  ` {{URL}}/egov-survey-services/egov-ss/survey/response/_submit  — Submit Survey
- `POST  ` {{URL}}/egov-survey-services/egov-ss/survey/response/_results  — Fetch Survey Results

## MDMS
- `POST  ` {{URL}}/egov-location/location/v11/boundarys/_search?hierarchyTypeCode=ADMIN&tenantId=mz  — Fetch Boundary Data
- `POST  ` {{URL}}/egov-mdms-service/v1/_search  — Fetch Actions Test
- `POST  ` {{URL}}/egov-mdms-service/v1/_search  — Fetch Role Action Mappings
- `POST  ` {{URL}}/egov-mdms-service/v1/_search  — Fetch HCM Project Types
- `POST  ` {{URL}}/egov-mdms-service/v1/_search  — Fetch HCM Project Id search
- `POST  ` {{URL}}/egov-mdms-service/v1/_search  — Fetch HCM Field App Configs
- `POST  ` {{URL}}/egov-mdms-service/v1/_search  — Fetch HCM Project Task Configs
- `POST  ` {{URL}}/egov-mdms-service/v1/_search  — Fetch HCM Service Registry

## ServiceRequest
- `POST  ` {{URL}}/service-request/service/definition/v1/_create  — Service Definition Create
- `POST  ` {{URL}}/service-request/service/definition/v1/_create  — Service Definition Create Copy
- `POST  ` {{URL}}/service-request/service/definition/v1/_search  — Service Definition Search
- `POST  ` /service-request/service/definition/v1/_update  — Service Definition Update
- `POST  ` /service-request/service/definition/v1/_update  — service definition update - status
- `POST  ` /service-request/service/v1/_create  — Service Create
- `POST  ` {{URL}}/service-request/service/v1/_search  — Service Search
- `POST  ` {{URL}}/user/oauth/token  — OAuth Token Copy

## HFReferral
- `POST  ` {{URL}}/referralmanagement/hf-referral/v1/_create  — HFReferral Create
- `POST  ` {{URL}}/referralmanagement/hf-referral/v1/bulk/_create  — HFReferral Bulk Create
- `POST  ` {{URL}}/referralmanagement/hf-referral/v1/_search?limit=100&offset=0&tenantId=mz  — HFReferral Search
- `POST  ` {{URL}}/referralmanagement/hf-referral/v1/_update  — HFReferral Update
- `POST  ` {{URL}}/referralmanagement/hf-referral/v1/bulk/_update  — HFReferral Bulk Update
- `POST  ` {{URL}}/referralmanagement/hf-referral/v1/_delete  — HFReferral Delete
- `POST  ` {{URL}}/referralmanagement/hf-referral/v1/bulk/_delete  — HFReferral Bulk Delete
- `POST  ` /referralmanagement/hf-referral/v1/_create  — hf referral create

## Localization
- `POST  ` {{URL}}/user/oauth/token  — OAuth Token
- `POST  ` {{URL}}/localization/messages/v1/_upsert  — Upsert Localization
- `POST  ` /localization/messages/v1/_search?locale=en_MZ&tenantId=default  — Fetch Localization DEV
- `POST  ` {{URL}}/localization/messages/v1/_search?locale=en_MZ&tenantId=mz  — Fetch Localization QA
- `POST  ` /localization/messages/v1/_search?locale=en_MZ&tenantId=mz  — Fetch Localization DEMO
- `POST  ` {{dev}}/project/resource/v1/_search  — Test
- `POST  ` {{URL}}/egov-mdms-service/v1/_search  — https://unified-dev.digit.org/egov-mdms-service/v1/_search
- `POST  ` {{URL}}/health-individual/v1/_create  — {{URL}}/health-individual/v1/_create
- `POST  ` {{URL}}/user/_logout?tenantId=mz&_=1709643100336  — logout Copy
- `POST  ` /boundary-service/boundary-relationships/_search?tenantId=mz&hierarchyType=BURUNDI&includeChildren=true  — https://moz-health-uat.digit.org/boundary-service/boundary-relationships/_search?tenantId=mz&hierarchyType=BURUNDI&includeChildren=true
- `POST  ` /boundary-service/boundary-relationships/_search?tenantId=mz&hierarchyType=BURUNDI&includeChildren=true  — https://moz-health-uat.digit.org/boundary-service/boundary-relationships/_search?tenantId=mz&hierarchyType=BURUNDI&includeChildren=true
- `POST  ` /boundary-service/boundary-relationships/_search?tenantId=mz&boundaryType=Country&codes=mz&includeChildren=true&hierarchyType=IRS  — https://salama.digit.org/boundary-service/boundary-relationships/_search?tenantId=mz&boundaryType=Country&codes=mz&includeChildren=true&hierarchyType=IRS
- `POST  ` /localization/messages/v1/_search?module=rainmaker-common,rainmaker-ng,rainmaker-works,rainmaker-hcm-common,rainmaker-hcm-dss,rainmaker-hr,rainmaker-pgr&locale=en_NG&tenantId=ng&_=1741166535232  — https://prod.hcm-kano.org/localization/messages/v1/_search?module=rainmaker-common,rainmaker-ng,rainmaker-works,rainmaker-hcm-common,rainmaker-hcm-dss,rainmaker-hr,rainmaker-pgr&locale=en_NG&tenantId=ng&_=1741166535232
- `POST  ` /attendance/attendee/v1/_create  — http://localhost:8080/attendance/attendee/v1/_create
- `POST  ` tsindamalariya.bi/localization/messages/v1/_upsert  — http://localhost:8387/localization/messages/v1/_upsert
- `POST  ` /health-expense/bill/v1/_search  — https://congo-hcm-uat.digit.org/health-expense/bill/v1/_search
- `POST  ` /user/oauth/token  — https://training-hcm-kano.digit.org/user/oauth/token
- `POST  ` /household/v1/bulk/_create  — https://sierraleone-hcm-uat.digit.org/household/v1/bulk/_create
- `GET   ` /  — New Request