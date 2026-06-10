# Health Cloud Debugger

**Trigger:** Health Cloud, care plans, care gaps, assessments, utilization management, benefit verification, `vlocity_ins` namespace (payer-side), HC-specific OmniStudio components, FHIR/HL7, provider network, virtual care, social determinants, DORA, account provisioning.

> **Scope distinction:** This vertical handles **payer-side** healthcare (claims adjudication, utilization management, benefit verification, care plans, care gaps, FHIR). For **carrier-side** insurance (policy admin, quoting, rating, enrollment, billing, commissions), route to the **Insurance** vertical.

---

## Product Areas

| Category | Features |
|---|---|
| Provider | Care Management, Life Sciences Program Mgmt, Clinical Data Model, Intelligent Appt Mgmt, Home Health, Provider Network, Social Determinants, Virtual Care |
| Payer | Claims, Code Sets, Health Insurance, Utilization Management |
| Med Device | Remote Monitoring, Device Registration |
| API | EHR (FHIR/HL7), Business APIs, Metadata APIs |
| Other | Slack for HC, Performance, Setup/Config, License/Permissions, Account Provisioning, DORA |

---

## Repository Architecture

### GitHub / git.soma Repos

| Repository | Path | Content |
|---|---|---|
| `healthcare` | `git.soma.salesforce.com/industries/healthcare` | Health Cloud managed package (primary) |
| `core` | `git.soma.salesforce.com/industries/core` | Shared Industries core |
| `build` | `git.soma.salesforce.com/industries/build` | Build infrastructure |
| `via_platform` | `github.com/sf-industries/via_platform` | OmniStudio Apex (vlocity_ins namespace, `package_ins.xml`) |
| `HealthCloudCCDAandEMRDatakit` | `github.com/sf-industries/HealthCloudCCDAandEMRDatakit` | CCDA and EMR data kit |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-healthcare-impl/                     ← HC Java implementation (FHIR, Care Plans, UM)
  core/ui-industries-common-components/omnistudio/     ← Shared components
  core/industries-interaction-ptc/apex/vlocity_ins/    ← PTC layer (shared with Insurance)
```

### PTC Layer (HC-Relevant)
```
AssessmentResponsesPtc.apex       ← Assessments (HC-specific)
InsuranceClaimServicePtc.apex     ← Claims adjudication
InsuranceRatingPtc.apex           ← Benefit verification
InsEnrollmentServicePtc.apex      ← Enrollment
InsCensusServicePtc.apex          ← Census management
InsContractServicePtc.apex        ← Contracts
```

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| Health Cloud Playbooks (Hub) | https://confluence.internal.salesforce.com/spaces/CCEIND/pages/1078971559/Health+Cloud+Playbooks |
| HC Developer Playbook | https://confluence.internal.salesforce.com/spaces/CCEIND/pages/1093963007/Health+Cloud+Developer+Playbook |
| HC Engineering Forum (HEF) | https://confluence.internal.salesforce.com/spaces/IN/pages/431752302/Health+Cloud+Engineering+Forum+-+HEF |
| HC Engineering | https://confluence.internal.salesforce.com/spaces/IN/pages/257593878/HC+Engineering |

---

## HC-Specific Objects

| Object | Description |
|---|---|
| `CarePlan` / `CarePlanProblem` / `CarePlanGoal` | Care management |
| `HealthCondition` | Patient conditions |
| `CareGap` | Care gap identification |
| `AssessmentQuestion` / `AssessmentResponse` | Assessments |
| `MemberPlan` | Insurance membership |
| `InsuranceCoverage` | Coverage details |
| `Claim` / `ClaimItem` | Insurance claims |
| `AuthorizationRequest` | Utilization management |
| `CarePreauth` | Prior authorization |

---

## Common Issues

| Symptom | Check |
|---|---|
| Assessment not saving | `AssessmentResponsesPtc.apex` changes, FLS on assessment objects |
| Care plan IP failing | `InsuranceClaimServicePtc.apex`, OIC flags |
| Enrollment errors | `InsEnrollmentServicePtc.apex`, `InsCensusServicePtc.apex` |
| Benefit verification failing | `InsuranceRatingPtc.apex`, callout config |
| HC DataRaptor missing fields | `EnforceDMFLSAndDataEncryption` OIC flag, `CheckFieldLevelSecurity__c` |
| Home Health visit error | Check scheduling objects, shift management |
| Provider Network setup | Provider relationship objects, network config |
| FHIR integration failing | Callout config, named credentials, external service |

---

## Sample SOQL Queries

### Care plans for a patient
```soql
SELECT Id, Name, Status, StartDate, EndDate, Subject.Name,
       (SELECT Id, Name, Priority, Status FROM CarePlanProblems),
       (SELECT Id, Name, Status, TargetDate FROM CarePlanGoals)
FROM CarePlan
WHERE Subject.Id = '<PATIENT_CONTACT_ID>'
ORDER BY StartDate DESC LIMIT 5
```

### Assessments for a patient
```soql
SELECT Id, Name, AssessmentType, Status, TotalScore, CompletedDateTime,
       (SELECT Id, AssessmentQuestion.Name, ResponseValue, IsRequired
        FROM AssessmentQuestionResponses)
FROM AssessmentResponse
WHERE Subject.Id = '<PATIENT_CONTACT_ID>'
ORDER BY CompletedDateTime DESC LIMIT 10
```

### Authorization requests
```soql
SELECT Id, CaseNumber, Status, Priority, AuthorizationType,
       StartDate, EndDate, Account.Name
FROM AuthorizationRequest
WHERE Account.Id = '<ACCOUNT_ID>'
ORDER BY CreatedDate DESC LIMIT 10
```

### Member plans
```soql
SELECT Id, Name, Status, MemberNumber, CoverageStartDate, CoverageEndDate,
       Account.Name
FROM MemberPlan
WHERE Account.Id = '<PATIENT_ACCOUNT_ID>'
  AND Status = 'Active'
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex uncaught exceptions (HC triggers, IP failures) |
| `ipipr` | Integration Procedures (HC uses many — care plan flows, enrollment) |
| `ipdar` | DataRaptors (patient data extraction, care plan creation) |
| `ipipa` | IP actions/blocks (individual step failures) |
| `gslog` | Platform Java exceptions (FHIR engine, UM, core HC Java) |
| `txerr` | Error logs (callout failures, external system errors) |

---

## Code Investigation Paths

### HC Managed Package
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:git.soma.salesforce.com/industries/healthcare content:<keyword>"
max_matches: 10
```

### HC Core Java Implementation
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-262-public path:core/industries-healthcare-impl content:<keyword>"
max_matches: 10
```

### HC PTC Layer (Insurance-shared)
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_ins/AssessmentResponsesPtc.apex"
```

### OmniStudio Package (INS namespace — HC uses this)
```
Tool: mcp__mcp-adaptor__search
query: "repo:github.com/sf-industries/via_platform content:<keyword> path:package_ins.xml"
```

---

## Symptom-Driven Fast Path

| Symptom | First Check |
|---|---|
| Assessment not saving | `AssessmentResponsesPtc.apex`, FLS on assessment objects, `EnforceDMFLSAndDataEncryption` OIC flag |
| Care plan IP failing | `InsuranceClaimServicePtc.apex`, OIC flags, IP step order (DML before callout?) |
| Enrollment errors | `InsEnrollmentServicePtc.apex`, `InsCensusServicePtc.apex`, data format |
| Benefit verification failing | `InsuranceRatingPtc.apex`, Named Credential config, callout timeout |
| HC DataRaptor missing fields | `EnforceDMFLSAndDataEncryption` = TRUE → check FLS on all DR fields |
| FHIR integration timeout/401 | Named Credential status, endpoint URL, FHIR R4 schema mismatch |
| Provider search empty | Criteria-Based Search permission set, search config, indexed fields |
| IAM setup failing | Run Health Cloud Troubleshooter (Setup), verify required records |
| UM authorization stuck | State machine configuration (`VlocityStateModel__c`), valid transitions |
| HC permission error | HC PSL provisioned? Permission sets assigned? |

---

## Escalation

- Slack: `#tmp-help-health-cloud` (C020DANSS94) — HC tech questions to engineering
- Community: `#industries-healthcare` (C01KUUZ3QCV) — providers + payers
- Signature: `#support-industry-fsc-hc` (C01L982KF7A)
- General: `#support-swarm-industries` (C02BEHKLWES)
- GUS product tag: `Health Cloud` or `Industries Interaction platform`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [Care Plans | Assessments | UM/Auth | FHIR | Provider Network | IAM | Enrollment | Other]
Payer or Provider context?:
Issue Description:
Reproduced in Demo org?:
Troubleshooting steps taken?:
HC Troubleshooter run?:
OmniScript/IP involved?:
```

---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `known-patterns.md` — 12 known issue patterns with triage tree and resolution steps
