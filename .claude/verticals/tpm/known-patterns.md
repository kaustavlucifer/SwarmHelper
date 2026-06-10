## A. Triage Decision Tree

**Step 1 — Which functional area?**

```
Customer reports a TPM problem
│
├── KPIs not updating / wrong values on UI
│     ├── After Nightly Calculation → Pattern 1: NC Batch Failure
│     ├── After push promotion → Pattern 3: Push Promotion / Child Spend
│     ├── P&L ≠ Hyperforce values → Pattern 5: Hyperforce KPI Sync
│     └── Manual input showing wrong value → Pattern 4: EditableKPI NaN Bug
│
├── Batch job failure or timeout
│     ├── CONFIGURATION_ERROR: Unknown reference level → Pattern 1
│     ├── CGT_Calculation_Batch / GCPCalculationBatch failure → Pattern 2
│     ├── ProcessQueueWorker_UNKNOWN with no error detail → Pattern 1
│     └── FlattenAccountHierarchyBatch 0 records → Pattern 9
│
├── Push Promotion issues
│     ├── Child promotions not created → Pattern 3
│     ├── Child spend reset to zero after push → Pattern 3
│     ├── Push very slow (5+ min for 40 accounts) → Pattern 3 (use OffPlatform mode)
│     └── Manual input values wrong on child after push → Pattern 4
│
├── Payment / Claim issues
│     ├── Payment stuck in ToBeClosed → Pattern 6
│     ├── NoGrossAccrualAgainsCreditRLIPayment error → Pattern 6
│     └── Incorrect payment amount reading → Pattern 6
│
├── Promotion record load failure (UI error)
│     ├── "Apex heap size too large" → Pattern 7
│     ├── "Apex CPU time limit exceeded" → Pattern 7
│     └── Promotion Info Card missing on first load (F5 fixes it) → Pattern 8
│
├── Licensing / access errors
│     ├── "Oops! Something went wrong" on Spend Planning Card → Pattern 10
│     ├── User can't see TPM objects → Pattern 10
│     └── KPI Map not populating tactic fund records → Pattern 10
│
├── Agentforce for TPM errors
│     ├── "report timeframe configuration is outside supported range" → Pattern 11
│     ├── "User does not have read access to at least one account" → Pattern 11
│     └── Agent not fetching account measures → Pattern 11
│
├── RTR Report / API data issues
│     ├── splitweeks=true causing missing KPI data → Pattern 12
│     ├── Monthly data inconsistencies → Pattern 12
│     └── API 500 error creating promotions → Pattern 13
│
└── Funding Grid not loading for Brand-level products → Pattern 14
```

**Quick Diagnostic Questions:**
1. What is the customer's CG Cloud package version? (e.g., 256.7, 258.x, 260.x, 262.x)
2. What is the Sales Org ID / Org ID?
3. Is this on Hyperforce or Classic infra?
4. What is the Batch Run Status record ID (if a batch failed)?
5. What is the exact error message (not paraphrased)?
6. When did this last work correctly?

---

## B. Known Issue Patterns

### Pattern 1: Nightly Calculation (NC) Batch Failure — CONFIGURATION_ERROR

**Frequency:** Very High (most common TPM case type)
**Severity:** Typically Signature/Premier (impacts dozens–hundreds of promotions)
**TTR Impact:** 3–7 days

**Symptoms:**
- `CONFIGURATION_ERROR: Unknown reference level = "ForecastingUnit" found while processing "<KPI_RULE_NAME>"`
- `CONFIGURATION_ERROR: Unknown reference level = "Package" found while processing "<KPI_RULE_NAME>"`
- Batch state shows `Error` in `cgcloud__Batch_Run_Status__c`
- `ProcessQueueWorker_UNKNOWN` job with no visible error in BRS Details
- KPIs not recalculating with latest values for 50–300 promotions
- Customers see stale promotion data in Spend Planning card

**Root Cause:**
The KPI Set configuration references a product hierarchy level (e.g., "ForecastingUnit", "Package") that no longer exists or was renamed in the org's product catalog. The off-platform processing service cannot resolve the reference and aborts the entire batch job chain for that sales org. This is a **configuration error**, not a product bug.

**Resolution Steps:**
1. Get the failing `kpisetid` and `promotionid` from Splunk logs or BRS Details
2. Navigate in the customer org: Setup → KPI Sets → open the KPI Set with that ID
3. Find the KPI rules referencing the unknown level (e.g., "ForecastingUnit")
4. Either: (a) Update the KPI rule to use the correct current level name, OR (b) remove the rule if it's no longer needed
5. Run **Update Configuration** on the KPI Set to push the change to the processing service
6. Re-trigger the Nightly Calculation manually for the affected sales org
7. Verify BRS shows `Done` state

**Splunk Query for investigation:**
```
(index=distapps) substrate=aws k8s_cluster=sam-processing1 k8s_namespace=rcg functional_domain=industries organizationId="<ORG_ID>*" jobChainId=<NC_JOB_CHAIN_ID> logLevel=error
```

**Verification:**
- BRS record for the next NC run shows `Done`
- KPIs on impacted promotions show updated values

**Escalation Criteria:**
- If the KPI Set config appears correct and the error persists after Update Configuration → escalate to R&D with Splunk logs + KPI Set export
- GUS reference: [W-22335677](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002ZHXUCYA5/view) (CGT Calculation batches not executing — resolved with internal tools)

---

### Pattern 2: CGT/GCP Calculation Batch Not Executing

**Frequency:** High (Signature customers, large orgs like P&G, Unilever)
**Severity:** Signature/Premier
**TTR Impact:** 2–5 days (resolved with internal tools)

**Symptoms:**
- `CGT_Calculation_Batch_GCPS`, `GCPCalculationBatch`, `CGT_CalculationBatch` not executing successfully
- Error logged in `cgcloud__Batch_Run_Status__c` on specific sales orgs
- Promotion P&L showing stale data in C00G and U60G style org codes

**Root Cause:**
GCP (Gross Contribution to Profit) calculation chain fails when the processing service worker queue is backlogged or when there are stuck/zombie jobs from previous failed runs in the customer's org.

**Resolution Steps:**
1. Check BRS for the org — identify which job in the chain first failed
2. Use internal tooling to **abort/cancel stuck NC jobs** (requires break-glass or IS tooling access)
   - Reference: [W-19096387](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE000029GoKKYA0/view) — Hyperforce job abort procedure
3. Verify no zombie jobs remain via Splunk
4. Re-trigger the full calculation chain from the beginning (Reorganization → AccountProductList → AccountPlanBasic → Promotion → BusinessPlan → RBF → Fund)
5. Monitor until all sub-jobs show `Done`

**Splunk Dashboard:**
```
https://splunk-web-noncore.log-analytics.monitoring.aws-esvc1-useast2.aws.sfdc.cl/en-US/app/publicSharing/cgcloud_processing_services__job_chain_jobs_scheduling_and_worker_processing
```
Parameters: `orgId=<ORG_ID>`, `jobChainId=<NC_JOB_CHAIN_ID>`

**Escalation Criteria:**
- Requires break-glass / IS-level access to abort stuck jobs — open internal incident if you cannot abort via standard tooling

---

### Pattern 3: Push Promotion — Child Spend Reset / Not Created / Slow

**Frequency:** Very High (recurring across Unilever, CCEP, P&G, Mars, BRF)
**Severity:** Premier/Signature
**TTR Impact:** 3–10 days

**Sub-patterns:**

**3a — Child promotions not created after push shows "successful":**
- Symptoms: BRS shows push completed, but no child `cgcloud__Promotion__c` records created; plan spend amount was 0
- Resolution: Verify "Tactics" is included in the **Copied Components** attribute on the Promotion Template Hierarchy. If not present, push will succeed but create empty children.
- Reference: [W-21746930](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002WwlUrYAJ/view)

**3b — Child spend reset to zero after push (Lump Sum tactics):**
- Error: "Deal Rate in UOM is getting copied at header level for child promotions for Lump Sum Tactic"
- Symptoms: After push, child `Planned Promo Spend` becomes 0 or incorrect; `cgcloud__Deal_Rate_UOM__c` incorrectly populated
- Resolution: Check the tactic type's compensation model — for Lump Sum, ensure Deal Rate UOM is NOT set at the header level before push. This is a known behavior. Workaround: clear the Deal Rate UOM field on parent tactic before pushing; re-push.
- GUS: [W-22137223](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002YVvk2YAD/view), [W-22193629](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002YhdoGYAR/view)

**3c — Child promotion spend values reset to zero in Ongoing/Amendment phase:**
- Root cause: Re-push in Ongoing phase resets manual inputs and spend on children. Working as designed — child promotions "retain the link to the parent and can't be edited." Changes must be made on parent and re-pushed.
- Escalation if customer insists it worked differently before: check if the Promotion Template was recently modified

**3d — Push is extremely slow (5+ minutes for 40 accounts):**
- Root cause: Legacy Apex Batch-based push mode. Expected with legacy version.
- **Fix (package ≥ 260):** Create a System Setting custom setting:
  - Name: `PushPromotionExecutionMode`
  - Value: `OffPlatform`
  - This enables off-platform push — handles up to 10,000 child promotions concurrently
  - Doc: https://help.salesforce.com/s/articleView?id=ind.tpm_task_enable_push_promotions_dedicated.htm&type=5
- Also ensure the Promotion Calculation Batch user has permission to any custom fields on promotions

**3e — Parent/child KPI discrepancy after push (11K+ difference in Sell-in):**
- Symptoms: Sum of child KPIs ≠ parent KPI in Spend Planning; Hyperforce shows correct parent value
- Resolution: Run REORGANIZATION, then Push, then RECALCULATION in that order. If discrepancy persists, check `cgcloud__Batch_Run_Status__c` for aggregation errors. The KPI writeback from Hyperforce to Salesforce Core may have a lag.
- GUS: [W-21627818](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002WQkNJYA1/view)

---

### Pattern 4: EditableCalculated KPI Manual Input — NaN / Incorrect UI Values

**Frequency:** High (recurring Luigi Lavazza, Mars Pet Nutrition)
**Severity:** Premier
**TTR Impact:** 2–5 days

**Symptoms:**
- Manual input values on Spend Planning Card show **different values than what was saved**
- Blank cells or 0 displayed when `applyFixedTotals=true`
- After push promotion, manual input on child promotions shows incorrect value
- Active bug: [W-22597542](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002aYZfYYAW/view) — `EditableCalculated KPI manual input renders blank cells due to NaN cascade when applyFixedTotals=true` (Status: **New P3 Bug**)

**Root Cause:**
When `applyFixedTotals=true` is set on the KPI definition, a NaN value in one cell cascades through the total calculation, rendering sibling cells blank. Separately, after push, the manual input field stores the pushed value but the UI renders a recalculated value before the writeback completes.

**Resolution Steps:**
1. Confirm `applyFixedTotals` setting on the KPI definition
2. If `applyFixedTotals=true`: Try setting it to `false` as a workaround and re-testing
3. For post-push incorrect values: trigger a manual save on the child promotion to force recalculation and writeback
4. For the "Promotion Information Card missing on first navigation" variant (fixes after F5): this is a navigation/caching issue — resolved without code change per [W-22571462](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002aQ688YAC/view). Workaround: navigate away and back, or press F5.
5. If none of the above resolve: log against W-22597542 as a known bug

---

### Pattern 5: Hyperforce KPI Sync — UI Values Don't Match Calculation Engine

**Frequency:** High
**Severity:** Standard–Signature
**TTR Impact:** 2–7 days

**Symptoms:**
- Salesforce Core UI shows different KPI values than Hyperforce calculation engine
- "UI values in P&L not matching cgps value" (Just Born, Inc. — [W-21970022](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002XomAWYAZ/view))
- KPI values "not getting synced to Hyperforce" ([W-22668967](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002asem9YAA/view))
- Trade Liability report shows stale data; live promotion shows updated data

**Root Cause (from Slack — Himanshu Sekhar Sahoo, #tmp-help-consumer-goods):**
Standard Salesforce reports do NOT query the calculation engine directly. They rely on **KPI Maps** to write calculated values back to standard Salesforce fields. The writeback only fires when the promotion record is saved. Simply attaching a claim does NOT trigger a promotion save, so KPI maps don't fire. Additionally:
- The backend may retain **legacy read code** (old dailymeasurereal/weeklymeasurereal table names) conflicting with new writeback logic
- Only **approved claims** are considered for P&L calculations

**Resolution Steps:**
1. Check `cgcloud__Batch_Run_Status__c` for the SFDataSyncWorker batch — look for Error state around the time the discrepancy started
2. If BRS shows error: identify the error in BRS Details, fix root cause (usually legacy read code or config), run **Update Configuration** on KPI Set
3. If BRS shows success but values still wrong: manually save/touch the Promotion record to trigger a live calculation and KPI Map writeback
4. For legacy read code conflict: remove old `dailymeasurereal`/`weeklymeasurereal` read code entries from the KPI Set, run Update Configuration, re-trigger NC batch
5. Verify using `cgcloud__Batch_Last_Successful_Run__c` — check the `cgcloud__Last_Successful_Run__c` timestamp

**Key diagnostic SOQL:**
```soql
SELECT Id, Name, cgcloud__Batch_State__c, cgcloud__Start_Date__c, cgcloud__End_Date__c
FROM cgcloud__Batch_Run_Status__c
WHERE cgcloud__Sales_Org__c = '<SALES_ORG_ID>'
ORDER BY cgcloud__Start_Date__c DESC
LIMIT 20
```

---

### Pattern 6: Payment / Claim Lifecycle — Stuck in ToBeClosed / NoGrossAccrual Error

**Frequency:** High (Duracell, Mondelēz, P&G)
**Severity:** Premier/Signature
**TTR Impact:** 3–10 days

**Sub-patterns:**

**6a — Payment stuck in "ToBeClosed" status:**
- Symptoms: Payment set to ToBeClosed, core closure job ran, status did NOT change to Closed; remains at ToBeClosed or jumps to Approved
- Root cause: Custom Apex trigger setting `ToBeClosed` before Claim Tactic records are inserted (SAP integration pattern). The core closure process requires claim tactics to exist BEFORE the status transition.
- Key diagnostic question: "Does the customer have a custom trigger that sets status to ToBeClosed on payment INSERT?" — if yes, this is the cause.
- Resolution: Modify the custom trigger to set ToBeClosed AFTER claim tactic records are created, not during insert. Alternatively, set status via a scheduled job after tactic creation.
- Reference: [W-22174491](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002YdZSoYAN/view) (Duracell — Payments Stuck in ToBeClosed)

**6b — "NoGrossAccrualAgainsCreditRLIPayment" error:**
- Symptoms: Resolution Line-Item records not resolving; error thrown on processing
- Root cause: PPAM (Promotion Payment Accrual Method) records not generated — this is done by core and requires the claim tactic records to be present
- Resolution: Ensure claim tactics are attached before triggering payment resolution. Verify the payment record has tactic payouts assigned.
- GUS: [W-21647010](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002WWS2eYAH/view) — resolved with internal tools

**6c — Incorrect payment amount calculated:**
- GUS: [W-22038297](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002YASJdYAP/view) — P&G ATPMI — Incorrect reading of payment amount → new bug logged
- If calculation is wrong but UI shows correct tactic payouts: check UPDATEPAYMENTONSF and PAYMENTCALCULATION batch status in BRS

---

### Pattern 7: Apex Governor Limits on Promotion Load — Heap / CPU

**Frequency:** Medium (large P&G-scale promotions)
**Severity:** Swarm/Signature
**TTR Impact:** Closed as "Working as Documented" — document limits

**Symptoms:**
- `"Apex heap size too large"` error when opening a specific promotion
- `"Apex CPU time limit exceeded"` error on promotion page load
- Affects promotions with 1,000+ products across 14+ tactics (P-00011152 pattern)
- `FlattenAccountHierarchyBatch` → CPU time exceeded with 237,000+ Account records

**Root Cause:**
The promotion loading process fetches all tactic-product combinations into heap memory on the Salesforce Core side. With very large promotions (1,493 products, 80 display components), this exceeds the 12MB heap limit. This is a platform governor limit, not a bug.
- Investigation: [W-21930828](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002XboySYAR/view) — Closed: No Fix, Working as Documented

**Resolution / Workarounds:**
1. Inform customer this is a known platform limit — enforce the **soft limit for number of products in CBP** (Customer Business Planning): [W-21336199](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002VFFluYAH/view) — P1 bug to update the limit
2. Reduce number of products per promotion tactic (split into multiple promotions or tactics)
3. For `FlattenAccountHierarchyBatch` CPU issue: reduce batch scope by scheduling during off-peak, or check `Number_Of_Extensions__c > 0` filter is working (first query returns rows but second returns 0 = `cgcloud__Account_Extensions__r` not populated correctly)
4. Do NOT increase batch size — reduce it to process fewer records per transaction

**Escalation:** Only if within documented limits and still hitting the error → raise P2 bug against CBP soft limit enforcement

---

### Pattern 8: Promotion Information Card Missing on First Navigation

**Frequency:** Medium
**Severity:** Premier
**TTR Impact:** 1–2 days (resolved without code change)

**Symptoms:**
- Promotion record page loads, but the Promotion Information Card is blank/missing
- Pressing F5 (page refresh) makes it appear
- Happens only on "first navigation from list view"
- GUS: [W-22571462](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002aQ688YAC/view) — Closed: Resolved Without Code Change

**Root Cause:**
LWC component navigation caching issue — the card's LWC component does not fully initialize on the first client-side navigation from a list view. Hard refresh forces a full page load.

**Resolution:**
1. Workaround (immediate): Press F5 or navigate away then back
2. Check if the customer is on a very old package version — upgrade may fix
3. Clear browser cache and re-test
4. If reproducible consistently: collect HAR file and LWC console errors, escalate to R&D

---

### Pattern 9: FlattenAccountHierarchyBatch — 0 Records Processed

**Frequency:** Medium
**Severity:** Standard
**TTR Impact:** 2–5 days

**Symptoms:**
- `ScheduleCGCloudServFlattenAccHierarchy` runs, shows 0 batches processed despite accounts existing
- Logs: First SOQL returns 714 rows (accounts with `Number_Of_Extensions__c > 0`), second SOQL returns 0 rows
- All jobs downstream of FlattenAccountHierarchy (account plan hierarchy, store hierarchy) also fail

**Root Cause:**
The second query uses `where Id in :fromAccounts` — the `fromAccounts` set is populated by reading `cgcloud__Account_Extensions__r` child records, but these records don't exist or the relationship isn't populated (customer never ran initial data load for Account Extensions).

**Resolution Steps:**
1. Verify `cgcloud__Account_Extensions__c` records exist for the accounts — query: `SELECT COUNT() FROM cgcloud__Account_Extension__c WHERE cgcloud__Account__c IN (SELECT Id FROM Account WHERE Number_Of_Extensions__c > 0)`
2. If count is 0: run the **Account Extension data load** process first (Admin → CG Cloud Setup → Account Extension)
3. After data load: re-run `ScheduleCGCloudServFlattenAccHierarchy`

---

### Pattern 10: Licensing / Permission Set Errors

**Frequency:** High (especially on new orgs or new users)
**Severity:** Standard–Signature
**TTR Impact:** 1–3 days

**Symptoms:**
- "Oops! Something went wrong, please contact your system administrator" on Spend Planning Card
- User can see some TPM objects but not others
- KPI Map not populating values on Tactic Fund record
- PSL assigned but issue persists
- Error: `"user does not have read access to at least one account and category provided in the request"`

**Required Permission Sets / Licenses (all must be present):**
1. **Org-level permission:** `OrgHasTradePromotionMgmt` (provisioned by Salesforce — check in Setup → Company Information)
2. **User PSL:** `Lightning Trade Promotion Management` PSL (from `cgcloud` namespace)
   - Confirmed by Sravani Gajula: "in order to make this work they need to have the Lightning Trade Promotion Management Psl"
   - Confirmed by Patricia Castillo Perez: "if they don't have Lightning Trade Promotion Management Psl, they don't have TPM"
3. **For Agentforce for TPM users:** Additionally need:
   - `TPM Master Data Admin`
   - `TPM Standard Object Admin`
   - `Access CGCloud AI Assistive Agent`
4. **Code reference** (`LicenseCheckUtils.cls` in rcg-retail-tpm):
   - `TPM_ORG_PERM = 'OrgHasTradePromotionMgmt'`
   - `TPM_USER_PERMISSION = 'PermissionsCGCTradePromotionManagement'`

**Resolution Steps:**
1. Check org permission: `SELECT PermissionsOrgHasTradePromotionMgmt FROM Organization LIMIT 1` — if false, submit a license provisioning request
2. Check user PSL: Setup → Permission Set Licenses → "Lightning Trade Promotion Management" → Assignees
3. Check individual permission set assignment: Setup → Users → [user] → Permission Set Assignments
4. For Spend Planning Card specifically: verify tactic has `$ off` or `% off` in its Compensation Model picklist — if only "Fixed Fees", certain KPIs that depend on Compensation Model will fail
5. If PSL is present but error persists: check if `cgcloud__CGC_Trade_Promotion__c` standard object permissions are granted

---

### Pattern 11: Agentforce for TPM — Account Measures Retrieval Failures

**Frequency:** Medium (emerging — P&G, Just Born orgs)
**Severity:** Support/Signature
**TTR Impact:** 3–7 days

**Symptoms:**
- `"The report timeframe configuration is outside supported range"` (HTTP 400)
- `"User <userId> does not have read access to at least one account and category provided in the request"` (HTTP 400)
- Agent retrieves some measures but fails for specific accounts/products
- "Measures for product not found. Product might not have measures or might belong to an invalid product level."
- Error appears in the `Retrieve Account Measures` action

**Root Cause:**
- The **timeframe error** occurs when the Business Year referenced in the account plan is outside the configured reporting window for the KAM processing service
- The **access error** means the KAM user running the agent action lacks the `Account Measure Reader` profile permission or the account/category record is not in the user's access scope
- The **product level error** occurs when the prompt specifies a product at a hierarchy level (Brand/Flavor) that the KAM action doesn't support for direct measure retrieval
- Related GUS: [W-22262501](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002YzrGUYAZ/view) — Agentforce KAM failing on retrieving account measures → new bug logged

**Resolution Steps:**
1. Verify package version ≥ 256.7 (minimum for Agentforce for TPM); 260.2 preferred
2. Assign required PSLs to the user:
   - `TPM Master Data Admin`
   - `TPM Standard Object Admin`
   - `Access CGCloud AI Assistive Agent`
3. For timeframe error: check that a valid Business Year record exists in the org for the fiscal year in the prompt; ensure Business Year configuration covers the requested date range
4. For access error: verify the user's **Promotion Access Definition Policy** — "Combined Anchors" is the correct setting
5. For product level error: the agent only supports SKU/product-level queries, not Brand or Flavor levels — inform customer to prompt with specific product names
6. Check Splunk for KAM service logs:
   ```
   (index=distapps) substrate=aws k8s_cluster=sam-processing1 k8s_namespace=rcg functional_domain=industries organizationId="<ORG_ID>*" logLevel=error
   ```
7. Reference work: [W-19090936](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002IaIBtYAN/view)

---

### Pattern 12: RTR Report / splitweeks Configuration Data Discrepancies

**Frequency:** Medium (Corrao Group, Mondelēz)
**Severity:** Standard
**TTR Impact:** 3–7 days

**Symptoms:**
- `splitweeks=true` in RTR Report Configurations causes missing KPI data
- CSV output when `splitweeks=true` differs from `splitweeks=false`
- RTR Monthly Integration Report showing inconsistent monthly data
- GUS: [W-22512248](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002a7UGvYAM/view) (still open — More Info Reqd)

**Root Cause:**
When `splitweeks=true`, the RTR report engine splits weekly measures into sub-period daily allocations. If a promotion spans partial weeks at month boundaries, the split produces different totals than whole-week aggregation. This is expected behavior but customers interpret it as incorrect.

**Resolution Steps:**
1. Confirm whether the customer's use case requires true daily-level data or monthly aggregation
2. If monthly reporting is the goal: set `splitweeks=false`
3. For "Period Sub-Formation" (Mondelēz — incorrect monthly revenue): this is documented behavior for promotions that span partial sub-periods — [W-19455806](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002KuvUhYAJ/view) closed as Will Not Fix
4. Provide documentation: RTR Report Configuration article in Salesforce Help

---

### Pattern 13: REST API 500 Error Creating Promotions

**Frequency:** Low-Medium
**Severity:** Signature
**TTR Impact:** 1–3 days (Doc/Usability)

**Symptoms:**
- REST API endpoint for Promotions creation returns HTTP 500
- GUS: [W-22506810](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002a5xQ5YAI/view) — Closed: Doc/Usability (P&G ATPMI)

**Root Cause:**
Customer using incorrect endpoint format or missing required fields. The TPM REST API for promotion creation requires specific fields that differ from standard Salesforce REST API.

**Resolution Steps:**
1. Reference the correct API documentation: Consumer Goods Cloud Developer Guide → Trade Promotion Management Custom Objects
2. Verify required fields are present: `cgcloud__Sales_Org__c`, `cgcloud__Promotion_Template__c`, `cgcloud__Valid_From__c`, `cgcloud__Valid_Thru__c`
3. Check API version — use v56.0 or later
4. Enable detailed error logging: add `?_HttpStatus=true` to the request to get detailed error body

---

### Pattern 14: Funding Grid Not Loading for Brand-Level Products

**Frequency:** Low-Medium
**Severity:** Signature
**TTR Impact:** 2–4 days

**Symptoms:**
- Funding Grid card shows spinner indefinitely for Brand or high-level product hierarchy entries
- Magnum Ice Cream Company (TMICC) pattern — [W-22412183](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002ZcLsmYAF/view) — Closed: Doc/Usability

**Root Cause:**
The Funding Grid component is designed for leaf-level (SKU/product) entries. Brand-level product nodes generate a query that returns too many children, causing a timeout before the component renders.

**Resolution Steps:**
1. Confirm the product hierarchy level of the entries causing the issue
2. If Brand/Category level: inform customer this is a known limitation — Funding Grid is supported at product/SKU level only
3. Workaround: navigate to the funding grid from a specific SKU-level product rather than brand
4. If the customer insists they need Brand-level grid: this is a feature request — log a GUS Enhancement

---

## C. Setup & Configuration Guide

### Minimum Required Setup Checklist
- [ ] `OrgHasTradePromotionMgmt` org permission enabled (provisioned by Salesforce)
- [ ] CG Cloud managed package installed (check version: recommend 260.x+)
- [ ] Sales Organization records configured (`cgcloud__Sales_Organization__c`)
- [ ] Business Year records configured (required for Agentforce and some batch jobs)
- [ ] KPI Sets configured and "Update Configuration" run for each Sales Org
- [ ] Promotion Templates configured with correct Copied Components attribute
- [ ] `Lightning Trade Promotion Management` PSL assigned to all TPM users
- [ ] Nightly Calculation scheduled (via `cgcloud.TPMCalculationChain.startProcess()`)
- [ ] KPI Maps configured to write back values to Salesforce Core fields

### Nightly Batch Setup (correct API call):
```apex
cgcloud.TPM_BC_Settings settings = new cgcloud.TPM_BC_Settings(currentSalesOrg);
settings.execReorganization = true;
settings.execAccountProductList = true;
settings.execAccountPlanBasic = true;
settings.execPromotion = true;
settings.execAccountPlanBusinessPlan = true;
settings.execRBFCalculation = true;
settings.execFundCalculation = true;
settings.batchSizePAAPLA = 200;
settings.daysAfter = 7;  // IMPORTANT: set this or 2025 account plans may be excluded
cgcloud.TPMCalculationChain tpmCalcChain = new cgcloud.TPMCalculationChain(settings);
tpmCalcChain.startProcess();
```

**Common Misconfiguration — `daysAfter` not set:** If `daysAfter` is omitted, the NC batch may exclude account plans from the current year. Always set this parameter.

### Push Promotion Mode (package ≥ 260):
```
System Settings custom setting:
  Name:  PushPromotionExecutionMode
  Value: OffPlatform
```
See: https://help.salesforce.com/s/articleView?id=ind.tpm_task_enable_push_promotions_dedicated.htm&type=5

---

## D. Licensing & Entitlements

### Decision Flowchart
```
User reports access issue
│
├── Can user see the TPM app at all?
│     NO → Check org permission OrgHasTradePromotionMgmt
│             If false → raise provisioning request to Salesforce
│
├── User sees app but features missing / "Something went wrong"
│     → Check "Lightning Trade Promotion Management" PSL assigned
│       → If not: assign from Setup → Permission Set Licenses
│
├── Agentforce for TPM not working
│     → Also need: TPM Master Data Admin + TPM Standard Object Admin
│                  + Access CGCloud AI Assistive Agent
│
└── TPM and RE on same org?
      → Possible: "we do not recommend it, but customers like Swedish Match do it"
        (Martin Burgard, #tmp-help-consumer-goods 2026-05-18)
        → May encounter cross-product permission conflicts
```

### SOQL to check org permission:
```soql
SELECT Id, Name, PermissionsOrgHasTradePromotionMgmt
FROM UserLicense
WHERE Name LIKE '%Consumer Goods%'
LIMIT 10
```

---

## E. Documentation Quick-Reference

| Topic | Link |
|-------|------|
| Agentforce for TPM | https://help.salesforce.com/s/articleView?id=ind.tpm_agentforce.htm&type=5 |
| Push a National Promotion | https://help.salesforce.com/s/articleView?id=ind.tpm_task_push_national_promotion.htm&type=5 |
| Enable OffPlatform Push | https://help.salesforce.com/s/articleView?id=ind.tpm_task_enable_push_promotions_dedicated.htm&type=5 |
| Configure Calculation Chain | https://help.salesforce.com/s/articleView?id=ind.tpm_admin_task_configure_calculation_chain.htm&type=5 |
| TPM Custom Objects (API) | Consumer Goods Cloud Developer Guide → dev_guides/retail_api |

**Key behavioral rules (non-obvious):**
- Standard Salesforce reports do NOT read from the Hyperforce calculation engine — they read KPI Map writeback values only
- KPI Maps only fire on Promotion record SAVE — attaching a claim does not trigger writeback
- Only **approved** claims are factored into P&L calculations
- Child promotions created by push CANNOT be edited independently — all changes must be made on parent then re-pushed
- Tactics are only copied to children if "Tactics" is in the Copied Components attribute on the Promotion Template Hierarchy
- `daysAfter` parameter controls how far into the future the NC batch processes — omitting it can silently exclude active account plans

---

## F. Escalation Paths

### When to Escalate (and what to collect first)

| Symptom | Escalate If | Collect Before Escalating |
|---------|-------------|---------------------------|
| NC batch CONFIGURATION_ERROR | Config is correct, error persists after Update Configuration | Splunk logs, KPI Set export, failing promotionid + kpisetid |
| CGT batches not executing | Cannot abort stuck jobs via standard tooling | BRS record IDs, org ID, Splunk job chain ID |
| Apex heap/CPU on promotion load | Within documented product limits | Promotion ID, tactic count, product count per tactic |
| Agentforce account measures | PSLs correct, Business Year exists, still 400 | txId from error, user ID, Splunk logs for KAM service |
| Payment stuck in ToBeClosed | Custom trigger ruled out, PPAM records exist | Payment ID, Claim ID, Tactic payout records, trigger code |

### GUS Engineering Team
- **Product Tag:** `Industries Consumer Goods - Trade Promotion (TPM)`
- **Eng team:** RE-Jarvis (Retail and Consumer Goods)
- **Key GUS contacts:** Search `#tmp-help-consumer-goods` for Martin Burgard, Sven Camenzind

### High-TTR Investigations to Reference
- [W-22335677](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002ZHXUCYA5/view) — CGT Calculation batch failures (P&G)
- [W-21930828](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002XboySYAR/view) — Apex heap too large on large promotions
- [W-22597542](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002aYZfYYAW/view) — **OPEN P3**: EditableCalculated KPI NaN bug
- [W-22262501](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002YzrGUYAZ/view) — Agentforce KAM failing on account measures

---

## G. Tribal Knowledge & Pro Tips

*(From #tmp-help-consumer-goods — sourced from support engineers)*

1. **KPI values show different between UI and report? Save the promotion.** Standard reports read KPI Map writeback values, not the engine directly. Touching/saving the promotion record forces a live calculation and KPI Map fire. (Himanshu Sekhar Sahoo, March 2026)

2. **Nightly Calculation failing with "Unknown reference level"? It's always a config problem.** Check the KPI Set for stale hierarchy level references. Run Update Configuration after fixing. Never a product bug. (Multiple threads, Dec 2025–Mar 2026)

3. **Push promotion slow? Check the package version.** Package ≥ 260 supports `PushPromotionExecutionMode = OffPlatform` — eliminates Flex Queue dependency, supports 10K concurrent child promotions. The legacy Apex Batch mode taking 5 minutes for 40 accounts is working as designed on older packages. (Sven Camenzind, #tmp-help-consumer-goods, March 2026)

4. **TPM and RE on the same org is not recommended but possible.** Swedish Match does it. Customers may hit cross-product permission conflicts. (Martin Burgard, #tmp-help-consumer-goods, May 2026)

5. **"Lightning Trade Promotion Management PSL" is the gate.** Without it, users get cryptic errors. Always check PSL first before deep diving. (Patricia Castillo Perez + Sravani Gajula, April 2026)

6. **Payments stuck in ToBeClosed? Ask about SAP integration triggers.** The most common cause is a custom trigger setting ToBeClosed on payment INSERT, before claim tactic records exist. Core requires tactics to process the closure. (Patricia Castillo Perez, November 2025)

7. **Funding Grid not loading for Brand-level? Feature limitation.** The Funding Grid is designed for leaf-level product nodes. Brand/Category nodes time out. Workaround: navigate from a SKU-level product.

8. **`daysAfter` omission in NC batch can silently exclude account plans.** Many customers copy the Nightly Batch code from docs without the `daysAfter` parameter. This can cause the batch to skip current-year account plans. Always verify this parameter is set.

9. **BRS shows Error but no detail in BRS Detail records? Check Splunk.** `ProcessQueueWorker_UNKNOWN` with no error in UI is a known display issue — the actual error is always in Splunk under the org's `k8s_namespace=rcg` logs. (Sravani Gajula, October 2025)

10. **Child promotion created without tactics? Check Copied Components.** Push copies components listed in the Promotion Template Hierarchy's "Copied Components" attribute. If "Tactics" is not in the list, push will succeed but children will be empty. (Sven Camenzind, March 2026)

11. **Promotion Information Card missing on first load? F5 fixes it.** This is a known LWC navigation caching issue on first navigation from list view. Document the workaround, no code fix. (W-22571462, closed without code change)

12. **For Agentforce TPM — the prompt level matters.** The KAM agent can only retrieve measures at the SKU/product level, not Brand or Flavor. Prompts specifying Brand-level products will return "Measures for product not found." Educate customers on prompt construction. (Tahoor Khan, #tmp-help-consumer-goods, April 2026)

---

## H. Active Known Issues (as of 2026-05-29)

| GUS ID | Subject | Status | Priority |
|--------|---------|--------|----------|
| [W-22597542](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002aYZfYYAW/view) | EditableCalculated KPI manual input renders blank cells due to NaN cascade when applyFixedTotals=true | **New** | P3 |
| [W-22689415](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002axg7uYAA/view) | Manual input values incorrect on UI for child promotions after push promotion (Luigi Lavazza) | More Info Reqd | - |
| [W-22512248](https://gus.my.salesforce.com/lightning/r/ADM_Work__c/a07EE00002a7UGvYAM/view) | splitweeks=true causing missing KPI data and CSV discrepancies in RTR | More Info Reqd | - |

---

## I. Code Reference Map

**Repo:** `git.soma.salesforce.com/Localization/rcg-retail-tpm` (branch: `release-262.0.0`)

| Class | Purpose | Relevant To |
|-------|---------|------------|
| `force-app/main/default/classes/LicenseCheckUtils.cls` | License/permission validation. Constants: `TPM_ORG_PERM = 'OrgHasTradePromotionMgmt'`, `TPM_USER_PERMISSION = 'PermissionsCGCTradePromotionManagement'` | Pattern 10 |
| `cgcloud.TPMCalculationChain` (managed package) | Main entry point for Nightly Calculation chain. Called via `startProcess()` | Pattern 1, 2 |
| `cgcloud.TPM_BC_Settings` (managed package) | Settings object for NC batch — controls which sub-jobs run and `daysAfter` parameter | Pattern 1 |
| `cgcloud__Batch_Run_Status__c` | Standard object tracking all batch job runs — primary diagnostic tool | All batch patterns |
| `cgcloud__Batch_Last_Successful_Run__c` | Tracks last successful run timestamp per batch type | Pattern 5 |
| `ScheduleCGCloudServFlattenAccHierarchy` (managed) | Account hierarchy flattening batch | Pattern 9 |

**Key permission constants (from `admin.xml`):**
- `org_provision_val_OrgPermissions_OrgHasTradePromotionMgmt` = "Consumer Goods Cloud: Trade Promotion Management"
- `PermissionsCGCTradePromotionManagement` = the user-level permission flag

---

## J. MCP Tools for Debugging TPM Cases

### GUS — Query investigations by product tag
```soql
SELECT Id, Name, Subject__c, Status__c, Priority__c, Type__c
FROM ADM_Work__c
WHERE Product_Tag__r.Name LIKE '%Trade Promo%'
AND Type__c IN ('Bug','Investigation')
AND Status__c NOT IN ('Closed','Closed - Defunct')
ORDER BY Priority__c NULLS LAST, CreatedDate DESC
LIMIT 20
```

### GUS — Find investigations by case number (from Subject)
```soql
SELECT Id, Name, Subject__c, Status__c, Priority__c
FROM ADM_Work__c
WHERE Subject__c LIKE '%<CASE_NUMBER>%'
AND Product_Tag__r.Name LIKE '%Trade Promo%'
LIMIT 10
```

### Slack — Search #tmp-help-consumer-goods
- Primary channel: `#tmp-help-consumer-goods` (C028DDZ05B6)
- Search queries to try:
  - `in:#tmp-help-consumer-goods <exact error message>`
  - `in:#tmp-help-consumer-goods <feature area> workaround fix resolved`
  - `in:#tmp-help-consumer-goods <customer name>`

### CodeSearch — Find relevant Apex
```
repo:git.soma.salesforce.com/Localization/rcg-retail-tpm content:"<class or method name>"
```

### Splunk — Processing service logs
```
(index=distapps) substrate=aws k8s_cluster=sam-processing1 k8s_namespace=rcg functional_domain=industries organizationId="<ORG_ID>*" logLevel=error
```
Dashboard: https://splunk-web-noncore.log-analytics.monitoring.aws-esvc1-useast2.aws.sfdc.cl/en-US/app/publicSharing/cgcloud_processing_services__job_chain_jobs_scheduling_and_worker_processing
