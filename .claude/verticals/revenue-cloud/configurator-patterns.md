## A. Triage & Classification

### Quick Diagnostic Questions

1. What is the exact error message (copy verbatim)?
2. Does it happen during: Browse Catalog / Configure / Save & Exit / Reprice / Amendment?
3. Is CML (Constraint Model Language / Product Configuration Rules) enabled in Revenue Settings?
4. Is this a new setup or a regression after a change?
5. Is the org using a custom Procedure Plan, or the standard Revenue Management Default Pricing Procedure?
6. Is Digital Insurance or Health Cloud also enabled on the same org?

### Decision Tree

```
Error clicking Configure / Browse Catalog?
├── "ConstraintEngineNodeStatus field hasn't been added…"
│   └── → Pattern 1: ConstraintEngineNodeStatus / CML Setup
├── "Something went wrong while hydrating the context"
│   └── → Pattern 3: Browse Catalog / Context Expiration
├── "Invalid Configuration. Please see Configurator ErrorMessage"
│   └── → Pattern 3 (context expiration) OR Pattern 2 (context definition mismatch)
├── "PRODUCT_DISCOVERY_SERVICE_FAILED" or pricing timeout
│   └── → Pattern 9: Performance / Pricing Timeout
└── Products simply not visible in catalog
    └── → Pattern 3B: Products Not Appearing

CML rule issues?
├── Rule fires but wrong behavior / wrong product added
│   └── → Pattern 5: CML Rules Not Firing Correctly
├── Error not shown on first add, only second
│   └── → Pattern 5B: CML Error Display Timing Bug
└── Error between two bundle products (SourceContextNode)
    └── → Pattern 4: Bundle Configuration Issues

Transaction Line Editor?
├── Inline edits not saving (Order object / STLE)
│   └── → Pattern 6: TLE Inline Editing Issues
├── Quantity updates, pricing display mismatch
│   └── → Pattern 6B: TLE Display / Pricing Issues
└── Date field won't accept typed input
    └── → Pattern 6C: Date Field Picker Required

Amendment / Renewal?
├── "PeriodBoundaryDay is required"
│   └── → Pattern 7A: Amendment PeriodBoundary Error
├── Prices become 0 or wrong price on renewal
│   └── → Pattern 7B: Amendment Pricing Errors
├── "Adjustment type must be set to Override"
│   └── → Pattern 7C: BundleBasedAdjustment Currency Bug
└── STPricingContractAggregationStrategy__std not accessible
    └── → Pattern 7D: Prehook ARC Line Access Error

Deployment / Migration?
├── Cannot deploy ExpressionSetDefinition
│   └── → Pattern 8A: ExpressionSet Deployment Error
├── CurrencyCode mismatch go-live blocker
│   └── → Pattern 8B: Currency Mismatch Deployment
└── ParentQuoteLineItemId read-only (CPQ migration)
    └── → Pattern 8C: CPQ Migration Field Issue

Licensing / Permissions?
├── Revenue Settings error for specific user (not sys admin)
│   └── → Pattern 10: Licensing / Permission Issues
└── Configure gear icon missing for products
    └── → Pattern 11: Gear Icon Missing
```

---

## B. Known Issue Patterns

### Pattern 1: ConstraintEngineNodeStatus / CML Setup Error

**Frequency:** High (36+ cases)
**Severity:** P1–P2
**TTR Impact:** 2–5 days

**Symptoms:**
```
Something went wrong while running configuration rules. : You can't open the 
Product Configurator because the ConstraintEngineNodeStatus field either hasn't 
been added to transaction line item objects or hasn't been mapped to the active 
context definition. Your Salesforce admin can help with that.
```

**Root Cause:**
The Product Configurator (Advanced Constraints mode) checks the active Context Definition's runtime schema for one of two tags before invoking the constraint engine:
- `ConstraintEngineNodeStatus__c`
- `ConstraintEngineNodeStatus`

Source: `BaseSalesTransactionContextConfigurationHandler.java:1015–1017`
```java
if (!contextRuntimeSchema.getTagAttributeMap().containsKey("ConstraintEngineNodeStatus__c")
    && !contextRuntimeSchema.getTagAttributeMap().containsKey("ConstraintEngineNodeStatus")) {
    throw new SalesforceGenericException("...ConstraintEngineNodeStatus field hasn't been added...");
}
```

This fails when:
1. CML was just enabled but the Context Definition was not updated with the field mapping
2. The org uses a **custom Procedure Plan** (e.g., Digital Insurance, Health Cloud) — the Insurance/Health context is used instead of `SalesTransactionContextAuto`, and that context lacks the tag
3. The SalesTransactionContextAuto context has the tag but the active Procedure Plan points to a different Context Definition

**Resolution Steps:**
1. Navigate to Setup → Revenue Settings and confirm "Product Configuration Rules" is enabled
2. Go to Setup → Context Management. Find the Context Definition that is **actively used** by the Procedure Plan (not just `SalesTransactionContextAuto`)
3. Verify `ConstraintEngineNodeStatus` field is present in the TLI node mappings of that Context Definition
4. If missing: add the field mapping on Transaction Line Item objects (Quote Line Item, Order Product, Asset) in that Context Definition
5. Activate the new version of the Context Definition
6. Test by navigating to a Quote → Browse Catalog → click Configure

**Insurance/Health Cloud Conflict:**
If Digital Insurance or Health Cloud is enabled, `InsuranceContextPricingMetadataResolver` takes over context resolution. The `SalesTransactionContextAuto` context has the mapping but the org's Procedure Plan points to an Insurance context. Resolution: either add the ConstraintEngineNodeStatus mapping to the Insurance Context Definition as well, or (if customer only wants RCA) revert the Insurance Procedure Plan configuration entirely.

**Verification:**
```soql
SELECT Id, ErrorCode, ErrorMessage, PrimaryRecord.Name, CreatedDate 
FROM RevenueTransactionErrorLog 
WHERE CreatedDate = LAST_N_DAYS:2 
ORDER BY CreatedDate DESC LIMIT 20
```
Error should no longer appear after fix.

**Escalation Criteria:** Escalate if Context Definition is already correctly configured, error persists after activation, or if this is a compound issue with Insurance/Health Cloud requiring product team guidance.

---

### Pattern 2: Context Definition Mismatch / Derived Price Products

**Frequency:** High (sub-pattern of 71 attribute/context cases)
**Severity:** P1–P2

**Symptoms:**
```
Unable to add Derived Price products to Quote failing with context definition mismatch
```
Or: Products added without error but pricing is wrong; attribute-based pricing not triggered.

**Root Cause:**
The Context Definition used by the Procedure Plan diverges from what the product's pricing step expects. Occurs after:
- Manual updates to Context Definition (activated new version without syncing)
- Deployment to new sandbox that inherited stale context
- Multi-cloud setup where the Procedure Plan was changed

**Resolution Steps:**
1. Run: `SELECT Id, Name, IsActive FROM ContextDefinition` — confirm which CD is active
2. In Revenue Settings → Pricing Setup, verify the Procedure Plan's Context Definition matches the active one
3. Go to Context Management → re-sync the Context Definition
4. If syncing fails, deactivate current version and activate a new one
5. Verify attribute mappings are complete for all product nodes (TLI, Asset Action Source, Order Product)

**Note from Cases:** "for transaction line items it was set up correctly but for asset action source and order product only the standard fields were there" — check all three object nodes, not just TLI.

**Escalation Criteria:** If Context Definition sync is stuck or results in a deployment loop.

---

### Pattern 3: Browse Catalog — Intermittent "Something Went Wrong" / Context Expiration

**Frequency:** High (64 catalog cases, multiple swarms)
**Severity:** P1–P2 when intermittent; P3 for first-load slow

**Symptoms:**
- `Something went wrong while hydrating the context.: We couldn't process your request. Try again.`
- `Invalid Configuration. Please see Configurator ErrorMessage for details.` (with no error code in RTEL)
- Intermittent across user profiles; self-resolves on page refresh
- First load > 10 seconds; subsequent loads fast

**Root Cause (Intermittent):**
Context instance TTL expiration. There are two TTL scopes:
- **Request scope**: recently reduced to ~10 seconds (not configurable by support)
- **Session scope**: tied to Context Definition TTL — configurable within the org, default 10 minutes, **max 45 minutes**

When a pricing procedure version is activated or refreshed mid-session, the context instance becomes stale.

**Slack Insight (Avinash Saklani, #support-rev-rlm-global-swarm-help, Mar 2026):**
> "Context instance gets expired... It is Editable. Max we can have is 45 minutes. Within the Org... If Configurator uses Session scope it will take TTL from CD."

**Resolution Steps:**
1. First: ask customer to page-refresh and retry — most intermittent cases self-resolve
2. Check Context Definition TTL in org: Setup → Context Management → edit active CD → look for Session TTL setting
3. Increase TTL to 45 minutes for the active Context Definition
4. If using custom LWC Browse Catalog: verify the LWC re-fetches context on component init; stale LWC state can cause the same symptom
5. For first-load slowness: enable Product Catalog Management Cache; first load always warms the cache, subsequent loads are fast (by design)

**RTEL Diagnostic Query:**
```soql
SELECT Id, ErrorCode, ErrorMessage, PrimaryRecord.Name, PrimaryRecordId, 
       CreatedBy.Name, CreatedDate 
FROM RevenueTransactionErrorLog 
WHERE CreatedDate = LAST_N_DAYS:2 AND ErrorCode = null
ORDER BY CreatedBy.Profile.Name, CreatedDate DESC
```
Null ErrorCode with "Invalid Configuration" message = context expiration, not a data issue.

**Pattern 3B — Products Not Appearing in Catalog:**
When products are invisible in Browse Catalog despite being set up:
1. Verify: Product Selling Model assigned, active Price Book Entry, Product Category assigned
2. Run: Rebuild Product Index (Revenue Settings → Product Catalog → Rebuild Index)
3. Run: Sync Pricing
4. Check product is Active and not filtered by any Catalog rules

**Escalation Criteria:** Escalate to product team (#rlm-office-hours tag Nilesh Shekhawat or Avinash Saklani) if TTL increase doesn't resolve intermittent errors, or if RTEL shows non-null error codes pointing to infrastructure issues.

---

### Pattern 4: Bundle Configuration Issues

**Frequency:** High (57 cases)
**Severity:** P1–P3

**Sub-patterns:**

**4A — CML Rules Between Two Different Bundles (SourceContextNode Not Supported):**
```
The annotation SourceContextNode isn't supported on the relation <bundle_name>
```
CML cannot write rules that reference across two distinct bundle product models. Each CML model operates on its own context. Workaround: restructure as a single parent bundle with both as children, or use prehook Apex to coordinate cross-bundle logic.

**4B — Prehook Error When Bundle Option Unselected:**
```
Something went wrong while pricing the transaction line item: not updatable state
```
Occurs when a bundle has a CML + prehook and a user unchecks a bundle option and clicks Save and Exit. Root cause: prehook receives an item in a "not updatable" state (already being processed by CML). Workaround: add null check in prehook; skip processing for items with `notUpdatable` state.

**4C — DKK Currency Number Format in Configurator:**
```
Error: Invalid pattern: kr.#,##0.00 there must be a number before the grouping separator
```
Occurs with DKK (Danish Krone) currency when Instant Pricing is enabled for bundle products. Not reproducible with EUR or other currencies. Known bug, fix targeted for r262 (Summer '26). Workaround: disable Instant Pricing for DKK-currency orgs, or use EUR display.

**4D — CurrencyCode Mismatch on Attribute (Go-Live Blocker):**
Known issue `a02Ka00000mFWYL`. Occurs during production deployment when attribute CurrencyCode doesn't match org currency settings. See official known issue page for recommended steps. If steps are followed but issue persists, escalate with reference to that known issue ID.

**4E — Proportional Cancel on Amendments:**
```
Specify null or 0 in Start Quantity of the QuoteLineItem with action type No Change
```
When canceling a component configured with quantity scale = Proportional during a renewal. Set `StartQuantity` to null or 0 on the No Change lines before submitting.

**4F — Unexpected Bundle Product Settings (Price Includes Component, Quantity Scaling):**
If "Price Includes Component" is not checked but products show as price-included, or Quantity Scaling Method = Constant but behaves differently: verify product settings at the bundle option level vs. the bundle header level — there are two places to configure, and a mismatch causes unexpected rendering.

**Escalation Criteria:** 4C (DKK bug) — raise investigation for customer, fix in r262. 4D (currency mismatch) — check known issue page, escalate if workaround doesn't resolve.

---

### Pattern 5: CML Rules Not Firing Correctly

**Frequency:** Medium (16+ cases)
**Severity:** P2–P3

**Sub-patterns:**

**5A — Rule Not Triggering on First Product Add (Only Second):**
CML validation fires asynchronously; on the very first TLE load, constraint evaluation hasn't initialized yet. The rule fires correctly on subsequent interactions. Workaround: use a prehook Apex class that blocks Save if required CML conditions are unmet — this fires synchronously at save time regardless of TLE initialization state.

**5B — CML Error Message Not Surfacing at Top of Browse Catalog UI:**
Rule executes correctly on backend (product deselected as expected) but error banner doesn't appear at top of Browse Catalog interface. Backend constraint is working; this is a UI display gap. Workaround: advise customer the constraint IS enforced; surface the error message via a custom LWC notification component.

**5C — CML Adds Wrong Sibling Product (Annual Subscription Scenario):**
When multiple subscription products (Monthly, Quarterly, Annual) coexist, a require rule targeting "Annual" may fire for siblings. Root cause: rule condition uses a loose match on product name/attribute rather than product ID. Fix: use `this.productId == "<exact_id>"` instead of name-based conditions.

**5D — Fractional Quantities in CML (int vs decimal):**
```
// Bug: int truncates fractional quantity
int qty = this.quantity  // ← wrong for fractional units

// Fix: use decimal
decimal qty = this.quantity
```
Products with fractional Unit of Measure (e.g., 1.5 units) silently truncate when CML uses `int`. Use `decimal` type in CML for all quantity references.

**5E — Duplicate Products Added in Amendment with Constraint Rules:**
When CML constraint rules are used on Root and Child products in ARM, amendment creates duplicate rows. Known issue — log a GUS investigation. Workaround: implement a uniqueness check in the CML model using `context.hasProduct("<productId>")` before adding.

**5F — Auto-Add Quantity Sum Not Working:**
CML `AutoAdd` with quantity aggregation requires correct syntax:
```
type LineItem {
    int qty = this.quantity;
}
// Use sum() function correctly — if aggregation breaks, verify rule syntax in RTEL logs
```
Check RTEL for the exact parsing error if the rule compiles but doesn't execute.

**5G — Constraint Rules for Second-Level Bundle Products Broke After Summer '26:**
Selection rules for second-level bundle products created before Summer '26 (r260) stopped triggering child product selection. Known regression in the Summer '26 release. Log investigation and reference prior cases. Check #rlm-office-hours for product team status.

---

### Pattern 6: Transaction Line Editor (TLE / STLE) Issues

**Frequency:** Medium (23 cases)
**Severity:** P3–P4

**6A — Inline Editing in STLE (Order Object) Not Persisting:**
Changes made via inline edit in the Sales Transaction Line Editor on the Order object don't save. The STLE on Order has limited inline edit support compared to the Quote TLE. Workaround: use the Configure action (gear icon) to edit fields rather than inline edit directly.

**6B — Net Total Mismatch: Simulator vs STLE:**
Simulator shows correct prorated Net Total; STLE shows wrong value. The Simulator recalculates proration dynamically; the STLE displays the stored value until repriced. Fix: click "Reprice All" after opening the STLE to force recalculation.

**6C — Date Field in TLE Only Accepts Date Picker (Not Typed Input):**
TLE date fields require selection via the calendar date picker. Typing formats like `Jan 24, 2026` or `1/24/26` is not accepted. This is known platform behavior for TLE date columns.

**6D — TLE Requires Page Refresh After Quantity Updates:**
Quote totals don't update until page refresh after quantity changes. Workaround: click "Reprice All" instead of waiting for auto-recalculation. Root cause: TLE may not trigger reactive re-rendering on all field updates.

**6E — Bundle Product Options Order in TLE:**
Product options in bundle don't display in the order configured by default. Use `SortOrder` field on the bundle option records to control display sequence. The system auto-assigns 5-digit values; set these manually on option records to reorder.

**6F — Warning on QLI Quantity Edit in TLE:**
Warning "We could not process your request because the asset was updated by another process" on quote lines. Check `RevenueTransactionErrorLog WHERE PrimaryRecordId = '<QuoteId>'`. Often caused by concurrent asset update process. GUS logged for this (doc bug, not product fix expected).

---

### Pattern 7: Amendment / Renewal Errors

**Frequency:** Medium (23 cases)
**Severity:** P1–P3

**7A — PeriodBoundaryDay Is Required Error:**
```
When PeriodBoundary is DayofPeriod, PeriodBoundaryDay is required
```
Occurs when amending an existing asset where `PeriodBoundary = 'Anniversary'` (not DayofPeriod) — the error is misleading. Check the asset's billing schedule: if `PeriodBoundary` was set incorrectly during initial setup, update it. If the field is locked, use the Override Amendment to set the correct value.

**7B — Amendment Prices Becoming 0 (NetUnitPrice / TotalPrice = 0):**
On amendment quotes, `NetUnitPrice` and `TotalPrice` show as 0 after repricing. Root cause confirmed: `Quantity` field on the amendment QLD is reverting to 0 before pricing runs. Verify the Quantity field has the correct value on the Quote Line Detail (QLD) record after the amendment is created.

**7C — "Adjustment Type Must Be Set to Override" with BundleBasedAdjustment:**
```
Your quote was not updated. The adjustment type for this line item must be set to Override.
```
Occurs when `BundleBasedAdjustment` CurrencyIsoCode doesn't match the quote currency. Related to DKK (Danish Krone) currency evaluation bug — same as Pattern 4C. If changing `AdjustmentType` to Override removes the error but applies wrong value (flat amount instead of %), the discount is being misinterpreted. Fix in r262. Workaround: use USD as org currency for testing; for production, raise investigation.

**7D — STPricingContractAggregationStrategy__std Not Accessible in Prehook on Amendment:**
```
Attribute: STPricingContractAggregationStrategy__std not Accessible of boPath: Quote
```
Occurs in custom Apex prehook (`RevSignaling.SignalingApexProcessor`) during amendment quote when prehook tries to access `STPricingContractAggregationStrategy__std` on ARC (Amendment/Renewal/Cancel) lines.

Root cause: ARC lines do not expose this attribute to the pre-hook context. The prehook logic only runs for net-new QLIs, not ARC lines, but the attribute reference still causes an access exception.

Fix: Add null check / ARC line type check in prehook:
```apex
// Before accessing STPricingContractAggregationStrategy__std
if (lineItem.getActionType() == 'Existing') {
    // Skip prehook logic for ARC lines
    return;
}
```

**7E — Renewal Taking New List Price Instead of Contracted Price:**
On renewal, `OrderProduct.UnitPrice` takes the new list price from the Price Table rather than the contracted price from the Contract. Root cause: renewal pricing procedure is hitting the ListPrice pricing step before checking the contracted price. Verify the renewal Pricing Procedure has a "Contracted Price" step that runs before ListPrice; contracted price step requires Contract Line Item data mapped in the context.

**7F — Attribute Picklist Value in Draft During Amendment:**
```
Error on Amendment when Attribute picklist value status is Draft
```
This issue does not exist in Production (running Spring '26 Patch 14.6/260.14.6) but does exist in affected sandboxes. Likely a patch-level fix. Upgrade the sandbox or work around by ensuring all attribute picklist values are Active before creating amendment quotes.

---

### Pattern 8: Deployment / Migration Issues

**Frequency:** Medium (23 cases)
**Severity:** P1–P2 (go-live blockers)

**8A — Cannot Deploy ExpressionSetDefinition to Production:**
Multiple cases. Common causes:
1. ExpressionSet dependencies (Pricing Procedure steps) have unresolvable references in target org
2. Version must be Activated before deployment — if you deploy inactive, it errors
3. "Can't enable the expression set version without steps" — the ES version has no steps or has steps referencing deleted components

Resolution: Ensure all steps are present and referenced components exist in target org. Activate in sandbox first, then deploy via Change Set or Metadata API.

**8B — CurrencyCode Mismatch of Attribute (Go-Live Blocker):**
Known issue `a02Ka00000mFWYL`. When deploying constraint model attributes to production, CurrencyCode on the attribute doesn't match org default currency. Follow the known issue workaround steps. If still blocked, escalate with the known issue ID.

**8C — ParentQuoteLineItemId Read-Only (CPQ → Revenue Cloud Migration):**
When migrating from CPQ, `ParentQuoteLineItemId` is read-only and cannot be set via DML to establish parent-child QLI relationships. Workaround: use the Revenue Cloud migration APIs (Place Sales Transaction) to create the hierarchy rather than direct DML.

**8D — Product Configuration Rules Cannot Be Migrated Between Orgs:**
Product Configuration Rules (CML) are org-specific — same as BRE Context Rules. They cannot be exported/imported between environments. Must be recreated manually. Workaround: document all rules in a spreadsheet; re-create via the CML editor in each target org.

**8E — TransactionProcessingType sObject Not Supported (Revenue Cloud Re-Enable):**
```
INVALID_TYPE: ...sObject type 'TransactionProcessingType' is not supported
```
Occurs when Revenue Cloud was previously disabled on an org and is being re-enabled. The SOQL query for `TransactionProcessingType` fails because the sObject isn't fully re-provisioned yet.

Resolution: Allow 24–48 hours for the org to fully re-initialize Revenue Cloud objects after re-enabling. If persists, raise a BT request to re-provision the org's Revenue Cloud objects. Error surfaces in Splunk at `RevLifecycleMgmtController.getTransactionProcessingTypes`.

---

### Pattern 9: Performance / Pricing Timeout

**Frequency:** Medium (cases + multiple swarms)
**Severity:** P1 when production-down

**Symptoms:**
```
PRODUCT_DISCOVERY_SERVICE_FAILED: Something went wrong while processing your 
configuration changes. Unknown error received from pricing service

Exception in ListPrice custom element: The Product Discovery Pricing Procedure 
pricing procedure has timed out. Try again later.
```

**Code Reference:**
`AbstractPricingKnowledgeModel.executeBusinessKnowledgeModel:662`
→ `PricingGaurdRailServiceImpl.checkProcedureTimeoutGuardRails`
→ Timeout = 60,000ms default

Stack trace pattern:
```
industries.pricing.guardrail.PricingGaurdRailServiceImpl.logGaurdRailFailException
industries.pricing.rulesengine.knowledgemodel.AbstractPricingKnowledgeModel.executeBusinessKnowledgeModel:662
```

**Root Causes:**
1. HBase unavailable — infrastructure issue, not customer-configurable
2. Context metadata stale/unavailable (same as Pattern 3 but manifesting as timeout)
3. Pricing Procedure has too many steps or lookup tables hitting row limits
4. **Decision Matrix used instead of Decision Table** — Decision Matrix is NOT supported by Salesforce Pricing. Error: `Lookup API name: X, but LookUpId is null`. GUS W-20474114 closed as "No Fix - Working as Documented" — must migrate to Decision Table.

**Resolution Steps:**
1. Confirm if error is infrastructure (check Splunk for HBase error in same timeframe): `index=<instanceIndex> <orgId> "hbase"` 
2. If infrastructure, raise production-down P1 with infrastructure team
3. If Decision Matrix: advise customer to migrate to Decision Table (the only supported lookup type in Salesforce Pricing)
4. If complex pricing procedure: review step count and lookup table sizes; split into sub-procedures if > 20 steps

**Intermittent Lookup API null (Reprice All):**
- Error: `Exception in pricing element. Lookup API name: AMLTierDiscountsV1, but LookUpId is null`
- Occurs intermittently during early morning hours PST, self-resolves in 2–4 hours
- Root cause: Decision Matrix (AMLTierDiscountsV1 is a Decision Matrix object, not a Decision Table)
- Fix: migrate to Decision Table. Intermittent behavior = cache eviction cycle on the matrix object

---

### Pattern 10: Licensing / Permission Issues

**Frequency:** Medium (from Slack patterns)
**Severity:** P2–P3

**10A — Revenue Settings Error for Non-Admin Users:**
```
Unfortunately, there was a problem. Please try again. Error ID: XXXXXXXXX
```
When a non-System-Admin user opens Revenue Settings but a System Admin user has no problem.

Common cause: user is missing one or more of:
- Business Rules Engine Designer PSL
- Business Rules Engine Runtime PSL
- Product Discovery User PSL

Check: `SELECT Id, PermissionSetLicense.DeveloperName FROM PermissionSetLicenseAssign WHERE AssigneeId = '<userId>'`

Also from Splunk: `INVALID_TYPE: SELECT Id, Name, ApiName FROM RuleLibrary` — RuleLibrary sObject not accessible to user = BRE Runtime PSL missing.

**10B — Cannot Add "Edit Calculation Status on Quote and Order" to PSG:**
Error when adding this system permission to a Permission Set inside a Permission Set Group.

Workaround (from Sashi Preetham Kotni, #support-rev-rlm-global-swarm-help, Mar 2026):
1. Take backup: `SELECT Id, PermissionSetGroupId, AssigneeId FROM PermissionSetAssignment WHERE PermissionSetGroupId='<PSGId>'`
2. Delete PermissionSetAssignments for the PSG
3. Add "Edit Calculation Status on Quote and Order" to the Permission Set directly (not via PSG)
4. Re-insert the PermissionSetAssignments from backup

**10C — Revenue Cloud + Subscription Management Coexistence:**
Once RCA is enabled, Subscription Management users cannot amend/renew/cancel assets via SM batch jobs. SM batch job (`SubscriptionAutoRenewalController`) throws NPE on legacy data shapes even with `AppUsageAssignment` pointing to SM.

Root cause: "Revenue Cloud takes precedence. Subscription Management users can't amend, renew, or cancel assets." — documented behavior.

Options: (1) test legacy renewals in a sandbox without RCA, or (2) overhaul ERP integration to push data in RCA/RLM shape.

**10D — Gear Icon (Configure) Missing for Products:**
Configure icon absent = product does NOT have a CML model attached, OR the user doesn't have "Product Discovery User" PSL.

Steps:
1. Check user's PSL assignments (Product Discovery User required)
2. Verify the product has a CML Constraint Model attached and active
3. If PSL is correct and CML model exists: check RTEL for backend errors preventing configurator initialization

---

### Pattern 11: Custom Prehook Context Attribute Not Rendering

**Frequency:** Low (4 cases)
**Severity:** P3

**Symptoms:**
Custom pricing step created for "Contract Product Family Discounts" executes correctly in backend (pricing math works), but the custom discount step does not appear in the Price Waterfall UI.

**Root Cause:**
Price Waterfall display requires the pricing step to be registered as a named step with a specific `PriceWaterfallComponentType`. A custom Apex pricing step that returns correct values will NOT automatically appear in the waterfall unless it is registered in the Pricing Procedure as a waterfall-visible step.

**Resolution:**
1. In the Pricing Procedure builder, verify the custom step has `Show in Price Waterfall` enabled (if available)
2. The step must return a `PricingResult` that populates `PricingAdjustmentGroup` records for the waterfall renderer to pick them up
3. If using a Prehook (not a Procedure Step): prehook outputs do NOT appear in Price Waterfall by design; only Procedure Steps do
4. Consider converting the prehook logic to a named Pricing Procedure Step for waterfall visibility

---

## C. Setup & Configuration Guide

### Required Permissions & PSLs

| Role | Required PSLs |
|------|--------------|
| All Revenue Cloud users | Revenue Cloud User (or Revenue Cloud Billing User) |
| Admin configuring CML | Business Rules Engine Designer |
| Users accessing configurator | Business Rules Engine Runtime, Product Discovery User |
| Users editing calculation status | "Edit Calculation Status on Quote and Order" system perm |

### Context Definition Setup Checklist

- [ ] `ConstraintEngineNodeStatus` field mapped in active Context Definition for TLI node
- [ ] Same field mapped for Asset Action Source node (if ARM/amendment)
- [ ] Same field mapped for Order Product node (if order-based flow)
- [ ] Context Definition is Active (not Draft)
- [ ] Context Definition is linked to the Procedure Plan being used (not just the default)
- [ ] TTL configured (recommend 30–45 minutes for complex orgs)

### Enabling CML — Complete Checklist

1. Revenue Settings → enable "Product Configuration Rules" (Advanced Configurator)
2. Verify `ConstraintEngineNodeStatus` field exists on Quote Line Item object — if not, create it (field type: Text, 255)
3. Map field in active Context Definition for all three nodes (TLI, Asset Action Source, Order Product)
4. Activate new Context Definition version
5. Assign Business Rules Engine Runtime PSL to all users who will use the configurator
6. Test in sandbox before enabling in production

### Common Misconfigurations

| Misconfiguration | Symptom | Fix |
|-----------------|---------|-----|
| CML enabled but CD not updated | ConstraintEngineNodeStatus error | Map field in active CD |
| Insurance/Health Cloud Procedure Plan active | ConstraintEngineNodeStatus error on RCA orgs | Add tag to Insurance context CD OR revert Insurance config |
| Decision Matrix in pricing procedure | Intermittent "LookupId is null" | Migrate to Decision Table |
| Context Definition TTL too short | Intermittent "hydrating context" error | Increase TTL to 45 min |
| CML uses `int` for fractional qty | Quantity truncation silently | Use `decimal` type |
| Prehook does not check ARC line type | Amendment fails on Save & Exit | Skip prehook for ARC lines |
| BRE Designer PSL missing | Cannot create/edit CML models | Assign PSL |
| Product Discovery User PSL missing | Configure gear icon absent | Assign PSL |

---

## D. Licensing & Entitlements

**Required PSL Stack for Full Configurator Feature:**
```
Revenue Cloud User PSL (minimum)
├── Browse Catalog, basic product addition
├── Product Discovery User PSL
│   ├── Configure gear icon, attribute rendering
│   └── CML rule evaluation
└── Business Rules Engine Runtime PSL
    └── Full CML constraint model execution
```

**Admin Setup Requires:**
- Business Rules Engine Designer PSL (to create/edit CML models)
- Revenue Cloud Administrator permission set (or equivalent)

**Common PSL Diagnostic:**
```soql
SELECT Id, PermissionSetLicense.DeveloperName, Assignee.Name 
FROM PermissionSetLicenseAssign 
WHERE AssigneeId = '<userId>'
```

**License Flowchart for "Configure Icon Missing":**
1. User has Revenue Cloud User PSL? → No → Assign it
2. User has Product Discovery User PSL? → No → Assign it
3. Product has active CML Constraint Model? → No → Create/attach CML model
4. Still missing? → Check Revenue Settings: "Product Configuration Rules" enabled?
5. Still missing? → Check RTEL for backend error during configurator initialization

---

## E. Documentation Quick-Reference

- CML Language Reference: Revenue Cloud Developer Guide → Constraint Model Language
- Context Definition Setup: Revenue Cloud Admin Guide → Set Up Context Definitions
- Browse Catalog Setup: Revenue Cloud Admin Guide → Set Up Product Discovery
- Procedure Plan Setup: Revenue Cloud Admin Guide → Set Up Procedure Plans
- Known Issue (CurrencyCode Mismatch): `a02Ka00000mFWYL`
- GUS W-20474114: Decision Matrix not supported in Salesforce Pricing (Closed - No Fix)

**Key Behavioral Rules / Design Decisions:**
1. CML models are org-specific — they cannot be migrated between environments
2. Decision Matrix ≠ Decision Table: only Decision Table is supported in Salesforce Pricing
3. Prehook outputs do NOT appear in Price Waterfall; only Procedure Steps do
4. Once RCA is enabled, SM batch jobs cannot process legacy-shape assets (coexistence design)
5. ProductSellingModelOption is a "behind the scenes" object — no custom fields, flows, or triggers allowed
6. Context Definition TTL defaults differ by scope: Request scope ~10s (not configurable), Session scope configurable up to 45 min

---

## F. Escalation Paths

### When to Escalate

| Situation | Action |
|-----------|--------|
| Production-down pricing timeout (HBase/context infrastructure) | P1 escalation to product team + infrastructure tag |
| ConstraintEngineNodeStatus error persists after correct CD setup | Escalate to #rlm-office-hours, tag Vinod Kumar Lakkakula |
| CurrencyCode mismatch at go-live after following known issue workaround | Escalate with known issue ID a02Ka00000mFWYL |
| DKK currency format bug | Log investigation, reference r262 fix timeline |
| CML regression after Summer '26 release (second-level bundle rules) | Log investigation, reference prior cases |
| Insurance + RCA compound context resolution issue | Escalate to #rlm-office-hours |

### Information to Gather Before Escalating

1. Org ID, sandbox/production
2. Exact error message from RTEL and/or Splunk
3. `RTEL query results`: `SELECT Id, ErrorCode, ErrorMessage, PrimaryRecordId FROM RevenueTransactionErrorLog WHERE CreatedDate = LAST_N_DAYS:1`
4. Active Context Definition name + version
5. Procedure Plan name and type
6. Whether Digital Insurance / Health Cloud is also enabled
7. Revenue Settings screenshot (which features enabled)
8. CML model name and version
9. Splunk log ID from error (visible in Splunk search via org ID + error string)

### Expert Contacts

- **#rlm-office-hours**: Vinod Kumar Lakkakula (pricing/context deep dives), Harsh Deep (escalation coordination), Avinash Saklani (CD/TTL issues), Frank Neumann (infrastructure/HBase)
- **#support-rev-rlm-global-swarm-help**: Prasad Rao (general configurator), Sashi Preetham Kotni (permissions/PSL issues), Vrushabh Mahendrakar (amendment/pricing), Bhuvaneswari Betala (known issues tracking)

---

## G. Tribal Knowledge & Pro Tips

1. **Context TTL is your first tool for intermittent Browse Catalog errors.** Default TTL may be 10 minutes — increase to 45 min before escalating. (Avinash Saklani, #support-rev-rlm-global-swarm-help, Mar 2026)

2. **RTEL with null ErrorCode = context expiration, not data.** When `ErrorCode = null` and `ErrorMessage = "Invalid Configuration..."`, it means the context instance expired mid-session, not a product data issue. Page refresh resolves it. (Prashant Bhutani, swarm, Mar 2026)

3. **Decision Matrix will always eventually fail Salesforce Pricing.** If a customer is using a legacy Decision Matrix in their pricing procedure, it will intermittently produce `LookUpId is null` errors, especially during release weekends when caches reset. Migration to Decision Table is mandatory. GUS W-20474114 closed no-fix. (Dishant J, swarm, Mar 2026)

4. **ConstraintEngineNodeStatus check: two tag names, one check.** The code at `BaseSalesTransactionContextConfigurationHandler.java:1015` checks for BOTH `ConstraintEngineNodeStatus__c` AND `ConstraintEngineNodeStatus` (without `__c`). Either will satisfy the check. (Vinod Kumar Lakkakula, swarm analysis, May 2026)

5. **Insurance context hijacks RCA pricing.** If `Digital Insurance` or `Health Cloud` is enabled alongside RCA, `InsuranceContextPricingMetadataResolver` takes over context resolution. The standard `SalesTransactionContextAuto` context is bypassed entirely. Customer must either add ConstraintEngineNodeStatus to the Insurance context, or cleanly separate their Insurance/Health config from RCA. (Vinod Kumar Lakkakula, swarm code trace, May 2026)

6. **STPricingContractAggregationStrategy__std is not accessible to prehooks on ARC lines.** Amendment/Renewal/Cancel lines do not expose this pricing attribute to pre-hook context. Prehook must check line action type before accessing ARC-restricted attributes. (Rahul Gupta/Prasad Rao swarm, May 2026)

7. **CML integer truncates fractional quantities.** If a product allows fractional UOM (e.g., 1.5 units), CML `int qty = this.quantity` silently truncates. Use `decimal qty = this.quantity`. (LiveRamp case, #rlm-office-hours discussion)

8. **Proportional cancel on amendments: set StartQuantity to null, not 0.** Error "Specify null or 0 in Start Quantity" — null works but 0 may not in all cases; use null to be safe. (Mimecast swarm)

9. **CML between two bundles requires SourceContextNode — which isn't supported.** Trying to write cross-bundle rules produces "SourceContextNode annotation isn't supported on the relation." Workaround: consolidate both bundles under a single parent, or use prehook Apex.

10. **Revenue Cloud + Subscription Management = SM batch jobs stop working.** Once RCA is on, the `SubscriptionAutoRenewalController` batch job crashes with NPE on legacy-data assets even if `AppUsageAssignment` explicitly points to SM. There is no bypass — platform design. (Isaac David Sánchez, swarm, Apr 2026)

11. **ProductSellingModelOption blocks customization.** Customers cannot add custom fields, flows, or triggers to this object. It is "behind the scenes" by design. Escalate to #rlm-office-hours for any customization request. (Kushagr Jaiswal, swarm, Mar 2026)

12. **Save & Exit success toast may not appear — but product IS added.** Missing success notification is a UI cosmetic issue. Product is added to the quote in the backend. Inform customer and document as known UI gap. (Subhodip Choudhury, swarm, Apr 2026)

13. **DKK currency bug: "Invalid pattern: kr.#,##0.00" — fix in r262.** Only affects Danish Krone with bundle Instant Pricing. Workaround: disable Instant Pricing for DKK orgs. (Beatriz Torres Alatriste / Bhuvaneswari Betala, swarm, Mar 2026)

14. **Revenue Settings error ≠ product error.** "Unfortunately there was a problem" at Revenue Settings for non-admin users is almost always a missing PSL: Business Rules Engine Designer, BRE Runtime, or Product Discovery User. Check `PermissionSetLicenseAssign` first. (Sanjana Kolu, swarm, May 2026)

15. **Re-enabling Revenue Cloud requires 24–48 hours.** Orgs that disabled and re-enabled Revenue Cloud will hit `TransactionProcessingType not supported` errors during the re-provisioning window. Do not escalate immediately — allow the platform time to reprovision.

16. **"Edit Calculation Status" perm can't be added to PSG directly.** Workaround: temporarily remove all PSG assignments, add the system perm to the individual Permission Set, then restore assignments. (Sashi Preetham Kotni, swarm, Mar 2026)

17. **Browse Catalog product sort is by Name — no native resequencing.** Products appear sorted by Name within a category. No standard sort order field is respected on Browse Catalog. Workaround: prefix product names for custom ordering, or build a custom LWC that applies `SortOrder`-based ordering.

18. **Pricing Analytics App requires Data Cloud data streams.** Installation fails with `ssot__SalesOrder__dlm not found` if required Data Cloud streams aren't deployed. Follow Revenue Management Intelligence App setup fully including all Datakit installation steps before attempting app installation. Known bug logged (per Bhuvaneswari Betala, May 2026).

19. **Context Definition sync issues after sandbox refresh.** After a sandbox refresh, active Context Definitions may show as Active but be out of sync. Re-sync all Context Definitions before testing configurator in a freshly refreshed sandbox.

20. **Prehook "not updatable state" = CML already processing.** If prehook fires and returns "not updatable state" on a bundle option uncheck, CML has already locked that line item. Add a state check in the prehook before attempting updates.

---

## H. Active Known Issues

| GUS / Known Issue ID | Summary | Status |
|---------------------|---------|--------|
| W-20474114 | Decision Matrix not supported in Salesforce Pricing (LookUpId null) | Closed - No Fix - Working as Documented |
| a02Ka00000mFWYL | CurrencyCode Mismatch of Attribute on deployment | Active known issue with workaround |
| DKK Currency Bug | Invalid pattern kr.#,##0.00 for DKK in configurator Instant Pricing | Fix targeted r262 (Summer '26) |
| CML Amendment Duplicate Rows | Amendment CML constraint rules duplicate products | Raise investigation per case |
| Second-Level Bundle Rules Post Summer '26 | Constraint rules for 2nd-level bundle products stopped triggering | Raise investigation |
| Pricing Analytics App Install Failure | ssot_EndDateTime_c/ssot_StartDateTime_c not found in DMO | Known bug, Bhuvaneswari Betala tracking |

---

## I. Code Reference Map

| Issue | Code Path | Key Location |
|-------|-----------|-------------|
| ConstraintEngineNodeStatus check | `BaseSalesTransactionContextConfigurationHandler.java:1015–1017` | checks tagAttributeMap for ConstraintEngineNodeStatus (with and without __c) |
| Pricing procedure timeout | `AbstractPricingKnowledgeModel.executeBusinessKnowledgeModel:662` → `PricingGaurdRailServiceImpl.checkProcedureTimeoutGuardRails` | Guard rail at 60,000ms |
| Context resolution (PlaceSalesTransaction) | `PlaceSalesTransactionPostableResource.post()` → `StandardPlaceSalesTransaction.resolveContext()` → `ContextResolverImpl.buildNewContext()` | Insurance conflict: `InsuranceContextPricingMetadataResolver.getContextBusinessMapping():131` |
| Intermittent pricing error | `PlaceSalesTransactionPricingStep.java:123` | Fires when Decision Matrix LookupId is null |
| Configurator rules execution | `LegacyConfigurationStep.execute()` → `BaseSalesTransactionContextConfigurationHandler.runConfiguratorRules()` → `executeAdvancedConstraints()` | Throws on missing ConstraintEngineNodeStatus |
| Revenue Settings failure | `RevLifecycleMgmtController.getTransactionProcessingTypes` | `common.api.ApiNameUddX.getEntityInfo` fails when TransactionProcessingType not provisioned |
