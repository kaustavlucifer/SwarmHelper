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

## Escalation

- Slack: `#support-rev-dev-amer`, `#support-omnistudio-collaboration`
- GUS product tag: `Revenue Cloud`


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
