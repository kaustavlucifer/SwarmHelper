# Life Sciences Cloud Debugger

**Trigger:** Life Sciences Cloud, clinical trials, drug programs, MedTech, patient services, consent management, referral management (LS context), REMS, companion diagnostics.

---

## Product Areas

| Area | Scope |
|---|---|
| Patient Services | Patient enrollment, consent, benefit verification |
| Clinical | Trial management, site selection, patient recruitment |
| Commercial | HCP engagement, samples, formulary |
| MedTech | Device tracking, remote monitoring (shared with HC) |

---

## Repository Architecture

### GitHub Repos

| Repository | Path | Content |
|---|---|---|
| `lifesciences` | `sf-industries-ls/lifesciences` | Life Sciences main repo (note: separate org) |
| `HealthCloudCCDAandEMRDatakit` | `sf-industries/HealthCloudCCDAandEMRDatakit` | CCDA/EMR data kit (shared with HC) |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-lifesciences-impl/                   ← Life Sciences Java (trial mgmt, research study)
  core/ui-industries-common-components/omnistudio/     ← Shared components
```

### Engineering Playbooks

| Resource | URL |
|---|---|
| Life Sciences | https://confluence.internal.salesforce.com/spaces/IN/pages/827098968/Life+Sciences |
| Healthcare and Life Sciences (CCEIND) | https://confluence.internal.salesforce.com/spaces/CCEIND/pages/601662751/Healthcare+and+Life+Sciences |
| Epic Narratives (Quip) | https://salesforce.quip.com/GEpaAnwMbhiH |

> **Note:** No dedicated engineering support playbook found for Life Sciences. This is a documentation gap — consider researching `#support-swarm-industries` Slack for tribal knowledge.

---

## Key Objects

| Object | Description |
|---|---|
| `ResearchStudy` | Clinical trial definitions |
| `ResearchStudySubject` | Trial participants |
| `ResearchSite` | Trial site locations |
| `MedicationDispense` | Drug dispensing records |
| `MedicationRequest` | Prescription / drug requests |
| `IndividualApplication` | Patient enrollment applications |
| `CareProgram` | Drug/care program definitions |
| `CareProgramEnrollee` | Program participants |
| `CareProgramProduct` | Products in program |
| `ContactPointConsent` | Consent management |
| `PersonAccount` | HCP and patient records |
| `Visit` | Provider visit records |
| `VisitedPlace` | Visit location tracking |
| `Briefing` | Account briefings (Agentforce) |

---

## Common Issues

| Symptom | Check |
|---|---|
| Patient enrollment IP failing | PTC layer, callout configuration, OIC flags |
| Consent management errors | `ContactPointConsent` permissions, data sharing model |
| Clinical trial data model | `ResearchStudy` configuration, site relationships |
| Drug program setup | `CareProgram` configuration, eligibility rules, product mapping |
| HCP engagement tracking | Activity capture config, compliance rules, Visit object |
| Sample management errors | Inventory objects, allocation rules, lot tracking |
| Briefing generation failing | Metadata cache configuration, object schema |
| Provider visit scheduling | Visit object permissions, scheduling configuration |
| Audio briefing not playing | Mobile app configuration, object schema setup |
| Presentation content assignment | Content definition hierarchy, assignment rules |

---

## Sample SOQL Queries

### Research studies
```soql
SELECT Id, Name, Status, Phase, StartDate, EndDate,
       (SELECT Id, Name, Status FROM ResearchStudySubjects)
FROM ResearchStudy
WHERE Status = 'Active'
ORDER BY StartDate DESC LIMIT 10
```

### Care program enrollees
```soql
SELECT Id, Name, Status, CareProgram.Name, EnrollmentDate,
       Account.Name
FROM CareProgramEnrollee
WHERE CareProgram.Id = '<PROGRAM_ID>'
ORDER BY EnrollmentDate DESC LIMIT 20
```

### Consent records
```soql
SELECT Id, Name, PrivacyConsentStatus, EffectiveFrom, EffectiveTo,
       PartyId, DataUsePurpose.Name
FROM ContactPointConsent
WHERE PartyId = '<CONTACT_ID>'
ORDER BY EffectiveFrom DESC
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex exceptions |
| `ipipr` | Integration Procedures (enrollment flows, consent flows) |
| `ipdar` | DataRaptors |
| `gslog` | Platform Java (Life Sciences core implementation) |

---

## Code Investigation Paths

### Life Sciences Core
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-262-public path:core/industries-lifesciences-impl content:<keyword>"
max_matches: 10
```

### Life Sciences Managed Package
```
Tool: mcp__plugin_git-emu_vmcp-git-emu__search_repositories
query: "lifesciences"
```

---

## Debugging Approach

1. **Identify the feature area** — Clinical, Patient Services, Commercial, MedTech
2. **Check OmniStudio components** — LS heavily uses IPs for enrollment and consent flows
3. **Verify permissions** — LS has specific permission sets per feature
4. **Check FHIR/HL7** — Clinical data often involves FHIR integration (shared with HC)
5. **Review compliance** — HCP engagement has regulatory constraints
6. **Check Agentforce** — Newer LS features use Agentforce (briefings, metadata cache)

---

## Escalation

- GUS product tag: `Life Sciences Cloud`
- Slack: `#support-swarm-industries`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [Clinical/Trials | Patient Services | HCP Engagement | Samples | Consent | Drug Programs | Other]
Issue Description:
Reproduced in Demo org?:
Troubleshooting steps taken?:
OmniScript/IP involved?:
```
