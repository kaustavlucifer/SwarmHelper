## Case Volume Statistics (OrgCS, trailing 365 days — snapshot May 2026)

> Snapshot date: **~May 2026**. Re-run the OrgCS volume query to refresh; treat counts as directional, not live.

**Total closed cases for "Revenue-CPQ Developer Support": 1,972 | Avg TTR: 9.47 days**

### By Feature Area

| Feature Area | Cases | Avg TTR (days) |
|---|---|---|
| Line Editor UI/UX | 383 | 9.17 |
| Contracts | 332 | 9.84 |
| Amendments and Renewals | 246 | 10.05 |
| Apex | 202 | 11.36 |
| Orders / Order Products | 181 | 9.45 |
| Pricing | 125 | 9.94 |
| Advanced Approvals | 95 | 9.39 |
| Product Selection / Bundle Configuration | 87 | 10.69 |
| Installation / Upgrade / Authorization | 82 | 8.65 |
| Licensing, Provisioning, Permissions | 81 | 6.56 |

### By Case Cause

| Cause | Sub-Cause | Cases | Avg TTR |
|---|---|---|---|
| Documentation / Training | User needs instructions or training | 838 | 7.96 days |
| Product/Infrastructure | Error message issue | 216 | 9.57 days |
| Documentation / Training | Need better documentation | 122 | 9.79 days |
| Product/Infrastructure | Feature limit / Feature activation | 28 | 9.82 days |
| Product/Infrastructure | **Product Bug** | **18** | **12.38 days** |
| Out of scope | Request for implementation/code dev | 27 | 14.0 days |

> **Note:** Apex area has the highest TTR (11.36 days). Product Bugs avg 12.4 days. "Request for code development" cases (out of scope) avg 14.7 days — close early with scope guidance.

## B. Known Issue Patterns

### Pattern 1: UNABLE_TO_LOCK_ROW from SBQQ Managed Package

**Frequency:** High — most reported QLE/developer error; recurs across all feature areas
**TTR Impact:** Avg 26–37 days when misdiagnosed; resolves in <5 days once root cause isolated
**Area:** QLE / Flow / Async Calculator

**Symptoms:**
- `SBQQ.QuoteService.QuoteServiceException: unable to obtain exclusive access to this record`
- `UNABLE_TO_LOCK_ROW issue from SBQQ managed package`
- Occurs when Flow or custom code inserts/updates Quote Lines while CPQ calculator is running

**Root Cause:**
Two concurrent transactions both attempt to lock the Quote record. The CPQ `QueueableCalculatorService` fires asynchronously when Quote Lines are inserted, and if the parent Flow/trigger transaction hasn't committed yet, the calculator cannot obtain an exclusive row lock.

**Resolution Steps:**
1. Confirm `Calculate Immediately` is **unchecked** in CPQ Package Settings
2. Bulkify all Quote Line DML into a single `Create Records` element as the **absolute last step** of the Flow
3. For bundle hierarchy (parent/child lines needing `SBQQ__RequiredBy__c`): use three sequential final steps — insert parent line → assign parent ID to children → insert child lines
4. Never insert Quote Lines then immediately update them in the same transaction
5. If using Apex: ensure all DML is in a single transaction boundary; no `@future` or `Queueable` chains that touch the Quote mid-transaction

**Verification:** Run the same Flow/trigger path without the lock error appearing in debug logs.

**Real Case Examples (OrgCS):**
- "UNABLE_TO_LOCK_ROW issue from SBQQ managed package" (37 days TTR)
- "Unable to Lock Row issue on saving Quote from Quote Line Editor" (26.1 days TTR)
- "Unable to obtain exclusive access to this record" (32.9 days TTR)

**Escalation Criteria:** If reproducible with no custom code and `Calculate Immediately` unchecked → raise GUS investigation.

---

### Pattern 2: "Failed to process Queueable job for class SBQQ" / QueueableCalculatorService Failure

**Frequency:** High — surfaces across QLE, Amendments, Orders areas
**TTR Impact:** Avg 22–35 days
**Area:** Async Calculator / Queueable Jobs

**Symptoms:**
- `Error: Failed to process Queueable job for class SBQQ`
- `CPQ Queueable Job Failure - SBQQ.QueueableCalculatorService`
- Quote Lines saved but prices not recalculated; quote stays in stale state
- Often intermittent — works sometimes, fails other times

**Root Cause (multiple):**
- QCP (Quote Calculator Plugin) throwing an unhandled exception inside the async job
- `Too many queueable jobs` limit hit when multiple quote operations are chained
- Heroku calculation service timeout during async callout
- `SBQQ__RequiredBy__c` null reference on a bundle child line mid-calculation

**Resolution Steps:**
1. Capture the **CPQ managed package debug log** (not standard Apex log) — set to `FINEST` for `SBQQ` namespace
2. Check for active QCP: `SELECT Id, Name, SBQQ__Code__c FROM SBQQ__CustomScript__c WHERE SBQQ__Active__c = true`
3. If QCP exists: temporarily disable it and retest — if job succeeds, the QCP has an unhandled exception
4. Check `SBQQ__QuoteCalculationStatus__c` on the Quote record — if stuck in "Calculating", the job may be orphaned
5. Check org's queueable job queue: `SELECT Id, Status, JobType FROM AsyncApexJob WHERE JobType = 'Queueable' ORDER BY CreatedDate DESC LIMIT 20`
6. Enable **Legacy Amend/Renew Service** temporarily to force synchronous calculation and capture full stack trace

**Real Case Examples (OrgCS):**
- "Error: Failed to process Queueable job for class SBQQ" (35 days TTR)
- "CPQ Queueable Job Failure - SBQQ.QueueableCalculatorService" (22.2 days TTR)
- "Too many queueable jobs error in Salesforce CPQ" (36.8 days TTR)
- "SBQQ: Too many queueable jobs error" (28.1 days TTR)

**Escalation Criteria:** If reproducible without QCP and without custom automation → raise GUS investigation with managed package log.

---

### Pattern 3: ApexPages.addMessage Called Outside Visualforce Context

**Frequency:** Medium
**TTR Impact:** Avg 50 days
**Area:** Custom Apex Trigger / Quote After Trigger

**Symptoms:**
- `CPQ Quote After Trigger: ApexPages.addMessage can only be called from a Visualforce page`
- Error fires in a trigger on Quote (AfterInsert/AfterUpdate) that uses `ApexPages.addMessage()`
- Works in some contexts but fails in API/Flow-triggered updates

**Root Cause:**
Customer has a custom Quote After Trigger that calls `ApexPages.addMessage()`. This method is only valid inside a Visualforce page context. When CPQ triggers a Quote update via API or async process, there is no VF page context, and the call throws.

**Resolution Steps:**
1. Locate the custom Quote trigger: search for `ApexPages.addMessage` in the org's Apex triggers on `SBQQ__Quote__c` or `Quote`
2. Wrap the call with a context check: `if (ApexPages.currentPage() != null) { ApexPages.addMessage(...); }`
3. Or replace with a Platform Event / custom notification for non-VF contexts
4. Test in both VF page context and via API to confirm fix

**Real Case Example (OrgCS):**
- "CPQ Quote After Trigger, ApexPages.addMessage can only be called from a Visualforce page" (50.3 days TTR)

---

### Pattern 4: CPQ Blocked by API Access Control / Experience Cloud

**Frequency:** Medium
**TTR Impact:** Avg 16–63 days
**Area:** Integration / Experience Cloud / API Access

**Symptoms:**
- `Experience cloud is not allowing Visualforce pages to access API when API Access Control is enabled`
- `CPQ error when API Access Control is enabled`
- `Unable to Query Quote and CPQ objects from Middleware`
- CPQ calculation works in internal org but fails for Experience Cloud / Community users or middleware integration

**Root Cause:**
When `API Access Control` is enabled on a Connected App or in Experience Cloud settings, it restricts which OAuth scopes can access specific APIs. CPQ's VF pages and calculation service make API calls that may be blocked by these restrictions.

**Resolution Steps:**
1. Check Connected App settings: verify `api` scope is included in the Connected App used by Experience Cloud / middleware
2. For Experience Cloud: go to Administration → Policies → check if API Access is restricted
3. For middleware: confirm the integration user has the `Salesforce CPQ` Permission Set assigned — this grants API access to SBQQ objects
4. Test with the integration user running a direct SOQL query on `SBQQ__Quote__c` — if it fails, it's a permission/scope issue, not a CPQ bug
5. If using Named Credentials for Heroku callouts: verify the Named Credential is authorized for the integration user

**Real Case Examples (OrgCS):**
- "Experience cloud is not allowing Visualforce pages to access API when API Access Control is enabled" (62.9 days TTR — Product Bug)
- "CPQ error when API Access Control is enabled" (16.1 days TTR)
- "Unable to Query Quote and CPQ objects from Middleware" (55.3 days TTR)

---

### Pattern 5: QCP Debugging — Custom Script Logic Issues

**Frequency:** High (202 Apex cases; QCP is involved in most developer escalations)
**TTR Impact:** Avg 40–55 days for complex QCP issues
**Area:** Quote Calculator Plugin (QCP)

**Symptoms:**
- QCP plugin failing with unhandled exception
- QCP logic not executing for specific products or scenarios
- `Need help debugging QCP logic`
- Group number not updating correctly after quote clone
- `SBQQ__RequiredBy__c` field not cloned/populated via `clone()`
- QCP script showing incorrect group numbers after clone

**Key Constraints:**
- Salesforce supports up to **200 lines of QCP code review** for Premier plans
- QCP runs **synchronously inside QLE** and **asynchronously via Heroku** outside QLE — behavior may differ
- QCP cannot make `@future` callouts from inside the synchronous QLE context
- `SBQQ__RequiredBy__c` is **not cloned** by the standard CPQ `clone()` method — must be set manually in QCP

**Resolution Steps:**
1. Capture the CPQ managed package debug log at `FINEST` level — standard logs miss QCP internals
2. Check QCP `onBeforeCalculate`, `onAfterCalculate`, `onBeforePriceRules`, `onAfterPriceRules` methods for unhandled null checks
3. To test QCP in isolation: use CPQ Quote API (`SBQQ.QuoteAPI.calculate`) in Execute Anonymous with the failing Quote ID
4. For `SBQQ__RequiredBy__c` not cloning: customer must explicitly set this field in QCP code after clone — it is by design excluded from the managed package clone
5. For group number issues after clone: group numbers are stored on `SBQQ__QuoteLineGroup__c` — customer QCP must re-sequence after clone
6. For Premier plans: inform customer of 200-line review limit upfront; for larger QCPs, request they isolate the failing method

**Real Case Examples (OrgCS):**
- "Need Help On the QCP logic — debugging it" (54.9 days TTR)
- "QCP Script showing the same group number after cloning" (55.3 days TTR)
- "QCP plugin Failing" (40.4 days TTR)
- "SBQQ__RequiredBy__c Field Not Cloned or Populated via clone()" (29.6 days TTR)

**Escalation Criteria:** QCP custom code issues are developer support scope — do not raise GUS unless the underlying managed package method is confirmed broken independent of QCP.

---

### Pattern 6: "Attempt to de-reference a null object" on Contract Creation

**Frequency:** High — #1 error in Contracts area (332 cases, avg TTR 9.84 days)
**TTR Impact:** Avg 41–57 days for high-severity cases
**Area:** Contract Creation / Order Contracting

**Symptoms:**
- `Error during Contract creation: Attempt to de-reference a null object`
- `Null pointer error on order contracting`
- `System.NullPointerException at SBQQ.OrderItemObjectManager.validateDateChanges`
- Contracts not created for specific orders — no error shown to user, just no contract record

**Root Cause (multiple):**
- Product ID mismatch between Quote Line and Order Product (most common)
- Backdated amendment contracted out of chronological order — later amendment already set `SBQQ__TerminatedDate__c`
- Custom trigger on Order / OrderItem firing during contracting and hitting a null reference
- `Master Contract` field on Quote populated when it shouldn't be (re-contracting edge case)

**Resolution Steps:**
1. Compare Quote Lines and Order Products: `SELECT Id, SBQQ__QuoteLine__c, SBQQ__QuoteLine__r.SBQQ__ProductName__c, PricebookEntryId, SBQQ__QuoteLine__r.SBQQ__PricebookEntryId__c FROM OrderItem WHERE OrderId = '<OrderId>'` — look for mismatched PricebookEntryIds
2. Check for custom OrderItem/Order triggers: temporarily deactivate and retest
3. For backdated amendment NPE: check `SBQQ__TerminatedDate__c` on original Order Products — if set by a later amendment, null it temporarily, contract the backdated order, then review
4. Check `Master Contract` field on the Quote — if set, clear it and retry contracting
5. Check `SBQQ__OrderingPreference__c` on the Account — misconfigurations here affect which orders generate contracts

**Real Case Examples (OrgCS):**
- "Error during Contract creation: Attempt to de-reference a null object" (57.1 days TTR)
- "Error During Contract Creation - Null Object Reference Exception" (41.9 days TTR)
- "Null pointer error on order contracting" (41 days TTR)
- "Contracts not created for specific orders" (37.1 days TTR)
- "Again Contracts not being created for specific orders" (19.9 days TTR)

**Escalation Criteria:** Reproducible with all custom triggers/flows disabled and no QCP → GUS investigation with full managed package stack trace.

---

### Pattern 7: Connection Timeouts / Heroku Callout Failures

**Frequency:** Medium
**TTR Impact:** Avg 15–70 days
**Area:** Calculation Service / Heroku / Integration

**Symptoms:**
- `Unable to connect to the server (transaction aborted: timeout)`
- `CPQ Record Jobs Failing at connect ETIMEDOUT`
- `Error executing Quote creation in CPQ after Hyperforce Migration`
- Quote calculation intermittently failing outside QLE (async path)
- Works in sandbox but fails in production (or vice versa)

**Root Cause:**
CPQ's asynchronous calculation (outside QLE) calls the Heroku calculation service. Network timeouts, Heroku pod cold starts, or post-Hyperforce DNS changes can break this callout.

**Resolution Steps:**
1. Confirm the calculation service is authorized: CPQ Package Settings → Pricing and Calculation → check authorization status
2. Check the Named Credential for the Heroku endpoint — after Hyperforce migration, the Named Credential URL may need updating
3. Re-authorize the calculation service: Connected Apps → SteelBrick CPQ → revoke OAuth → clear `App Refresh Token` in CPQ Custom Settings → re-authorize
4. Check if the issue only occurs async (outside QLE) — if QLE works but Quote layout Calculate button fails, the Heroku service is the issue
5. For Hyperforce migrations specifically: verify `Remote Site Settings` include the new Heroku endpoint URL

**Real Case Examples (OrgCS):**
- `"Unable to connect to the server (transaction aborted: timeout)"` (70.7 days TTR)
- "CPQ Record Jobs Failing at connect ETIMEDOUT" (18 days TTR)
- "Error executing Quote creation in CPQ after Hyperforce Migration" (15.7 days TTR)

---

### Pattern 8: Subscriptions Getting Terminated / Duplicate Order Products

**Frequency:** Medium
**TTR Impact:** Avg 16–49 days
**Area:** Contracts / Subscriptions / Order Products

**Symptoms:**
- `EA subscriptions are getting terminated` unexpectedly
- `Duplicate Opportunity Products (OEM) are creating`
- Cancelled child subscriptions linked to wrong parent subscriptions after contracting
- Subscription lines issue on bulk operations

**Root Cause:**
- Custom triggers or batch Apex that updates `SBQQ__TerminatedDate__c` on subscriptions incorrectly
- Bundle contracting where child subscription `SBQQ__RequiredBy__c` is null — CPQ links them to wrong parent
- Bulk operations that hit SOQL limits and partially process subscription records

**Resolution Steps:**
1. Query termination timestamps: `SELECT Id, SBQQ__TerminatedDate__c, SBQQ__Product__r.Name, SBQQ__Contract__c FROM SBQQ__Subscription__c WHERE SBQQ__Contract__c = '<ContractId>' AND SBQQ__TerminatedDate__c != null ORDER BY SBQQ__TerminatedDate__c`
2. Check for custom batch jobs or triggers that set `SBQQ__TerminatedDate__c`
3. For wrong parent subscription links: `SELECT Id, SBQQ__RequiredBy__c, SBQQ__Product__r.Name FROM SBQQ__Subscription__c WHERE SBQQ__Contract__c = '<ContractId>'` — verify `SBQQ__RequiredBy__c` hierarchy matches product bundle structure
4. For bulk operations: add chunking to avoid governor limits; process subscriptions in batches of 200

**Real Case Examples (OrgCS):**
- "EA subscriptions are getting terminated" (16.1 days TTR)
- "On big contract, cancelled child Subscriptions linked to wrong parent" (49 days TTR)
- "Duplicate Opportunity Products (OEM) are creating" (28 days TTR)
- "Error during bulk updates on subscription object" (28.2 days TTR)

---

### Pattern 9: Apex Heap Size Too Large (Contract Creation / Large Quotes)

**Frequency:** High — Apex area avg TTR 11.36 days; heap errors are the most reported Apex sub-issue
**TTR Impact:** Avg 19–42 days
**Area:** Apex / Governor Limits / Large Quotes

**Symptoms:**
- `System.LimitException: Apex heap size too large: 7959103` (or similar multi-MB values)
- `Apex heap size too large: 14362190`
- Occurs on order contracting with large quote line counts
- Occurs during multi-group quote amendment (180+ lines)
- Occurs when processing Quote Line Items via external tool (e.g., Valorx)

**Root Cause (multiple):**
- QCP loading large or unused fields (Long Text Areas) into the in-memory quote model
- Large `SBQQ__QuoteLineGroup__c` sets with many child lines building oversized object graphs
- External integrations passing large payloads alongside CPQ's own in-memory model
- Debug logging itself adds heap overhead — error may only appear when dev console is open

**Resolution Steps:**
1. Check QCP Quote Line Fields — remove any Long Text Area fields not needed in calculation
2. Enable **Large Quote Performance Settings**: `Setup → CPQ Package Settings → Large Quote Performance`
3. Enable `'Enable Large Configurations'` setting in CPQ Package Settings
4. Use `transient` keyword on large variables in custom Apex that co-runs with CPQ
5. For external integrations: process Quote Lines in smaller batches; avoid loading full quote objects alongside CPQ models
6. Reference: https://help.salesforce.com/s/articleView?id=000385712&type=1 (Transient Variables)
7. Reference: https://help.salesforce.com/s/articleView?id=sales.cpq_large_quote_performance.htm&type=5

**Real Case Examples (OrgCS):**
- "When creating contracts from an order, it is throwing System.LimitException: Apex heap size too large: 7959103" (41.9 days TTR)
- "Apex heap size too large: 14362190" (19.3 days TTR)
- "Multi Group Quote — performing amendment for 180 lines encountering Apex heap size issue" (27.3 days TTR)
- "Heap Size and 503 Service Unavailable Errors While Processing Quote Line Items via Valorx" (32.7 days TTR)
- "Apex CPU Time Limit Issue when creating quote lines through custom code for Large Quotes (100+ lines)" (27.2 days TTR)

---

### Pattern 10: "Too Many Queueable Jobs" / Queueable Chain Depth Exceeded

**Frequency:** High
**TTR Impact:** Avg 28–37 days
**Area:** Apex / Async / Amendments

**Symptoms:**
- `SBQQ: Too many queueable jobs error`
- `Too many queueable jobs error in Salesforce CPQ`
- Errors in `CaseDebookService Queueable` (stack depth exceeded)
- Occurs during amendment processing with large bundle structures or multiple concurrent quote operations

**Root Cause:**
Salesforce enforces a limit of 50 queueable jobs per transaction (and max chain depth of 5 in sync contexts). CPQ's internal queueable jobs (calculator, contract creation) plus customer custom queueable jobs can exceed this limit during complex amendment processing.

**Resolution Steps:**
1. Identify all active queueable jobs: `SELECT Id, ApexClass.Name, Status FROM AsyncApexJob WHERE JobType = 'Queueable' AND Status IN ('Queued', 'Processing') LIMIT 50`
2. Check for custom Apex classes implementing `Queueable` that fire on Quote/Order events — these compete with CPQ's internal jobs
3. Enable `Legacy Amend/Renew Service` to process amendments synchronously and bypass queueable chains
4. Batch processing: use `Database.executeBatch` instead of chained Queueables for bulk operations
5. For `CaseDebookService` stack depth errors: reduce bundle complexity or split into sequential amendment operations

**Real Case Examples (OrgCS):**
- "SBQQ: Too many queueable jobs error" (28.1 days TTR)
- "Too many queueable jobs error in Salesforce CPQ" (36.8 days TTR)
- "Errors Encountered During Bundle Amendments in CaseDebookService Queueable (Stack Depth & Missing Field)" (35 days TTR)

---

### Pattern 11: "Amend" Button Not Generating Amendment Opportunity

**Frequency:** Medium
**TTR Impact:** Avg 33 days
**Area:** Amendments / Contract

**Symptoms:**
- Clicking `Amend` on a Contract record does nothing or shows a generic error
- No amendment opportunity or quote is created
- `Requesting assistance: "Amend" button on the Contract is not generating an amendment opportunity`
- Sometimes produces `invalid cross reference id` on the new Opportunity

**Root Cause (multiple):**
- A Flow on Opportunity fires after CPQ creates the amendment opportunity and fails (invalid owner ID, record type issue)
- `SBQQ__AmendmentOpportunityStage__c` in CPQ Package Settings references a Stage value that no longer exists on the Opportunity Stage picklist
- Contract's `SBQQ__RenewalForecast__c` or amendment-related fields in an inconsistent state

**Resolution Steps:**
1. Enable debug logs for the user clicking Amend; look for errors in the Opportunity AfterInsert context
2. Check CPQ Package Settings → `Amend/Renew` section → `Amendment Opportunity Stage` — verify the stage value exists in the picklist
3. Query Flows on Opportunity: look for any Flow with owner assignment, record type, or required field that could fail
4. Check if the Contract has `SBQQ__AmendedContract__c` already populated (indicates a prior amendment exists)
5. Verify the Contract's `Status` is `Activated` — CPQ will not amend non-activated contracts

**Real Case Example (OrgCS):**
- `"Amend" button on the Contract is not generating an amendment opportunity` (33 days TTR)
- "Contract is not Pulling over when Amending" (52.1 days TTR)

---

### Pattern 12: "Too Many SOQL Queries: 101/301" in CPQ Apex Context

**Frequency:** High — Apex area; SOQL limit errors are the second most common developer sub-issue
**TTR Impact:** Avg 21–108 days
**Area:** Apex / Governor Limits

**Symptoms:**
- `Too Many SOQL Queries: 101` during contract creation
- `SBQQ: Too many SOQL aggregate queries: 301`
- `CPQ - Too MANY SOQL Query during Contract creation`
- Occurs in specific orgs on a specific package version but not others (version-specific regression)

**Root Cause:**
- CPQ's internal contract creation logic performs multiple SOQL queries per order product
- Custom Apex triggers on Order/OrderItem add additional queries in the same transaction
- After package upgrades, internal query counts may increase, pushing the org over the 100/300 limits
- Middleware or integration tools performing SOQL inside CPQ trigger contexts

**Resolution Steps:**
1. Capture the full stack trace — identify whether queries originate from SBQQ namespace or customer namespace
2. Check for custom triggers on Order, OrderItem, SBQQ__Quote__c, SBQQ__QuoteLine__c that fire during contracting — SOQL inside loops is the most common culprit
3. Bulkify custom triggers: move SOQL outside loops, use Maps for lookups
4. If all queries are from SBQQ namespace and the issue started after a version upgrade → raise GUS investigation with the specific version number
5. Check `SOQL Query Limits` in the debug log trace — identify which class/method is consuming the most queries

**Real Case Examples (OrgCS):**
- "CPQ - Error TOO MANY SOQL Query during Contract creation" (108.6 days TTR)
- "Too Many SOQL Queries: 101 in Salesforce CPQ (v256.0), not reproducible in Staging (v258.0)" (21.9 days TTR)
- "SBQQ: Too many SOQL aggregate queries: 301" (33.7 days TTR)
- "CPQ - Too Many SOQL Queries" (42.3 days TTR)

---

### Pattern 13: CPQ API Integration Issues (addProductsToQuote / calculate)

**Frequency:** Medium
**TTR Impact:** Avg 13–55 days
**Area:** CPQ API / Integration / Middleware

**Symptoms:**
- `Error: Uncommitted Work Pending in CPQ_ApiService.addProductsToQuote`
- `CPQ API throwing Null pointer exception`
- API data load taking too long
- `nextRecordUrl is not showing all the records in Postman` (pagination issue)
- Quote Lines inserted via API but prices not calculated

**Root Cause:**
- `Uncommitted Work Pending`: customer code calls `SBQQ.QuoteAPI.addProducts()` inside a transaction that already has uncommitted DML — CPQ API requires a clean transaction boundary
- Null pointer in API: Quote or Product records passed to API have missing required fields
- Pagination: CPQ REST API returns max 2,000 records per page; `nextRecordUrl` must be followed for large result sets
- Prices not calculating after API insert: `SBQQ.QuoteAPI.calculate()` must be called explicitly after `addProducts()`

**Resolution Steps:**
1. For `Uncommitted Work Pending`: ensure `SBQQ.QuoteAPI.addProducts()` is the first DML in the transaction — no prior inserts/updates allowed before the call
2. For null pointer: validate all required fields on the `SBQQ.ProductModel` objects before passing to the API
3. For pagination: loop on `nextRecordUrl` until null — do not assume a single API call returns all records
4. For prices not updating: always call `SBQQ.QuoteAPI.calculate(quoteModel)` after `addProducts()` to trigger recalculation
5. Reference CPQ API documentation: https://developer.salesforce.com/docs/atlas.en-us.cpq_dev_guide.meta/cpq_dev_guide/

**Real Case Examples (OrgCS):**
- "Error: Uncommitted Work Pending in CPQ_ApiService.addProductsToQuote" (13 days TTR)
- "CPQ API throwing Null pointer exception" (32 days TTR)
- "Integration: nextRecordUrl is not showing all the records in Postman" (52.9 days TTR)
- "Api data load is taking lot of time" (55 days TTR)

---

### Pattern 14: CPQ Quote API — Null Pointer / Missing Fields

**Frequency:** Medium
**TTR Impact:** Avg 32 days
**Area:** CPQ API / Apex

**Symptoms:**
- `CPQ API throwing Null pointer exception` when calling `SBQQ.QuoteAPI` methods
- Quote status not updating via automation but works manually
- Primary Quote not linked to related Opportunity after API insert

**Root Cause:**
- `SBQQ__Primary__c = true` must be explicitly set on the Quote when creating via API — it is not automatically set
- `SBQQ__Opportunity__c` lookup must be populated before calling Quote API methods
- Quote status update via Apex works differently than UI — `SBQQ__Status__c` field may be locked by approval state

**Resolution Steps:**
1. For Primary Quote not linking: explicitly set `SBQQ__Primary__c = true` on insert and ensure `SBQQ__Opportunity__c` is populated
2. For status update failures: check if an active approval process locks the status field; approval must be withdrawn before status update
3. For null pointer on API: log the full `QuoteModel` JSON to identify which field is null before passing to API

**Real Case Examples (OrgCS):**
- "CPQ API throwing Null pointer exception" (32.1 days TTR)
- "Quote status is not updating via automation but works manually" (23.5 days TTR)
- "Primary Quote Exists but not linked to the related Opportunity" (29.1 days TTR)

---

### Pattern 15: SBQQ__RequiredBy__c Not Populated via clone()

**Frequency:** Low-Medium
**TTR Impact:** Avg 29 days
**Area:** Apex / Bundle Cloning

**Symptoms:**
- `SBQQ__RequiredBy__c Field Not Cloned or Populated on SBQQ__QuoteLine__c via clone()`
- Bundle child lines lose their parent reference after programmatic clone
- Quote appears correct in UI but bundle hierarchy is broken after API insert

**Root Cause:**
By design, Salesforce's `SObject.clone()` method does not clone relationship fields like `SBQQ__RequiredBy__c` — this is a platform behavior. CPQ's managed package does not override this.

**Resolution Steps:**
1. After cloning Quote Lines: explicitly copy `SBQQ__RequiredBy__c` from original lines to cloned lines using a Map:
   ```apex
   Map<Id, Id> oldToNewLineId = new Map<Id, Id>();
   // populate from insert result...
   for (SBQQ__QuoteLine__c line : clonedLines) {
       if (line.SBQQ__RequiredBy__c != null) {
           line.SBQQ__RequiredBy__c = oldToNewLineId.get(line.SBQQ__RequiredBy__c);
       }
   }
   update clonedLines;
   ```
2. Alternatively, use the CPQ Quote API's `cloneQuote` method which handles this automatically

**Real Case Example (OrgCS):**
- "SBQQ__RequiredBy__c Field Not Cloned or Populated on SBQQ__QuoteLine__c via clone()" (29.6 days TTR)

---

### Pattern 16: Managed Apex Trigger Failing — Orders Stuck in Draft

**Frequency:** Medium
**TTR Impact:** Avg 22 days
**Area:** Orders / Managed Package Trigger

**Symptoms:**
- `Managed Apex Trigger Failing - Orders Stuck in Draft Status`
- Orders not transitioning from Draft to Activated
- Custom code calling `Order.Status = 'Activated'` fails silently or throws

**Root Cause:**
- CPQ's managed Order trigger requires specific preconditions before an order can be activated (Quote must be Primary, all required fields populated)
- Custom Apex activating orders bypasses CPQ's UI validation — missing fields cause the managed trigger to fail
- If `SBQQ__Contracted__c` was set to true before activation, the order state machine is in an invalid state

**Resolution Steps:**
1. Check Order record for required fields: `SBQQ__Quote__c`, `SBQQ__Contracted__c = false` before activation attempt
2. Verify the related Quote is `SBQQ__Primary__c = true` and `SBQQ__Ordered__c = false`
3. Set `Order.Status = 'Activated'` only — do not set `SBQQ__Contracted__c = true` in the same DML; let CPQ's trigger handle contracting
4. Check for validation rules on Order that block activation

**Real Case Example (OrgCS):**
- "Managed Apex Trigger Failing - Orders Stuck in Draft Status" (22.9 days TTR)

---

## C. Setup & Developer Configuration Guide

### CPQ API Developer Checklist
- [ ] Integration user has `Salesforce CPQ` permission set assigned
- [ ] Integration user has `CPQ Integration` PSL if doing API-level operations
- [ ] `Calculate Immediately` is **unchecked** if Flow/Apex inserts Quote Lines
- [ ] All Quote Line DML is in a single transaction boundary (no split inserts/updates)
- [ ] `SBQQ.QuoteAPI.calculate()` called explicitly after `addProducts()`
- [ ] `SBQQ__Primary__c = true` set explicitly when creating Quote via API
- [ ] `SBQQ__Opportunity__c` populated before calling any Quote API method

### Common Developer Misconfigurations

| Misconfiguration | Symptom | Fix |
|---|---|---|
| `Calculate Immediately` enabled with Flow DML | UNABLE_TO_LOCK_ROW | Uncheck in Package Settings; bulkify all DML to last step |
| QCP with unused Long Text Area fields | Apex heap size too large | Remove unused fields from QCP Quote Line Fields section |
| Custom queueable jobs chained with CPQ | Too many queueable jobs | Use `Database.executeBatch` instead; deconflict with CPQ's internal jobs |
| `addProducts()` called mid-transaction | Uncommitted Work Pending | Ensure it is the first DML in the transaction |
| `SBQQ__RequiredBy__c` not re-mapped after clone | Broken bundle hierarchy | Explicitly re-map using old-to-new line ID map after clone |
| Apex trigger calling `ApexPages.addMessage` | Fails in API/async context | Wrap with `if (ApexPages.currentPage() != null)` check |
| Heroku Named Credential stale after Hyperforce migration | Connection timeouts | Update Named Credential URL; re-authorize calculation service |

---

## D. Licensing & Developer Entitlements

### Developer Support Scope

| License / Entitlement | What it Covers |
|---|---|
| Salesforce CPQ (standard) | Config + dev support for SBQQ managed package |
| CPQ Integration PSL | API-level operations, integration user access |
| Premier / Signature | QCP code review up to **200 lines** |
| Developer Support queue | Cases requiring Apex/code-level investigation — this skill's primary scope |

### Permission Set Requirements for Integrations
- Integration user must have `Salesforce CPQ` permission set — grants access to all SBQQ__ objects
- `CPQ Integration` PSL required for API operations that bypass UI
- After CPQ 228+ upgrade: profiles cannot grant SBQQ__ permissions — must use permission sets

---

## E. Documentation Quick-Reference

| Topic | Link |
|---|---|
| CPQ Developer Guide (API) | https://developer.salesforce.com/docs/atlas.en-us.cpq_dev_guide.meta/cpq_dev_guide/ |
| CPQ Package Install | https://install.steelbrick.com/# |
| Large Quote Performance | https://help.salesforce.com/s/articleView?id=sales.cpq_large_quote_performance.htm&type=5 |
| Transient Variables for Heap | https://help.salesforce.com/s/articleView?id=000385712&type=1 |
| QCP (Quote Calculator Plugin) Guide | https://help.salesforce.com/s/articleView?id=sf.cpq_qcp.htm&type=5 |
| CPQ + Profile Permissions (228 upgrade) | https://help.salesforce.com/s/articleView?id=000389406&type=1 |
| MDQ Guidelines | https://help.salesforce.com/s/articleView?id=sf.cpq_mdq_guidelines_general.htm&type=5 |
| CPQ Trailhead Dev Org | https://trailhead.salesforce.com/promo/orgs/cpqtrails |

### Key Developer Behavioral Rules
- CPQ Quote API (`SBQQ.QuoteAPI`) requires a **clean transaction** — no uncommitted DML before `addProducts()`
- `SBQQ__RequiredBy__c` is **never cloned** by `SObject.clone()` — always re-map manually
- Calculations inside QLE = **synchronous, no Heroku** callout. Calculations outside QLE = **async, Heroku callout** — behaviors differ
- CPQ managed package logs require `SBQQ` namespace at `FINEST` — standard Apex logs miss the internals
- `QueueableCalculatorService` fires on **any** Quote Line insert/update — design custom code to avoid competing DML
- Premier QCP review limit: **200 lines**. State this upfront on Sev 1 cases with large QCPs

---

## F. Escalation Paths

### When to Escalate to Engineering (GUS Investigation)
- Error stack trace shows only SBQQ namespace (no customer code)
- Reproducible with all custom triggers, flows, and QCP disabled
- Started after a specific CPQ package version upgrade (document exact version)
- Multiple customers reporting the same error in the same release window
- High TTR (>10 days) with no workaround identified

### What to Gather Before Escalating
- [ ] Full CPQ managed package debug log (SBQQ namespace at FINEST)
- [ ] CPQ package version (exact: e.g., 258.0)
- [ ] Org ID (production + sandbox where reproducible)
- [ ] Steps to reproduce with specific record IDs
- [ ] Confirmed: reproducible with QCP disabled?
- [ ] Confirmed: reproducible with all custom triggers/flows disabled?
- [ ] Confirmed: which version introduced the issue?
- [ ] Internal org reproduction attempt (yes/no, result)

### Channel Routing

| Issue Type | Channel |
|---|---|
| AMER CPQ Dev escalation | #support-rev-dev-amer (C0275QG40LE) |
| IST/EMEAPAC CPQ Dev escalation | #support-rev-dev-emeapac (C021MMRBRR6) |
| Technical / cross-team escalations | #support-rev-technical (G01GF9PFUH3) |
| Salesforce Billing Apex | #C026Y0ENETH |
| Revenue Cloud Advanced (RCA/RCB) | #C05TFMXB3RN |
| License / permission issues | #C020Y1K42F8 |

### Key SMEs
- **Apoorva Chava / Vinay Kumar K** — CPQ dev support (AMER), Splunk log analysis, null pointer on contracting
- **Jhansi Vipulasri Chodisetti** — CPQ dev support (AMER), Sev 1 assistance
- **Conall OMalley** — CPQ dev escalations (EMEAPAC), managed package internals, GUS triage
- **Arpit Kaushik** — Calculation service auth / OAuth issues (EMEAPAC)
- **Bart Lis** — QCP behavior, CPQ API design decisions, amendment edge cases
- **Sam Faulkner / Rashi Saraswat** — Dev support queue management (AMER)

---

## G. Tribal Knowledge & Pro Tips

1. **Always capture the CPQ managed package log, not the standard Apex log.** Set `SBQQ` namespace to `FINEST`. Standard logs miss all managed package internals — this is the #1 reason dev cases take long. (#support-rev-dev-emeapac, ongoing)

2. **UNABLE_TO_LOCK_ROW almost always means competing DML.** Before spending hours on root cause analysis, check if `Calculate Immediately` is enabled and if the Flow/trigger has non-bulkified DML. Fixing those two things resolves 80% of lock row cases. (#support-rev-dev-amer)

3. **QCP code review limit is 200 lines for Premier.** Tell the customer this upfront on any Sev 1 with a large QCP. If they have 2,000 lines, we can only review 200 — they need to isolate the failing method first. (#support-rev-config-emeapac — Bhuvaneswari Betala, May 2026)

4. **Calculation inside QLE vs. outside QLE are completely different code paths.** Inside QLE = synchronous, no Heroku. Outside QLE (Quote layout Calculate button, async) = Heroku callout. If an error only appears outside QLE, suspect Heroku/calculation service, not CPQ logic. (#support-rev-config-emeapac, May 2026)

5. **`SBQQ__RequiredBy__c` is never auto-cloned.** Customers using Apex to clone Quote Lines almost always hit broken bundle hierarchies because of this. Point them to explicitly re-map using a Map<Id,Id> of old-to-new line IDs. (#support-rev-dev-amer)

6. **For Hyperforce migrations: always re-check the calculation service auth.** Named Credentials for the Heroku endpoint may have stale URLs. Re-authorize from scratch: revoke Connected App → clear Custom Settings token → re-authorize. (#support-rev-dev-emeapac — Arpit Kaushik)

7. **`Uncommitted Work Pending` in CPQ API means DML happened before `addProducts()`.** The CPQ API requires a completely clean transaction. Even a single DML before the API call causes this. The fix is always: make `addProducts()` the first DML in the transaction. (#support-rev-dev-amer)

8. **Too Many SOQL errors that start on a specific package version are GUS bugs.** If a customer is on v256 and it fails but v258 passes (or vice versa), document the exact versions and raise a GUS investigation — this is not a customer code issue. (#support-rev-dev-amer)

---

## H. Active Known Issues (as of May 2026)

| Issue | GUS / KI | Workaround |
|---|---|---|
| Product selection checkboxes blocked in Console App (post Summer '26) | W-22449442, W-22488848 / KI: a02Ka00000mGGGF | Enable Split View in Console App |
| Licensed Objects error editing profiles in Sandboxes | W-22439807, W-22453566 / KI: a02Ka00000mGAxyIAG | None — track KI |
| Summary Variables not recalculating on OOB renewal creation | KI: a028c00000uXaL6AAK | Manually recalculate renewal quote after creation |
| Quote Line Group Actions not rendering when first action has unmet conditions | GUS Bug W-SBQQ8SpHvIAK | Reorder custom actions so first action has no unmet conditions |

---

## I. Code Reference Map

> Connect CodeSearch to verify file paths. Key managed package classes referenced in developer cases:

| Class / Component | Pattern | Notes |
|---|---|---|
| `SBQQ.QueueableCalculatorService` | Patterns 1, 2, 10 | Async CPQ calculator; fires on any Quote Line DML |
| `SBQQ.OrderItemObjectManager.validateDateChanges` | Pattern 6 | Date validation during amendment contracting |
| `SBQQ.OrderProductDAO.updateTerminatedDateOfOrderProducts` | Pattern 6 | Sets TerminatedDate on order products during contracting |
| `SBQQ.QuoteAPI.addProductsToQuote` / `calculate` | Pattern 13 | CPQ public API entry points |
| `SBQQ.CaseDebookService` (Queueable) | Pattern 10 | Bundle amendment processing — hits stack depth limit |
| `CXO_CPQAPIModels.QuoteSaver.save` | Pattern 9 | Quote save orchestration; heap issues here |
| `SBQQ.JSQC` (JavaScript Quote Calculator) | Pattern 5 | QCP host inside QLE — exception here surfaces as `sbqq__JSQC script exception` |
| `SBQQ.QuoteService` | Pattern 1 | Throws `unable to obtain exclusive access` on row lock |

## Skill Completeness Report

## Scope Limits

This skill **cannot resolve:**
- Pure CPQ configuration issues (price rules setup, product rules, quote templates) — route to config channels
- QCP custom code debugging beyond identifying the failing method — customer is responsible for their custom code
- Issues in PROS Smart CPQ, AvaTax for CPQ, or 3rd-party CPQ products not at install.steelbrick.com
- Revenue Cloud Advanced (RCA/RCB) Apex issues — route to #C05TFMXB3RN
- Salesforce Billing Apex — route to #C026Y0ENETH
- Issues requiring Splunk log access — only Dev Support engineers can access Splunk
- Full QCP code reviews beyond 200 lines (Premier plan limit)