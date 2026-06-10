# Communications Cloud Debugger

**Trigger:** Communications Cloud, Telco, EPC, CPQ (Industries), order management, product catalog, pricing, order decomposition, orchestration plans, MSM, `vlocity_cmt` namespace in comms context, service qualification.

---

## Repository Architecture

### GitHub Repos

| Repository | Path | Content |
|---|---|---|
| `via_platform` | `sf-industries/via_platform` | Managed package Apex (vlocity_cmt namespace) |
| `via_telco` | `sf-industries/via_telco` | Telco managed package |
| `via_telco_unmanaged` | `sf-industries/via_telco_unmanaged` | Telco unmanaged package |
| `vpl_comms` | `sf-industries/vpl_comms` | Vlocity Platform for Communications |
| `communications_apps_automation` | `sf-industries/communications_apps_automation` | Communications apps automation |

Key managed package classes: `CPQService.cls`, `CalculationService.cls`, `DroService.cls`, `AvailabilityService.cls`

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-communications-msm-impl/omnistudio/  ← MSM IPs + OmniScripts
  core/industries-communications-msm-impl/             ← Billing inquiry, quick quote, agentforce
  core/ui-industries-communications-msm-components/    ← MSM LWC
  core/ui-industries-cpq-components/                   ← CPQ components
  core/ui-industries-serviceprocess-components/omnistudio/
```

### PTC Layer
```
core/industries-interaction-ptc/apex/vlocity_cmt/
  CPQServicePtc.apex
  CalculationServicePtc.apex
  AvailabilityServicePtc.apex
  DroServicePtc.apex
  OrchestrationPlanCompositionServicePtc.apex

core/industries-interaction-ptc/apex/vlocity_ins_fsc/
  OrderDecompService.apex         ← NOTE: in vlocity_ins_fsc, not vlocity_cmt
  OrderMgmtLoggingService.apex
```

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| VLCOM Space (Comms Cloud) | https://confluence.internal.salesforce.com/spaces/VLCOM/ |
| AITB1 - Index of Resources | https://confluence.internal.salesforce.com/spaces/VLCOM/pages/449598012/AITB1+-+Index+of+Resources |

---

## Comms-Specific Objects

| Object | Description |
|---|---|
| `Product2` / `vlocity_cmt__Product2__c` | Product catalog |
| `vlocity_cmt__PriceList__c` | Pricing |
| `Order` / `OrderItem` | Orders and line items |
| `vlocity_cmt__OrchestrationPlan__c` | Order orchestration |
| `vlocity_cmt__ContextDimension__c` | Context-based pricing |
| `vlocity_cmt__CalculationMatrix__c` | Calculation matrices |
| `vlocity_cmt__CalculationProcedure__c` | Calculation procedures |

---

## Common Issues

| Symptom | Check |
|---|---|
| CPQ pricing errors | `CalculationServicePtc.apex`, Calculation Matrix data |
| Order decomposition failing | `OrderDecompService.apex`, `OrchestrationPlanCompositionServicePtc.apex` |
| Service qualification failing | `AvailabilityServicePtc.apex` |
| DR extract missing fields | FLS on `vlocity_cmt__*` fields, `CheckFieldLevelSecurity__c` |
| IP DML before callout | Reorder: HTTP callout BEFORE DataRaptor Post Action |
| Pricing not loading | `DroServicePtc.apex`, Pricebook configuration |
| Cart timeout / slow | Cache configuration, SOQL limits in pricing |
| EPC attribute issues | `vlocity_cmt__Attribute__c` config, EPC Designer |

---

## Service Catalog

All services use the **Vlocity Open Interface** pattern (same as Insurance):
```
InvokeService.invoke(className, methodName, inputJSON, optionsJSON)
```

| Service | Key Methods | Domain |
|---|---|---|
| `CPQService` | `getCartsItems`, `addToCart`, `deleteFromCart`, `postCartsItems` | Cart/quoting |
| `CalculationService` | `calculate`, `getCalculationMatrix` | Pricing |
| `DroService` | `getDro`, `executeDro` | Dynamic Revenue Orchestration |
| `AvailabilityService` | `checkAvailability`, `getServiceQualification` | Service qualification |
| `CatalogService` | `getProducts`, `getProductsWithPricing` | Product catalog |
| `OrderManagementService` | `submitOrder`, `cancelOrder` | Order lifecycle |
| `OrchestrationPlanService` | `executeOrchestrationPlan`, `getOrchestrationPlan` | XOM |
| `AttributeService` | `getAttributes`, `setAttributes` | EPC attributes |

---

## Sample SOQL Queries

### Cart/Quote with line items
```soql
SELECT Id, Name, vlocity_cmt__Status__c, vlocity_cmt__AccountId__c,
       (SELECT Id, vlocity_cmt__Product2Id__r.Name, vlocity_cmt__LineNumber__c,
               vlocity_cmt__OneTimeTotal__c, vlocity_cmt__RecurringTotal__c
        FROM vlocity_cmt__OrderItems__r)
FROM Order
WHERE vlocity_cmt__AccountId__c = '<ACCOUNT_ID>'
  AND vlocity_cmt__Status__c != 'Cancelled'
ORDER BY CreatedDate DESC LIMIT 10
```

### Orchestration plans for an order
```soql
SELECT Id, Name, vlocity_cmt__OrderId__c, vlocity_cmt__State__c,
       (SELECT Id, Name, vlocity_cmt__State__c, vlocity_cmt__TaskType__c,
               vlocity_cmt__OrchestrationItemType__c
        FROM vlocity_cmt__OrchestrationItems__r ORDER BY vlocity_cmt__Sequence__c)
FROM vlocity_cmt__OrchestrationPlan__c
WHERE vlocity_cmt__OrderId__c = '<ORDER_ID>'
ORDER BY CreatedDate DESC LIMIT 5
```

### Products in catalog with pricing
```soql
SELECT Id, Name, ProductCode, IsActive, vlocity_cmt__Type__c,
       vlocity_cmt__SpecificationType__c
FROM Product2
WHERE vlocity_cmt__Type__c IN ('Product', 'Service')
  AND IsActive = true
LIMIT 50
```

### Calculation matrices
```soql
SELECT Id, Name, vlocity_cmt__IsActive__c, vlocity_cmt__InputType__c
FROM vlocity_cmt__CalculationMatrix__c
WHERE vlocity_cmt__IsActive__c = true
ORDER BY Name
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `axerr` | Apex uncaught exceptions (CPQ, decomposition) |
| `ipipr` | Integration Procedures (order submission, cart operations) |
| `ipdar` | DataRaptors (product catalog, attribute extraction) |
| `ipipa` | IP actions/blocks (individual step failures) |
| `axlim` | Governor limit consumption (cart pricing, decomposition) |
| `gslog` | Platform Java exceptions (orchestration engine) |

---

## Code Investigation Paths

### CPQ Service (PTC)
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_cmt/CPQServicePtc.apex"
```

### Order Decomposition (PTC — note: in vlocity_ins_fsc!)
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_ins_fsc/OrderDecompService.apex"
```

### MSM Components
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/industries-communications-msm-impl/omnistudio/"
```

### Managed Package (via_telco)
```
Tool: mcp__plugin_git-emu_vmcp-git-emu__get_file_contents
owner: "sf-industries"
repo: "via_telco"
path: "classes/<ClassName>.cls"
```

---

## Symptom-Driven Fast Path

| Symptom | First Check |
|---|---|
| Cart pricing wrong | `CalculationServicePtc.apex`, Calculation Matrix data, pricing context |
| Order decomposition failing | `OrderDecompService.apex` (in vlocity_ins_fsc!), product hierarchy |
| Orchestration plan stuck | `OrchestrationPlanCompositionServicePtc.apex`, AutoTask config |
| Service qualification error | `AvailabilityServicePtc.apex`, external callout config |
| IP DML before callout | Reorder: HTTP callout BEFORE DataRaptor Post Action |
| Cart timeout / performance | Cache config (see `dc-cache-patterns.md`), SOQL limits |
| EPC attribute issues | `vlocity_cmt__Attribute__c` config, attribute category |
| Product not in catalog | Product active? Price list assignment? Context dimension? |
| Order submission fails | IP step order, validation rules on Order/OrderItem |

---

## Escalation

- Slack: `#support-rev-dev-amer` (Revenue/CPQ), `#support-omnistudio-collaboration`
- GUS product tags: `Industries Interaction platform`, `Communications Cloud`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Component: [CPQ/Cart | Order Decomposition | Orchestration | Pricing | Catalog | MSM | Other]
Namespace: [vlocity_cmt | omnistudio]
Issue Description:
Order/Cart ID (if applicable):
Reproduced in Demo org?:
Troubleshooting steps taken?:
Cache recently regenerated?:
```

---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `dc-cache-patterns.md` — DC Regenerate Cache APIs, batch jobs (48 cases + 34 GUS)
- `order-management-patterns.md` — Order decomposition, XOM codebase (665 cases)
- `performance-patterns.md` — Performance, timeouts, cache, SOQL limits (238 cases analyzed)
- `promotion-discount-patterns.md` — CPQ promotions, discounts, cart integration, ESM
