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

## Escalation

- Slack: `#support-rev-dev-amer` (Revenue/CPQ), `#support-omnistudio-collaboration`
- GUS product tags: `Industries Interaction platform`, `Communications Cloud`


---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `dc-cache-patterns.md` — DC Regenerate Cache APIs, batch jobs (48 cases + 34 GUS)
- `order-management-patterns.md` — Order decomposition, XOM codebase (665 cases)
- `performance-patterns.md` — Performance, timeouts, cache, SOQL limits (238 cases analyzed)
- `promotion-discount-patterns.md` — CPQ promotions, discounts, cart integration, ESM
