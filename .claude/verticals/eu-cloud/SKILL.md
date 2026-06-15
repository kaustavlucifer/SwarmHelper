---
name: eu-cloud
description: Energy & Utilities Cloud troubleshooting — CAM, multisite orders, DC API cache, billing & usage, E&U Business App. Routed by /swarm-helper.
---

# Energy & Utilities Cloud Debugger

**Trigger:** Energy & Utilities Cloud, E&U, CAM (Customer Acquisition & Management), multisite orders, VEEDigitalGetBasket, DC API cache, package upgrade errors, billing & usage components, E&U Business App performance.

---

## Repository Architecture

### GitHub Repos

| Repository | Path | Content |
|---|---|---|
| `via_energy` | `sf-industries/via_energy` | Energy managed package |
| `vpl_energy` | `sf-industries/vpl_energy` | Energy VPL package |
| `vex_express_energy` | `sf-industries/vex_express_energy` | Energy Express package |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public:
  core/industries-energy-utilities-udd/                 ← E&U Cloud UDD + configuration
  core/ui-industries-energy-utilities-components/       ← E&U UI (agent console, billing, self-serve)
  core/ui-industries-energy-utilities-sales-components/ ← E&U sales components
  core/industries-sustainability/                       ← Sustainability (energy → carbon footprint)
```

---


### PTC Layer

```
core/industries-interaction-ptc/apex/vlocity_cmt/
  CPQServicePtc.apex              ← E&U pricing/cart
  CalculationServicePtc.apex      ← Calculation matrices
  AvailabilityServicePtc.apex     ← Service qualification
  EUCInstrumentationPtc.apex      ← E&U-specific instrumentation
```

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| E&U Functional Architecture | https://confluence.internal.salesforce.com/spaces/IN/pages/938345591/Energy+Utilities+Functional+Architecture |
| Energy and Utilities Cloud | https://confluence.internal.salesforce.com/spaces/IN/pages/630891843/Energy+and+Utilities+Cloud |
| EUC Overview (Onboarding) | https://confluence.internal.salesforce.com/spaces/IN/pages/1240629424/EUC+Overview |
| E&U Tech Talk | https://confluence.internal.salesforce.com/spaces/IN/pages/1226185780/Energy+Utilities+Tech+Talk |

---

## Product Areas

| Area | Scope |
|---|---|
| CAM | Customer acquisition, quote/order management, multisite |
| DC APIs | Digital Commerce APIs, Regenerate Cache, Batch Jobs |
| Billing & Usage | Billing components, usage tracking, metering |
| EPC | Product catalog (shared with Comms Cloud) |

---

## Common Issues

| Symptom | Check |
|---|---|
| CAM performance degradation | SOQL limits in VEEDigitalGetBasket, cache config |
| DC API cache job failures | Batch job status, regenerate API config |
| Too many SOQL queries in basket | N+1 patterns in custom IP, product hierarchy depth |
| Integration Procedure timeouts | HTTP callout config, external system response time |
| Package upgrade record locks | Concurrent upgrade jobs, lock contention |
| Multisite order performance | Order line count, decomposition complexity |
| CPU time limit exceeded | Trigger stacking, complex pricing calculations |
| Billing component loading | LWC access, data visibility rules |
| Cache regeneration failing | Cache size limits, data volume |

---

## Debugging Approach

1. **Check DC API cache status** — Batch Run records for cache jobs
2. **Basket performance** — Debug log for SOQL count in VEEDigitalGetBasket
3. **Multisite scaling** — Order line count × sites = total processing
4. **Package upgrade** — Check for concurrent jobs, record lock patterns

---

## Key Objects

| Object | Description |
|---|---|
| `Product2` / `vlocity_cmt__Product2__c` | Product catalog (shared with Comms) |
| `Order` / `OrderItem` | Orders and line items |
| `vlocity_cmt__PriceList__c` | Price lists |
| `vlocity_cmt__CalculationMatrix__c` | Calculation matrices (pricing, eligibility) |
| `vlocity_cmt__CalculationProcedure__c` | Calculation procedures |
| `vlocity_cmt__OrchestrationPlan__c` | Order orchestration |
| `ServicePoint__c` | Energy service/delivery point |
| `Premise__c` | Physical location (address) |
| `EnergyUsage__c` | Usage/consumption records |
| `vlocity_cmt__ContextDimension__c` | Context-based pricing |

---

## Sample SOQL Queries

### Service points for an account
```soql
SELECT Id, Name, ServicePointType__c, Status__c, Premise__r.Name,
       Account.Name
FROM ServicePoint__c
WHERE Account.Id = '<ACCOUNT_ID>'
ORDER BY Name
```

### Multisite orders
```soql
SELECT Id, OrderNumber, Status, vlocity_cmt__AccountId__c,
       (SELECT Id, vlocity_cmt__Product2Id__r.Name, vlocity_cmt__ServicePointId__c
        FROM OrderItems)
FROM Order
WHERE vlocity_cmt__AccountId__c = '<ACCOUNT_ID>'
  AND RecordType.DeveloperName LIKE '%Multisite%'
ORDER BY CreatedDate DESC LIMIT 10
```

### Cache batch job status
```soql
SELECT Id, Name, vlocity_cmt__Status__c, vlocity_cmt__StartTime__c,
       vlocity_cmt__EndTime__c, vlocity_cmt__ErrorMessage__c
FROM vlocity_cmt__BatchRun__c
WHERE vlocity_cmt__Type__c = 'CacheRefresh'
ORDER BY CreatedDate DESC LIMIT 5
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex uncaught exceptions (basket, pricing) |
| `ipipr` | Integration Procedures (VEEDigitalGetBasket, multisite flows) |
| `ipdar` | DataRaptors (product/pricing extraction) |
| `axlim` | Governor limits (multisite scaling, basket SOQL count) |
| `gslog` | Platform Java exceptions |

---

## Code Investigation Paths

### Energy Managed Package
```
Tool: mcp__plugin_git-emu_vmcp-git-emu__get_file_contents
owner: "sf-industries"
repo: "via_energy"
path: "classes/<ClassName>.cls"
```

### E&U Core Components
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/industries-energy-utilities-udd/"
```

### E&U UI Components
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/ui-industries-energy-utilities-components/"
```

---

## Escalation

- GUS product tag: `Energy & Utilities Cloud`
- Slack: `#energy_cloud-cx-support` (C070CSJGH0Q) — E&U CX requests/blockers
- Community: `#industries-energy-oil-and-gas` (C01K79X82ER)
- General: `#support-swarm-industries` (C02BEHKLWES)

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [CAM/Basket | DC APIs/Cache | Multisite | Billing/Usage | EPC/Pricing | Other]
Issue Description:
Order line count (if multisite):
Cache recently regenerated?:
Reproduced in Demo org?:
Troubleshooting steps taken?:
```

---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `known-patterns.md` — Known issue patterns with resolution steps
