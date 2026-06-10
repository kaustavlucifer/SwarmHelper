# Nonprofit Cloud Debugger

**Trigger:** Nonprofit Cloud, NPSP (Nonprofit Success Pack), fundraising, donations, grants, gift entry, donor management, `npsp__`, `npe01__`, `npo02__`, program management, volunteer management, outcomes.

---

## Product Areas

| Area | Scope |
|---|---|
| Fundraising & Donations | Gift entry, recurring donations, batch processing, pledges |
| NPSP Core | Household management, affiliations, relationships, address management |
| Grants Management | Grant applications, disbursements, milestones, reporting |
| Program Management | Programs, cohorts, participants, outcomes tracking |
| Volunteer Management | Volunteer assignments, hours tracking, scheduling |
| Advancement | Major gifts, planned giving, prospect management |
| Accounting | Fund accounting, allocations, GAU (General Accounting Units) |
| Engagement | Constituent engagement, event management, campaigns |

---

## Repository Architecture

### Managed Packages

| Package | Namespace | Description |
|---|---|---|
| NPSP | `npsp__` | Nonprofit Success Pack (core) |
| Affiliations | `npe5__` | Organization affiliations |
| Relationships | `npe4__` | Contact relationships |
| Recurring Donations | `npe03__` | Recurring donation management |
| Households | `npo02__` | Household management |
| Contacts & Organizations | `npe01__` | Contact extensions |
| Gift Entry | `sfdo__` | Gift Entry Manager (GEM) |
| Nonprofit Cloud | `sfdo_np__` | Newer Nonprofit Cloud objects |
| Outcomes | `sfdo__` | Program outcomes |

### Core Monorepo Paths (CONFIRMED via CodeSearch)

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-nonprofit-udd/                     ← Nonprofit UDD
  core/industries-nonprofit-api/                     ← Nonprofit API
  core/industries-nonprofit-impl/                    ← Nonprofit implementation
  core/industries-fundraising/                       ← Fundraising feature (related)
```

> **Note:** Nonprofit Cloud (core) is separate from traditional NPSP managed package. Core Nonprofit is integrated into the Industries platform.

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| Nonprofit / NGO (CSG) | https://confluence.internal.salesforce.com/spaces/CSGPAK/pages/206541174/Nonprofit+NGO+salesforce.org |
| Nonprofit Cloud CSG Learning (Quip) | https://salesforce.quip.com/PXWiAwsgKljl |
| 256 Design Documents | https://confluence.internal.salesforce.com/spaces/IN/pages/994153578/256+Design+Documents+Nonprofit+Cloud |

---

## Key Objects

| Object | Description |
|---|---|
| `Opportunity` | Donations (NPSP uses Opportunity for gifts) |
| `npe03__Recurring_Donation__c` | Recurring donation schedules |
| `npsp__Allocation__c` | Fund allocations (GAU splits) |
| `npsp__General_Accounting_Unit__c` | Fund/GAU definitions |
| `npsp__Batch__c` | Gift entry batches |
| `npsp__DataImport__c` | NPSP data import records |
| `Account` (Household/Organization) | Household accounts, organization accounts |
| `npe5__Affiliation__c` | Organization affiliations |
| `npe4__Relationship__c` | Contact-to-contact relationships |
| `npsp__Address__c` | NPSP address management |
| `Program__c` | Program definitions |
| `ProgramCohort__c` | Program cohorts |
| `Participant__c` | Program participants |
| `Outcome__c` | Outcomes tracking |

---

## Common Issues

| Symptom | Check |
|---|---|
| Recurring donation installment not creating | RD schedule configuration, `npsp__RecurringDonations2__c` settings |
| Gift entry batch errors | Batch template configuration, field mapping, permissions |
| Household naming not updating | NPSP Household Naming settings, trigger handler config |
| Donation rollup wrong | NPSP Rollup settings, hard credit vs soft credit rules |
| Affiliation not auto-creating | Affiliation mapping configuration, trigger enabled |
| NPSP trigger conflicts | Custom trigger vs NPSP trigger handler framework |
| Data import failing | DataImport field mapping, required field validation |
| Address management conflicts | NPSP address model vs standard address fields |
| Allocation total exceeds 100% | GAU allocation rules, default allocation |
| Relationship reciprocal not creating | Relationship settings, reciprocal method |
| Engagement plan tasks not generating | Engagement plan template, dependency rules |
| Soft credit not rolling up | Contact Roles configuration, soft credit rules |

---

## Sample SOQL Queries

### Donations for a household
```soql
SELECT Id, Name, Amount, StageName, CloseDate, 
       npsp__Primary_Contact__r.Name, RecordType.Name
FROM Opportunity
WHERE AccountId = '<HOUSEHOLD_ACCOUNT_ID>'
  AND StageName = 'Closed Won'
ORDER BY CloseDate DESC LIMIT 20
```

### Recurring donations
```soql
SELECT Id, Name, npe03__Amount__c, npsp__Status__c,
       npsp__InstallmentFrequency__c, npsp__Day_of_Month__c,
       npe03__Contact__r.Name, npsp__Next_Payment_Date__c
FROM npe03__Recurring_Donation__c
WHERE npe03__Contact__c = '<CONTACT_ID>'
ORDER BY CreatedDate DESC
```

### Fund allocations for a donation
```soql
SELECT Id, npsp__General_Accounting_Unit__r.Name,
       npsp__Amount__c, npsp__Percent__c
FROM npsp__Allocation__c
WHERE npsp__Opportunity__c = '<OPPORTUNITY_ID>'
```

---

## Code Investigation Paths

### Core Implementation
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-262-public content:Nonprofit lang:java"
max_matches: 15
```

### NPSP Source (if accessible)
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "content:npsp__ content:RecurringDonation"
max_matches: 10
```

---

## Debugging Approach

1. **Identify the package** — NPSP, Gift Entry, Nonprofit Cloud core, or custom
2. **Check NPSP settings** — Custom Settings and Custom Metadata control trigger behavior
3. **Verify trigger handlers** — NPSP uses a trigger handler framework (`npsp__Trigger_Handler__c`)
4. **Check record types** — NPSP uses Account record types (Household, Organization)
5. **Review rollup settings** — NPSP rollups have complex configuration
6. **Check batch processing** — Gift entry uses batch processing with specific error handling
7. **Verify namespace conflicts** — Multiple packages (npsp, npe01, npo02, npe03, npe4, npe5)

---

## Escalation

- GUS product tag: `Nonprofit Cloud`
- Slack: `#ask-sfdo-tech-expert`, `#sfdo-architects`
- Related: `#support-swarm-industries`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Package: [NPSP | Gift Entry | Nonprofit Cloud Core | Recurring Donations | Other]
NPSP Version:
Issue Description:
Reproduced in Demo org?:
Troubleshooting steps taken?:
Custom triggers present?:
```
