# Industry Insurance Debugger

**Trigger:** Industries Insurance services, insurance rating, quoting, policy administration, claims processing, enrollment, billing, commissions, `vlocity_ins` namespace (carrier-side), FSC Insurance objects (`InsurancePolicy`, `Claim`), Insurance-specific OmniStudio components, `InsPolicyService`, `InsQuoteService`, `InsClaimService`, `InsProductService`, `InsEnrollmentService`.

> **Scope distinction:** This vertical handles **carrier-side** insurance (policy admin, quoting, rating, enrollment, billing, commissions, product catalog). **Health Cloud** handles **payer-side** (claims adjudication, utilization management, benefit verification, care plans).

---

## Critical First Step: Identify Data Model Variant

Before investigating, determine which data model the org uses:

| Dimension | Harmonized FSC Model (Current) | Legacy Vlocity Model |
|---|---|---|
| Policy object | `InsurancePolicy` (standard) | `Asset` |
| Coverage | `InsurancePolicyCoverage` | `AssetCoverage__c` |
| Insured items | `InsurancePolicyAsset` | `AssetInsuredItem__c` |
| Participants | `InsurancePolicyParticipant` | `AssetPartyRelationship__c` |
| Package | `via_ins` + `via_ins_fsc` | `via_ins` only |

Check the org's harmonization state by looking for `InsurancePolicy` records or `Asset`-based policies in the case context.

---

## Product Areas

| Area | Scope |
|---|---|
| Policy Administration | Create, endorse, renew, cancel, reinstate policies; payment schedules; OOSE |
| Quoting | Quote creation, rating, cloning, endorsement quotes, product rules |
| Claims | Claim creation, coverage verification, payments, recoveries |
| Products & Rating | Product eligibility, async rating, rate bands, product rules |
| Enrollment & Census | Member enrollment, census management, group benefits |
| Billing & Revenue | Billing statements, revenue schedules, policy terms |
| Contracts | Contract creation, group class management |
| Commissions | Commission calculation, schedule assignments |

---

## Repository Architecture

```
┌───────────────────────────────────────────────────┐
│  via_components (UI)        — shared LWC library  │
├───────────────────────────────────────────────────┤
│  via_ins (Business Logic)   — 150+ Insurance Apex │
├───────────────────────────────────────────────────┤
│  via_ins_fsc (Bridge)       — INS ↔ FSC mapping   │
├───────────────────────────────────────────────────┤
│  via_platform (OmniStudio)  — DR/IP/OS/FC engine  │
├───────────────────────────────────────────────────┤
│  via_core (Foundation)      — shared runtime      │
├───────────────────────────────────────────────────┤
│  Salesforce Platform (FSC + Standard Objects)     │
└───────────────────────────────────────────────────┘
```

### GitHub Repos

| Repository | GitHub Path | Key Content |
|---|---|---|
| `via_ins` | `sf-industries/via_ins` | Core Insurance services, triggers, LWC |
| `via_ins_fsc` | `sf-industries/via_ins_fsc` | FSC bridge (Get*/Post* classes, field mappings) |
| `via_ins_ps` | `sf-industries/via_ins_ps` | Insurance Professional Services extensions |
| `via_insx` | `sf-industries/via_insx` | Insurance extensions |
| `via_xom_ins` | `sf-industries/via_xom_ins` | Insurance XOM (order management) |
| `via_fincommon` | `sf-industries/via_fincommon` | Financial common (shared INS/FSC) |
| `via_core` | `sf-industries/via_core` | DREngine, InvokeService, StateTransition, SecurityChecker |
| `via_platform` | `sf-industries/via_platform` | OmniStudio engine, DataRaptor, IntProc |
| `via_components` | `sf-industries/via_components` | Shared LWC (wizard, form, datasource) |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-insurance/                        ← Insurance UDD (filters, profiles)
  core/industries-insurance-impl/                   ← Insurance core implementation
  core/industries-insurance-claim-impl/             ← Claims implementation
  core/industries-insurance-billing-impl/           ← Insurance billing
  core/industries-insurance-policy-common-impl/     ← Policy common implementation
  core/industries-insurance-groupbenefits-impl/     ← Group benefits
  core/industries-insurance-brokerage-impl/         ← Brokerage implementation
  core/industries-insurance-reinsurance/            ← Reinsurance
  core/industries-interaction-ptc/apex/vlocity_ins/ ← PTC layer (protected Apex)
```

### PTC Layer Files

```
InsuranceClaimServicePtc.apex          InsuranceRatingPtc.apex
InsuranceLoggingServicePtc.apex        InsEnrollmentServicePtc.apex
InsCensusServicePtc.apex               InsContractServicePtc.apex
InsGroupClassServicePtc.apex           InsUserFinancialAuthorityServicePtc.apex
InsAsyncBulkServicePtc.apex
```

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| Insurance Knowledge Base | https://confluence.internal.salesforce.com/spaces/IN/pages/601939902/Insurance+Knowledge+Base |
| On-Call Support Playbook | https://salesforce.quip.com/LqfPAPuNJmpg |
| Insurance Known Issues | https://confluence.internal.salesforce.com/spaces/IN/pages/630530890/Insurance+Known+Issues+and+Limitations |
| Insurance GCC (Best Practices) | https://confluence.internal.salesforce.com/spaces/CSGPAK/pages/342398772/Insurance+GCC |

---

## Insurance Services Catalog (29 Services)

All services use the **Vlocity Open Interface** pattern:
```
InvokeService.invoke(className, methodName, inputJSON, optionsJSON)
  → SecurityChecker.canAccessClass(className)
    → <Service>.invokeMethod(methodName, input, output, options)
```

### Policy Administration
| Service | Key Methods |
|---|---|
| `InsPolicyService` | `createUpdatePolicy`, `cancelPolicy`, `createPolicyVersion`, `createRenewalPolicy`, `createPaymentSchedule`, `verifyCoverage` |
| `InsurancePolicyService` | `createPolicyVersion`, `cancelPolicy`, `reinstatePolicy` |
| `InsurancePolicyRenewalService` | batch renewal processing |
| `InsPolicyBatchRenewalService` | bulk renewal jobs |
| `OutOfSequenceEndorsementService` | OOSE mid-term modifications |
| `InsurancePolicyTransactionService` | `reverseTransaction` |

### Quoting
| Service | Key Methods |
|---|---|
| `InsQuoteService` | `createUpdateQuote`, `getQuoteDetail`, `priceRootItem`, `cloneQuote`, `createEndorsementQuote`, `invokeProductRules` |
| `InsuranceQuoteService` | quote engine with rating and attributes |
| `InsuranceQuotePolicyService` | quote-to-policy conversion |

### Claims
| Service | Key Methods |
|---|---|
| `InsClaimService` | `createUpdateClaim`, `verifyCoverage`, `invokeProductRules`, `invokeRules` |
| `InsClaimItemService` | `add`, `delete`, `update`, `calculateCoverages`, `createPayments`, `cancelPayments` |
| `InsClaimCoverageService` | `createUpdateCoverage`, `invokeProductRules` |
| `InsClaimRecoveryService` | `getRecoveries`, `save` |

### Products & Rating
| Service | Key Methods |
|---|---|
| `InsProductService` | `getEligibleProducts`, `getRatedProducts`, `repriceProduct`, `cloneProduct`, `runRules` |
| `InsProductAsyncRatingService` | `startAsyncRating`, `fetchProductsPrice`, `repriceProduct` |
| `InsProductJSONService` | `getAttributes`, `getCoverages`, `updateUserValues` |

### Enrollment & Census
| Service | Key Methods |
|---|---|
| `InsEnrollmentService` | `enrollMembers`, `enrollPlans`, `findEnrollees`, `getMemberEnrollments` |
| `InsEnrollmentServiceStd` | same methods (Standard/FSC model) |
| `InsCensusService` | `addMembers`, `getMembers`, `updateMembers`, `createAccounts` |
| `InsCensusServiceStd` | same methods (Standard/FSC model) |

### Billing & Revenue
| Service | Key Methods |
|---|---|
| `InsPolicyBillingService` | `generateBillingAccountStatements`, `generateDirectBillingStatements` |
| `InsPolicyRevenueScheduleService` | `createRevenueSchedule`, `modifyRevenueSchedule`, `cancelRevenueSchedule` |
| `InsPolicyTermsService` | `getCurrentStanding` |

### Other
| Service | Key Methods |
|---|---|
| `InsContractService` / `InsContractServiceStd` | `createUpdateContract`, `getContractDetail` |
| `InsCommissionService` | `calculate`, `adjust` |
| `InsGroupClassService` | `getGroupClassByAccount`, `getGroupClassesByContract` |
| `InsAsyncBulkService` | `getRequestStatusById`, `getRequestStatusByUser` |
| `InsScheduledAutomatedTransitionService` | `scheduleAutomatedTransition` |
| `InsVlocityActionService` | `assignRecordToQueue`, `notifyApplicant` |
| `StateRuleService` | `invokeRules`, `getRuleLogs`, `getTransitionStates` |

---

## Data Model (Harmonized FSC)

```
InsurancePolicy (224KB object — heavily extended)
├── InsurancePolicyCoverage (master-detail)
├── InsurancePolicyAsset (master-detail)
├── InsurancePolicyParticipant (master-detail, 56KB)
├── InsurancePolicyTerms (master-detail)
│   └── InsurancePolicyTermTrackingEntry__c (master-detail)
├── InsurancePolicyPaymentScheduleEntry__c (lookup)
│   └── InsPaymentScheduleEntryDetail__c (child)
├── InsurancePolicyTransaction__c
├── InsurancePolicyPricingAdjustment__c
├── InsurancePolicyRevenueScheduleEntry__c
├── InsurancePolicyCommission__c
└── InsurancePolicySurcharge

Claim
├── ClaimItem
│   └── ClaimCoverage
│       ├── ClaimCoveragePaymentDetail
│       └── ClaimCoverageReserveAdjustment
└── ClaimParticipant

Quote → QuoteLineItem
GroupCensus → GroupCensusMember → GroupCensusMemberPlan
Contract → ContractGroupPlan → GroupClassContribution
CommissionSchedule → CommissionScheduleAssignment
```

---

## Insurance-Specific Debug Log Patterns

| Log pattern | Meaning |
|---|---|
| `InsPolicyService.invokeMethod` | Entry point for policy operations |
| `InsurancePolicyService.createPolicyVersion` | Endorsement/MTA operation |
| `InsurancePolicyService.createRenewalPolicy` | Renewal operation |
| `InsPolicyPayScheduleDetailService` | Payment schedule detail operations |
| `OutOfSequenceEndorsementService` | Out-of-sequence endorsement |
| `AttributeRatingService` | Rating calculation |
| `InsuranceProductRuleService` | Product rule evaluation |

---


---

## Sample SOQL Queries

### Policies for an account
```soql
SELECT Id, Name, PolicyType, Status, EffectiveDate, ExpirationDate,
       (SELECT Id, Name, CoverageAmount FROM InsurancePolicyCoverages)
FROM InsurancePolicy
WHERE NameInsuredId = '<ACCOUNT_ID>'
ORDER BY EffectiveDate DESC LIMIT 10
```

### Claims for a policy
```soql
SELECT Id, CaseNumber, Subject, Status, CreatedDate,
       (SELECT Id, Name, Type FROM ClaimItems)
FROM Claim
WHERE PolicyNumberId = '<POLICY_ID>'
ORDER BY CreatedDate DESC LIMIT 10
```

### Quotes
```soql
SELECT Id, Name, Status, ExpirationDate, GrandTotal,
       Account.Name
FROM Quote
WHERE Account.Id = '<ACCOUNT_ID>'
ORDER BY CreatedDate DESC LIMIT 10
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex uncaught exceptions (service classes, triggers) |
| `ipipr` | Integration Procedures (Insurance IPs for quoting, enrollment) |
| `ipdar` | DataRaptors (Insurance data operations) |
| `axlim` | Governor limits (bulk rating, large policy operations) |
| `gslog` | Platform Java exceptions (InsuranceClaimServicePtc, InsuranceRatingPtc) |

## Known Error Patterns

| Error | Root Cause | Resolution |
|---|---|---|
| `NullPointerException` in `createPaymentDetailRecordsForFuturePSEs` | PSE records exist without child `InsPaymentScheduleEntryDetail__c` records. Occurs when `createRenewalPolicy` called without `useIsPaidFlag=true`. | Add `useIsPaidFlag=true` to `createRenewalPolicy` input. GUS: W-14605992 (Closed). |
| `NullPointerException` in OOSE with payment schedule | Same root cause — out-of-sequence endorsement on renewal policies missing PSE details. | Same fix: ensure `useIsPaidFlag=true` on renewal. |
| "You have uncommitted work pending" | DML before callout in rating/quoting flows | Reorder operations: callouts before DML, or use `@future`/queueable for callout |
| `NullPointerException` in `InsuranceRatingService` | Missing product configuration or plan data | Verify product catalog activation and effective date ranges |
| `FIELD_CUSTOM_VALIDATION_EXCEPTION` on InsurancePolicy | Validation rules blocking service writes | Identify and deactivate/adjust the validation rule for the service user context |
| `System.LimitException: Too many SOQL queries` | Bulk rating with complex product hierarchies | Reduce batch size; check for recursive triggers |
| Timeout on external rating callout | Named Credential misconfiguration or endpoint down | Check Named Credential config and endpoint availability |
| `System.LimitException: Too many DML statements` | Endorsement on policy with many coverages/assets | Process in smaller batches; check trigger recursion |
| `JSONException` or `Unable to deserialize` | Malformed input payload to Insurance service | Validate JSON structure against expected schema |
| Empty response from `getRatedProducts` | Product not active for effective date, or missing rating facts | Verify `EffectiveDate__c` range and required rating attributes |
| `InsurancePolicyLockingHandler` error | Concurrent modification of same policy | Retry after lock release; check for conflicting automation |

---

## Symptom-Driven Fast Path

| Symptom | First Check |
|---|---|
| NPE in payment schedule during endorsement | `InsPolicyPayScheduleDetailService` — check `useIsPaidFlag=true` on renewal (W-14605992) |
| Rating returns incorrect premium | Product/plan configuration; rating factor tables; `AttributeRatingService` |
| Quote generation fails | `InsuranceQuoteService`; input payload validation; product eligibility rules |
| Policy issuance error | `InsurancePolicyService`; validation rules on `InsurancePolicy`; required fields |
| Claims processing failure | `InsuranceClaimServicePtc.apex`; claim lifecycle state validation |
| Enrollment errors | `InsEnrollmentService` / `EnrollmentHandlerFSC`; census data format |
| Timeout on rating callout | Named Credential config; external endpoint availability; callout timeout |
| Bulk operation limit exception | `InsAsyncBulkService`; batch size configuration; governor limits |
| Product not found | `InsuranceProductService`; product catalog activation; effective date ranges |
| Financial authority denied | `InsUserFinancialAuthorityServicePtc.apex`; user permission setup |
| Contract service error | `InsContractServicePtc.apex`; contract state/lifecycle validation |
| State transition failure | `StateTransitionService` (via_core); `VlocityStateModel__c` config |
| Record locking error | `InsurancePolicyLockingHandler` (via_ins_fsc); concurrent modifications |
| Commission calculation wrong | `InsCommissionService`; `CommissionSchedule` setup; `ProductionCode__c` |
| Revenue schedule error | `InsPolicyRevenueScheduleService`; `InsurancePolicyRevenueScheduleEntry__c` |

---

## Code Investigation Paths

### via_ins — Insurance Service Classes (PRIMARY)
```
Tool: mcp__plugin_git-emu_vmcp-git-emu__get_file_contents
owner: "sf-industries"
repo: "via_ins"
path: "classes/<ClassName>.cls"
```

Key service classes by domain:
| Domain | Classes |
|---|---|
| Policy | `InsPolicyService`, `InsurancePolicyService`, `InsurancePolicyRenewalService`, `InsPolicyBatchRenewalService`, `OutOfSequenceEndorsementService`, `InsPolicyPayScheduleDetailService` |
| Quoting | `InsuranceQuoteService`, `InsuranceQuotePolicyService`, `InsQuoteCompareService`, `InsuranceQuoteClonerService` |
| Claims | `InsuranceClaimService`, `InsuranceClaimItemService`, `InsuranceClaimCoverageService`, `InsuranceClaimRecoveryService`, `InsuranceClaimDataService` |
| Products/Rating | `InsuranceProductService`, `AttributeRatingService`, `TaxAndFeeRatingService`, `ProductRateBandService`, `InsuranceProductRuleService` |
| Enrollment | `InsEnrollmentService`, `InsCensusService`, `CensusMemberService` |
| Billing | `InsurancePolicyBillingService` |
| Async | `InsAsyncJobService`, `InsAsyncBulkService` |
| Triggers | `InsuranceTriggerFactory` (factory pattern for all 29 triggers) |

### via_ins_fsc — FSC Bridge Classes
```
Tool: mcp__plugin_git-emu_vmcp-git-emu__get_file_contents
owner: "sf-industries"
repo: "via_ins_fsc"
path: "classes/<ClassName>.cls"
```

Key bridge classes:
| Pattern | Classes |
|---|---|
| Policy CRUD | `GetInsurancePolicy`, `PostInsurancePolicy`, `PostInsurancePolicyFromQuote` |
| Payment Schedule | `GetInsurancePolicyPaymentSchedule`, `PostInsurancePolicyPaymentSchedule`, `PostInsPolicyPaymentScheduleDetail` |
| Claims CRUD | `GetClaim`, `PostClaim`, `GetClaimItem`, `PostClaimItem` |
| Enrollment | `EnrollmentHandlerFSC`, `FSCEnrollmentService`, `FSCGroupClassService` |
| Mapping | `FSCApexMappings`, `FSCAbstractCallableImpl` |

### via_core — Platform Foundation

Relevant when the issue is in platform-level code:
- `InvokeService` — service dispatch routing
- `SecurityChecker` / `PlatformSecurityChecker` — FLS/CRUD failures
- `StateTransitionService` — policy state machine errors
- `AsyncProcessEngine` — batch job failures
- `OrgCacheManager` — platform cache issues
- `DREngine2` / `DRProcessor` — DataRaptor failures in Insurance flows

---

## Invocable Actions (Flow-Compatible)

| Action | Description |
|---|---|
| Create Accounts | Creates person accounts for group census members |
| Create Contacts | Creates contacts for group census members |
| Create Portal Users | Creates portal user records for census members |
| Enroll Members | Creates insurance policies by enrolling members into plans |
| Generate User Inputs | Generates user inputs and fact ratings |
| Rate Products | Rates insurance products for Group Benefits |

---

## Trigger Architecture

Insurance uses a **factory pattern** via `InsuranceTriggerFactory` with 29 triggers:
- Controlled by `TriggerSetup__c` custom setting (enable/disable per trigger)
- Key triggers: `Policy`, `PolicyAsset`, `InsuranceQuote`, `InsuranceClaim`, `ClaimLineItem`, `ClaimCoverage`, `InsuranceProduct`, `GroupCensus`, `FeeSchedule`

When debugging trigger-related issues, check:
1. Is the trigger enabled? (`TriggerSetup__c`)
2. Is there trigger recursion? (multiple DML in same transaction)
3. Are there conflicting triggers from other packages?

---

## Escalation

| Channel | Use |
|---|---|
| `#insurance-cce-support-all` (C0AH1JFCJ3D) | Insurance CCE support |
| `#industries-insurance` (C01PMNSFUKZ) | Insurance industry community |
| `#industries-insurance-service` (C02765RD8FK) | Internal expertise (no SLA) |
| `#industries-ins-pct` (C027DRQ28AW) | Insurance package PCT |
| `#support-omnistudio-collaboration` (C03GSNY2GVC) | OmniStudio-layer issues |
| `#support-swarm-industries` (C02BEHKLWES) | General Industries swarm |

**GUS product tags:** `Industries Insurance`, `Industries Interaction platform`, `Health Cloud`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Service: [InsPolicyService | InsQuoteService | InsClaimService | InsProductService | InsEnrollmentService | Other]
Data Model: [Harmonized FSC | Legacy Asset-based | Unknown]
Issue Description:
Reproduced in Demo org?:
Troubleshooting steps taken?:
Debug/Splunk logs verified?:
useIsPaidFlag checked? (if payment schedule related):
```
