# Manufacturing Cloud Debugger

**Trigger:** Manufacturing Cloud, warranty, plant management, sales agreements, account forecasting, fleet management, sample management, dealer search, `mfg`, `ManufacturingCloud`, rebate management, program-based business.

---

## Product Areas

| Area | Scope |
|---|---|
| Sales Agreements | Schedule-based ordering, actuals tracking, account manager targets |
| Account Forecasting | Advanced account forecast, product category forecasting |
| Program Based Business | Program forecasts, manufacturing program components, DPE processing |
| Sample Management | Sample inventory tracking, sample requests, distribution |
| Fleet Management | Vehicle fleets, maintenance scheduling, telematics |
| Warranty Management | Goodwill repairs, warranty claims, OEM/dealer workflows |
| Dealer Management | Dealer search, dealer network, partner performance |
| Inventory Management | Stock tracking, distribution, demand planning |
| Asset Service Management | Field service for manufacturing assets |
| Analytics | CRM Analytics for Manufacturing, dashboards |

---

## Repository Architecture

### GitHub / git.soma Repos

| Repository | Path | Content |
|---|---|---|
| `via_platform` | `sf-industries/via_platform` | OmniStudio Apex (shared) |
| `via_core` | `sf-industries/via_core` | Platform foundation (shared) |

> **Note:** Manufacturing Cloud is core-only — no dedicated managed package repo.

### Core Monorepo Paths (CONFIRMED via CodeSearch)

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-manufacturing/                    ← Manufacturing Cloud implementation
  core/industries-manufacturing-udd/                ← Manufacturing UDD (objects, config)
  core/ui-industries-manufacturing-components/      ← Manufacturing LWC components
  core/industries-rebate-impl/                      ← Rebate Management implementation
```

### Ownership Teams
- `MFG-Warriors`, `MFG-Core`, `MFG-Titans`, `MFG-Vikings`, `MFG-Strikers`
- `Industries Einstein - GPT Usecases` (AI features)

---


### PTC Layer

Manufacturing is core-only (no PTC layer). Uses shared OmniStudio PTC when OmniStudio components are involved:
- `DataRaptorUtilsPtc.apex`, `IntegrationProcedureUtilsPtc.apex`, `OmniScriptServicePtc.apex`

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| Manufacturing Cloud (IN space) | https://confluence.internal.salesforce.com/spaces/IN/pages/188429750/ |

> **Note:** Limited dedicated engineering playbook documentation found. Check `#support-swarm-industries` for tribal knowledge.

---

## Key Objects

| Object | Description |
|---|---|
| `SalesAgreement` | Schedule-based ordering agreement |
| `SalesAgreementProduct` | Products within a sales agreement |
| `AccountForecast` | Account-level forecast records |
| `AccountForecastAdjustment` | Manual forecast adjustments |
| `ManufacturingProgram__c` | Manufacturing program definition |
| `ManufacturingProgramForecast__c` | Program forecast records |
| `Asset` | Managed assets (fleet, equipment) |
| `Claim` / `ClaimItem` / `ClaimCoverage` | Warranty claims |
| `Product2` / `Pricebook2` | Product catalog |
| `WarrantyTerm` | Standard warranty term definition |
| `MaintenancePlan` / `MaintenanceAsset` | Maintenance scheduling |

---

## Common Issues

| Symptom | Check |
|---|---|
| Sales Agreement actuals not syncing | DPE job status, data source mapping, sync schedule |
| Account forecast not generating | Forecast configuration, DPE definition, product category mapping |
| Program forecast data volume issues | DPE batch size, governor limits on large data volumes |
| Warranty claim denied unexpectedly | Warranty term effective dates, coverage verification rules |
| Goodwill repair flow failing | Flow configuration, OEM vs dealer user context |
| Fleet management data not loading | Asset configuration, telematics integration |
| Sample management limits exceeded | Custom metadata configurations, inventory object permissions |
| Analytics dashboard errors | CRM Analytics app permissions, dataset refresh |
| Dealer search not returning results | Criteria-Based Search permission set, search configuration |
| Inventory Management agent errors | Agentforce template configuration, supported functionality |
| DPE definition run failing | Data Processing Engine definition, input/output mapping |

---

## Sample SOQL Queries

### Sales Agreement with products
```soql
SELECT Id, Name, Status, StartDate, EndDate, Account.Name,
       (SELECT Id, Product2.Name, PlannedQuantity, ActualQuantity FROM SalesAgreementProducts)
FROM SalesAgreement
WHERE AccountId = '<ACCOUNT_ID>'
ORDER BY StartDate DESC LIMIT 10
```

### Account Forecasts
```soql
SELECT Id, AccountId, Account.Name, ForecastingTypeId, StartDate, EndDate
FROM AccountForecast
WHERE AccountId = '<ACCOUNT_ID>'
ORDER BY StartDate DESC LIMIT 10
```

### Warranty Claims
```soql
SELECT Id, CaseNumber, Status, CreatedDate, Asset.Name,
       (SELECT Id, Name, Type FROM ClaimItems)
FROM Claim
WHERE Asset.Account.Id = '<ACCOUNT_ID>'
  AND RecordType.DeveloperName LIKE '%Warranty%'
ORDER BY CreatedDate DESC LIMIT 20
```

---

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
query: "repo:gitcore.soma.salesforce.com/core-2206/core-262-public content:Manufacturing lang:java"
max_matches: 15
```

### LWC Components
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/ui-industries-manufacturing-components/"
```

---

## Debugging Approach

1. **Identify the feature area** — Sales Agreements, Forecasting, Warranty, Fleet, etc.
2. **Check DPE definitions** — Many MFG features rely on Data Processing Engine
3. **Verify permissions** — Manufacturing Cloud requires specific permission sets
4. **Check batch/async jobs** — Forecast generation and actuals sync are batch processes
5. **Review custom metadata** — TPM custom metadata configurations affect behavior
6. **Check governor limits** — Large data volumes (forecasts, actuals) can hit limits

---

## Escalation

- GUS product tag: `Manufacturing Cloud`
- Slack: `#support-swarm-industries`
- Related: `#support-consumer-goods` (shared team)

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [Sales Agreements | Forecasting | Warranty | Fleet | Sample Mgmt | Other]
Issue Description:
DPE Job Status (if applicable):
Reproduced in Demo org?:
Troubleshooting steps taken?:
```
