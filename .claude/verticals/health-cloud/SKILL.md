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

## Escalation

- Slack: `#support-health-cloud` or `#support-omnistudio-collaboration`
- GUS product tag: `Health Cloud` or `Industries Interaction platform`


---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `known-patterns.md` — Known issue patterns with resolution steps
