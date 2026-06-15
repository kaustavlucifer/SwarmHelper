---
name: public-sector
description: Public Sector Solutions troubleshooting — permits, licensing, inspections, complaints, constituent services, vlocity_ps. Routed by /swarm-helper.
---

# Public Sector Solutions Debugger

**Trigger:** Public Sector, PSS, government, permits, licensing, inspections, complaints, case management (government context), `vlocity_ps` namespace, constituent services, compliance management.

---

## Product Areas

| Area | Scope |
|---|---|
| License, Permit & Inspection (LPI) | License applications, permit management, inspections, renewals |
| Complaint Management | Intake, investigation, resolution, participant notifications |
| Compliance Management | Regulatory compliance, visit summaries, enforcement |
| Case Management (Gov) | Government case workflows, benefits management, recertification |
| Recruitment & HR | Candidate sourcing, job applications, job posting search |
| Provider Management | Provider search, caseworker-provider collaboration |
| Constituent Services | Citizen portals, Agentforce for constituents |
| Benefits Management | Benefits eligibility, recertification events |

---

## Repository Architecture

### GitHub / git.soma Repos (CONFIRMED via CodeSearch)

| Repository | Path | Content |
|---|---|---|
| `via_ps` | `sf-industries/via_ps` | **Public Sector managed package** (primary Apex: PSUtils, PSPymnt, BuildingInfo, ApplicationAssets, IncidentMapController) |
| `via_ps_test` | `sf-industries/via_ps_test` | PS test package |
| `via_platform` | `sf-industries/via_platform` | OmniStudio Apex (vlocity_ps namespace, `package_ps.xml`) |
| `via_core` | `sf-industries/via_core` | Platform foundation (shared) |

> **Note:** Public Sector is primarily an Apex managed package (not Java-based like most other Industry Clouds). Uses `vlocity_ps__` namespace.

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public:
  core/ui-industries-public-sector-components/omnistudio/  ← PSS OmniStudio components
  core/industries-interaction-ptc/apex/vlocity_ps/         ← PTC layer (PS namespace)
```

### Key Package Classes (from via_ps)
| Class | Domain |
|---|---|
| `PSUtils` | General PS utilities |
| `PSPymnt` | Payment processing |
| `BuildingInfo`, `TenantDetailInfo` | Housing/facility management |
| `ApplicationAssets`, `ApplicationIncome`, `ApplicationExpenses` | Benefit application processing |
| `IncidentMapController`, `MapGeoServices` | Geolocation/incident mapping |

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| PSS Product Overview | https://confluence.internal.salesforce.com/spaces/IN/pages/290299084/PSS+Product+Overview |
| PSS Onboarding | https://confluence.internal.salesforce.com/spaces/IN/pages/290299082/PSS+Onboarding |
| LPI & ERM | https://confluence.internal.salesforce.com/spaces/IN/pages/809929338/LPI_ERM |

---

## Key Objects

| Object | Description |
|---|---|
| `RegulatoryAuthorizationType` | License/permit type definitions |
| `BusinessLicenseApplication` | License applications |
| `BusinessLicense` | Issued licenses/permits |
| `InspectionType` / `Inspection` | Inspection definitions and records |
| `Complaint` | Complaint records |
| `ActionPlan` / `ActionPlanItem` | Government action plans |
| `CarePlan` | Benefits management care plans |
| `Referral` | Service referrals |
| `DocumentChecklistItem` | Document requirements |
| `Case` | Government cases (benefits, complaints) |
| `BMRecertEvent__e` | Benefits management recertification platform event |

---

## Common Issues

| Symptom | Check |
|---|---|
| License application flow failing | OmniScript configuration, vlocity_ps PTC layer |
| Inspection not creating from license | Inspection type configuration, trigger setup |
| Complaint intake agent errors | Agentforce template config, prompt template |
| Provider search not working | Criteria-Based Search permission set, search config |
| Benefits recertification not triggering | Platform event subscription, `BMRecertEvent__e` |
| Document checklist not showing | DocumentChecklistItem permissions, template assignment |
| Compliance visit summary not sending | Email template, send action configuration |
| Action plan tasks not progressing | Task dependencies, action plan template config |
| Constituent portal access denied | Community user permissions, sharing rules |
| Candidate sourcing agent scope | Agent is FAQ-only, not designed for full workflow |
| LPI Assistance agent data privacy | Policy manual uploads, data exposure boundaries |

---

## Sample SOQL Queries

### License applications by account
```soql
SELECT Id, Name, Status, ApplicationType, Account.Name,
       CreatedDate, LastModifiedDate
FROM BusinessLicenseApplication
WHERE Account.Id = '<ACCOUNT_ID>'
ORDER BY CreatedDate DESC LIMIT 20
```

### Inspections for a license
```soql
SELECT Id, Name, Status, InspectionType.Name, AssessmentIndication,
       ActualStartDateTime, ActualEndDateTime
FROM Inspection
WHERE RegulatoryAuthorizationId = '<LICENSE_ID>'
ORDER BY ActualStartDateTime DESC LIMIT 10
```

### Complaints with action plans
```soql
SELECT Id, Subject, Status, Priority, CreatedDate,
       (SELECT Id, Name, Status FROM ActionPlans)
FROM Case
WHERE RecordType.DeveloperName LIKE '%Complaint%'
  AND Account.Id = '<ACCOUNT_ID>'
ORDER BY CreatedDate DESC LIMIT 10
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries instrumentation (filter by `instKey`) — applies to the `vlocity_ps` managed-package / OmniStudio layer (LPI, assessments, benefits use OmniScript heavily) |
| `gslog` | Platform Java exceptions/gacks — core LPI/Benefits implementation |
| `axerr` | Apex uncaught exceptions |
| `axlim` | Governor limit consumption |
| `ipipr` | Integration Procedures (OmniStudio — assessments, application flows) |
| `ipdar` | DataRaptors (OmniStudio data ops) |

## Code Investigation Paths

### PTC Layer (PS namespace)
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_ps/"
```

### Managed Package (PS classes)
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:github.com/sf-industries/via_ps content:PublicSector"
```

### Core Implementation
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public content:PublicSector lang:java"
max_matches: 15
```

---

## Debugging Approach

1. **Identify the feature area** — LPI, Complaint, Compliance, Benefits, Recruitment
2. **Check OmniStudio components** — PSS heavily uses OmniScripts and Integration Procedures
3. **Verify permissions** — PSS has specific permission sets per feature area
4. **Check Agentforce config** — Many PSS features now have Agentforce templates
5. **Review platform events** — Benefits recertification uses `BMRecertEvent__e`
6. **Check Criteria-Based Search** — Provider and job search depend on this framework

---

## Escalation

- GUS product tag: `Vlocity Public Sector` (verified 2026-06-15 — the string `Public Sector Solutions` returns 0 bugs; if a search is empty, widen with `Product_Tag__r.Name LIKE '%Public Sector%'`)
- Slack: `#sfdo-architects` (C01GXEZABDF, validated)
- Related: `#support-swarm-industries`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [LPI | Complaint | Compliance | Benefits | Recruitment | Provider | Portal | Other]
Issue Description:
Reproduced in Demo org?:
Troubleshooting steps taken?:
OmniScript/IP involved?:
```

---

## Detailed Pattern Files

- `known-patterns.md` — Public Sector Solutions diagnostic patterns (symptom → subsystem → confirm → GUS search).
