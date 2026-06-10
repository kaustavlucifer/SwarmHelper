## Case Volume Statistics (Last 365 Days — OrgCS)

**Total closed cases for "Revenue-Salesforce CPQ": 9,023 | Avg TTR: 8.06 days**

### By Feature Area (cases with confirmed issues)

| Feature Area | Cases | Avg TTR (days) |
|---|---|---|
| Line Editor UI/UX | 1,700 | 6.94 |
| Amendments and Renewals | 1,237 | 8.82 |
| Pricing | 960 | 8.28 |
| Contracts | 890 | 9.85 |
| Advanced Approvals | 704 | 9.78 |
| Licensing, Provisioning, Permissions | 663 | 6.50 |
| Apex | 640 | 10.08 |
| Product Selection / Bundle Configuration | 615 | 6.74 |
| Orders / Order Products | 564 | 8.36 |
| Installation / Upgrade / Authorization | 357 | 7.92 |
| Proration | 17 | 11.90 |

### By Case Cause

| Cause | Sub-Cause | Cases | Avg TTR |
|---|---|---|---|
| Documentation / Training | User needs instructions or training | 4,756 | 6.6 days |
| Product/Infrastructure | Error message issue | 381 | 8.9 days |
| Documentation / Training | Need better documentation | 371 | 7.3 days |
| Product/Infrastructure | Feature limit / Feature activation | 111 | 10.2 days |
| Product/Infrastructure | **Product Bug** | **86** | **13.4 days** |
| Out of scope | Other / Implementation request | 278 | 11.2 days |

> **Note:** Product Bugs (avg 13.4 days TTR) take 2× longer to resolve than documentation cases (avg 6.6 days). Contracts feature area has the highest combined TTR (9.85 days).

---

## Prerequisites & Tool Access

| Tool | Purpose | If not connected |
|------|---------|-----------------|
| CodeSearch | Trace SBQQ class/trigger root causes | Mark code references as NEEDS CODE REFERENCES |
| GUS (SF CLI) | Look up bug records, investigations, known issues | Mark as NEEDS GUS DATA |
| Slack MCP | Search live tribal knowledge in CPQ channels | Mark as NEEDS SLACK INSIGHTS |
| Google Workspace | Pull case data spreadsheets | Ask user for spreadsheet URL |

### Quick Diagnostic Questions

Ask these first before diving into patterns:

1. What is the **exact error message**? (copy-paste, not paraphrase)
2. What **org type**? (Production / Sandbox)
3. What **CPQ package version**? (Setup → Installed Packages → Salesforce CPQ)
4. Is this happening for **all users** or specific users?
5. Is this **reproducible** in a sandbox with no QCP/custom triggers?
6. **When did it start?** After a release, config change, or sandbox refresh?
7. Is a **QCP (Quote Calculator Plugin)** active in the org?

### Decision Tree

> Branches ordered by case volume (OrgCS, last 365 days).

```
Customer reports CPQ issue
│
├── [#1 Volume: 1,700 cases] Error in Quote Line Editor (QLE)?
│   ├── "Cannot read properties of null (reading 'lineVO')"  → Pattern 1
│   ├── "Unexpected end of input"                            → Pattern 2 (Price Rule formula)
│   ├── "You may not specify additional discount as both..."  → Pattern 3
│   ├── "No metadata was retrieved for field SBQQ__..."      → Pattern 4 (orphaned Price Rule)
│   ├── Product selection checkbox unclickable (Console App) → Pattern 5 (Summer '26 Bug)
│   ├── Custom actions disappear after adding product        → Pattern 6
│   └── Apex heap size / SOQL limit error                    → Pattern 7
│
├── [#2 Volume: 1,237 cases] Amendment / Renewal issue?
│   ├── "Add a subscription pricing value to all products..."→ Pattern 8
│   ├── Short-term product not co-terming correctly          → Pattern 9 (WAD/Config)
│   ├── Backdated amendment NullPointerException             → Pattern 10
│   ├── Invalid cross reference id on renewal opportunity    → Pattern 11
│   └── Renewal Opportunity not generated / missing products → Pattern 8 + check Renewal Forecast field
│
├── [#3 Volume: 960 cases] Pricing / Calculation issue?
│   ├── Price rule not firing on first save                  → Pattern 2 / check eval event
│   ├── "MDQ segments missing or quantity 0 during amendment"→ Pattern 2 (MDQ + pricing rules)
│   ├── Summary variable not calculating on renewal          → Pattern 15
│   ├── Formula field incorrect in QLE but correct in DB     → Pattern 2 / calculation sequence
│   ├── Quotes Showing $0                                    → Pattern 2 / check Price Rule + Pricebook
│   └── Calculation error: Internal Server Error (async)     → Check QCP, Heroku callout
│
├── [#4 Volume: 890 cases] Order Contracting failure?
│   ├── "Attempt to de-reference a null object"              → Pattern 12
│   ├── "bad value for restricted picklist field"             → Pattern 13
│   ├── UNABLE_TO_LOCK_ROW on Quote during Flow              → Pattern 14
│   ├── "Contracted = true" but no Contract created          → Check Order status, SBQQ settings
│   └── Order activated but contract not generated           → Pattern 12 / check Opportunity Pricebook
│
├── [#5 Volume: 704 cases] Advanced Approvals issue?
│   ├── Approval not routing to correct approver             → Pattern 19
│   ├── Approval status stuck / not advancing                → Pattern 19
│   ├── "Modify All" bypassing approval                      → Pattern 19 (WAD)
│   └── Approval emails not sending                          → Pattern 19
│
├── [#6 Volume: 663 cases] License / PSL / Permission issue?
│   ├── "Read on Quote can't be granted"                     → Pattern 16
│   ├── Licensed Objects error when editing Profile          → Pattern 17
│   ├── CPQ License Expired cases                            → IR "CPQ License" → #C020Y1K42F8
│   └── Calculation service authorization "Oh no" error     → Pattern 18
│
└── Scope / routing question?
    ├── PROS Smart CPQ / 3rd-party CPQ                        → Out of scope, direct to vendor
    ├── Conga Doc Gen (CQG) license active                    → Still in scope until license expires
    ├── RCA / RCB (Revenue Cloud Advanced)                    → Route to #C05TFMXB3RN
    └── Salesforce Billing                                    → Route to #C026Y0ENETH
```

---

## B. Known Issue Patterns

### Pattern 1: "Cannot read properties of null (reading 'lineVO')"

**Frequency:** High — QLE/UI is the #1 case area (1,700 cases/year, avg TTR 6.94 days)
**Severity:** Typically Sev 2–3  
**Area:** QLE / Quote calculation

**Real Case Examples (OrgCS):**
- "Edit Lines not functioning on quote" (17.9 days TTR)
- "Duplicate ID in list error in Quote Line Editor" (16.3 days TTR)
- "sbqq__JSQC script exception in California location" (12.8 days TTR)
- "Users In QA Sandbox Blocked by Invisible Sidebar" (14 days TTR)

**Symptoms:**
- Error: `Salesforce CPQ Error: Cannot read properties of null (reading 'lineVO')`
- Occurs during quote save or renewal processing (often async)

**Root Cause:**
QCP (Quote Calculator Plugin) is causing the issue — a null reference in the line model during async calculation.

**Resolution Steps:**
1. Capture the CPQ managed package debug log (not just standard Apex log)
2. Check if a QCP is active: `SELECT Id, Name FROM SBQQ__CustomScript__c WHERE SBQQ__Active__c = true`
3. If QCP exists, try removing it temporarily and retesting
4. If async: enable **Legacy Amend/Renew Service** temporarily to process synchronously and capture full stack trace
5. Reference KA: https://help.salesforce.com/s/articleView?id=004637113&type=1

**Verification:** Reproduce without QCP — if issue disappears, QCP custom code is root cause (dev support case).

**Workaround (from #support-rev-config-amer):** Switch to synchronous processing via Legacy Service to isolate whether QCP async is the cause.

**Escalation Criteria:** If reproducible with no QCP and no custom automation → raise GUS investigation.

---

### Pattern 2: "Unexpected end of input" / Price Rule Formula Error in QLE

**Frequency:** High — Pricing is the #3 case area (960 cases/year, avg TTR 8.28 days); error/bug sub-cause avg 8.9–39.4 days TTR
**Severity:** Sev 2–3  
**Area:** Price Rules / QLE

**Real Case Examples (OrgCS):**
- "CPQ Price rule not working" (39.4 days TTR)
- "Error: MDQ segments missing or quantity 0 during amendment" (36 days TTR)
- "When Cloning a Quote, it gets created but not auto-calculated" (35 days TTR)
- "Price rule not triggered in sandbox" (10 days TTR)
- "CPQ Quotes Showing $0 — Emergency" (6.1 days TTR)

**Symptoms:**
- Error `Unexpected end of input` when adding products in QLE
- Specific products fail to add; others work fine
- Price rule with formula receives unexpected data

**Root Cause:**
A Price Rule formula is incorrectly formatted or receives null/unexpected data during calculation. Often a single active rule is the culprit among many.

**Resolution Steps:**
1. Query all active price rules: `SELECT Id, Name FROM SBQQ__PriceRule__c WHERE SBQQ__Active__c = true`
2. Binary search — deactivate half, test; narrow down to the single offending rule
3. Check Price Action formula for null-safety (missing ISBLANK checks, dividing by zero, etc.)
4. Check if adding an **Error Condition** to fire the rule only when `COUNT of Quote Lines > 0` resolves it (fixes false-trigger on empty QLE)

**Verification:** Deactivating the offending rule resolves the error immediately.

**Workaround:** Deactivate the problematic rule while fixing the formula.

**Tribal Knowledge (#support-rev-config-emeapac):** Resolution case reference pattern — narrow from 109 active rules using binary deactivation. Add error condition `COUNT(Quote Lines) > 0` to prevent rules firing on empty QLE state.

---

### Pattern 3: "You may not specify additional discount as both percentage and absolute amount"

**Frequency:** Medium  
**Severity:** Sev 3  
**Area:** Price Rules / Discount

**Symptoms:**
- Error on QLE save: `You may not specify additional discount as both percentage and absolute amount`
- Triggered by Price Rule updating `SBQQ__AdditionalDiscountAmount__c`
- `SBQQ__Discount__c` not explicitly set in any rule

**Root Cause:**
CPQ validates that discount fields are mutually exclusive. If a Price Rule sets `SBQQ__AdditionalDiscountAmount__c` and simultaneously the `SBQQ__Discount__c` (percentage) field has a value (from user input or another rule), CPQ throws this error.

**Resolution Steps:**
1. Identify which Price Rule updates `SBQQ__AdditionalDiscountAmount__c`: `SELECT Id FROM SBQQ__PriceAction__c WHERE SBQQ__Field__c = 'SBQQ__AdditionalDiscountAmount__c'`
2. Ensure `SBQQ__Discount__c` is nulled out in the same rule execution if using amount-based discount
3. Or restructure logic so only one discount type is used at a time

**Verification:** Deactivating the Price Rule stops the error.

---

### Pattern 4: "No metadata was retrieved for field SBQQ__QuoteLine__c.<CustomField__c>"

**Frequency:** Medium  
**Severity:** Sev 2–3  
**Area:** Price Rules / Referenced Fields

**Symptoms:**
- Error when adding products in QLE: `No metadata was retrieved for field SBQQ__QuoteLine__c.<FieldName>`
- Field no longer exists or is not active

**Root Cause:**
A deleted or renamed custom field is still referenced in an active Price Rule (as a formula field, Price Action target, or Price Condition). The Referenced Field metadata record still points to it.

**Resolution Steps:**
1. Query referenced fields: `SELECT Id, Name, SBQQ__FieldName__c, SBQQ__ObjectName__c FROM SBQQ__ReferencedField__c WHERE SBQQ__FieldName__c = '<FieldName>'`
2. Also check: `SELECT Id FROM SBQQ__CalculatorReferencedField__c WHERE SBQQ__FieldName__c = '<FieldName>'`
3. Check active Price Rule conditions and formulas for the field reference
4. Remove or update the stale reference

**Verification:** Adding products in QLE completes without error.

---

### Pattern 5: Product Selection Checkboxes Unclickable in QLE (Summer '26 Bug)

**Frequency:** High (post Summer '26 release) — Product Selection/Bundle Config area: 615 cases/year, avg TTR 6.74 days
**Severity:** Sev 2  
**Area:** QLE Product Selection / UI Bug  
**Known Issue:** https://help.salesforce.com/s/issue?id=a02Ka00000mGGGF  
**GUS:** W-22449442, W-22488848

**Symptoms:**
- Cannot click/select product checkboxes on QLE Product Selection page
- Occurs specifically in **Console Apps where Utility Bar is NOT enabled**
- A UI element (`navexSplitViewWrapper`) overlaps the product selection area

**Root Cause:**
Post Summer '26 release bug — residual UI element renders on the left side of the screen in Console Apps without Utility Bar, blocking the product selection checkboxes.

**Resolution Steps:**
1. Confirm org is Console App without Utility Bar enabled
2. Apply workaround immediately
3. Attach case to Known Issue W-22449442 and W-22488848

**Workaround (#support-rev-technical):** Enable **Split View mode** in the Console App. This correctly allocates width for the split view toggle button and prevents the overlay.

**Escalation:** Map incoming cases to the KI. No fix ETA — track the Known Issue.

---

### Pattern 6: Custom Actions Disappear After Adding Product in QLE

**Frequency:** Medium  
**Severity:** Sev 2  
**Area:** QLE / Custom Actions  
**GUS Bug:** W-SBQQ (Quote Line Group Actions dropdown bug)

**Symptoms:**
- Custom actions (e.g., "Add Products") disappear when a product is added in QLE
- Reappear after clicking Save or reloading
- Works in one org (QA) but not another (Staging/Production)

**Root Cause:**
Known CPQ bug: **Quote Line Group Actions dropdown does not render if an action first in display order has conditions to display that aren't met.** Also triggered by QCP changes or product/pricing rule interactions.

**Resolution Steps:**
1. Check Custom Action display orders: look for duplicate display order values or an action first in display order with unsatisfied conditions
2. Check if any custom action with the lowest display order has **display conditions that are not currently met** — this blocks all subsequent actions from rendering
3. Fix: Change the display order of the problematic action so it is not first, OR remove its condition, OR give it a higher display order number
4. Also check QCP changes between working and non-working orgs

**Verification:** Custom actions remain visible after adding products without requiring a save.

**Tribal Knowledge (#support-rev-config-emeapac):** In one case, a 'Quick Quote Products' action had display order 899 (before 'Add Products' at 900) with unmet conditions — deactivating it resolved the issue immediately.

---

### Pattern 7: Apex Heap Size Too Large in CPQ

**Frequency:** High — Apex is the #7 case area (640 cases/year, avg TTR 10.08 days); highest TTR among non-bug patterns
**Severity:** Sev 2–3  
**Area:** Performance / Large Quotes / QCP

**Real Case Examples (OrgCS):**
- "CPQ - Error TOO MANY SOQL Query during Contract creation" (108.6 days TTR)
- "Too Many SOQL Queries: 101 Error in Salesforce CPQ (v256.0)" (21.9 days TTR)
- "System.LimitException: Apex heap size too large error in CPQ package trigger when finalizing order" (25.2 days TTR)
- "Apex CPU time limit exceeded in CPQ" (18 days TTR)
- "CPU time limit exceeded during approval process in Advanced Approvals" (26.1 days TTR)
- "Error: Uncommitted Work Pending in CPQ_ApiService.addProductsToQuote" (13 days TTR)  

**Symptoms:**
- `Apex heap size too large` error on Quick Save in QLE
- Or during Order contracting / batch processing
- May only appear when developer console is open or debug logs are active (debug logging increases heap usage)

**Root Cause (multiple):**
- Too many quote lines with QCP enabled
- QCP loading large fields (e.g., Long Text Area fields) that aren't needed
- External image processing during document generation (images from S3 buckets)
- Large batch processes building too many Quote/QuoteLine objects in memory

**Resolution Steps:**
1. Check if error correlates with debug logs being active — if so, heap inflation from logging is a factor
2. Check QCP Quote Line Fields section — remove any large/unused fields (especially Long Text Area fields)
3. Enable **Large Quote Performance Settings**: https://help.salesforce.com/s/articleView?id=sales.cpq_large_quote_performance.htm&type=5
4. For batch processes: check `transient` keyword usage — https://help.salesforce.com/s/articleView?id=000385712&type=1
5. For document generation: move images to Salesforce Files (internal) rather than external S3 bucket to reduce download overhead
6. Enable **'Enable Large Configurations'** setting in CPQ Package Settings

**GUS Investigation (#support-rev-dev-emeapac):** https://gus.lightning.force.com/lightning/r/ADM_Work__c/a07EE00001bSvm1YAC/view (contracting orders with <500 order products limit)

**Tribal Knowledge (#support-rev-dev-amer):** From Splunk logs — `CXO_CPQAPIModels.QuoteSaver.save` class is often in the stack trace. Heap issue with specific fields in QCP that are unexpectedly large.

---

### Pattern 8: "Add a subscription pricing value to all products associated with your subscriptions" on Amend

**Frequency:** High — Amendments and Renewals is the #2 case area (1,237 cases/year, avg TTR 8.82 days); product bugs in this area avg 13.4 days
**Severity:** Sev 2  
**Area:** Amendment / Contract / Subscription Pricing

**Real Case Examples (OrgCS):**
- "Decision on problem: 'Amend Contract' from Opportunity in Console pulling in previous Amendment" (114.7 days TTR)
- "Subscriptions billed again in amendment even though already contracted in earlier amendment" (50 days TTR)
- "Issue with Amendment Cancellation Behavior for Bundle Products" (37.2 days TTR)
- "Error on creating and Amending Proposals via Conga CPQ (Spring '26 regression)" (9 days TTR)
- "Renewal Opportunity Missing Recurring Products and Discounts" (9.2 days TTR)
- "Dates are different on renewal quote lines compared to previous contract end date" (25.7 days TTR)

**Symptoms:**
- Error: `Add a subscription pricing value to all products associated with your subscriptions and try again` when clicking Amend on a Contract
- Backfilling `SBQQ__SubscriptionPricing__c` and `SBQQ__SubscriptionType__c` on QLIs does not resolve it

**Root Cause:**
A subscription exists for a product bundle where subscriptions are missing for component products. The managed package logic requires all products in the amendment to have subscription pricing values.

**Resolution Steps:**
1. Query subscriptions for the contract: `SELECT Id, SBQQ__Product__r.Name, SBQQ__SubscriptionPricing__c FROM SBQQ__Subscription__c WHERE SBQQ__Contract__c = '<ContractId>'`
2. Identify products missing `SBQQ__SubscriptionPricing__c` (null or empty)
3. For bundle products: ensure all bundle components have subscriptions created
4. Backfill missing subscriptions or update `SBQQ__SubscriptionPricing__c` = 'Fixed Price' on missing records
5. If a subscription was manually created (e.g., "A La Carte Core Fulfillment Products" bundle), verify all child product subscriptions also exist

**Verification:** Clicking Amend proceeds without error.

---

### Pattern 9: Short-Term / Non-Co-Term Product in Amendment (Working as Designed)

**Frequency:** Medium  
**Severity:** Sev 3  
**Area:** Amendments / Subscription Term

**Symptoms:**
- CPQ enforces co-terming even with `Disable Amendment Co-Term` checkbox enabled
- Null errors or product not adding correctly in amendment
- Customer wants to add a short-term product (e.g., 3 months) that ends before contract end date

**Root Cause / Design Behavior:**
CPQ by design requires amendment quote lines to co-term with the parent contract. `Disable Amendment Co-Term` affects specific scenarios. If the product is a **POT (Percent of Total)** or excluded by package settings, additional restrictions apply.

**Resolution:**
1. Confirm if product is POT type — POT products have restrictions in amendment co-term scenarios
2. Check CPQ Package Settings for subscription amendment settings
3. If truly non-co-term is needed: **recommended approach is a separate quote/contract** for the short-term product
4. Define end date at the **Quote Line level** (not just quote level) as an alternative

**Tribal Knowledge (#support-rev-technical — Bart Lis):** "Log a swarm — this seems config related and we'd want to see the records. Seems more likely the product is POT or something and the package settings exclude it."

---

### Pattern 10: NullPointerException on Backdated Amendment Contracting

**Frequency:** Medium — Contracts is #4 case area (890 cases/year, avg TTR 9.85 days); highest non-Apex TTR
**Severity:** Sev 1–2  
**Area:** Order Contracting / Amendment Sequence

**Real Case Examples (OrgCS):**
- "Getting error while checking the Contracted Checkbox on the Order" (19.9 days TTR)
- "Amend-Cancel Contract creation failing in DEV" (18.9 days TTR)
- "Service contract creation issue" (19.1 days TTR)
- "Downsell on Amendment Quote Creates New Subscription Line Instead of Updating Existing Line" (11.8 days TTR)
- "Order Activated but Contract Not Generated in Salesforce CPQ" (9.1 days TTR)

**Symptoms:**
- `System.NullPointerException` at `Class.SBQQ.OrderItemObjectManager.validateDateChanges`
- Occurs when contracting a backdated amendment order when a later amendment was already contracted
- First amendment (e.g., 5/13) contracted successfully; second backdated amendment (e.g., 5/5) fails

**Root Cause:**
CPQ requires amendments to be contracted in **strictly chronological order**. When a later amendment sets `SBQQ__TerminatedDate__c` on the original Order Product, contracting an earlier backdated amendment hits an unhandled date overlap scenario.

**Resolution Options (from #support-rev-dev-amer — Shihai Chen):**

**Option 1 (Best Practice):** Contract in chronological order
- Revert/delete the later amendment
- Contract the backdated amendment first
- Re-create and contract the later amendment

**Option 2 (Data Manipulation):**
- Temporarily clear `SBQQ__TerminatedDate__c` on the original Order Product (set to null)
- Contract the backdated order
- Review billing/revenue schedules after — date change may affect recognized revenue

**Option 3 (Break Amendment Linkage):**
- Clear `SBQQ__RevisedOrderProduct__c` on the backdated order's order products
- Treats as net-new rather than amendment (skips termination logic entirely)

**Escalation:** If none of these options are viable, GUS investigation needed.

---

### Pattern 11: "Error creating Renewal Opportunity: invalid cross reference id"

**Frequency:** Medium  
**Severity:** Sev 3  
**Area:** Renewal / Opportunity Creation

**Symptoms:**
- Error: `Error creating Opportunity: invalid cross reference id` when setting Renewal Forecast to true on a Contract

**Root Cause:**
A **Flow** on the Opportunity object is assigning an Owner ID that doesn't exist in the org (e.g., a user ID from a different sandbox or a deleted user). CPQ's renewal process creates the Opportunity and the Flow fires, hitting the invalid ID.

**Resolution Steps:**
1. Review Flows on Opportunity: check for any flow that updates Owner ID, Record Type, or other lookup fields
2. Query flow executions for the relevant timestamp in debug logs
3. In the Flow, find the hardcoded/invalid user ID assignment and replace with a valid ID or dynamic lookup
4. Common: after sandbox refresh, user IDs from production are invalid in the sandbox

**Tribal Knowledge (#support-rev-dev-amer — Jhansi Chodisetti):** "In this flow, they are trying to assign the Owner ID of the opportunity to an invalid user ID... Please ask them to check this point and assign an appropriate ID to the Owner ID field."

---

### Pattern 12: "Attempt to de-reference a null object" on Order Contracting

**Frequency:** High — Orders/Order Products is #9 case area (564 cases/year, avg TTR 8.36 days)
**Severity:** Sev 1–2  
**Area:** Order Contracting

**Real Case Examples (OrgCS):**
- "Error when clicking Edit Order Products on a sales order" (41 days TTR)
- "Mismatch in Order Product Dates in Salesforce CPQ" (19 days TTR)
- "CPQ populating incorrect end dates in OrderItem" (16.1 days TTR)
- "Order line item dates mismatch with Quote line item dates" (13.1 days TTR)

**Symptoms:**
- `System.NullPointerException: Attempt to de-reference a null object` when checking Contracted = true
- Error type shown in email notification: `Error type: System.NullPointerException / Reason for error: Attempt to de-reference a null object`

**Root Cause:**
Mismatch between Quote Line products and Order Product products. A specific Order Product has a different Product ID than expected (often missed in manual comparison when there are many order products).

**Resolution Steps:**
1. Run this query to compare: `SELECT Id, SBQQ__QuoteLine__c, SBQQ__QuoteLine__r.SBQQ__ProductName__c, Product_Name__c, SBQQ__QuoteLine__r.SBQQ__Quote__c, SBQQ__SubscriptionPricing__c, SBQQ__QuoteLine__r.SBQQ__PricebookEntryId__c, PricebookEntryId FROM OrderItem WHERE OrderId = '<OrderId>'`
2. Check `PricebookEntryId` matches between QL and OP for all line items
3. Check if the `Master Contract` field on the Quote is populated (can cause issues on re-contracting)
4. Also check: custom `QuoteLineTriggerHandler` or similar apex triggers that run during order processing

**Tribal Knowledge (#support-rev-dev-amer — Apoorva Chava):** "Two interconnected errors — one from Account, one from QuoteLineTriggerHandler.updateOpptyFields (line 95)... Deactivate the QuoteLineTriggerHandler class to isolate."

---

### Pattern 13: "bad value for restricted picklist field" on Amendment Opportunity

**Frequency:** Medium  
**Severity:** Sev 1  
**Area:** Amendment / Opportunity Record Type

**Symptoms:**
- Error: `Error creating Opportunity: bad value for restricted picklist field: <value>` when creating an amendment opportunity
- Picklist value exists on the object but not available for the record type being assigned

**Root Cause:**
The default record type assigned when CPQ creates an amendment opportunity does not include the required picklist value. CPQ uses the default record type, which may not have all required picklist values configured.

**Resolution Steps:**
1. Identify which record type CPQ assigns for amendment opportunities (usually 'Sales Opportunity' or default)
2. Go to Setup → Object Manager → Opportunity → Record Types → [that record type] → Edit
3. Find the picklist field mentioned in the error
4. Ensure the failing value is in the **Selected Values** column for that record type

**Tribal Knowledge (#support-rev-dev-amer — Jhansi Chodisetti):** "It seems the default record type assigned when an amendment opportunity is created is the Sales Opportunity record type... ensure the picklist value is added under the Selected Values column."

---

### Pattern 14: UNABLE_TO_LOCK_ROW on Quote During Flow-Based Quote Creation

**Frequency:** Medium  
**Severity:** Sev 2  
**Area:** Flow Integration / CPQ Calculation

**Symptoms:**
- `SBQQ.QuoteService.QuoteServiceException: unable to obtain exclusive access to this record`
- Occurs when a Flow creates an Opportunity + Quote + Quote Lines
- CPQ background calculator (`QueueableCalculatorService`) fires mid-transaction and tries to update the Quote while the Flow transaction holds the lock

**Root Cause:**
Quote Lines are inserted during a Flow while the parent Quote record is still locked by the same transaction. This spawns the CPQ calculator, which then fails to obtain exclusive lock on the Quote.

**Resolution Steps (from #support-rev-dev-amer):**
1. Bulkify DML in the Flow: collect ALL Quote Lines into a single record collection, then insert with **one** `Create Records` element as the **absolute last step**
2. For bundle hierarchy (parent/child lines needing `SBQQ__RequiredBy__c`): use THREE distinct final steps:
   - Step 1: Insert the 1 parent Quote Line
   - Step 2: Assign the generated parent ID to `SBQQ__RequiredBy__c` on child lines
   - Step 3: Insert the collection of child Quote Lines
3. Verify `Calculate Immediately` is **unchecked** in CPQ Package Settings
4. Do NOT insert Quote Lines then immediately update them in the same transaction (triggers multiple competing calculator jobs)

---

### Pattern 15: Summary Variables Not Calculating on Renewal Quote Creation

**Frequency:** Medium  
**Severity:** Sev 3  
**Area:** Pricing / Renewals / Summary Variables  
**Known Issue:** https://issues.salesforce.com/issue/a028c00000uXaL6AAK

**Symptoms:**
- Price Actions using Summary Variables do not fire correctly when a Renewal Quote is created via OOB function
- Custom fields populated by price rules (using summary variables) show incorrect/stale values on renewal
- Manually recalculating the renewal quote fixes the values

**Root Cause:**
CPQ Known Issue: Price Actions with Summary Variables are not re-evaluated during the out-of-box renewal creation process.

**Resolution Steps:**
1. Confirm the issue is isolated to renewal quotes (not amendments or new sale quotes)
2. Workaround: manually recalculate the Renewal Quote after it's created (open QLE → Calculate → Save)
3. For automation: use a post-renewal trigger or Process Builder to kick off a recalculation
4. Reference: https://help.salesforce.com/s/articleView?id=000388745&type=1 (Pricing Requires Multiple Calculations)

---

### Pattern 16: "Read on Quote can't be granted" When Editing Profile

**Frequency:** High — Licensing/Permissions is #6 case area (663 cases/year, avg TTR 6.50 days)
**Severity:** Sev 2–3  
**Area:** Permissions / Profiles / CPQ Licensing

**Real Case Examples (OrgCS):**
- "Unable to upgrade CPQ package in sandbox" (35.1 days TTR)
- "Error in Salesforce CPQ: Calculation error on quote — UNAUTHORIZED" (31.1 days TTR)
- "Error assigning permission set" (26.3 days TTR)
- "Profile contains custom objects requiring a license" (8.2 days TTR)
- "Custom profile not editable due to CPQ license error" (7.2 days TTR)
- "Unable to grant edit access to Admin Profile for CPQ objects" (11.3 days TTR)

**Symptoms:**
- Error: `Read on Quote can't be granted. Grant the permission using a permission set with the required license or use a permission set not associated with a specific license`
- Occurs when trying to edit/save a Profile
- Happens after CPQ 228+ upgrade

**Root Cause:**
Post CPQ 228 upgrade, profiles cannot grant CPQ object permissions unless users have the appropriate PSL (Permission Set License). Any custom object with a Master-Detail relationship to a CPQ licensed object also requires the PSL.

**Resolution Steps:**
1. Remove all SBQQ__ and SBAA__ object permissions from the profile
2. Also remove permissions for **custom objects in Master-Detail relationship with Quote**: commonly `Quote Lines Summary` and `Payment Portal Transaction`
3. Grant CPQ access via **Permission Sets** with the appropriate PSL instead of profiles
4. Query dependent objects: check any custom object using CPQ objects in a Master-Detail relationship

**Key SOQL for investigation:**
```sql
SELECT Id, Name FROM PermissionSet WHERE IsCustom = true AND PermissionsViewSetup = false
```

**Tribal Knowledge (#support-rev-config-emeapac — Nitesh Kumar Singi):** "Dependent objects can include any custom object that uses a CPQ or AA licensed object in a Master-Detail relationship. Permissions granted to a custom object may also require the user to have a PSL after the upgrade to CPQ 228 or AA 228."

**After resolution:** Remove the failing objects from the profile. Two common culprits: `Quote Lines Summary` and `Payment Portal Transaction`. Profile save succeeds once these are removed.

---

### Pattern 17: Licensed Objects Error When Editing Profile (Sandbox — Known Issue)

**Frequency:** High (active KI as of May 2026)  
**Severity:** Sev 2–3  
**Area:** Profiles / Licensing  
**Known Issue:** https://help.salesforce.com/s/issue?id=a02Ka00000mGAxyIAG  
**GUS:** W-22439807, W-22453566

**Symptoms:**
- Error regarding Licensed Objects when modifying profiles (e.g., System Administrator) in **Sandboxes**
- No workaround currently available

**Resolution Steps:**
1. Attach these GUS work records to the case: W-22439807 and W-22453566
2. Direct customer to track the Known Issue for updates
3. Tell customers a revert of the change is being attempted

---

### Pattern 18: Calculation Service Authorization "Oh no, we got an error undefined"

**Frequency:** Medium  
**Severity:** Sev 2  
**Area:** Calculation Service / OAuth

**Symptoms:**
- Error: `Oh no, we got an error undefined` when trying to authorize the calculation service in CPQ Package Settings
- Reproducible after sandbox refresh
- Standard troubleshooting (incognito, different browser, revoke/re-authorize) doesn't help

**Root Cause:**
Stale OAuth token in CPQ Custom Settings. The calculation service OAuth token from before the sandbox refresh is still present but invalid.

**Resolution Steps (from #support-rev-dev-emeapac — Arpit Kaushik):**
1. Go to Setup → Connected Apps → OAuth Usage → find **SteelBrick CPQ**
2. Remove SteelBrick CPQ from Connected Apps OAuth Usage
3. Go to CPQ Custom Settings → empty the **Salesforce CPQ App Refresh Token** field
4. Re-authorize the calculation service from CPQ Package Settings

**Verification:** Calculation service shows as authorized without error.

---

### Pattern 19: Advanced Approvals — Approval Not Routing Correctly / Stuck

**Frequency:** High — Advanced Approvals is the #5 case area (704 cases/year, avg TTR 9.78 days)
**Severity:** Sev 2–3  
**Area:** Advanced Approvals (SBAA)

**Symptoms:**
- Approval submitted but goes to wrong approver, or approval status doesn't advance
- `sbaa__TrackedValue__c` duplication causing approval chain issues
- Users unable to reassign or approve requests assigned to different users
- Approval rule condition field not visible in tested field dropdown
- `Modify All` permission causing unexpected bypass of approval routing
- Advanced Approvals not sending notification emails

**Real Case Examples (OrgCS):**
- "Incorrect update of Approval Status upon submitting Quote for approval" (28 days TTR)
- "sbaa__TrackedValue__c duplication issue" (24.9 days TTR — Product Bug)
- "Commercial Finance-Deal Desk users unable to reassign or approve approval requests assigned to different users" (10.4 days TTR)
- "Approval engine not recognizing 'RVP Approver' field value" (6.8 days TTR)
- "Approval issue with Modify All permission" (5.9 days TTR)
- "Salesforce CPQ Advanced Approvals - Not Sending Emails" (7.3 days TTR)

**Root Cause (multiple):**
- Users with `Modify All` on Quote can bypass approval routing — this is WAD (Advanced Approvals checks object-level permissions)
- `sbaa__TrackedValue__c` records duplicated when approval is submitted multiple times or recalled incorrectly
- Approval rule condition using a field that isn't exposed in the SBAA condition builder (licensing/field-level security)
- Delegation/reassignment requires the delegatee to have specific SBAA permission set

**Resolution Steps:**
1. For routing errors: query active approval rules: `SELECT Id, Name, SBAA__ApproverField__c, SBAA__ConditionsMet__c FROM SBAA__ApprovalRule__c WHERE SBAA__Active__c = true ORDER BY SBAA__StepNumber__c`
2. Check if the approver field value is populated on the Quote: confirm `SBAA__ApproverField__c` value maps to a valid User lookup
3. For "Modify All" bypass: this is WAD — document for customer, recommend removing Modify All from roles that shouldn't bypass approvals
4. For email not sending: check that approval process notification template is active and the approver user has a valid email address; also check if the org has email deliverability set to "System Email Only"
5. For delegation/reassignment: the user must have the `Salesforce CPQ` permission set plus the approver's delegation must be set up in `SBAA__Approver__c` records

**Verification:** Submit a test quote through approval; confirm approval step transitions and emails fire.

**Escalation Criteria:** If `sbaa__TrackedValue__c` duplication is confirmed and reproducible without custom triggers → raise GUS investigation.

---

## C. Setup & Configuration Guide

### CPQ Package Installation Checklist
- [ ] Correct package version from https://install.steelbrick.com/#
- [ ] CPQ Integration Permission Set License (CPQIntegration) assigned
- [ ] Salesforce CPQ and Salesforce CPQ Admin Permission Sets assigned to admin
- [ ] Calculation service authorized (CPQ Package Settings → Pricing and Calculation → Authorize)
- [ ] Usage-Based Pricing enabled (if applicable — log swarm in licensing channel)
- [ ] For sandbox: run **Match Production Licenses** tool after sandbox refresh

### Common Misconfiguration Table

| Misconfiguration | Symptom | Fix |
|-----------------|---------|-----|
| CPQ license not in sandbox | Install error: "ConsumptionSchedule.Category__c Entity not available" | Run Match Production Licenses tool or ask Licensing to push CPQIntegration PSL to sandbox |
| Calculate Immediately enabled with Flow DML | UNABLE_TO_LOCK_ROW on Quote | Uncheck Calculate Immediately in Package Settings; bulkify Flow DML |
| Stale OAuth token after sandbox refresh | Calculation service authorization error | Clear Connected App OAuth + App Refresh Token custom setting; re-authorize |
| Profile with CPQ object permissions (no PSL) | "Read on Quote can't be granted" | Move CPQ permissions to Permission Set with PSL; remove from profile |
| QCP with large fields loaded | Apex heap size too large | Remove large/unused fields from QCP Quote Line Fields |
| Multiple Custom Actions same display order + conditions | Actions disappear in QLE | Assign unique display orders; ensure first action in order has no unmet conditions |

---

## D. Licensing & Entitlements

### License Decision Flowchart

```
Customer reports CPQ license issue
│
├── License expired?
│   └── Track with IR "CPQ License" → route to #C020Y1K42F8
│
├── License not in sandbox?
│   └── Run Match Production Licenses tool in sandbox org
│
├── License provisioning needed (new install)?
│   ├── Check Asset Lines in BT for active CPQ license
│   └── If missing → AE contact needed for procurement
│
├── Conga Quote Generation (CQG) license issue?
│   ├── Customer has active CQG licenses? → Still supported through us until expiry
│   └── Want to renew/add CQG licenses? → Log case via Conga Partner Community (partnercommunity.conga.com)
│       └── Use License/Entitlement support option; mention "Revenue Lifecycle edition"
│
├── RevenueLifecycleManagement Platform License?
│   └── Included in certain Industries products (e.g., Digital Insurance Unlimited Edition)
│       Gives partial RCA access — confirm with RCA team (#C05TFMXB3RN) for scope
│
└── Partner org license provisioning?
    └── Partner support team can provision licenses directly
```

### Permission Set Requirements
- **Salesforce CPQ** — required for all CPQ users
- **Salesforce CPQ Admin** — required for CPQ admins configuring rules, templates, etc.
- **CPQ Integration** (PSL) — required for API-level access and integration users
- Approvals (**SBAA**): submitter needs Edit access on Quote to flip to 'Pending'; **final approver** needs Edit access to flip to 'Approved'

---

## E. Documentation Quick-Reference

| Topic | Link |
|-------|------|
| CPQ Package Install | https://install.steelbrick.com/# |
| Large Quote Performance | https://help.salesforce.com/s/articleView?id=sales.cpq_large_quote_performance.htm&type=5 |
| Pricing Requires Multiple Calculations | https://help.salesforce.com/s/articleView?id=000388745&type=1 |
| Transient Variables for Heap | https://help.salesforce.com/s/articleView?id=000385712&type=1 |
| MDQ Guidelines (Amendments/Renewals) | https://help.salesforce.com/s/articleView?id=sf.cpq_mdq_guidelines_amendments_renewals.htm&type=5 |
| MDQ General Guidelines | https://help.salesforce.com/s/articleView?id=sf.cpq_mdq_guidelines_general.htm&type=5 |
| CPQ + Profile Permissions (228 upgrade) | https://help.salesforce.com/s/articleView?id=000389406&type=1 |
| Summary Variables in Price Actions (Renewal KI) | https://issues.salesforce.com/issue/a028c00000uXaL6AAK |
| lineVO null error KA | https://help.salesforce.com/s/articleView?id=004637113&type=1 |
| CPQ Professional Edition limitations | https://help.salesforce.com/s/articleView?id=sales.cpq_prof_edition.htm&type=5 |
| CPQ Trailhead Org (free dev org) | https://trailhead.salesforce.com/promo/orgs/cpqtrails |

### Key Behavioral Rules / Design Decisions
- Amendments must be contracted in **strictly chronological order** — backdated amendments after a later one is contracted require special handling
- `SBQQ__QuoteDocument__c.Name` field max length is **80 characters** (managed package — cannot be changed). Use `CustomName__c` formula field for longer names
- CPQ does **not support negative quantity** amendments (quantity must be ≥ 0) — no built-in validation, but behavior is unsupported
- MDQ with daylight savings time end dates creates an extra zero-value pricing segment — working as designed; workaround: null the End Date or use subscription term only
- Calculations inside QLE do **not** trigger Heroku callouts; calculations outside QLE (async) **do** call Heroku — explains why issues appear on Quote layout but not in QLE
- QCP code review: Salesforce supports up to **200 lines** of QCP code review for Premier success plan

---

## F. Escalation Paths

### When to Escalate to Engineering (GUS Investigation)
- Error is reproducible in a clean internal org with no customizations
- Error traceable to SBQQ managed package code (not customer automation)
- Known pattern without a confirmed resolution
- High TTR (>5 days) with no workaround available
- Multiple customers reporting same issue after a release

### What to Gather Before Escalating
- [ ] Exact error message (copy-paste from debug log)
- [ ] CPQ package version
- [ ] Org ID (production + sandbox)
- [ ] Debug log (CPQ managed package log, not just standard)
- [ ] Steps to reproduce (with specific record IDs)
- [ ] Confirmed: reproducible with QCP disabled?
- [ ] Confirmed: reproducible with all custom triggers/flows disabled?
- [ ] Internal org reproduction attempt (yes/no, result)

### Channel Routing
| Issue Type | Channel |
|-----------|---------|
| AMER CPQ Config | #support-rev-config-amer (C027R84HLKB) |
| AMER CPQ Dev | #support-rev-dev-amer (C0275QG40LE) |
| IST/EMEAPAC CPQ Config | #support-rev-config-emeapac (C02836Z3SF2) |
| IST/EMEAPAC CPQ Dev | #support-rev-dev-emeapac (C021MMRBRR6) |
| Salesforce Billing | #C026Y0ENETH |
| Revenue Cloud Core (RCA/RCB) | #C05TFMXB3RN |
| CPQ Document Generation | #C05SQUC3G9K |
| License expired / install errors | #C020Y1K42F8 |
| Technical escalations / cross-team | #support-rev-technical (G01GF9PFUH3) |

### Key SMEs (from channel observations)
- **Bart Lis** — CPQ config, pricing rules, QCP, POT behavior, amendment/renewal design
- **Tori (Victoria) Olson** — CPQ technical, licensing, scope decisions, channel routing
- **Shaun Ray** — CPQ licensing, Conga CQG, RCA entitlements, partner provisioning
- **Conall OMalley** — CPQ dev escalations (EMEAPAC), MDQ, known bugs, GUS triage
- **Apoorva Chava / Vinay Kumar K** — CPQ dev support (AMER), Splunk log analysis
- **Jhansi Vipulasri Chodisetti** — CPQ dev support (AMER), Sev 1 assistance
- **Sam Faulkner / Rashi Saraswat** — Dev support queue management (AMER)

---

## G. Tribal Knowledge & Pro Tips

1. **Debug log type matters.** Standard Apex logs miss CPQ managed package internals. Always capture the CPQ package debug log specifically. (#support-rev-dev-emeapac, ongoing)

2. **Calculation inside vs. outside QLE is fundamentally different.** Inside QLE = synchronous, no Heroku callout. Outside QLE (Quote layout Calculate button, async) = Heroku callout. If an error only appears outside QLE, suspect the Heroku calculation service or QCP callout limits. (#support-rev-config-emeapac — Bhuvaneswari Betala, May 2026)

3. **"Solution deployed" on a KI doesn't mean fixed.** Sometimes the KI is marked "solution deployed" for a doc bug — meaning documentation was updated to acknowledge the limitation, not that the underlying issue was fixed. Always read the GUS bug record details. (#support-rev-dev-emeapac — Conall OMalley, Oct 2023)

4. **Calculation service authorization "undefined" error after sandbox refresh:** always clear Connected App OAuth + Custom Settings App Refresh Token before re-authorizing. Just revoking/re-authorizing is not enough. (#support-rev-dev-emeapac, Nov 2024)

5. **CPQ only supports packages at install.steelbrick.com.** Third-party packages (PROS Smart CPQ, AvaTax for CPQ) are out of scope unless the customer also has Salesforce CPQ (SBQQ) installed. AvaTax upgrades go to the AvaTax AppExchange vendor. (#support-rev-technical, Apr 2026)

6. **Binary search for price rule issues.** With 50+ active price rules, deactivate half and test. Narrow in 3-4 iterations. Adding `COUNT(Quote Lines) > 0` as an error condition prevents rules from firing on an empty QLE state. (#support-rev-config-emeapac, Aug 2025)

7. **Renewal quote custom action disappearing:** it's often a duplicate display order combined with a custom action that has unsatisfied display conditions. Changing the display order of the first action (even by 1) resolves it without touching conditions. (#support-rev-dev-emeapac — Conall OMalley, Oct 2024)

8. **Amendment opportunity picklist errors:** the default record type assigned during amendment is not always the primary record type. Check which record type CPQ uses and ensure all picklist values exist in it. (#support-rev-dev-amer — Jhansi Chodisetti, Mar 2026)

9. **QCP code review limit:** for Premier success plan, Salesforce will only review up to 200 lines of QCP code. Document this early in Sev 1 cases with large QCPs. (#support-rev-config-emeapac — Bhuvaneswari Betala, May 2026)

10. **For Signature/Sev 1 cases:** loop in AE early (especially for QCP issues, large configuration requirements, or product limitations the customer won't accept). AE engagement often unblocks cases faster than engineering escalation. (#support-rev-config-emeapac, May 2026)

---

## H. Active Known Issues (as of May 2026)

| Issue | GUS / KI | Workaround |
|-------|---------|-----------|
| Product selection checkboxes blocked in Console App (post Summer '26) | W-22449442, W-22488848 / KI: a02Ka00000mGGGF | Enable Split View in Console App |
| Licensed Objects error editing profiles in Sandboxes | W-22439807, W-22453566 / KI: a02Ka00000mGAxyIAG | None — track KI |
| MDQ extra pricing segment with daylight savings time end dates | Doc Bug — WAD | Null End Date on Quote; use subscription term only |
| Summary Variables not recalculating on OOB renewal creation | KI: a028c00000uXaL6AAK | Manually recalculate renewal quote after creation |
| Quote Line Group Actions not rendering when first action has unmet conditions | GUS Bug W-SBQQ8SpHvIAK | Reorder custom actions so first action has no unmet conditions |
| Incorrect Order StartDate when using Create Order button (OSCPQOrderManagementPlugin) | KI: a028c00000gAxWkAAK (marked "solution deployed" — see tribal knowledge #3) | Per KI: solution was documentation update, not a fix |

---

## I. Code Reference Map

> Connect CodeSearch to get specific file paths. Key classes in the CPQ managed package (SBQQ) referenced in Slack investigations:

| Class / Component | Issue Pattern | Notes |
|------------------|--------------|-------|
| `SBQQ.OrderItemObjectManager.validateDateChanges` | Pattern 10 (backdated amendment NPE) | Validates date overlap on amendment contracting |
| `SBQQ.OrderProductDAO.updateTerminatedDateOfOrderProducts` | Pattern 10 | Updates TerminatedDate when contracting amendments |
| `QueueableCalculatorService` | Pattern 14 (row lock), Pattern 7 (heap) | Async CPQ calculator; triggered by Quote Line DML |
| `QuotePriceHandler.refresh` / `loadAndIndexPricebookEntries` | Pricebook popup on amendment (null Opp pricebook) | Called when null pricebook ID triggers popup |
| `CXO_CPQAPIModels.QuoteSaver.save` | Pattern 7 (heap) | Quote save orchestration |
| `OrderGenerator.isStandAlonePOT` | POT/co-term amendment behavior | Controls SubscriptionPricing for standalone POT products |
| `CPQ_OrderTriggerHandler.bulkAfter` / `filterRecords` | Custom triggers on orders (Pattern 12 related) | Customer custom trigger — not SBQQ managed package |

---

## Update Cadence

- Refresh after every Salesforce release (Spring, Summer, Winter — 3x/year)
- After every new Known Issue published to #support-rev-technical
- Monthly scan of Slack channels for new recurring patterns
- When GUS investigations for CPQ close with root cause documented

---

## Test Cases

> Sourced from real OrgCS cases, last 365 days.

| Scenario (from real cases) | Expected Skill Response |
|---------|------------------------|
| "Customer can't select products in QLE — checkboxes are blocked / invisible sidebar" (14 days TTR) | Check if Console App without Utility Bar → Pattern 5, Summer '26 KI W-22449442, enable Split View workaround |
| "Getting NullPointerException when contracting an order — error type: Attempt to de-reference null" (19.9 days TTR) | Ask for debug log; query Order Products for product ID mismatch → Pattern 12; check for custom QuoteLineTriggerHandler; check if Master Contract field populated |
| "CPQ Quotes Showing $0 — Emergency" (6.1 days TTR) | Check active Price Rules → Pattern 2; verify Pricebook is populated on Opportunity; check if any Price Action sets Net Price to 0 |
| "Amend Contract from Opportunity pulling in previous Amendment" (114.7 days TTR) | Pattern 8 / amendment sequence issue; check if Subscription records are duplicated; verify amendment quote is based on latest contract; may need GUS investigation |
| "Unable to grant edit access to Admin Profile for CPQ objects" (11.3 days TTR) | Pattern 16 — post CPQ 228 upgrade restriction; remove SBQQ__ / SBAA__ object permissions from profile; move to Permission Sets with PSL |
| "Approval engine not recognizing 'RVP Approver' field value" (6.8 days TTR) | Pattern 19 — check approval rule SBAA__ApproverField__c mapping; verify user lookup field is populated; check if `Modify All` permission bypassing approval |
| "Too Many SOQL Queries: 101 Error during contract creation" (108.6 days TTR) | Pattern 7 — Apex limits in SBQQ trigger chain; capture managed package debug log; check for custom triggers running alongside CPQ contracting; check SOQL in QCP if present |

---

## Skill Completeness Report

```
=== SKILL COMPLETENESS REPORT ===

✅ Case data (OrgCS): 9,023 cases analyzed, 19 patterns identified
   — Feature area breakdown: 11 areas, TTR data included
   — Case cause breakdown: 20 sub-cause combinations analyzed
   — Real case subjects included in each pattern as examples
❌ CodeSearch: not connected — code references sourced from Slack/manual (managed package class names)
❌ GUS: not connected — investigation IDs sourced from Slack (W-22449442, W-22488848, W-22439807, W-22453566)
✅ Slack: 5 channels mined — tribal knowledge in Sections G and per-pattern workarounds
✅ Help Docs: 12 documentation links included

GAPS:
- Code references are managed package class names from Slack, not verified CodeSearch file paths
- GUS investigation details (root cause briefs, found-in-build) not pulled — W-IDs only
- Feature areas: Proration (17 cases, avg 11.9 days TTR) and Installation/Upgrade (357 cases) lack dedicated patterns
- Advanced Approvals pattern added (Pattern 19) from case data but not enriched with GUS root cause

RECOMMENDATION:
- Connect CodeSearch → verify managed package class paths in Section I
- Connect GUS → pull root cause briefs for W-22449442, W-22439807, and top 5 open CPQ bugs
- Run skill quarterly: re-query OrgCS to update case volume and TTR data
```

---

## Scope Limits

This skill **cannot resolve:**
- Issues in PROS Smart CPQ, AvaTax for CPQ, or other 3rd-party CPQ products not listed at install.steelbrick.com
- Revenue Cloud Advanced (RCA/RCB) issues — route to #C05TFMXB3RN
- Salesforce Billing issues — route to #C026Y0ENETH
- QCP custom code debugging beyond identifying the class/method — this is dev support scope
- Conga Quote Generation license renewal/provisioning — route to Conga Partner Community
- Issues requiring internal Splunk log access — only Dev Support engineers can access Splunk