---
name: revenue-lifecycle-mgmt
description: Revenue Lifecycle Management troubleshooting — asset lifecycle, quote-to-order, sales transactions, usage selling, CPI uplift, amendments/renewals. Routed by /swarm-helper.
---

# Revenue Lifecycle Management (RLM) Debugger

**Trigger:** Revenue Lifecycle Management, RLM, Asset Lifecycle Management, Quote to Order Capture, sales transactions, transaction rollbacks, contract cotermination, usage selling, usage-based assets, grant management, consumption tracking, CPI uplift, renewal price uplift, swap amendment, upgrade/downgrade amendment, quote detail lines, CSV import into quote, order orchestration, `ConstraintEngineNodeStatus__c`, product variations, product attributes, product classifications, price lists, price adjustments, asset add/amend/renew/cancel operations.

> **Scope distinction:** This vertical handles **Revenue Lifecycle Management** — the next-generation asset lifecycle and subscription management platform (Quote-to-Order-to-Asset-to-Renew). It is **distinct** from:
> - **Revenue Cloud (Core)** — BRE, Advanced Approvals, CLM/DocGen, DRO, billing/invoicing (see `revenue-cloud/`)
> - **Salesforce CPQ (SBQQ)** — legacy Steelbrick quote-to-cash (see `cpq/`)
> - **Industries CPQ (vlocity_cmt)** — Industries-specific CPQ with EPC/order decomposition (see `comms-cloud/`)

---

## Product Areas

| Area | Scope |
|---|---|
| Asset Lifecycle Management | Add, amend, renew, cancel, swap, upgrade/downgrade assets; lifecycle state tracking; amendment operations |
| Quote to Order Capture | Quote detail lines, CSV import, order orchestration, quote-to-order conversion |
| Transaction Management | Sales transactions, transaction rollbacks, transaction history on assets |
| Product Configurator | Product variations, attributes, classifications, CML constraint rules, browse catalog |
| Pricing | Price lists, price adjustments, CPI/renewal price uplift, usage-based pricing |
| Usage Selling | Usage-based assets, grant management, consumption tracking, overage calculation |
| Contract Cotermination | Aligning subscription end dates across contract assets, proration |

---

## Repository Architecture

```
┌───────────────────────────────────────────────────────────┐
│  UI Components (LWC)          — Product Configurator UI,  │
│                                  Quote Editor, Asset UI   │
├───────────────────────────────────────────────────────────┤
│  Industries CPQ Impl           — CML rules, constraint    │
│                                  engine, configurator     │
├───────────────────────────────────────────────────────────┤
│  Industries CPQ Pricing        — Price lists, adjustments │
│                                  CPI uplift, renewals     │
├───────────────────────────────────────────────────────────┤
│  BRE (Business Rules Engine)   — Decision Tables,         │
│                                  Expression Sets          │
├───────────────────────────────────────────────────────────┤
│  Salesforce Platform (Standard Objects)                   │
│  Quote, QuoteLineItem, Order, OrderItem, Asset,          │
│  Product2, Contract, PricebookEntry                       │
└───────────────────────────────────────────────────────────┘
```

### Core Monorepo Paths

> Core paths + class names below were live-probed against `core-262-public` on 2026-06-15. Modern RLM is **core-based** (these `core/*` modules), NOT the legacy `via_rm` managed package.

```
gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public:
  # Configurator / pricing / rules
  core/industries-cpq/                         ← Industries CPQ core (NGP pricing service)
  core/industries-cpq-core/                    ← CPQ core pricing services (PricingService, CartPricingService, PreviewPricingService)
  core/industries-cpq-impl/                    ← CPQ implementation (CpqPricingService, asset services, configurator model)
  core/industries-cpq-core-connect-api/        ← CPQ connect API (ConstraintEngineConstants)
  core/industries-cpq-core-connect-impl/       ← CPQ connect impl (ConstraintEngineOverallOutput, instant pricing)
  core/industries-cpq-pricing-common-api/      ← CPQ pricing common API
  core/industries-constraint-studio-connect-impl/ ← CML parser (CmlParseResult) — Constraint Studio
  core/cpq/                                    ← Core CPQ functionality (shared)
  core/cpq-config-rules/                       ← CPQ configuration rules (CML)
  # Asset lifecycle / amend-renew / cancel
  core/revenue-amend-renew-api/                ← Amend/Renew/Swap (SwapService, PearCancelService)
  core/asset-management-impl/                  ← Asset cancel (CalmCancelService, PhoenixCalmCancelService)
  # Sales transactions
  core/sales-transaction-api/                  ← Sales transaction API (PlaceSalesTransactionService, PreprocessSalesTransactionService)
  core/revenue-place-sales-transaction-impl/   ← Place sales transaction impl (ReadSalesTransactionService)
  core/revenue-clone-sales-transaction-api/    ← Clone sales transaction (RevenueCloneSalesTransactionService)
  # BRE (Decision Tables / Expression Sets used in pricing)
  core/industries-bre-near-core-impl/          ← BRE implementation
  core/industries-bre-near-core-api/           ← BRE API layer
  core/industries-bre-engine-runtime/          ← BRE runtime engine
  # Billing
  core/billing-impl/                           ← Billing implementation (usage billing, revenue schedules)
  core/billing-services/                       ← Billing schedule amend/renew (BSAmendmentService, BSGRenewalService)
  core/ui-industries-cpq-components/           ← CPQ UI components (LWC)
  core/industries-interaction-ptc/apex/        ← PTC layer (protected Apex)
```

### PTC Layer (Protected Apex)

RLM shares only two PTC-layer classes with Industries CPQ (verified 2026-06-15 — asset/transaction/configurator logic lives in the Java impl modules above, NOT as PTC classes):
```
CPQServicePtc.apex                  ← Quote/Order/cart operations
CalculationServicePtc.apex          ← Pricing calculation entry
```

### GitHub Repos (Industries Managed Packages)

| Repository | Path | Content |
|---|---|---|
| `via_rm` | `sf-industries/via_rm` | ⚠️ **Legacy Vlocity Revenue Management** (Story/recommendation/scoring engine — `Story`, `VqMachine`, `VqService`, Vlocity Content Attributes). **NOT** modern RLM asset lifecycle — for asset/transaction/pricing use the `core/*` paths above. |
| `via_cpq` | `sf-industries/via_cpq` | Industries CPQ managed package (configurator) — indexed by deep-research codesearch |
| `via_core` | `sf-industries/via_core` | Foundation (InvokeService, SecurityChecker, StateTransition) |
| `via_platform` | `sf-industries/via_platform` | OmniStudio engine (DR/IP/OS used in flows) |

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| CPQ/RLM Onboarding (Asset Lifecycle) | https://confluence.internal.salesforce.com/spaces/QUOTETOC/pages/1187813893/Salesforce+CPQ+Onboarding+Asset+Lifecycle |
| Revenue Cloud Runbooks | https://confluence.internal.salesforce.com/spaces/QUOTETOC/pages/1294353243/Revenue+Cloud+Runbooks |
| Industries CPQ: Project CPQNext | https://confluence.internal.salesforce.com/spaces/VLCOM/pages/449598012/Industries+CPQ+Project+CPQNext |
| Rev Cloud Development (internal) | https://sites.google.com/salesforce.com/rev-cloud-development/ |

---

## Key Objects

> **Partially verified 2026-06-15.** `PriceAdjustmentSchedule` and `PriceAdjustmentTier` confirmed as **standard objects** (HTTP 200 across multiple orgs). Standard objects (Quote, Order, Asset, Contract, Product2, Pricebook2) are reliable. The transaction/usage objects (`SalesTransaction`, `SalesTransactionItem`, `UsageGrant`, `ConstraintEngineNodeStatus__c`) **could not be verified** — the two revenue-flavored orgs probed (Comms CPQ + a Revenue demo org) both returned 404 (neither was RLM-SalesTransaction–enabled). ⚠️ Confirm these via `EntityDefinition`/describe in a true **Revenue Lifecycle Management–enabled** org before relying on exact names. Do not assume a custom field exists.

### Quote & Order

| Object | Purpose |
|---|---|
| `Quote` | Standard Quote object — entry point for RLM transactions |
| `QuoteLineItem` | Quote detail lines — each represents a product/asset action |
| `Order` | Resulting order from quote acceptance |
| `OrderItem` | Order line items — converted from QuoteLineItem |
| `OpportunityLineItem` | Opportunity products (upstream from Quote) |

### Asset & Contract

| Object | Purpose |
|---|---|
| `Asset` | Represents subscriptions/products owned by customer; tracks lifecycle state |
| `Contract` | Groups assets under a subscription term; cotermination target |
| `Product2` | Product catalog — variations, attributes, classifications |
| `PricebookEntry` | Price list entries per product/pricebook |
| `Pricebook2` | Price lists (standard + custom) |

### Configuration & Rules

| Object | Purpose |
|---|---|
| `ConstraintEngineNodeStatus__c` | CML rule evaluation status — tracks which constraint rules ran |
| `ProductAttribute` | Product attribute definitions for configurator |
| `ProductAttributeValue` | Allowed values per product attribute |
| `ProductClassification` | Product classification hierarchy |

### Transaction & Pricing

| Object | Purpose |
|---|---|
| `SalesTransaction` | Sales transaction records (RLM-specific) |
| `SalesTransactionItem` | Individual items within a sales transaction |
| `PriceAdjustmentSchedule` | Price adjustment rules (discounts, surcharges) |
| `PriceAdjustmentTier` | Tiered pricing adjustments |

---

## Common Issues

### Asset Lifecycle Management

| Symptom | Root Cause | Resolution |
|---|---|---|
| Zero-quantity quote detail lines appear | Lifecycle state preservation — system adds lines with qty=0 to maintain full asset picture during amendments | Expected behavior. Lines represent existing assets not being changed in this transaction. Filter UI display if needed. |
| Amend operation fails with missing asset | Asset not in active state or missing required lifecycle fields | Verify `Asset.Status` = "Active" and `Asset.LifecycleStartDate` / `LifecycleEndDate` populated |
| Swap/upgrade creates duplicate asset | Original asset not cancelled before new one created | Check transaction sequence; swap must cancel original in same transaction |
| Renew pricing incorrect after CPI uplift | CPI uplift calculation uses stale price or wrong base | Verify CPI configuration: base price source, uplift percentage, effective date range |
| Cancel amendment leaves orphaned records | Cancel does not cascade to child assets/related records | Check cascade settings; may need to manually clean related records or use full cancel flow |
| Upgrade/downgrade restrictions not enforced | Product classification hierarchy not properly configured | Verify `ProductClassification` records and allowed upgrade/downgrade paths |

### Quote to Order Capture

| Symptom | Root Cause | Resolution |
|---|---|---|
| CSV import fails silently | Missing `ProductCode`, `RowNumber`, or required fields in CSV | Validate CSV against required schema: `ProductCode` (mandatory), `RowNumber` (mandatory), all required custom fields |
| CSV import — product not found | `ProductCode` in CSV does not match any active `Product2.ProductCode` | Verify product code matches exactly (case-sensitive); product must be active in the target pricebook |
| Quote-to-order conversion error | Required fields on Order/OrderItem not populated by mapping | Check field mapping configuration; validate all required Order fields have source values |
| Order orchestration not firing | Order status not at expected stage for orchestration trigger | Verify order activation flow; check status field transitions |
| Quote detail line pricing mismatch | Price list entry not found or wrong pricebook assigned | Verify `Quote.Pricebook2Id` matches expected pricebook; check `PricebookEntry` for product |

### Transaction Management

| Symptom | Root Cause | Resolution |
|---|---|---|
| Transaction rollback fails | Attempting to roll back non-latest transaction | Rollback only reverses the MOST RECENT transaction on an asset. Cannot skip transactions. |
| Transaction rollback partial | Some assets in transaction were subsequently modified | Assets with newer transactions cannot be rolled back as part of an older transaction |
| Sales transaction stuck "In Progress" | Async processing failure or timeout | Check batch job status; look for governor limit errors in debug logs |

### Product Configurator

| Symptom | Root Cause | Resolution |
|---|---|---|
| "Something went wrong while running configuration rules" | CML rule syntax error or infinite loop | Check `ConstraintEngineNodeStatus__c` for error details; validate CML rule logic |
| Product variations not showing | Product classification or attribute configuration incomplete | Verify `ProductAttribute` and `ProductAttributeValue` records; check classification hierarchy |
| Configurator timeout | Complex product hierarchy with many CML rules | Reduce rule complexity; check for circular references in constraint rules |
| Save & Exit loses configuration | Cart/transaction state not properly persisted | Check field-level security on configuration fields; verify session/state management |

### Pricing

| Symptom | Root Cause | Resolution |
|---|---|---|
| Price list entry not found | Product not in assigned pricebook or inactive entry | Verify `PricebookEntry.IsActive = true` and product in quote's pricebook |
| CPI uplift not calculating | CPI configuration missing or product type not supported | CPI uplift supports specific product types only; verify product eligibility and CPI schedule |
| Renewal price uplift wrong percentage | Multiple adjustment schedules conflicting | Check `PriceAdjustmentSchedule` priority and applicability conditions |
| Price adjustment not applying | Adjustment schedule conditions not met | Verify `PriceAdjustmentSchedule` criteria match the transaction context |

### Usage Selling

| Symptom | Root Cause | Resolution |
|---|---|---|
| Usage-based asset not tracking consumption | Grant management configuration missing or incorrect | Verify usage grant setup, consumption tracking enabled, and metering integration active |
| Overage calculation incorrect | Grant depletion logic not accounting for rollovers | Check grant rollover settings and period boundaries |
| Grant not replenishing on renewal | Renewal flow not triggering grant reset | Verify renewal process includes grant replenishment step |

### Contract Cotermination

| Symptom | Root Cause | Resolution |
|---|---|---|
| Cotermination not aligning end dates | Cotermination has limitations on subscription end date alignment | Check that all assets are eligible for cotermination; verify contract end date is the target |
| Proration calculation wrong after coterm | Proration denominator using wrong period | Verify proration method configuration and the cotermed subscription's effective dates |

---

## Sample SOQL Queries

### Asset Lifecycle State
```sql
SELECT Id, Name, Product2.Name, Status, LifecycleStartDate, LifecycleEndDate,
       Quantity, Price, Contract.EndDate, Contract.ContractNumber
FROM Asset
WHERE AccountId = ':accountId'
AND Status = 'Active'
ORDER BY LifecycleStartDate DESC
```

### Quote Detail Lines (including zero-qty lifecycle lines)
```sql
SELECT Id, QuoteId, Product2.Name, Product2.ProductCode, Quantity, UnitPrice,
       TotalPrice, LineNumber, Description
FROM QuoteLineItem
WHERE QuoteId = ':quoteId'
ORDER BY LineNumber ASC
```

### Sales Transactions on an Asset
```sql
SELECT Id, Name, Status, CreatedDate, LastModifiedDate
FROM SalesTransaction
WHERE Id IN (SELECT SalesTransactionId FROM SalesTransactionItem WHERE Asset.Id = ':assetId')
ORDER BY CreatedDate DESC
```

### CML Rule Evaluation Status
```sql
SELECT Id, Name, Status__c, ErrorMessage__c, ConstraintRule__c, CreatedDate
FROM ConstraintEngineNodeStatus__c
WHERE CreatedDate = TODAY
AND Status__c = 'Error'
ORDER BY CreatedDate DESC
```

### Price List Entries for a Product
```sql
SELECT Id, Product2.Name, Product2.ProductCode, Pricebook2.Name,
       UnitPrice, IsActive, UseStandardPrice
FROM PricebookEntry
WHERE Product2Id = ':productId'
AND IsActive = true
```

### Contract Assets (for Cotermination Analysis)
```sql
SELECT Id, Name, Product2.Name, Status, LifecycleStartDate, LifecycleEndDate,
       Quantity, Contract.EndDate, Contract.ContractNumber
FROM Asset
WHERE Contract.Id = ':contractId'
AND Status IN ('Active', 'Suspended')
ORDER BY LifecycleEndDate ASC
```

### Orders Generated from Quote
```sql
SELECT Id, OrderNumber, Status, EffectiveDate, EndDate, Quote.QuoteNumber,
       (SELECT Id, Product2.Name, Quantity, UnitPrice, OrderItemNumber FROM OrderItems)
FROM Order
WHERE QuoteId = ':quoteId'
```

### Usage Grants on Asset
```sql
SELECT Id, Name, Asset.Name, GrantAmount, ConsumedAmount, RemainingAmount,
       StartDate, EndDate, RolloverEnabled
FROM UsageGrant
WHERE Asset.Id = ':assetId'
AND Status = 'Active'
```

---

## Code Investigation Paths

### Core Monorepo — Industries CPQ (PRIMARY for RLM)

```
Tool: mcp__plugin_deep-research_codesearch__search
query: "<class or method name>"
repo: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
path_filter: "core/industries-cpq"
```

Key paths by domain (class names live-probed against `core-262-public` 2026-06-15 — only confirmed classes listed):
| Domain | Path | Key Classes (verified) |
|---|---|---|
| Amend / Renew / Swap | `core/revenue-amend-renew-api/` | `SwapService`, `PearCancelService` |
| Asset cancel/disconnect | `core/asset-management-impl/` | `CalmCancelService`, `PhoenixCalmCancelService` |
| Asset pricing/scheduling | `core/industries-cpq-impl/.../service/` | `AssetPriceRetainService`, `AssetPriceMaintenanceService`, `AssetToOrderSchedulerService`, `AssetDisconnectionSchedulerService`, `CpqAssetToBasketService`, `CMEAssetizationService` |
| Billing amend/renew | `core/billing-services/` | `BSAmendmentService`, `BSGRenewalService` |
| Sales transactions | `core/sales-transaction-api/`, `core/revenue-place-sales-transaction-impl/` | `PlaceSalesTransactionService`, `PreprocessSalesTransactionService`, `ReadSalesTransactionService` |
| Transaction clone | `core/revenue-clone-sales-transaction-api/` | `RevenueCloneSalesTransactionService` |
| Configurator / constraint engine | `core/industries-cpq-impl/`, `core/industries-cpq-core-connect-impl/`, `core/industries-cpq-core-connect-api/` | `ConstraintEngineNodeStatus`, `ConstraintEngineConstants`, `ConstraintEngineOverallOutput` |
| CML parser (Constraint Studio) | `core/industries-constraint-studio-connect-impl/` | `CmlParseResult` |
| Pricing | `core/industries-cpq-core/.../pricing/service/`, `core/industries-cpq-impl/`, `core/industries-cpq/` | `PricingService`, `CartPricingService`, `PreviewPricingService`, `CpqPricingService`, `CompilePricingService`, `CpqExternalPricingService`, `NGPPricingService` |
| Transaction context | `core/industries-cpq-impl/.../service/` | `CpqTransactionContextService` |

> The earlier generic names (`AssetLifecycleManager`, `AmendmentService`, `RenewalService`, `CancelService`, `SalesTransactionService`, `TransactionRollbackService`, `ConstraintEngine`, `CMLRuleEvaluator`, `ProductConfigurationService`, `PriceCalculationService`, `PriceAdjustmentEngine`, `CPIUpliftCalculator`, `QuoteLineItemService`, `OrderCaptureService`, `CSVImportService`, `ConstraintModelLanguageParser`, `ConstraintNodeEvaluator`) did **not** resolve in core-262 and have been replaced with the probe-confirmed names above. If you need a class not listed, search by feature keyword rather than assuming a name.

### Core Monorepo — BRE (Decision Tables/Expression Sets used in pricing)

```
Tool: mcp__plugin_deep-research_codesearch__search
query: "<BRE class or method>"
repo: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
path_filter: "core/industries-bre"
```

### Core Monorepo — PTC Layer

```
Tool: mcp__plugin_deep-research_codesearch__search
query: "<PTC class name>"
repo: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
path_filter: "core/industries-interaction-ptc"
```

### Industries Managed Package — via_rm (Revenue Management)

```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "github.com/sf-industries/via_rm"
ref: "HEAD"
file_path: "classes/<ClassName>.cls"
```

### Industries Managed Package — via_cpq (Industries CPQ)

```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "github.com/sf-industries/via_cpq"
ref: "HEAD"
file_path: "classes/<ClassName>.cls"
```

### Industries Foundation — via_core

```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "github.com/sf-industries/via_core"
ref: "HEAD"
file_path: "classes/<ClassName>.cls"
```

Relevant foundation classes:
- `InvokeService` — service dispatch routing (all RLM services use this)
- `SecurityChecker` / `PlatformSecurityChecker` — FLS/CRUD enforcement
- `StateTransitionService` — asset/order state machine
- `AsyncProcessEngine` — batch job framework
- `OrgCacheManager` — platform cache

---

## Debugging Approach

### Step 1: Classify the Issue

Determine the RLM sub-domain:
1. **Asset Lifecycle** — add/amend/renew/cancel/swap/upgrade/downgrade
2. **Quote to Order** — quote detail lines, CSV import, order conversion
3. **Transaction** — sales transaction creation/rollback
4. **Configurator** — CML rules, product attributes, variations
5. **Pricing** — price lists, adjustments, CPI uplift, usage
6. **Contract** — cotermination, subscription alignment

### Step 2: Gather Context

```
Tool: mcp__orgcs__soqlQuery
Query the case to identify org, pod, and taxonomy:
  SELECT Id, Subject, Description, CaseNumber, InstanceId__c,
         Account.CustomerOrgId15__c, ProductArea__c, SubArea__c
  FROM Case WHERE CaseNumber = ':caseNum'
```

### Step 3: Check for Known Issues (GUS)

```
Tool: mcp__plugin_gus_gus_server__query_gus_records
Query:
  SELECT Id, Name, Subject__c, Status__c, Priority__c, Found_in_Build__c
  FROM ADM_Work__c
  WHERE Product_Tag__r.Name LIKE '%Revenue%Lifecycle%'
  AND Status__c NOT IN ('Closed', 'Never Fix', 'Duplicate')
  AND Subject__c LIKE '%<keyword>%'
  ORDER BY Priority__c, CreatedDate DESC
  LIMIT 20
```

### Step 4: Search Splunk Logs

```
Tool: mcp__plugin_monitoring_vmcp-monitoring__query_splunk
index=<pod_name>
sourcetype=sf:core:applog
"<org_id_15char>" AND ("AssetLifecycle" OR "SalesTransaction" OR "ConstraintEngine" OR "PriceCalculation")
| head 50
```

For transaction-specific errors:
```
index=<pod_name>
sourcetype=sf:core:applog
"<org_id_15char>" AND "TransactionRollback" AND ("error" OR "exception" OR "failed")
| table _time, message
| sort -_time
```

### Step 5: Investigate Source Code

Based on the error class/method identified in logs:

1. **Search for the class in core monorepo:**
   ```
   Tool: mcp__plugin_deep-research_codesearch__search
   query: "SwapService"  (or whatever class from the actual stack trace)
   repo: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
   ```

2. **Read the specific file:**
   ```
   Tool: mcp__plugin_deep-research_codesearch__read_file
   repo: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
   path: "<path from search results>"
   ```

3. **Check PTC layer for regressions:**
   ```
   Tool: mcp__plugin_deep-research_codesearch__search
   query: "<service name>Ptc"
   repo: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
   path_filter: "core/industries-interaction-ptc"
   ```

4. **Check managed package if needed:**
   ```
   Tool: mcp__plugin_deep-research_codesearch__read_file
   repository: "github.com/sf-industries/via_rm"
   ref: "HEAD"
   file_path: "classes/<ClassName>.cls"
   ```

### Step 6: Check for Recent Patches/Regressions

```
Tool: mcp__plugin_deep-research_codesearch__commit_search
query: "<class name or feature keyword>"
repo: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
```

Look for recent commits that may have introduced regressions.

### Step 7: Validate Customer Configuration

Run SOQL against the customer org (if accessible via OrgCS/case context) to verify:
- Product catalog setup (active products in correct pricebook)
- Asset states (lifecycle dates, status values)
- Contract configuration (cotermination settings)
- CML rules (constraint engine node status for errors)

---

## P&T Routing References

| P&T Entry | Maps To |
|---|---|
| Revenue Lifecycle Management - Asset Lifecycle Management | Asset add/amend/renew/cancel/swap/upgrade/downgrade |
| Revenue Lifecycle Management - Quote to Order Capture | Quote detail lines, CSV import, order orchestration |
| Revenue Lifecycle Management - Developer Support - OmniStudio and DocGen | OmniStudio/DocGen within RLM context (route to `omnistudio/` or `docgen/` if pure OS/DG issue) |
| Revenue Lifecycle Management - Developer Support - Product, Pricing, Config | Configurator, pricing, CML rules |
| Revenue Cloud (Core) - Transaction Management | Sales transactions, rollbacks (shared with RLM) |
| Revenue Cloud (Core) - Advanced Configurator | Product configurator, CML (shared with RLM) |

---

## Key Behavioral Notes

1. **Zero-quantity lines are expected** — During amendments, the system preserves the full asset picture by including lines with `Quantity = 0` for assets not being modified. Do not treat these as bugs.

2. **Transaction rollback is sequential** — Only the MOST RECENT transaction on an asset can be rolled back. You cannot skip or selectively roll back older transactions.

3. **CSV import is strict** — `ProductCode` and `RowNumber` are mandatory columns. All custom required fields must be present. Product codes are case-sensitive and must match active products in the pricebook.

4. **CPI uplift has product-type restrictions** — Not all product types support CPI (Consumer Price Index) or renewal price uplift. Check product eligibility before assuming a bug.

5. **Swap = Cancel + Add in one transaction** — Swap amendments cancel the original asset and add the new one atomically. Both operations must succeed or the transaction fails entirely.

6. **Cotermination limitations** — Contract cotermination cannot always align subscription end dates, particularly when assets have conflicting billing frequencies or fixed-term constraints.

7. **Configurator shares code with Revenue Cloud** — The Product Configurator (CML rules, constraint engine) is shared infrastructure between RLM and Revenue Cloud Core. Issues in `core/industries-cpq-impl/` or `core/cpq-config-rules/` may affect both.

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex uncaught exceptions (transaction processing, asset operations) |
| `ipipr` | Integration Procedures (RLM flows) |
| `axlim` | Governor limits (bulk asset operations, CSV import) |
| `gslog` | Platform Java exceptions (constraint engine, pricing) |

For Falcon services (off-platform):
```spl
index=distapps functional_domain=core1 k8s_namespace=revenue-cloud earliest=-7d
| head 50
```

---

## Escalation

| Channel | Use |
|---|---|
| `#support-rev-rlm-global-swarm-help` (C05TFMXB3RN) | RLM case swarming/collaboration |
| `#support-rev-rlm` (C06BXE12Y4Q) | RLM support |
| `#support-rev-dev-amer` (C0275QG40LE) | Rev Dev swarms (Americas) |
| `#support-omnistudio-collaboration` (C03GSNY2GVC) | OmniStudio-layer issues |
| `#support-swarm-industries` (C02BEHKLWES) | General Industries swarm |

**GUS product tags:** `Revenue Cloud`, `Revenue Lifecycle Management`, `Industries CPQ`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Product Area: [Asset Lifecycle | Quote to Order | Transaction Mgmt | Configurator | Pricing | Usage Selling | Cotermination]
Operation: [Add | Amend | Renew | Cancel | Swap | Upgrade | Downgrade | Rollback | CSV Import | Other]
Issue Description:
Error message / stack trace:
Reproduced in scratch/demo org?:
Troubleshooting steps taken:
Debug/Splunk logs verified?:
Transaction rollback attempted?:
CML rules checked (if configurator)?:
```
