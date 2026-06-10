# Automotive Cloud Debugger

**Trigger:** Automotive Cloud, vehicle, dealer, VIN, auto, `AutomotiveCloud`, vehicle lifecycle, dealer management, auto finance, warranty claims (automotive context), parts and accessories.

---

## Product Areas

| Area | Scope |
|---|---|
| Vehicle Lifecycle Management | VIN tracking, vehicle records, ownership history, vehicle definitions |
| Dealer Management | Dealer network, dealer search, dealer performance, partner portals |
| Auto Finance | Loan management, asset finance, lease management, loan payoff |
| Warranty Claims | Automotive warranty, goodwill repairs, OEM/dealer workflows |
| Document Validation | AI-powered document validation for automotive processes |
| Sales Concierge | Agentforce for automotive sales, partner sales assistance |
| Parts & Accessories | Product and parts search, related accessories |
| Data Processing | DPE for bulk record creation (products, assets, vehicles) |
| Fleet Management | Shared with Manufacturing Cloud |

---

## Repository Architecture

### GitHub / git.soma Repos

| Repository | Path | Content |
|---|---|---|
| `via_platform` | `sf-industries/via_platform` | OmniStudio Apex (shared) |
| `via_core` | `sf-industries/via_core` | Platform foundation (shared) |

> **Note:** Automotive Cloud is core-only — no dedicated managed package repo. Shares teams with Manufacturing.

### Core Monorepo Paths (CONFIRMED via CodeSearch)

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-automotive/                         ← Automotive Cloud implementation
  core/industries-automotive-udd/                     ← Automotive UDD (objects, config)
  core/industries-automotive-setup-home/              ← Setup/config UI
  core/industries-manufacturing/                      ← Shared Manufacturing infrastructure
```

### Ownership Teams
- `MFG-Knights`, `MFG-Vikings`

### Key Core Classes
- `VehicleDefinition` — `core/shared-pathassistant/java/src/common/runtime/pipelineview/VehicleDefinition.java`
- Uses `VehicleDefinitionStandardEntity` from `com.force.commons.sobject.industriesautomotiveudd.entities`

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| Automotive Developer 101 | https://confluence.internal.salesforce.com/spaces/IN/pages/718878719/Automotive+Developer+101 |
| Automotive Cloud (IN space) | https://confluence.internal.salesforce.com/spaces/IN/pages/718518461/Automotive+Cloud |
| Automotive Foundation Tech DD | https://docs.google.com/presentation/d/1s147Y-hOd3CIVS6ANNqLY_MINdfdK2jODeN6jlHzRK8/edit |
| Team Agreement | https://salesforce.quip.com/WbhvACpufqt3 |

---

## Key Objects

| Object | Description |
|---|---|
| `Vehicle` / `VehicleDefinition` | Vehicle records and definitions |
| `Asset` | Vehicle as managed asset (finance, warranty) |
| `Account` | Dealer accounts, customer accounts |
| `Claim` / `ClaimItem` / `ClaimCoverage` | Warranty claims |
| `DocumentChecklistItem` | Document validation records |
| `Product2` | Parts, accessories catalog |
| `Order` / `OrderItem` | Parts orders, service orders |
| `Quote` / `QuoteLineItem` | Vehicle/service quotes |
| `Referral` | Dealer referrals |
| `Lead` / `Opportunity` | Sales pipeline |
| `WarrantyTerm` | Warranty term definitions |
| Custom: `VIN__c`, `DealerAmount__c`, `RepairOrderAmount__c` | Vertical-specific fields |

---

## Common Issues

| Symptom | Check |
|---|---|
| Vehicle record not syncing | DPE job status, data source mapping |
| Dealer search not working | Criteria-Based Search permission set, search configuration |
| Warranty claim denied | Warranty term effective dates, coverage verification rules |
| Goodwill repair flow error | Flow configuration, OEM vs dealer user context |
| Document validation failing | AI model configuration, document format requirements |
| Auto finance calculation wrong | Finance product setup, interest rate configuration |
| Parts search returning empty | Product catalog activation, parts relationship mapping |
| DPE bulk import failing | CSV format validation, required fields (ProductCode, RowNumber) |
| Sales concierge agent errors | Agentforce template configuration, prompt template setup |
| VIN lookup not resolving | External service configuration, Named Credential setup |
| Fleet data not loading | Asset configuration, telematics integration |

---

## Sample SOQL Queries

### Vehicle assets with warranty
```soql
SELECT Id, Name, SerialNumber, Status, Account.Name,
       Product2.Name, InstallDate, UsageEndDate
FROM Asset
WHERE Account.Id = '<ACCOUNT_ID>'
  AND Product2.Family = 'Vehicle'
ORDER BY InstallDate DESC LIMIT 20
```

### Warranty claims for dealer
```soql
SELECT Id, CaseNumber, Subject, Status, CreatedDate,
       Asset.Name, Asset.SerialNumber
FROM Case
WHERE Account.Id = '<DEALER_ACCOUNT_ID>'
  AND RecordType.DeveloperName LIKE '%Warranty%'
ORDER BY CreatedDate DESC LIMIT 20
```

### Document checklist items
```soql
SELECT Id, Name, Status, ParentRecordId, DocumentTypeId,
       CreatedDate, LastModifiedDate
FROM DocumentChecklistItem
WHERE ParentRecordId = '<RECORD_ID>'
ORDER BY CreatedDate DESC
```

---

## Code Investigation Paths

### Core Implementation
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-262-public content:Automotive lang:java"
max_matches: 15
```

### LWC Components
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/ui-industries-automotive-components/"
```

---

## Debugging Approach

1. **Identify the feature area** — Vehicle lifecycle, dealer, finance, warranty, parts
2. **Check DPE definitions** — Bulk operations rely on Data Processing Engine
3. **Verify permissions** — Automotive Cloud requires specific permission sets and licenses
4. **Check external integrations** — VIN lookup, telematics, finance systems use Named Credentials
5. **Review Agentforce config** — Sales Concierge and document validation use AI/Agentforce
6. **Check data model** — Vehicle → Asset → Warranty Term relationships

---

## Escalation

- GUS product tag: `Automotive Cloud`
- Slack: `#automotive-cloud-experts`, `#industries-all-manufacturing`, `#industries-engg-manufacturing`
- Related: `#support-swarm-industries`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [Vehicle Lifecycle | Dealer Mgmt | Auto Finance | Warranty | Parts | DocValidation | Other]
Issue Description:
VIN/Asset involved?:
Reproduced in Demo org?:
Troubleshooting steps taken?:
```
