---
name: revenue-cloud
description: Revenue Cloud (Core) troubleshooting — billing, BRE, configurator, advanced approvals, CLM, DRO, product-to-order, Expression Sets, Decision Tables. Routed by /swarm-helper.
---

# Revenue Cloud (Core) Debugger

**Trigger:** Revenue Cloud (Core), billing & invoicing, BRE (Business Rules Engine), configurator, advanced approvals, CLM (Contract Lifecycle Management), DocGen, DRO (Dynamic Revenue Orchestration), product-to-order, pricing, Expression Sets, Decision Tables, Context Definitions.

---

## Product Areas

| Area | Scope |
|---|---|
| Billing & Invoicing | Billing schedules, invoice batch runs, tax engine (AvaTax/Vertex), payment, credit/debit memos, milestone billing, usage billing, DPE |
| BRE | Decision Tables, Expression Sets, Context Definitions, Pricing Procedures, Lookup Tables, Configuration Rules, CML |
| Configurator | Product Configurator, CML rules, Browse Catalog, TLE/STLE/QLE, bundle config, amendment configurator |
| Advanced Approvals | Approval flows, Smart Approvals, Background Step, ApprovalWorkItem, recall, orchestration flow |
| CLM & DocGen | Contract creation/activation, document templates, merge fields, e-signature, Word connector, clause library |
| DRO | Revenue recognition, revenue schedules, ASC 606/IFRS 15 compliance |
| Product-to-Order | Product catalog, bundles, attributes, pricing config, product rules, quote-to-order |

---

## Repository Architecture

### GitHub Repos

| Repository | Path | Content |
|---|---|---|
| `via_rm` | `sf-industries/via_rm` | Revenue Management package |
| `via_contract` | `sf-industries/via_contract` | Contract management (CLM) |
| `via_docgen` | `sf-industries/via_docgen` | Document generation package |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/billing-impl/                              ← Billing implementation (invoice, tax)
  core/billing-hpp-connect-impl/                  ← Billing HPP Connect
  core/ui-revenue-common-components/              ← Revenue common UI components
  core/ui-industries-cpq-components/              ← CPQ components (shared)
  core/ui-industries-epc-components/              ← EPC components
  core/industries-interaction-ptc/apex/vlocity_cmt/
    CPQServicePtc.apex
    CalculationServicePtc.apex
```

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| Revenue Cloud Runbooks | https://confluence.internal.salesforce.com/spaces/QUOTETOC/pages/1294353243/Revenue+Cloud+Runbooks |

---

## Common Issues — Billing & Invoicing

| Symptom | Check |
|---|---|
| Billing schedule not generating | Billing schedule trigger, order status, billing treatment |
| Invoice batch failing | Batch job logs, DPE errors, record locks |
| Invoice stuck "Draft In Progress" | Batch completion status, DPE step failure |
| Tax engine error (AvaTax/Vertex) | Named credentials, callout config, tax line mapping |
| Credit memo error | Credit note allocation, source invoice status |
| Payment schedule issue | Payment allocation rules, billing account config |

---

## Common Issues — BRE

| Symptom | Check |
|---|---|
| Decision Table not activating | Activation In Progress stuck, record limit exceeded |
| Expression Set deployment error | Metadata deployment, namespace conflicts |
| Context Definition won't deactivate | Active references from Expression Sets |
| BRE Runtime PSL missing | License assignment, permission set availability |
| Lookup Table refresh failed | Special characters in keys, hash key group limit |
| Expression Set stops in sandbox | Sandbox-specific PSL availability |

---

## Common Issues — Configurator

| Symptom | Check |
|---|---|
| "Something went wrong while running configuration rules" | CML syntax, ConstraintEngineNodeStatus |
| Product not showing in catalog | Product visibility rules, Context Definition |
| Save & Exit not working | Cart state, field-level access |
| Amendment configurator errors | Amendment product matching, asset state |
| Bundle configuration failing | Nested bundle limits, attribute cardinality |

---

## Common Issues — Advanced Approvals

| Symptom | Check |
|---|---|
| Approval flow not triggering | Entry criteria, approval process config |
| Smart Approvals not auto-approving | Threshold config, approver matrix |
| Background Step license error | `UseBackgroundSteps` permission, PSL |
| ApprovalWorkItem stuck | Orphaned records, recall availability |
| Duplicate approval records | Concurrent submission, flow re-entry |

---

## Common Issues — CLM & DocGen

| Symptom | Check |
|---|---|
| DocGen permission error | CLM permission sets, Named Credential |
| Template merge not working | Merge field syntax, context service mapping |
| External storage (SharePoint) | SharePoint config, OAuth, file naming |
| E-signature (DocuSign) config | Integration settings, envelope routing |
| Contract status not changing | Status transition rules, validation rules |
| Font/formatting issues | Template format, PDF generation engine |

---

## Key Objects

| Object | Domain | Description |
|---|---|---|
| `BillingSchedule` | Billing | Billing schedule for recurring charges |
| `BillingScheduleGroup` | Billing | Groups related billing schedules |
| `Invoice` | Billing | Generated invoice records |
| `InvoiceLine` | Billing | Individual invoice line items |
| `CreditMemo` / `DebitMemo` | Billing | Adjustments to invoices |
| `PaymentSchedule` | Billing | Payment plan records |
| `ExpressionSet` | BRE | Expression set definitions |
| `ExpressionSetVersion` | BRE | Versioned expression set |
| `DecisionTable` | BRE | Decision table definitions |
| `DecisionTableParameter` | BRE | DT input/output parameters |
| `ContextDefinition` | BRE | Data context for rules |
| `LookupTable` | BRE | Lookup reference tables |
| `ProductConfigurationInstance` | Configurator | Config session state |
| `ConstraintEngineNodeStatus__c` | Configurator | CML rule evaluation status |
| `ApprovalWorkItem` | Approvals | Pending approvals |
| `SBAA__Approval__c` | Approvals | Advanced Approval records |

---

## Sample SOQL Queries

### Billing schedules for an order
```soql
SELECT Id, BillingScheduleGroupId, ReferenceEntityId, Status,
       NextBillingDate, BillingFrequency, TotalAmount
FROM BillingSchedule
WHERE ReferenceEntityId = '<ORDER_ID>'
ORDER BY NextBillingDate ASC
```

### Invoice with lines
```soql
SELECT Id, InvoiceNumber, Status, TotalAmount, DueDate, BillingAccountId,
       (SELECT Id, Product2.Name, Quantity, TotalAmount, Type FROM InvoiceLines)
FROM Invoice
WHERE BillingAccountId = '<ACCOUNT_ID>'
ORDER BY DueDate DESC LIMIT 10
```

### Decision tables by status
```soql
SELECT Id, DeveloperName, Status, LastModifiedDate, LastModifiedBy.Name
FROM DecisionTable
WHERE Status IN ('Activation In Progress', 'Active', 'InProgress')
ORDER BY LastModifiedDate DESC LIMIT 20
```

### Expression set versions
```soql
SELECT Id, ExpressionSetId, ExpressionSet.DeveloperName, VersionNumber,
       Status, StartDateTime, EndDateTime
FROM ExpressionSetVersion
WHERE Status = 'Active'
ORDER BY ExpressionSet.DeveloperName
LIMIT 20
```

### Pending approvals
```soql
SELECT Id, SBAA__Approver__r.Name, SBAA__Status__c, SBAA__ApprovalStep__c,
       SBAA__RecordField__c, CreatedDate
FROM SBAA__Approval__c
WHERE SBAA__Status__c = 'Requested'
ORDER BY CreatedDate ASC LIMIT 20
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex uncaught exceptions (billing batch, approval triggers) |
| `ipipr` | Integration Procedures (BRE execution, configurator flows) |
| `ipdar` | DataRaptors (product catalog, pricing data) |
| `axlim` | Governor limits (invoice batch, DPE processing) |
| `axque` | Queueable jobs (async billing, approval processing) |
| `gslog` | Platform Java exceptions (billing engine, DPE) |

---

## Code Investigation Paths

### Revenue Management Package
```
Tool: mcp__plugin_git-emu_vmcp-git-emu__get_file_contents
owner: "sf-industries"
repo: "via_rm"
path: "classes/<ClassName>.cls"
```

### CLM / Contract Package
```
Tool: mcp__plugin_git-emu_vmcp-git-emu__get_file_contents
owner: "sf-industries"
repo: "via_contract"
path: "classes/<ClassName>.cls"
```

### Billing Core Implementation
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-262-public path:core/billing-impl content:<keyword>"
max_matches: 10
```

### CPQ PTC Layer
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_cmt/CPQServicePtc.apex"
```

---

## Symptom-Driven Fast Path

| Symptom | First Check |
|---|---|
| Decision Table stuck "Activation In Progress" | Row limit (50/200), source object permissions changed |
| Expression Set not working in sandbox | Sandbox-specific PSL availability, org refresh impact |
| Billing schedule not generating | Order status, billing treatment, trigger configuration |
| Invoice batch failing | Batch job logs, DPE errors, record locks |
| Configurator "Something went wrong" | CML syntax errors, `ConstraintEngineNodeStatus__c` |
| Advanced Approval not triggering | Entry criteria, approval process config, PSL |
| CLM DocGen permission error | CLM permission sets, Named Credential |
| BRE Runtime PSL missing | License assignment, permission set availability |
| Context Definition won't deactivate | Active references from Expression Sets |
| Product not showing in catalog | Product visibility rules, Context Definition |

---

## Escalation

- Slack: `#support-rev-dev-amer`, `#support-omnistudio-collaboration`
- GUS product tag: `Revenue Cloud`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Product Area: [Billing | BRE | Configurator | Approvals | CLM/DocGen | DRO | Product Catalog | Other]
Issue Description:
Decision Table / Expression Set name (if BRE):
Reproduced in Demo org?:
Troubleshooting steps taken?:
Recent deployment or sandbox refresh?:
```

---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `approvals-patterns.md` — Advanced Approvals, orchestration flows, ApprovalWorkItem, Smart Approvals (18 patterns)
- `billing-patterns.md` — Billing schedules, invoice batch runs, tax engine, PDF generation, DPE (10 patterns)
- `bre-patterns.md` — Decision Tables, Expression Sets, Context Definitions, Lookup Tables, CML (9 patterns)
- `clm-docgen-patterns.md` — CLM contracts, DocGen templates, merge fields, e-signature, SharePoint (13 patterns)
- `configurator-patterns.md` — Product Configurator, CML rules, Browse Catalog, TLE, bundle config (11 patterns)
- `dro-patterns.md` — Revenue recognition, revenue schedules, ASC 606/IFRS 15, performance obligations (10 patterns)
- `product-to-order-patterns.md` — Product catalog, bundles, pricing config, quote-to-order mapping (10 patterns)
