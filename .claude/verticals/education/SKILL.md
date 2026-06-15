---
name: education
description: Education Cloud / EDA troubleshooting — student success, admissions, academic terms, courses, hed__/sfedo__. Routed by /swarm-helper.
---

# Education Cloud Debugger

**Trigger:** Education Cloud, EDA (Education Data Architecture), student success, admissions, enrollment (education context), academic terms, courses, program plans, student records, `hed__`, `sfedo__`.

---

## Product Areas

| Area | Scope |
|---|---|
| Education Data Architecture (EDA) | Core data model — accounts, contacts, affiliations, relationships |
| Admissions | Application management, recruitment, enrollment decisions |
| Student Success | Alerts, advising, appointment scheduling, success plans |
| Academic Operations | Courses, terms, program plans, course connections |
| Advancement | Fundraising, gift management, donor engagement |
| K-12 | Grade management, attendance, guardian relationships |
| Student Experience | Student portals, self-service, degree audit |
| Recruiting | Prospect management, recruitment campaigns, events |

---

## Repository Architecture

### GitHub / git.soma Repos

> **Note:** Education Cloud has no via_* managed package repo. It uses independent managed packages (EDA, Admissions Connect, SSH, GEM) hosted separately from the sf-industries org. Primary development is in core monorepo.

### Managed Packages

| Package | Description |
|---|---|
| EDA (Education Data Architecture) | Core data model and automation |
| Admissions Connect (AC) | Admissions workflow package |
| Student Success Hub (SSH) | Student success and advising |
| Gift Entry Manager (GEM) | Advancement/fundraising |
| Accounting Subledger | Financial management |
| Data Mover | Data migration utilities |

### Core Monorepo Paths (CONFIRMED via CodeSearch — 9 modules)

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-education/                         ← Education Cloud base
  core/industries-education-api/                     ← Public interfaces
  core/industries-education-impl/                    ← Service implementations
  core/industries-education-udd/                     ← UDD entities (54 enums)
  core/industries-education-connect-api/             ← Connect API
  core/industries-education-connect-impl/            ← Connect implementation
  core/ui-industries-education-api/                  ← UI API layer
  core/ui-industries-education-impl/                 ← UI implementation
  core/ui-industries-education-components/           ← LWC components
```

> **Note:** Education Cloud is core-only (no via_* managed package). Java root package: `industries.education.*`
> Cross-dependencies: public-sector (care plans), actionplan, program-mgmt, bre-guardrail, rules-engine, pricing

### Ownership Teams
- `EDU-Rad-ish`, `EDU-Turmerific`, `EDU-Sweet Potatoes`, `EDU-Turnip the Beets`, `EDU-Garlic Genies`, `EDU-Gingersnaps`, `NPC Cartwheel`

---


### PTC Layer

Education Cloud is core-only (no PTC layer). No managed package Apex — Java-based implementation across 9 modules.

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| Education Cloud (IN space) | https://confluence.internal.salesforce.com/spaces/IN/pages/449594336/Education+Cloud |
| EDU Product Overviews | https://confluence.internal.salesforce.com/spaces/IN/pages/449594342/Education+Cloud+Product+Overviews |
| EDU Core Architecture | https://confluence.internal.salesforce.com/spaces/IN/pages/449594410/EDU+Core+Architecture |
| EDU Engineering Onboarding | https://confluence.internal.salesforce.com/spaces/IN/pages/449594347/EDU+Engineering+Onboarding |
| Education Brown Bag Sessions | https://confluence.internal.salesforce.com/spaces/IN/pages/941728111/Education+Brown+Bag+Sessions |
| GUS Group | https://gus.lightning.force.com/lightning/r/CollaborationGroup/0F9B0000000c83vKAA/view |

---

## Key Objects

| Object | Description |
|---|---|
| `Account` (Academic) | Educational institutions, departments |
| `Contact` | Students, faculty, staff, prospects |
| `hed__Affiliation__c` | Person-to-organization relationships |
| `hed__Course__c` | Course definitions |
| `hed__Course_Enrollment__c` | Student course registrations |
| `hed__Course_Offering__c` | Scheduled course sections |
| `hed__Term__c` | Academic terms/semesters |
| `hed__Program_Plan__c` | Degree/program requirements |
| `hed__Plan_Requirement__c` | Individual program requirements |
| `hed__Application__c` | Admissions applications |
| `hed__Relationship__c` | Contact-to-contact relationships |
| `hed__Address__c` | Address management |
| `Success_Plan__c` | Student success plans |
| `Alert__c` | Student alerts (advising) |

---

## Common Issues

| Symptom | Check |
|---|---|
| EDA trigger errors after upgrade | EDA trigger handler configuration, custom trigger conflicts |
| Affiliation not auto-creating | EDA Affiliation Mappings configuration, trigger enabled |
| Course enrollment permissions | EDA permission sets, record type access |
| Admissions application flow failing | Application OmniScript, FlexCard configuration |
| Student Success alert not firing | Alert criteria, evaluation frequency |
| Program plan requirements not mapping | Plan requirement hierarchy, course connection |
| Duplicate contacts on import | EDA duplicate rules, matching rules configuration |
| Address management conflicts | EDA address model (hed__Address__c vs standard) |
| Relationship creation failing | Relationship type configuration, reciprocal relationships |
| Term-based automation not triggering | Term status (Current/Active), effective dates |

---

## Sample SOQL Queries

### Student enrollments with courses
```soql
SELECT Id, hed__Contact__r.Name, hed__Course_Offering__r.Name,
       hed__Status__c, hed__Grade__c, hed__Credits_Attempted__c
FROM hed__Course_Enrollment__c
WHERE hed__Contact__c = '<CONTACT_ID>'
ORDER BY hed__Course_Offering__r.hed__Term__r.hed__Start_Date__c DESC
LIMIT 20
```

### Affiliations for a contact
```soql
SELECT Id, hed__Account__r.Name, hed__Role__c, hed__Status__c,
       hed__StartDate__c, hed__EndDate__c, hed__Primary__c
FROM hed__Affiliation__c
WHERE hed__Contact__c = '<CONTACT_ID>'
ORDER BY hed__StartDate__c DESC
```

### Program plan requirements
```soql
SELECT Id, Name, hed__Category__c, hed__Credits__c,
       (SELECT Id, Name, hed__Course__r.Name FROM hed__Plan_Requirements__r)
FROM hed__Program_Plan__c
WHERE Id = '<PROGRAM_PLAN_ID>'
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex uncaught exceptions |
| `axlim` | Governor limit consumption |
| `ipipr` | Integration Procedures (if OmniStudio components used) |
| `ipdar` | DataRaptors (if OmniStudio components used) |
| `gslog` | Platform Java exceptions (core implementation) |

## Code Investigation Paths

### Core Implementation
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-262-public content:Education lang:java"
max_matches: 15
```

### EDA Package (if accessible)
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "content:hed__ content:Affiliation"
max_matches: 10
```

---

## Debugging Approach

1. **Identify the package** — EDA core, Admissions Connect, Student Success Hub, or custom
2. **Check EDA version** — Trigger behavior varies significantly between EDA versions
3. **Verify EDA settings** — Custom Settings control trigger behavior (`hed__Trigger_Handler__c`)
4. **Check record types** — EDA uses Account record types (Academic, Business, etc.)
5. **Review automations** — EDA has its own trigger framework; conflicts with custom triggers are common
6. **Check namespace** — `hed__` for EDA, `sfedo__` for newer Education Cloud objects

---

## Escalation

- GUS product tag: `Education Cloud`
- Slack: `#sfdo-architects` (C01GXEZABDF, validated)
- ⚠️ `#ask-sfdo-tech-expert` could not be confirmed via channel search (2026-06-15) — verify before relying on it
- Related: `#support-swarm-industries`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Package: [EDA | Admissions Connect | Student Success Hub | Education Cloud Core | Other]
EDA Version:
Issue Description:
Reproduced in Demo org?:
Troubleshooting steps taken?:
Custom triggers present?:
```
