---
name: fsc
description: Financial Services Cloud troubleshooting — financial accounts, action plans, referrals, rollup summaries, relationship maps, goals/life events. Routed by /swarm-helper.
---

# FSC Debugger

**Trigger:** Financial Services Cloud, FSC, financial accounts, account hierarchy, referrals, relationship maps, action plans, `vlocity_ins_fsc` namespace, rollup summary rules, document checklist items, goals, life events, interaction summaries.

---

## Repository Architecture

### GitHub / git.soma Repos

| Repository | Path | Content |
|---|---|---|
| `via_ins_fsc` | `github.com/sf-industries/via_ins_fsc` | Insurance Industries Extension for FSC |
| `via_platform` | `github.com/sf-industries/via_platform` | OmniStudio Apex (vlocity_ins_fsc namespace) |
| `fsc-next-gen-apps` | `github.com/salesforce-internal/fsc-next-gen-apps` | FSC Next-Gen Apps |

> **FSC core source lives in the core monorepo** (paths below), searchable via `mcp__plugin_deep-research_codesearch__search`. The legacy `git.soma/industries/wealth1`, `industries/core`, `industries/build` repos no longer resolve (validated 2026-06-15) — FSC migrated into core.

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public:
  core/ui-fsc-components/omnistudio/               ← FSC OmniStudio components
  core/ui-fsc-components/java/                     ← FSC Java
  core/ui-fsc-components/modules/                  ← FSC LWC (analytics, KYC, wealth)
  core/ui-fsc-api/                                 ← FSC API layer
  core/industries-interaction-ptc/apex/vlocity_ins_fsc/   ← PTC layer
  core/industries-interaction-ptc/apex/vlocity_fsc_gs0/   ← PTC layer (GS0)
```

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| FSC Main Page | https://confluence.internal.salesforce.com/spaces/IN/pages/187545546/Financial+Services+Cloud+-+FSC |

---

## FSC-Specific Objects

| Object | Description |
|---|---|
| `FinancialAccount` | Core financial account |
| `FinancialAccountRole` | Account roles (owner, beneficiary) |
| `FinancialHolding` | Holdings within accounts |
| `ActionPlan` / `ActionPlanItem` | FSC action plans |
| `Referral` | Referral tracking |
| `AccountContactRelationship` | Relationship maps |
| `RollupSummaryRule__c` | Rollup summary rules |
| `DocumentChecklistItem` | Document checklists |

---

## Common Issues

| Symptom | Check |
|---|---|
| Relationship map not loading | FlexCard datasource, FinancialAccount sharing rules |
| Action plan items not created | ActionPlan triggers, IP logic, field FLS |
| FSC OmniScript errors | `vlocity_ins_fsc` namespace, PTC layer changes |
| Financial account not visible | OWD, sharing rules, person account setup |
| Referral not visible | Referral sharing rules, routing config |
| RSR not calculating | Rollup summary rule config, stale values |
| Document checklist permissions | `DocumentChecklistItem` CRUD access, Site visibility |
| Action plan template access | Licensing (Sales Action Plans vs FSC), permission sets |
| Successor tasks not creating | Task dependency config, action plan item ordering |
| Timeline not showing | Component configuration, record-level access |
| Interaction summary not saving | Related record linking, field permissions |

---

## Data Model Relationships

```
Account (Person Account / Business Account / Household)
├── FinancialAccount (master-detail to Account)
│   ├── FinancialAccountRole (junction: Account ↔ FinancialAccount)
│   ├── FinancialHolding (master-detail)
│   └── FinancialAccountTransaction (lookup)
├── AccountContactRelationship (relationship map)
├── Referral (lookup to Account)
├── ActionPlan (lookup — can also be on Contact, Opportunity)
│   └── ActionPlanItem (master-detail)
│       └── TaskRelay (successor task config)
├── DocumentChecklistItem (lookup)
├── RollupSummaryRule__c (config object — rollup calculations)
└── InteractionSummary (lookup)
    └── InteractionSummaryItem (related records)
```

---

## Sample SOQL Queries

### Financial accounts for a client
```soql
SELECT Id, Name, FinServ__FinancialAccountType__c, FinServ__Balance__c,
       FinServ__Status__c, FinServ__PrimaryOwner__r.Name,
       (SELECT Id, FinServ__Role__c, FinServ__RelatedAccount__r.Name FROM FinServ__FinancialAccountRoles__r)
FROM FinServ__FinancialAccount__c
WHERE FinServ__PrimaryOwner__c = '<ACCOUNT_ID>'
ORDER BY FinServ__Balance__c DESC LIMIT 20
```

### Action plans on a record
```soql
SELECT Id, Name, ActionPlanTemplateVersion.Name, ActionPlanState, StartDate,
       (SELECT Id, Subject, Status, Priority, TaskDefinition.Subject, ActionPlanItemState 
        FROM ActionPlanItems ORDER BY ItemEntityOrder ASC)
FROM ActionPlan
WHERE TargetId = '<RECORD_ID>'
ORDER BY StartDate DESC LIMIT 5
```

### Rollup summary rules
```soql
SELECT Id, FinServ__ParentObject__c, FinServ__ChildObject__c, 
       FinServ__FieldToAggregate__c, FinServ__AggregateOperation__c,
       FinServ__Active__c
FROM FinServ__RollupSummaryRule__c
WHERE FinServ__Active__c = true
ORDER BY FinServ__ParentObject__c
```

### Referrals for an account
```soql
SELECT Id, Name, Status, ReferredRecord.Name, Owner.Name,
       CreatedDate, ClosedDate
FROM Referral
WHERE AccountId = '<ACCOUNT_ID>'
ORDER BY CreatedDate DESC LIMIT 10
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex uncaught exceptions (triggers, action plan logic) |
| `ipipr` | Integration Procedures (FSC OmniScripts) |
| `ipdar` | DataRaptors (FSC data operations) |
| `axlim` | Governor limit consumption (RSR batch, action plan bulk) |
| `gslog` | Platform Java exceptions |

---

## Code Investigation Paths

### FSC Core Source (core monorepo)
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public content:<keyword> file:core/ui-fsc-"
max_matches: 10
```

### FSC Core Components
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/ui-fsc-components/"
```

### FSC PTC Layer
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_ins_fsc/<ClassName>.apex"
```

### FSC Next-Gen Apps
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:github.com/salesforce-internal/fsc-next-gen-apps content:<keyword>"
max_matches: 10
```

---

## Symptom-Driven Fast Path

| Symptom | First Check |
|---|---|
| Relationship map not loading | FlexCard datasource config, FinancialAccount sharing rules |
| Action plan items not created | ActionPlan triggers, IP logic, field FLS, PSL assigned |
| Action plan "Couldn't create successor tasks" | Task dependency ordering, `ItemEntityOrder` field |
| Financial account not visible | OWD (Private), sharing rules, Person Account config |
| RSR not calculating | Rollup summary rule active? Batch job running? Field mapping correct? |
| Document checklist not showing | `DocumentChecklistItem` CRUD, record type assignment |
| Referral routing broken | Referral routing config, queue assignment, sharing rules |
| Timeline empty | Timeline component configuration, record-level access, object support |
| Interaction summary not saving | Related record linking, field permissions, object access |
| FSC OmniScript errors | `vlocity_ins_fsc` namespace, PTC layer, OIC flags |

---

## Escalation

- Slack: `#industries-fsc` (C022BF4TH7B), `#tech-prod-help-financial-services-cloud` (C020H26JCP7), `#fsc-support-cce-eng` (C08MH09LEH3)
- Escalation: `#support-industry-fsc-hc` (C01L982KF7A) — Signature tier
- General: `#support-swarm-industries` (C02BEHKLWES)
- GUS product tag: `Industries Financial Services Cloud`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [Action Plans | Financial Accounts | Referrals | RSR | Relationship Map | Timeline | Other]
Issue Description:
Person Account enabled?:
FSC Permission Sets assigned?:
Reproduced in Demo org?:
Troubleshooting steps taken?:
```

---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `action-plans-patterns.md` — Action plan templates, tasks, dependencies, permissions
- `known-patterns.md` — Known issue patterns with resolution steps
