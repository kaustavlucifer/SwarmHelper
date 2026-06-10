# Salesforce CPQ (SBQQ) Debugger

**Trigger:** Salesforce CPQ, SBQQ, QCP (Quote Calculator Plugin), quote line editor (QLE), amendments, contracts, pricing rules, CPQ API, subscription pricing, renewals, order contracting, MDQ, document generation (CPQ flavor).

---

## Key Differences: Salesforce CPQ vs Revenue Cloud CPQ

| | Salesforce CPQ (SBQQ) | Revenue Cloud (Industries CPQ) |
|---|---|---|
| Namespace | `SBQQ__` | `vlocity_cmt__` or standard objects |
| Package | Managed package (Steelbrick) | Industries package or core |
| Objects | `SBQQ__Quote__c`, `SBQQ__QuoteLine__c` | `Quote`, `QuoteLineItem` or standard |
| Pricing | QCP (JavaScript), Pricing Rules | Calculation Procedures, Decision Tables |

---

## Repository Architecture

### GitHub Repos

| Repository | Path | Content |
|---|---|---|
| `via_cpq` | `sf-industries/via_cpq` | Industries CPQ managed package |
| `CPQ` | `git.soma.salesforce.com/Steelbrick/CPQ` | Salesforce CPQ (SBQQ) managed package — Apex, VF, JS, CSS |
| `CPQ-REST` | `git.soma.salesforce.com/Steelbrick/CPQ-REST` | CPQ JS services (calculator, amend/renew) on Heroku |
| `org62-cpq-aws-*` | `SF-BT/org62-cpq-aws-*` | CPQ pricing, approval rules, config loader, evaluation engine |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-cpq/                    ← Industries CPQ core (rule engine)
  core/industries-cpq-impl/              ← Industries CPQ implementation
  core/industries-cpq-pricing-common-api/ ← CPQ pricing common API
  core/cpq/                              ← Core CPQ functionality
  core/cpq-config-rules/                 ← CPQ configuration rules
```

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| CPQ Onboarding (Asset Lifecycle) — includes full investigation workflow | https://confluence.internal.salesforce.com/spaces/QUOTETOC/pages/1187813893/Salesforce+CPQ+Onboarding+Asset+Lifecycle |
| How to Troubleshoot CPQ Issues (Industries CPQ) | https://confluence.internal.salesforce.com/spaces/CSGPAK/pages/432075488/How+to+Troubleshoot+CPQ+Issues |
| Industries CPQ: Project CPQNext | https://confluence.internal.salesforce.com/spaces/VLCOM/pages/449598012/Industries+CPQ+Project+CPQNext |
| CPQ (IN Space) | https://confluence.internal.salesforce.com/spaces/IN/pages/591534674/CPQ |
| Rev Cloud Development Briki (internal) | https://sites.google.com/salesforce.com/rev-cloud-development/ |

---

## Common Issues

| Symptom | Check |
|---|---|
| QLE not loading | QCP errors, JavaScript console, Quote status |
| Apex heap size error | Large quote (1000+ lines), QCP complexity |
| Too many SOQL queries | Triggers stacking on quote save, flow loops |
| Amendment failing | Contract status, subscription fields, amendment start date |
| Contract not creating | Order → Contract flow, required fields, validation rules |
| Pricing rule not firing | Rule order, evaluation event, conditions |
| Renewal quote issue | Renewal model, subscription term, opportunity stage |
| Order contracting error | Contract field mapping, order status transition |
| UNABLE_TO_LOCK_ROW | Concurrent quote saves, background jobs on same records |
| Calculation service auth | Service URL, certificate, endpoint config |
| MDQ (Multi-Dimensional Quoting) | Segment config, dimension mapping |
| CPQ document generation | Template merge fields, PDF generation |

---

## Developer-Tier Issues

| Symptom | Check |
|---|---|
| QCP custom code failing | JavaScript syntax, async/await patterns, API limits |
| CPQ API integration | REST API endpoints, bulk operations, field mapping |
| Managed trigger failures | Trigger order, recursion guards, namespace conflicts |
| Custom Flow + CPQ | Flow trigger order, CPQ calculator not re-firing |
| Queueable job failures | Chaining limits, async context restrictions |
| Middleware query failures | Connected App config, API versioning |

---

## Debugging Approach

1. **Browser console** (F12) — QLE/QCP errors appear here
2. **Debug logs** — Apex execution for quote save/calculate
3. **SBQQ Settings** — Custom Setting `SBQQ__QuoteSetting__c` controls behavior
4. **Calculation order** — Price Rules → QCP → Calculator → Plugins
5. **Package version** — Check installed version vs known bug fixes

---

## SBQQ Object Reference

| Object | Description | Key Fields |
|---|---|---|
| `SBQQ__Quote__c` | Quote record | `SBQQ__Status__c`, `SBQQ__Primary__c`, `SBQQ__NetAmount__c`, `SBQQ__Opportunity2__c` |
| `SBQQ__QuoteLine__c` | Quote line item | `SBQQ__Product__c`, `SBQQ__Quantity__c`, `SBQQ__NetPrice__c`, `SBQQ__Number__c` |
| `SBQQ__ProductOption__c` | Bundle options | `SBQQ__OptionalSKU__c`, `SBQQ__ConfiguredSKU__c`, `SBQQ__Type__c` |
| `SBQQ__PricingRule__c` | Pricing rules | `SBQQ__TargetObject__c`, `SBQQ__EvaluationEvent__c`, `SBQQ__Active__c` |
| `SBQQ__PriceCondition__c` | Rule conditions | `SBQQ__Field__c`, `SBQQ__Operator__c`, `SBQQ__Value__c` |
| `SBQQ__PriceAction__c` | Rule actions | `SBQQ__TargetObject__c`, `SBQQ__Field__c`, `SBQQ__Value__c` |
| `SBQQ__Subscription__c` | Active subscriptions | `SBQQ__Contract__c`, `SBQQ__RenewalQuantity__c`, `SBQQ__TermStartDate__c` |
| `SBQQ__ContractedPrice__c` | Contracted prices | `SBQQ__OriginalQuoteLine__c`, `SBQQ__Price__c` |

### Custom Settings (Critical for Debugging)

| Setting | Key Fields |
|---|---|
| `SBQQ__QuoteSettings__c` (org-wide) | `SBQQ__CalculatorServiceTimeout__c`, `SBQQ__EnableQuoteCalculator__c`, `SBQQ__CalculationMethod__c` |
| `SBQQ__LineEditorSettings__c` | `SBQQ__EnableMultiLineEditing__c` |
| `SBQQ__Plugins__c` (org-wide) | `SBQQ__PricingPlugin__c` — class name for QCP |

---

## Sample SOQL Queries

### Quote with lines
```soql
SELECT Id, Name, SBQQ__Status__c, SBQQ__Primary__c, SBQQ__NetAmount__c,
       SBQQ__Opportunity2__r.Name, SBQQ__Account__r.Name,
       (SELECT Id, SBQQ__Product__r.Name, SBQQ__Quantity__c, SBQQ__NetPrice__c,
               SBQQ__Number__c, SBQQ__SubscriptionTerm__c
        FROM SBQQ__LineItems__r ORDER BY SBQQ__Number__c)
FROM SBQQ__Quote__c
WHERE SBQQ__Opportunity2__c = '<OPPORTUNITY_ID>'
ORDER BY CreatedDate DESC LIMIT 5
```

### Active pricing rules
```soql
SELECT Id, Name, SBQQ__TargetObject__c, SBQQ__EvaluationEvent__c,
       SBQQ__Active__c, SBQQ__ConditionsMet__c,
       (SELECT Id, SBQQ__Field__c, SBQQ__Operator__c, SBQQ__Value__c FROM SBQQ__PriceConditions__r),
       (SELECT Id, SBQQ__TargetObject__c, SBQQ__Field__c, SBQQ__Value__c FROM SBQQ__PriceActions__r)
FROM SBQQ__PricingRule__c
WHERE SBQQ__Active__c = true
ORDER BY Name LIMIT 20
```

### Subscriptions for a contract
```soql
SELECT Id, SBQQ__Product__r.Name, SBQQ__Quantity__c, SBQQ__NetPrice__c,
       SBQQ__TermStartDate__c, SBQQ__TermEndDate__c, SBQQ__RenewalQuantity__c
FROM SBQQ__Subscription__c
WHERE SBQQ__Contract__c = '<CONTRACT_ID>'
ORDER BY SBQQ__TermStartDate__c
```

### QCP plugin configuration
```soql
SELECT SBQQ__PricingPlugin__c FROM SBQQ__Plugins__c LIMIT 1
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex uncaught exceptions (quote save, calculation, triggers) |
| `axlim` | Governor limits (large quotes, 1000+ lines) |
| `axque` | Queueable jobs (async calculation, renewal processing) |
| `axftr` | @future methods (background pricing) |
| `axapx` | Apex execution summary (quote save timing) |

---

## Code Investigation Paths

### SBQQ Managed Package (Steelbrick)
```
Tool: mcp__plugin_git-soma_vmcp-git-soma__get_file_contents
owner: "Steelbrick"
repo: "CPQ"
path: "classes/<ClassName>.cls"
```

### CPQ REST Services (Heroku Calculator)
```
Tool: mcp__plugin_git-soma_vmcp-git-soma__get_file_contents
owner: "Steelbrick"
repo: "CPQ-REST"
path: "src/<path>"
```

### CPQ Core
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-262-public path:core/cpq content:<keyword>"
max_matches: 10
```

---

## Symptom-Driven Fast Path

| Symptom | First Check |
|---|---|
| QLE not loading / blank | Browser console (JS errors), SBQQ__Quote__c status field |
| Apex heap size on quote save | Line count (>500?), QCP complexity, nested bundles |
| Too many SOQL queries | Trigger stacking, flow loops on quote save |
| Amendment not working | Contract status, subscription fields, amendment start date |
| Pricing rule not firing | Rule order, evaluation event, conditions met criteria |
| Renewal quote failing | Renewal model, subscription term, SBQQ__RenewalModel__c |
| UNABLE_TO_LOCK_ROW | Concurrent quote saves, background jobs on same records |
| Calculation service timeout | `SBQQ__CalculatorServiceTimeout__c` setting, QCP performance |
| Contract not creating from order | Order status, field mapping, validation rules |
| QCP custom code failing | JavaScript syntax, async/await, console.log debugging |

---

## Escalation

- Slack: `#support-rev-dev-amer`
- GUS product tag: `Revenue Cloud` (CPQ bugs filed here)

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Component: [QLE | QCP | Pricing Rules | Amendments | Renewals | Contracts | API | Other]
Quote Line Count:
SBQQ Package Version:
Issue Description:
Browser console errors?:
Reproduced in Demo org?:
Troubleshooting steps taken?:
```

---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `developer-patterns.md` — Developer-tier issues: QCP, API integration, Apex governor limits
- `known-patterns.md` — Known issue patterns with resolution steps
