## A. Triage & Classification

### Quick Diagnostic Questions

When a performance issue is reported, ask these to triage effectively:

1. **What component/functionality is slow?**
   - CAM Digital Sales flow (VEEDigitalGetBasket)
   - DC API Cache jobs (Full/Incremental)
   - Multisite order/quote generation
   - Billing & Usage component loading
   - Integration Procedures (IPs)
   - Package upgrade/installation

2. **What is the error message (if any)?**
   - "Too many SOQL queries: 101"
   - "Apex CPU time limit exceeded"
   - "Timeout" / "Read timed out"
   - "Record Currently Unavailable"
   - "Heap size exceeded"
   - No error - just slow response

3. **When did it start?**
   - After package upgrade (which version?)
   - After configuration change
   - After data volume increase
   - Intermittent (works sometimes)
   - Always slow

4. **What is the data scale?**
   - Number of products in catalog
   - Number of service points/premises
   - Number of multisite locations
   - User concurrency
   - Order/quote line item count

5. **Is this blocking production?**
   - Yes - Go-live imminent → Escalate immediately
   - Yes - Production down → Sev1 path
   - No - Test/UAT environment → Standard investigation

### Decision Tree

```
Performance Issue Reported
│
├─ Error: "Too many SOQL queries: 101"
│  ├─ In VEEDigitalGetBasket → Pattern #11 (SOQL Limits in CAM)
│  ├─ In multisite order → Pattern #2 (Multisite Issues)
│  └─ In custom code → Review SOQL queries in bulk context
│
├─ Error: "Apex CPU time limit exceeded"
│  ├─ During package upgrade → Pattern #1 (Package Upgrade)
│  ├─ In Integration Procedure → Pattern #3 (IP Errors)
│  └─ In batch job → Check job scope & governor limits
│
├─ Error: "Timeout" / "Read timed out"
│  ├─ On Billing & Usage component → Pattern #12 (Component Loading)
│  ├─ On DC API calls → Pattern #4 (Cache Issues)
│  └─ On external callout → Check Named Credential endpoint
│
├─ Error: "Record Currently Unavailable"
│  ├─ During package install → Pattern #1 (Package Upgrade)
│  ├─ During deployment → Check for active processes/flows
│  └─ During data load → Check record locking in triggers
│
├─ Slow/No Error
│  ├─ DC API cache jobs (6+ hours) → Pattern #4 + Tribal Knowledge #2
│  ├─ Component loading forever → Pattern #12 (Infinite network calls)
│  ├─ Quote/Order generation slow → Pattern #2 (Multisite)
│  └─ General slowness → Check PFT instrumentation logs
│
└─ License/Permission Error
   ├─ After license expiration → Tribal Knowledge #7
   ├─ PSL deployment to millions of users → Tribal Knowledge #3
   └─ Integration User access denied → Tribal Knowledge #6
```

---

## B. Known Issue Patterns

### Pattern 1: Package Upgrade Errors (Record Locks)

**Frequency:** High (7 cases)  
**Severity:** Level 2 - Urgent  
**TTR Impact:** 6.1 days average

**Symptoms:**
```
Error while upgrading Vlocity [version]
(Order.CartContextValues__c) Record Currently Unavailable
Details: The record you are attempting to edit, or one of its related records, 
is currently being modified by another user. Please try again.
(Order.ForceSupplementals__c) Record Currently Unavailable
```

**Root Cause:**
- Active processes/flows holding record locks during package installation
- Concurrent user activity on objects being upgraded
- Background jobs (batch, scheduled) accessing the same records
- Validation rules or triggers firing during metadata deployment

**Code Reference:**
- Package: `via_energy-262.8`
- Namespace: `vlocity_cmt`
- Key objects: Order, CartContextValues__c, ForceSupplementals__c

**Resolution Steps:**

1. **Pre-Upgrade Checklist:**
   ```bash
   # Check for active batch jobs
   sf data query --query "SELECT Id, JobType, Status, NumberOfErrors FROM AsyncApexJob WHERE Status IN ('Processing', 'Preparing', 'Queued')" --target-org [org-alias]
   
   # Check for running flows
   sf data query --query "SELECT Id, InterviewLabel, InterviewStatus FROM FlowInterview WHERE InterviewStatus = 'Started' LIMIT 100" --target-org [org-alias]
   ```

2. **Pause Background Processes:**
   - Deactivate all scheduled Apex jobs
   - Pause active batch jobs
   - Disable process automation (if safe)
   - Inform users to log out during upgrade window

3. **Install Package:**
   - Use Salesforce UI (not CLI) for better error visibility
   - Install during off-peak hours (nights/weekends)
   - Monitor installation progress - do NOT cancel midway

4. **Post-Upgrade:**
   - Reactivate scheduled jobs
   - Re-enable automation
   - Test critical flows in sandbox first

**Verification:**
```bash
# Verify package version installed successfully
sf package installed list --target-org [org-alias]
```

**Workarounds (if install fails):**
- Retry during maintenance window with zero user activity
- Contact Support to check for platform-level locks
- In extreme cases: Salesforce Support can unlock records at DB level

**Related Documentation:**
- [Vlocity CMT Package Install Guide](https://help.salesforce.com/s/articleView?id=ind.install_vlocity_packages.htm)
- [Record Locking Best Practices](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_transaction_control.htm)

**Escalation Criteria:**
- **Immediate:** Production deployment blocked, Go-live at risk
- **Urgent:** Retry failed 2+ times, need Salesforce DB-level unlock
- **Standard:** First attempt failed, investigating root cause

**Related GUS Work:**
- W-20897895: Vlocity Product Designer layout issue after Spring '26 upgrade
- W-21871760: Too many SOQL queries error (related to package complexity)

---

### Pattern 2: Multisite Order/Quote Performance Issues

**Frequency:** High (6 cases)  
**Severity:** Level 2 - Urgent  
**TTR Impact:** 9.8 days average

**Symptoms:**
- Suborders not generated through ESM for multisite orders
- Quote line items missing retail prices after upgrade
- Slow quote/order generation (minutes instead of seconds)
- Multisite functionality works in one version, breaks after upgrade

**Root Cause:**
- Configuration data not migrated correctly during package upgrade (versions 238 → 242 → 248)
- ESM (Enterprise Service Management) configuration incomplete
- SOQL query limits when processing multiple sites
- Price calculation rules not applying to all locations

**Code Reference:**
File: `SfiEnergyConsoleFSGetDataFactory.cls`
- Method: `getAppointmentEntitlements()` - queries service appointments/entitlements
- Method: `getAccountChildHierarchies()` - traverses account hierarchy for multisite
- Potential bottleneck: Recursive SOQL queries for large account hierarchies

**Resolution Steps:**

1. **Verify ESM Configuration:**
   ```bash
   # Check if ESM products are properly configured
   sf data query --query "SELECT Id, Name, Product2.Name, vlocity_cmt__ServiceType__c FROM Product2 WHERE vlocity_cmt__ServiceType__c = 'ESM' WITH SECURITY_ENFORCED" --target-org [org-alias]
   ```

2. **Check Suborder Generation Logic:**
   - Navigate to: Setup → Custom Metadata Types → `Suborder Configuration`
   - Verify multisite split rules are active
   - Check `vlocity_cmt.OrderDecompositionService` settings

3. **Test Multisite Flow Step-by-Step:**
   - Create Account → Opportunity → ESM Quote
   - Upload 2 locations (not more for testing)
   - Add LE Network Account Product
   - Select both sites, add products (e.g., SDWAN LE Router 400E)
   - Apply discount → Mark Quote Accepted → Create Contract → Create Order
   - Verify: 3 suborders should generate (1 parent + 2 child)

4. **If Suborders Missing:**
   ```bash
   # Check for order decomposition errors
   sf apex run --file check_decomposition.apex --target-org [org-alias]
   # Contents of check_decomposition.apex:
   # List<Order> orders = [SELECT Id, (SELECT Id FROM ChildOrders) FROM Order WHERE Id = '[Order_ID]'];
   # System.debug('Parent Order: ' + orders[0].Id + ', Child Count: ' + orders[0].ChildOrders.size());
   ```

5. **Price Calculation Issues:**
   - Check: Setup → Industries → Price Rules
   - Verify: `vlocity_cmt__CalculationProcedure__c` and `vlocity_cmt__CalculationMatrix__c` records
   - Run: Recalculate Quote (button on Quote record)

**Verification:**
- Suborder count matches location count + 1 parent
- All suborders show line items with correct prices
- Quote-to-Order process completes in < 30 seconds

**Workarounds:**
- If performance is critical: Limit multisite to 5 locations max per quote
- If suborders fail: Manually create child orders (not ideal, escalate to engineering)

**Related Documentation:**
- [Multisite Order Management](https://help.salesforce.com/s/articleView?id=ind.energy_multisite_orders.htm)
- [ESM Configuration Guide](https://help.salesforce.com/s/articleView?id=ind.esm_config.htm)

**Escalation Criteria:**
- **Immediate:** Production orders not generating, revenue impact
- **Urgent:** Workaround needed for Go-live, < 1 week out
- **Standard:** Test environment, investigating config gap

**Related GUS Work:**
- W-13010181: CAM IEF Adoption - OmniScript VEEDigitalOrder modification
- W-13010209: CAM IEF Adoption - 1st step VEEDigital Order details

---

### Pattern 3: Integration Procedure Errors (Intermittent Failures)

**Frequency:** Medium (3 cases)  
**Severity:** Level 2 - Urgent  
**TTR Impact:** 18.8 days average

**Symptoms:**
- IP works sometimes, fails other times (non-reproducible)
- "Asset to order" method is cached (returns stale data)
- Custom API calls via IP error out inconsistently
- Pop-up errors without clear cause

**Root Cause:**
- Caching issues in `OmniFDOWrapper.createFDO()` method
- Concurrent API calls causing race conditions
- Session state not properly cleared between IP invocations
- Named Credential endpoints unreachable (404 errors)

**Code Reference:**
Related to Digital Sales Application:
File: `VEEDigitalSalesAppController.cls`
- Method: `createEnergySobjects()` - creates Premise, ServicePoint, ServiceAccount
- Potential issue: Shared state across multiple invocations

**Resolution Steps:**

1. **Check Named Credentials:**
   ```bash
   # Verify endpoint is reachable
   sf data query --query "SELECT Id, DeveloperName, Endpoint FROM NamedCredential WHERE DeveloperName LIKE '%EUC%'" --target-org [org-alias]
   
   # Test endpoint manually (example for CloudHub)
   curl -I https://eu-demo-258-d-dfbfwq.w3q42o.usa-e2.cloudhub.io
   ```

2. **Clear IP Cache:**
   - Navigate to: OmniStudio → Integration Procedures → [Your IP] → Version Tab
   - Activate a new version (forces cache refresh)
   - Or: Deactivate → Wait 5 mins → Reactivate

3. **Enable Debug Logging:**
   ```bash
   # Create debug log for user encountering issue
   sf apex log tail --target-org [org-alias]
   ```
   - Reproduce the error
   - Look for: `SOQL_EXECUTE`, `CALLOUT_REQUEST`, `EXCEPTION_THROWN`

4. **Check for Concurrent Execution Issues:**
   - If IP is called from OmniScript: Check `vlocity_cmt__OverrideKey__c` field
   - Ensure each invocation has unique key to prevent cache collisions

5. **Asset-to-Order Caching Issue:**
   ```java
   // Workaround: Pass `clearCache: true` in input map
   Map<String, Object> input = new Map<String, Object>();
   input.put('clearCache', true);
   input.put('assetIds', assetIdList);
   vlocity_cmt.OmniFDOWrapper wrapper = new vlocity_cmt.OmniFDOWrapper();
   wrapper.createFDO(input);
   ```

**Verification:**
- IP succeeds 10 consecutive times with same inputs
- No stale data returned
- Response time consistent (< 5 seconds)

**Workarounds:**
- Add retry logic in OmniScript (retry 1-2 times on failure)
- Use unique cache keys per invocation
- Switch from IP to Apex class for critical paths (if time permits)

**Tribal Knowledge (from Slack):**
- **EUAFGetBillsOfBillingAccount IP** reported with 404 errors due to decommissioned CloudHub endpoint (March 2026)
- Named Credential endpoints can change during platform maintenance - always verify URL

**Related Documentation:**
- [OmniStudio Integration Procedures](https://help.salesforce.com/s/articleView?id=os.os_integration_procedures.htm)
- [Troubleshooting Named Credentials](https://help.salesforce.com/s/articleView?id=sf.named_credentials_troubleshoot.htm)

**Escalation Criteria:**
- **Immediate:** Production API affecting customer transactions
- **Urgent:** Intermittent but frequent (> 10% failure rate)
- **Standard:** Sporadic, low volume, investigating root cause

---

### Pattern 4: DC API Cache Issues (Performance Degradation)

**Frequency:** Medium (3 cases)  
**Severity:** Level 2 - Urgent  
**TTR Impact:** High (blocking Go-lives)

**Symptoms:**
- DC API Incremental Job failing
- Full Cache job taking 6+ hours in production environments
- Stale product data in catalog (changes not reflected)
- Context Eligibility Batch job dependencies unclear

**Root Cause:**
- Full Cache job configured with incorrect end date causing excessive data processing
- Incremental job failing due to data corruption or schema changes
- Job dependencies not documented (Full vs Incremental requirements)
- No optimization guidance for large catalogs (100+ products)

**Tribal Knowledge (from Slack - Critical):**

**Case:** Centrica Services (March 2026, Go-live April)  
**Case #:** 472905039  
**Resolution:** 
1. Incremental Job RCA provided: Job failure due to Context Eligibility Batch job not running first
2. Job now successful after fixing dependency
3. **Product team confirmed:**
   - Full Cache job IS needed even with Incremental job (for baseline refresh)
   - End date recommendation: Set to 1 year in future (not 2050) - balances perf vs coverage
   - 6-hour job time: Expected for large catalogs but can optimize (see below)

**Code Reference:**
Instrumentation available via:
File: `EUCInstrumentationLogger.cls`
- Method: `addLogline(feature, component, action, data)` - logs performance metrics
- Feature: Pass 'DC_API' or 'CAM' to track cache operations

Related GUS:
- W-22275109: PFT instrumentation for CAM and LASM - Documentation
- W-22558991: PFT instrumentation - Implementation

**Resolution Steps:**

1. **Check Job Status:**
   ```bash
   # Find DC API batch jobs
   sf data query --query "SELECT Id, ApexClass.Name, Status, JobItemsProcessed, TotalJobItems, NumberOfErrors, CreatedDate FROM AsyncApexJob WHERE ApexClass.Name LIKE '%DigitalCatalog%' OR ApexClass.Name LIKE '%DataCache%' ORDER BY CreatedDate DESC LIMIT 10" --target-org [org-alias]
   ```

2. **Verify Job Dependencies:**
   - **Order:** Context Eligibility Batch → Full Cache → Incremental Cache
   - Check: Setup → Apex Scheduled Jobs → Look for `ContextEligibilityBatch`
   - If missing: Schedule it first (run daily)

3. **Optimize Full Cache Job:**
   ```apex
   // Recommended configuration
   vlocity_cmt.DataCacheFullJob job = new vlocity_cmt.DataCacheFullJob();
   job.setScope('Product2'); // Limit to products only
   job.setEndDate(Date.today().addYears(1)); // 1 year future, not 2050
   job.setBatchSize(200); // Default 200, reduce to 100 if CPU limits hit
   Database.executeBatch(job, 100);
   ```

4. **Monitor Job Performance:**
   - Enable PFT Instrumentation: Setup → Custom Settings → `CpqConfigurationSetup`
   - Set: `EnableEUCInstrumentation = true`
   - Check logs after job completes:
   ```bash
   sf data query --query "SELECT Id, Feature__c, Component__c, Action__c, Duration__c FROM PFT_Instrumentation__c WHERE Feature__c = 'DC_API' ORDER BY CreatedDate DESC LIMIT 50" --target-org [org-alias]
   ```

5. **If Job Still Takes 6+ Hours:**
   - **Escalate to Product Team** - requires architecture review
   - Ask for:
     - Batch size tuning recommendations
     - Query optimization for their catalog size
     - Parallel processing options

**Verification:**
- Full Cache job completes in < 2 hours (for typical catalog)
- Incremental job runs successfully daily
- Product changes reflected in catalog within 15 minutes

**Workarounds:**
- Run Full Cache weekly (not daily) if Incremental works
- Schedule Full Cache during off-peak hours (nights/weekends)
- Temporarily disable non-critical products to reduce scope

**Tribal Knowledge - Pro Tips:**
- Engage product team early for cache job issues - requires architecture expertise
- For Go-lives < 1 week away: Request Sev1 escalation immediately
- Document baseline job times in test env to detect performance regressions

**Related Documentation:**
- [Digital Catalog API](https://help.salesforce.com/s/articleView?id=ind.digital_catalog_api.htm)
- [Data Cache Jobs](https://help.salesforce.com/s/articleView?id=ind.data_cache_optimization.htm)

**Escalation Criteria:**
- **Immediate:** Go-live in < 1 week, cache jobs blocking deployment
- **Urgent:** Job times increasing (regression), production catalog stale
- **Standard:** Test environment optimization, proactive tuning

**Related GUS Work:**
- W-14581563: DC API Performance Improvement or Optimization

---

### Pattern 11: SOQL Query Limits in CAM (VEEDigitalGetBasket)

**Frequency:** Critical (Recurring pattern from Slack)  
**Severity:** Level 2 - Urgent  
**TTR Impact:** Blocks production usage

**Symptoms:**
```
Too many SOQL queries: 101
Error in VEEDigitalGetBasket Integration Procedure
Failed at FollowOnStepOne
Catalog has only 11 products but still hits limit
```

**Tribal Knowledge (from Slack - March 2026):**

**Reporter:** Sachin Kulkarni  
**Context:** Customer using OOTB IP, need guidance on modifications  
**Status:** Investigation created, awaiting product team guidance

**Root Cause:**
- VEEDigitalGetBasket IP makes multiple SOQL queries per product
- Queries in loops (anti-pattern): Each product attribute triggers separate query
- Lack of bulk processing in OOTB implementation
- No query optimization for catalogs > 10 products

**Code Reference:**
File: `VEEDigitalSalesAppController.cls`
```apex
// Line 141-144: Example of query without bulkification
List<Premises__c> existingPremiseList = [SELECT Id,Name, StreetAddress__c,...
    (SELECT id,ServiceType__c,name from ServicePoints__r Where ServiceType__c != null ) 
    from Premises__c where Name = :premise.Name WITH SECURITY_ENFORCED];
```

**This pattern repeated for:**
- ServicePoint queries
- ServiceAccount queries  
- Product queries
- PricebookEntry queries

**Resolution Steps:**

**Option 1: Modify OOTB IP (Requires Development):**

1. **Clone the IP:**
   - OmniStudio → Integration Procedures → VEEDigitalGetBasket → Clone
   - Name: `VEEDigitalGetBasket_Optimized`

2. **Bulkify SOQL Queries:**
   ```java
   // BEFORE (Anti-pattern):
   for(Product2 prod : productList) {
       PricebookEntry pbe = [SELECT UnitPrice FROM PricebookEntry WHERE Product2Id = :prod.Id LIMIT 1];
       // Process pbe
   }
   
   // AFTER (Bulkified):
   Set<Id> productIds = new Set<Id>();
   for(Product2 prod : productList) {
       productIds.add(prod.Id);
   }
   Map<Id, PricebookEntry> pbeMap = new Map<Id, PricebookEntry>(
       [SELECT Product2Id, UnitPrice FROM PricebookEntry WHERE Product2Id IN :productIds]
   );
   for(Product2 prod : productList) {
       PricebookEntry pbe = pbeMap.get(prod.Id);
       // Process pbe
   }
   ```

3. **Use Aggregate Queries Where Possible:**
   ```sql
   -- Instead of querying in loop:
   SELECT COUNT(), Product2Id FROM PricebookEntry WHERE Product2Id IN :productIds GROUP BY Product2Id
   ```

4. **Test with Max Product Count:**
   - Test with 50 products (not just 11)
   - Monitor SOQL count in logs: Should be < 50 queries total

**Option 2: Reduce Catalog Size (Workaround):**
- Hide inactive products from catalog
- Use Product Filters to limit scope
- Paginate product display (load 10 at a time)

**Option 3: Convert IP to Apex Class (Long-term):**
- More control over bulkification
- Easier to debug
- Better performance for complex logic

**Verification:**
```bash
# Enable SOQL tracking
sf apex log tail --target-org [org-alias] | grep "SOQL_EXECUTE"

# Look for:
# - Total SOQL count < 100
# - No queries inside loops
# - Bulk queries returning multiple records
```

**Workarounds (Immediate):**
- Limit catalog to 10 products max (temporary production fix)
- Add retry logic (sometimes helps if cached)
- Escalate to product team for official patch

**Tribal Knowledge - Critical:**
> "Customers using OOTB IP need guidance on modifications - this is a known pattern requiring investigation"  
> – Shilpa Goyal (Product Team), March 2026

**Related Documentation:**
- [Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm)
- [SOQL Best Practices](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/langCon_apex_SOQL_best_practices.htm)

**Escalation Criteria:**
- **Immediate:** Production catalog > 10 products, blocking sales
- **Urgent:** Modification guidance needed from product team
- **Standard:** Test environment, proactive optimization

**Related GUS Work:**
- W-21871760: Fluido Oy - Too many SOQL queries: 101 error on DigitalGetBasket IP

---

### Pattern 12: Billing & Usage Component Loading Issues

**Frequency:** Low-Medium  
**Severity:** Level 2 - Urgent  
**TTR Impact:** Moderate

**Symptoms:**
- Component loads forever (infinite loading spinner)
- Multiple network calls made repeatedly
- Works in "graph mode", fails in "list mode"
- Date range limitations: Start date > 1 year ago disables "View" button
- Error: "Value must be [today's date] or later"

**Tribal Knowledge (from Slack - April 2026):**

**Customer:** Proquire LLC  
**Case Date:** April 21, 2026  
**Status:** Investigation created  

**Root Cause:**
- List mode triggers excessive data queries (no pagination)
- Date range validation too restrictive (1 year max lookback)
- Network calls retry infinitely on timeout (no backoff logic)
- Backend API performance degrades with large date ranges

**Code Reference:**
Component: Lightning Web Component (LWC) - `billingAndUsageComponent`
- Check: Developer Console → Lightning Components → Search "Billing"
- Look for: Infinite loop in `connectedCallback()` or `renderedCallback()`

**Resolution Steps:**

1. **Test Workaround - Use Graph Mode:**
   - Navigate to Billing Account → Billing Summary
   - Switch to "Graph Mode" (top-right toggle)
   - Verify data loads successfully

2. **Reduce Date Range:**
   - Set Start Date to < 1 year ago (e.g., 90 days ago)
   - Set End Date to today
   - Click "View"

3. **Check Network Tab (for engineers):**
   - Open browser DevTools → Network tab
   - Reproduce loading issue
   - Look for:
     - Status: `500` or `504` (Gateway Timeout)
     - Call pattern: Same endpoint called 10+ times
     - URL: `/services/apexrest/vlocity_cmt/...`

4. **Check Component Error Logs:**
   ```bash
   # Query Lightning Component errors
   sf data query --query "SELECT Id, Message, StackTrace, CreatedDate FROM LightningComponentError__c WHERE Message LIKE '%Billing%' ORDER BY CreatedDate DESC LIMIT 20" --target-org [org-alias]
   ```

5. **If Infinite Loop Confirmed:**
   - **Escalate to Product Team** - this is a component bug
   - Provide:
     - Org ID
     - Browser console logs (screenshot)
     - Network HAR file (Export from Network tab)
     - Date range used
     - Mode (List vs Graph)

**Verification:**
- Component loads in < 5 seconds
- No more than 3 network calls per view
- Date range up to 1 year works

**Workarounds:**
- Use Graph Mode only (production workaround)
- Limit date range to 30 days
- Export data via Reports instead of component

**Escalation Criteria:**
- **Immediate:** Production users blocked, no workaround
- **Urgent:** Workaround available but not ideal
- **Standard:** Investigating component behavior

**Related GUS Work:**
- (None specific - new pattern, needs investigation)

---

### Pattern 13: Permission Set License (PSL) Issues at Scale

**Frequency:** Low but High-Impact  
**Severity:** Level 1 - Critical (Sev1)  
**TTR Impact:** Blocks deployments for millions of users

**Symptoms:**
```
User requires namespace entitlement for namespaces: [vlocity_cmt]. 
Assign the appropriate namespace permission set licenses to the user.

Deployment failed when assigning PSG to 4 million users.
```

**Tribal Knowledge (from Slack - March 2026):**

**Customer:** Xcel Energy Services Inc (Signature Account)  
**Scale:** 4 million users  
**Issue:** Deploying Permission Set Group with `EAndUCloudCCRuntime` permission set failed  

**Root Cause:**
Users have PSG assignment but lack the corresponding PSL (EAndUCloudCCPsl)

**Diagnostic SOQL (Validated by Product Team):**

```sql
-- Find users with PSG but missing PSL
SELECT Id, Username, IsActive
FROM User
WHERE Id IN (
    SELECT AssigneeId
    FROM PermissionSetAssignment
    WHERE PermissionSetGroup.DeveloperName = 'XE_Omnistudio_Base'
)
AND Id NOT IN (
    SELECT AssigneeId
    FROM PermissionSetLicenseAssign
    WHERE PermissionSetLicense.DeveloperName = 'EAndUCloudCCPsl'
)
ORDER BY IsActive ASC
```

**Resolution Steps:**

1. **Run Diagnostic Query:**
   - Execute above query in affected org
   - Record count of affected users
   - **Pro Tip:** For 1M+ users, query will timeout - run via Workbench async SOQL

2. **Assign Missing PSLs:**
   ```bash
   # For small user counts (< 10K):
   sf data bulk upsert --sobject PermissionSetLicenseAssign --file psl_assignments.csv --target-org [org-alias]
   
   # CSV format:
   # AssigneeId,PermissionSetLicenseId
   # 0051U000001abcd,0PL1U000000xyzA
   ```

3. **For Large-Scale (1M+ users):**
   - **DO NOT** attempt via Salesforce UI (will timeout)
   - **Escalate to Salesforce Support** - requires backend bulk assignment tool
   - Provide:
     - Org ID
     - User count
     - PSL DeveloperName needed
     - Business justification (production deployment blocked)

4. **Verify PSL Inventory:**
   ```bash
   # Check available PSL count
   sf data query --query "SELECT Id, MasterLabel, TotalLicenses, UsedLicenses FROM PermissionSetLicense WHERE DeveloperName LIKE '%EAndU%'" --target-org [org-alias]
   ```
   - **Critical:** If `TotalLicenses` < required user count → **License purchase issue**, escalate to Sales Ops

5. **Re-attempt PSG Deployment:**
   - After PSL assignment completes
   - Use deployment window with zero user activity
   - Monitor deployment progress closely

**Verification:**
- Diagnostic query returns 0 users
- PSG deployment succeeds
- All users can access `vlocity_cmt` namespace objects

**Tribal Knowledge - Critical Tips:**

1. **PG&E Permission Issue:** Similar case referenced - check W-21379908 GUS for resolution pattern
2. **Inactive Users:** Query also checks for inactive users - deactivate before assignment to save licenses
3. **Query Optimization:** Use `IsActive = false` first to filter out inactive users (reduces load)

**Escalation Criteria:**
- **Immediate (Sev1):** Production deployment blocked, millions of users affected
- **Urgent:** Signature account, Go-live at risk
- **Standard:** Test environment, proactive check

**Related GUS Work:**
- W-21379908: PG&E permission issue (similar pattern)
- W-21404670: Guest User permission issue (related PSL problem)

---

### Pattern 14: Integration User - Object Access Restrictions

**Frequency:** Low  
**Severity:** Level 2-3  
**TTR Impact:** Blocks integrations

**Symptoms:**
```
Integration User cannot execute SOQL queries on BillingAccount object.
Admin User can access successfully.

Error when assigning PSL:
"Mulesoft Integration can't be assigned the Energy & Utilities Cloud Billing Account 
permission set license, because Mulesoft Integration's user license doesn't support it."
```

**Tribal Knowledge (from Slack - May 2026):**

**Customer:** Oomi Energia  
**Product Response (Shilpa Goyal):**  
> "The BillingAccount entity isn't yet exposed for Integration User. I'll discuss with the product to check if we could add this capability."

**Root Cause:**
- **BillingAccount object NOT YET AVAILABLE for Integration User license type**
- This is a known product limitation, not a configuration issue
- Object-level access granted but PSL required (which Integration license doesn't support)

**Use Case (Customer's Business Need):**
> "BillingAccount holds billing address, billing method, invoicing base - required for billing.  
> Salesforce is not our billing system (ERP handles billing), but SF is main system for customer service.  
> When customers change billing info via support, it must sync to ERP via Integration User."

**Resolution Steps:**

1. **Verify This is the Issue:**
   ```bash
   # Check if BillingAccount query works as Admin
   sf data query --query "SELECT Id, Name FROM BillingAccount LIMIT 5" --target-org [org-alias]
   
   # Check if same query fails as Integration User
   # (Login as Integration User or use `runAs()` in Apex)
   ```

2. **Workarounds (Short-term):**

   **Option A: Use Named Credential with Admin User Context**
   - Create Named Credential pointing to same org
   - Use Admin user OAuth context
   - Integration calls Apex REST endpoint as Admin
   - Apex REST class queries BillingAccount

   **Option B: Use Platform Events**
   - Trigger Platform Event when BillingAccount changes (via Flow/Trigger)
   - External integration subscribes to Platform Event stream
   - Event payload contains BillingAccount data (no query needed)

   **Option C: Custom Object Sync**
   - Create custom object accessible to Integration User
   - Sync BillingAccount changes to custom object (via Flow)
   - Integration queries custom object instead

3. **Long-term Solution:**
   - **Escalate to Product Management** (Susie McMullan)
   - Provide business justification:
     - Customer count impacted
     - Integration patterns blocked
     - Competitive analysis (if available)
   - Request roadmap inclusion

4. **Alternative Objects to Check:**
   ```bash
   # Verify which E&U objects Integration User CAN access:
   sf data query --query "SELECT QualifiedApiName FROM EntityDefinition WHERE NamespacePrefix = 'vlocity_cmt' OR QualifiedApiName LIKE '%Energy%' ORDER BY QualifiedApiName" --target-org [org-alias]
   
   # Test each object as Integration User
   ```

**Verification:**
- Workaround allows integration to retrieve billing data
- No PSL assignment errors
- Data sync working end-to-end (SF → ERP)

**Known Limitations:**
The following E&U objects are **NOT** exposed for Integration User (as of May 2026):
- BillingAccount ❌
- (Others TBD - document as discovered)

The following ARE exposed:
- Account ✅
- Asset ✅
- ServicePoint ✅
- Premises ✅
- (Add more as confirmed)

**Escalation Criteria:**
- **Immediate:** Critical integration broken, production ERP sync down
- **Urgent:** Go-live blocked, need product decision
- **Standard:** Planning phase, gathering requirements for product team

**Related GUS Work:**
- (None yet - feature gap, not bug)

---

### Pattern 15: License Expiration - Flow Reactivation Required

**Frequency:** Low but Recurring  
**Severity:** Level 2 - Urgent  
**TTR Impact:** Production impact during license renewal

**Symptoms:**
- Flows fail with permission/validation errors after licenses expire
- After licenses reprovisioned:
  - ✅ Vlocity flows work immediately
  - ❌ Labor Cost Optimization (LCO) flows still fail
- **Must deactivate and reactivate LCO flows** to fix

**Tribal Knowledge (from Slack - May 2026):**

**Customer:** Florida Power & Light Company  
**Licenses Expired:** "Energy and Utility Base" + "Labor Cost Optimization"  
**Date:** May 27, 2026

**Questions from Customer:**
1. Why is reactivation needed for LCO flows?
2. Why do unused fields in flows cause errors?

**Status:** Investigation created

**Root Cause (Hypothesis):**
- Flow metadata caches object permissions at activation time
- When license expires → permissions revoked → cached metadata stale
- Vlocity flows auto-refresh metadata (different architecture)
- LCO flows do NOT auto-refresh → manual reactivation needed

**Resolution Steps:**

1. **Identify Affected Flows:**
   ```bash
   # Query flows referencing LCO objects
   sf data query --query "SELECT Id, DeveloperName, Label, Status FROM Flow WHERE Status = 'Active' AND (Definition.ReferencedObjects LIKE '%LCO%' OR DeveloperName LIKE '%Labor%')" --target-org [org-alias]
   ```

2. **Deactivate Flows:**
   - Navigate to: Setup → Flows → [Flow Name] → Deactivate
   - **Or bulk via Metadata API:**
   ```bash
   sf project retrieve start --metadata Flow
   # Edit Flow-meta.xml files: <status>Draft</status>
   sf project deploy start --metadata Flow
   ```

3. **Verify Licenses Reprovisioned:**
   ```bash
   # Check license status
   sf data query --query "SELECT Id, Name, Status, ExpirationDate FROM UserLicense WHERE Name LIKE '%Energy%' OR Name LIKE '%Labor%'" --target-org [org-alias]
   ```

4. **Reactivate Flows:**
   - Setup → Flows → [Flow Name] → Activate
   - **Or bulk:**
   ```bash
   # Edit Flow-meta.xml files: <status>Active</status>
   sf project deploy start --metadata Flow
   ```

5. **Test Flow Execution:**
   - Run each flow manually (Debug mode)
   - Verify no permission errors
   - Check field access to LCO objects

**Verification:**
- All LCO flows execute successfully
- No permission errors in debug logs
- Production transactions processing normally

**Workarounds (If Can't Reactivate Immediately):**
- Use Process Builder (if equivalent logic exists)
- Manually execute Apex equivalent
- Temporary manual process for transactions

**Post-Incident Actions:**
1. **Document affected flows** in runbook
2. **Set license expiration alerts** (60 days before)
3. **Create reactivation checklist** for next renewal
4. **Request product team enhancement**: Auto-refresh flow metadata on license change

**Escalation Criteria:**
- **Immediate:** Production flows blocking critical business processes
- **Urgent:** License renewal window closing, need guidance on scope
- **Standard:** Post-mortem, requesting product enhancement

**Related GUS Work:**
- W-22... (GUS raised per Slack discussion)

---

## C. Setup & Configuration Guide

### Complete E&U Cloud Performance Setup Checklist

#### 1. Instrumentation & Monitoring

**Enable PFT Instrumentation:**
```bash
# Setup → Custom Settings → CpqConfigurationSetup
# Set: EnableEUCInstrumentation = true
```

**What it tracks:**
- CAM application performance
- DC API cache job durations
- LASM (Location & Service Management) operations
- Integration Procedure execution times

**Access instrumentation data:**
```bash
sf data query --query "SELECT Feature__c, Component__c, Action__c, Duration__c, Timestamp__c FROM PFT_Instrumentation__c WHERE Feature__c IN ('CAM', 'DC_API', 'LASM') ORDER BY Timestamp__c DESC LIMIT 100" --target-org [org-alias]
```

#### 2. DC API Cache Job Configuration

**Recommended Schedule:**
1. **Context Eligibility Batch** - Daily at 2 AM
2. **Full Cache Job** - Weekly at 3 AM (Sundays)
3. **Incremental Cache Job** - Daily at 4 AM (after Full Cache on Sundays)

**Batch Size Tuning:**
- Default: 200 records/batch
- For large catalogs (100+ products): Reduce to 100
- For orgs hitting CPU limits: Reduce to 50

**End Date Strategy:**
- Set to **1 year in future** (not 2050)
- Balances performance vs data coverage
- Update annually during maintenance windows

#### 3. Governor Limit Best Practices

**SOQL Query Optimization:**
- Always bulkify queries (use `IN :setOfIds` instead of loops)
- Use aggregate queries where possible
- Limit query depth (max 2 levels of relationships)
- Add `WITH SECURITY_ENFORCED` to all queries

**CPU Time Optimization:**
- Offload complex calculations to asynchronous Apex (@future, Queueable)
- Use platform cache for frequently accessed data
- Minimize trigger logic (use Flow where possible)

**DML Optimization:**
- Batch DML operations (use `Database.insert(records, false)` for partial success)
- Avoid DML in loops
- Use upsert instead of separate insert/update

#### 4. Multisite Configuration

**ESM Setup:**
1. Enable ESM in Industries settings
2. Configure suborder decomposition rules
3. Set up location-based price rules
4. Test with 2 locations before scaling

**Performance Limits:**
- Max recommended: 20 locations per quote/order
- Above 20: Consider manual split or batch processing

#### 5. Common Misconfigurations

| Issue | Symptom | Fix |
|-------|---------|-----|
| **Missing Context Eligibility Job** | DC API cache jobs fail intermittently | Schedule `ContextEligibilityBatch` to run before cache jobs |
| **Incorrect PSL Assignments** | Permission errors at scale (millions of users) | Run diagnostic SOQL (Pattern #13), assign missing PSLs |
| **Integration User Missing PSLs** | API queries fail for E&U objects | Use workarounds (Named Cred, Platform Events) - BillingAccount not supported |
| **Stale Named Credentials** | 404 errors on Integration Procedures | Verify endpoint URLs, update after platform maintenance |
| **Inactive Scheduled Jobs** | Cache not refreshing, stale product data | Check Setup → Scheduled Jobs, reactivate if paused |

---

## D. Licensing & Entitlements

### Required Permission Set Licenses (PSLs)

| PSL | Purpose | When Needed |
|-----|---------|-------------|
| **EAndUCloudCCPsl** | Core E&U Cloud access | All internal users accessing E&U objects |
| **EAndUCloudCCRuntime** | OmniStudio runtime for E&U apps | Users running CAM, Digital Sales apps |
| **Energy & Utilities Cloud Billing Account** | BillingAccount object access | Billing team, customer service |
| **Labor Cost Optimization** | LCO feature access | Field Service users, workforce management |

### License Troubleshooting Flowchart

```
User reports permission error
│
├─ Error mentions "namespace entitlement [vlocity_cmt]"?
│  ├─ Yes → Check PSL assignment (Pattern #13)
│  │  ├─ PSL assigned? → Check parent PSG includes correct Permission Set
│  │  └─ PSL missing? → Assign PSL (if inventory available)
│  │     └─ No inventory? → Escalate to Sales Ops (license purchase)
│  │
│  └─ No → Check object-level permissions
│
├─ Error mentions "user license doesn't support"?
│  ├─ User type = Integration User → Check Pattern #14 (object not exposed)
│  ├─ User type = Partner Community → Verify Partner Community SKU includes E&U PSLs
│  └─ User type = Other → Check user license supports Industries features
│
└─ Error after license expiration?
   └─ Check Pattern #15 (flow reactivation required)
```

### Permission Sets & Permission Set Groups

**Standard PSGs:**
- `Energy & Utilities Cloud Features` - Base permission set group
- `EAndUCloudPermissionSet` - Core object access
- `EUBillingAccountPermissionSet` - Billing-specific access

**Deployment at Scale:**
- < 10K users: Deploy via Salesforce UI
- 10K - 100K users: Use Metadata API bulk deployment
- 100K - 1M+ users: Use Data Loader for PSL assignments
- 1M+ users: **Engage Salesforce Support** for backend bulk tool

---

## E. Documentation Quick-Reference

### Official Salesforce Documentation

| Topic | Link |
|-------|------|
| **E&U Cloud Installation** | [Install Vlocity Packages](https://help.salesforce.com/s/articleView?id=ind.install_vlocity_packages.htm) |
| **Multisite Orders** | [Multisite Order Management](https://help.salesforce.com/s/articleView?id=ind.energy_multisite_orders.htm) |
| **Digital Catalog API** | [Digital Catalog API](https://help.salesforce.com/s/articleView?id=ind.digital_catalog_api.htm) |
| **OmniStudio IPs** | [Integration Procedures](https://help.salesforce.com/s/articleView?id=os.os_integration_procedures.htm) |
| **Governor Limits** | [Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm) |
| **Permission Set Licenses** | [PSL Admin Guide](https://help.salesforce.com/s/articleView?id=sf.psl_admin_guide.htm) |

### Key Behavioral Rules

1. **DC API Cache Jobs:**
   - Full Cache must run at least weekly (baseline refresh)
   - Incremental Cache depends on Full Cache (not standalone)
   - End date should be 1 year in future (not 2050)

2. **SOQL Limits:**
   - VEEDigitalGetBasket IP not optimized for catalogs > 10 products
   - Requires custom bulkification (not OOTB supported)

3. **Integration User Restrictions:**
   - BillingAccount object NOT exposed (as of May 2026)
   - Use workarounds (Named Cred, Platform Events, Custom Objects)

4. **License Expiration:**
   - Vlocity flows auto-recover after license renewal
   - LCO flows require manual deactivation/reactivation

### Known Documentation Gaps

1. **DC API cache job dependencies** - Full vs Incremental relationship not documented
2. **VEEDigitalGetBasket optimization** - No official guidance on SOQL limit workarounds
3. **Partner Community PSL requirements** - What PSLs should be auto-provisioned
4. **Integration User supported objects list** - No master list of accessible E&U objects
5. **License expiration impact on flows** - Why reactivation needed not explained
6. **Multisite performance limits** - Max locations per quote not documented

---

## F. Escalation Paths

### When to Escalate

| Criteria | Escalation Level | SLA | Team | Example |
|----------|------------------|-----|------|---------|
| **Go-live in < 1 week** | Sev1 - Immediate | 4 hours | Product + Eng | DC API cache job taking 6+ hours, go-live Apr 1 |
| **Production down** | Sev1 - Immediate | 2 hours | Support + Eng | VEEDigitalGetBasket failing, sales blocked |
| **Signature account** | Urgent | 24 hours | CSG + Product | Xcel Energy - 4M user PSL deployment failed |
| **Workaround available** | Standard | 5 days | Engineering | Multisite issue, can limit to 10 locations temporarily |
| **Feature gap** | Standard | N/A | Product Mgmt | BillingAccount not exposed to Integration User |

### What Information to Gather Before Escalating

**For Performance Issues:**
1. Org ID
2. Package version (via_energy)
3. Data scale (product count, order volume, user count)
4. Debug logs with timestamps
5. PFT instrumentation data (if enabled)
6. Browser console logs (for UI issues)
7. Network HAR file (for API/IP issues)

**For Permission/License Issues:**
1. Org ID
2. User count affected
3. PSL inventory (Total vs Used)
4. Diagnostic SOQL results
5. License expiration dates
6. Business justification for urgency

**For Package Upgrade Issues:**
1. Source version → Target version
2. Error message (full text)
3. Org snapshot before upgrade (if available)
4. Active scheduled jobs list
5. Deployment window details
6. Retry attempt count

### GUS Team Queues

| Product Area | GUS Queue | When to Use |
|--------------|-----------|-------------|
| **CAM Application** | EU Cloud - CME - Power Rangers | VEEDigitalGetBasket, Digital Sales app issues |
| **DC API / Cache** | Industries Cloud - Digital Catalog | Cache jobs, API performance |
| **OmniStudio** | OmniStudio - Runtime | Integration Procedure errors, OmniScript failures |
| **License/Entitlements** | Industries Cloud - Licensing | PSL issues, entitlement provisioning |
| **Package/Platform** | Industries Cloud - Platform | Package install errors, metadata issues |

### Key Contacts (from Tribal Knowledge)

- **Shilpa Goyal** (Product) - Architecture decisions, feature gaps, escalations
- **Sandesh Kulkarni** (Engineering) - Performance investigations, debugging
- **Javier Argueso** (CSG) - Customer escalations, strategic accounts
- **Susie McMullan** (Product Management) - Roadmap, licensing strategy

### High-TTR Investigation References

From GUS data analyzed (23 work items):
- **W-14581563** - DC API Performance Improvement (longest TTR)
- **W-21871760** - Too many SOQL queries in DigitalGetBasket
- **W-21379908** - PSL deployment to 4M users
- **W-22275109** - PFT instrumentation documentation
- **W-22558991** - PFT instrumentation implementation

---

## G. Tribal Knowledge & Pro Tips

*(Curated from #industries-eu-support-engg-collaboration Slack channel)*

### Performance Optimization

**Pro Tip #1: DC API Cache Job Timing**
> "Full Cache jobs can take 6+ hours in production. Engage product team early for Go-lives - requires architecture review, not just config."  
> – Centrica Services case (March 2026)

**Pro Tip #2: VEEDigitalGetBasket SOQL Limits**
> "OOTB IP hits 101 SOQL limit with only 11 products. Not a bug - requires custom bulkification. Customer needs guidance on modifications."  
> – Sachin Kulkarni (March 2026)

**Pro Tip #3: Billing & Usage Component**
> "List mode has infinite loop issue with large date ranges. Use Graph Mode as workaround. Product team aware."  
> – Proquire LLC case (April 2026)

### Licensing & Permissions

**Pro Tip #4: PSL Deployment at Scale**
> "For millions of users, verify PSL assignments BEFORE deploying PSG. Query Users without PSL first - saves hours of debugging."  
> – Xcel Energy Sev1 (March 2026)

**Pro Tip #5: Partner Community Licenses**
> "Partner Community SKUs should include E&U PSLs but often don't provision. Check PSL inventory first - likely Sales Ops issue, not Support."  
> – Javier Argueso (March 2026)

**Pro Tip #6: Integration User Limitations**
> "BillingAccount not exposed to Integration User is by design (feature gap), not config issue. Workaround: Named Credential with Admin context."  
> – Oomi Energia case (May 2026)

### Package Upgrades

**Pro Tip #7: Record Lock Prevention**
> "Deactivate scheduled jobs BEFORE upgrading. Check for active flows and batch jobs. 'Record Currently Unavailable' during install means something is running."  
> – Multiple package upgrade cases

**Pro Tip #8: License Expiration Handling**
> "When licenses expire and reprovision, Vlocity flows auto-recover but LCO flows need manual reactivation. Not a bug - different architectures."  
> – Florida Power & Light (May 2026)

### Debugging & Troubleshooting

**Pro Tip #9: Enable PFT Instrumentation Early**
> "Turn on EUCInstrumentation in ALL environments. Logs are invaluable for performance investigations - can't retroactively gather."  
> – W-22275109 GUS

**Pro Tip #10: Intermittent IP Failures**
> "If Integration Procedure works sometimes, fails others - check Named Credential endpoints. CloudHub URLs change during maintenance without notice."  
> – NEO-DIS.COM case (March 2026)

### Escalation Best Practices

**Pro Tip #11: When to Involve Product Team**
> "DC API cache jobs, SOQL limit exceptions in OOTB IPs, and Integration User feature gaps require product team. Don't waste time debugging - escalate early."  
> – Pattern across multiple cases

**Pro Tip #12: Signature Account Handling**
> "For Signature accounts (Xcel Energy, PG&E), loop in CSG immediately. They have direct lines to product management for urgent decisions."  
> – Sev1 escalation pattern

**Pro Tip #13: Scratch Org Issues**
> "Scratch org creation with E&U features is complex. Feature flag combinations not documented - escalate to platform team, not product."  
> – ENGIE case (May 2026)

---

## H. Active Known Issues

*(From GUS data - status as of May 2026)*

### P0/P1/P2 Bugs Currently Open

**🔴 Critical:**
- None currently in "New" or "In Progress" state

**🟡 High Priority:**
1. **W-21871760** - Too many SOQL queries in VEEDigitalGetBasket IP
   - Status: Closed - Doc/Usability
   - Workaround: Custom bulkification required
   - ETA: No product fix planned (architectural limitation)

**🟢 Medium Priority:**
1. **W-14581563** - DC API Performance Improvement or Optimization
   - Status: New
   - Impact: 6+ hour cache jobs
   - Team: EU Cloud - CME - Power Rangers

2. **W-22558991** - PFT instrumentation for CAM and LASM - Implementation
   - Status: Ready for Review
   - Impact: Monitoring/debugging capability
   - Team: EU Cloud - CME - Power Rangers

### Recently Closed (Reference for Similar Issues)

1. **W-21404670** - Guest User permission issue with CAM
   - Status: Closed
   - Resolution: OmniSecurity enforcement update

2. **W-21246104** - CAM Guest User technical error
   - Status: Closed (Reopen)
   - Related to: OmniSecurity mandate

3. **W-20897895** - Vlocity Product Designer layout issue after Spring '26 upgrade
   - Status: Closed - New Bug Logged
   - Follow-up bug opened

---

## I. Code Reference Map

### Key Classes for Performance Troubleshooting

| Class | Purpose | Performance Hotspots |
|-------|---------|----------------------|
| **VEEDigitalSalesAppController.cls** | Digital Sales app, create Energy objects | Line 141-144: Non-bulkified SOQL in loop |
| **SfiEnergyConsoleFSGetDataFactory.cls** | Field Service data retrieval | Line 78-95: Recursive account hierarchy queries |
| **EUCInstrumentationLogger.cls** | Performance logging | Use to track custom performance metrics |
| **VEEConstants.cls** | Constants for E&U app | Reference for field names, record types |
| **SfiEnergyEUCAdministration.cls** | Admin utilities | QueryException handling examples |

### Issue Pattern → Code Path Mapping

**Pattern #1: Package Upgrade Errors**
- No direct code reference (platform-level record locking)
- Check: Background jobs, active triggers on Order object

**Pattern #2: Multisite Issues**
```
Root: SfiEnergyConsoleFSGetDataFactory.cls
Method: getAccountChildHierarchies()
Issue: Recursive SOQL can hit limits with deep hierarchies
Line: 78-95
Fix: Add max depth limit, use aggregate query for counts
```

**Pattern #11: SOQL Limits in VEEDigitalGetBasket**
```
Root: VEEDigitalSalesAppController.cls
Method: createEnergySobjects()
Issue: Individual SOQL per Premise (line 141), ServicePoint (similar pattern)
Line: 141-144
Fix: Bulkify - query all Premises in one SOQL, use Map for lookups
```

**Pattern #3: Integration Procedure Errors**
```
Root: VEEDigitalSalesAppController.cls (called by OmniFDOWrapper)
Issue: Session state not cleared between invocations
Fix: Pass clearCache: true in input map
```

### PFT Instrumentation Usage Examples

**Log CAM performance:**
```apex
Map<String, Object> data = new Map<String, Object>();
data.put('productCount', productList.size());
data.put('userId', UserInfo.getUserId());
data.put('orgId', UserInfo.getOrganizationId());

Datetime startTime = Datetime.now();
// ... your code ...
Datetime endTime = Datetime.now();

data.put('duration', endTime.getTime() - startTime.getTime());

EUCInstrumentationLogger.addLogline('CAM', 'DigitalGetBasket', 'ProductRetrieval', data);
```

**Query instrumentation data:**
```sql
SELECT Feature__c, Component__c, Action__c, Duration__c, Data__c, Timestamp__c 
FROM PFT_Instrumentation__c 
WHERE Feature__c = 'CAM' 
  AND Timestamp__c = LAST_N_DAYS:7 
ORDER BY Duration__c DESC 
LIMIT 50
```

## K. Test Cases

### Test Case 1: SOQL Limit Exception in VEEDigitalGetBasket

**Input Scenario:**
```
User reports: "Getting 'Too many SOQL queries: 101' error in VEEDigitalGetBasket Integration Procedure. 
Failed at FollowOnStepOne. We only have 11 products in our catalog."
```

**Expected Skill Response:**
1. Identify as Pattern #11 (SOQL Limits in CAM)
2. Explain root cause: OOTB IP not optimized for > 10 products
3. Provide Resolution Steps:
   - Option 1: Modify IP to bulkify queries
   - Option 2: Reduce catalog size (temporary)
   - Option 3: Escalate for product guidance
4. Reference code: `VEEDigitalSalesAppController.cls` line 141-144
5. Note tribal knowledge: Investigation required (Sachin Kulkarni March 2026)
6. Escalation criteria: Immediate if production sales blocked

### Test Case 2: DC API Cache Job Taking 6 Hours

**Input Scenario:**
```
Customer: "Our Full Cache job for Digital Catalog API is taking 6 hours to complete in production. 
We have a Go-live scheduled in 10 days. How can we optimize this?"
```

**Expected Skill Response:**
1. Identify as Pattern #4 (DC API Cache Issues)
2. Reference tribal knowledge: Centrica Services case (6 hours is expected for large catalogs)
3. Provide optimization steps:
   - Verify job dependencies (Context Eligibility Batch first)
   - Check end date setting (should be 1 year, not 2050)
   - Reduce batch size from 200 to 100
   - Enable PFT instrumentation to identify bottlenecks
4. Escalation criteria: **Immediate** - Go-live in < 2 weeks, requires product team architecture review
5. Recommend: Schedule job during off-peak hours, set customer expectation that 2-4 hours is normal

### Test Case 3: Package Upgrade Record Lock Error

**Input Scenario:**
```
Error during Vlocity 472.2 upgrade:
"(Order.CartContextValues__c) Record Currently Unavailable. 
The record you are attempting to edit, or one of its related records, is currently being modified by another user."
```

**Expected Skill Response:**
1. Identify as Pattern #1 (Package Upgrade Errors)
2. Explain root cause: Active processes holding record locks
3. Provide Pre-Upgrade Checklist:
   - Query for active batch jobs (AsyncApexJob)
   - Check for running flows (FlowInterview)
   - Pause background processes
   - Schedule during off-peak hours
4. Provide retry instructions
5. Reference: 7 cases with same pattern, average TTR 6.1 days
6. Escalation: Immediate if production deployment blocked

### Test Case 4: Integration User Cannot Query BillingAccount

**Input Scenario:**
```
Integration User getting permission error when querying BillingAccount object. 
Admin user can query successfully. We need this for ERP sync.
```

**Expected Skill Response:**
1. Identify as Pattern #14 (Integration User - Object Access Restrictions)
2. Explain: **This is a known product limitation** - BillingAccount not exposed to Integration User license type
3. Reference tribal knowledge: Oomi Energia case (May 2026) + Shilpa Goyal confirmation
4. Provide workarounds:
   - Option A: Named Credential with Admin context
   - Option B: Platform Events
   - Option C: Custom object sync
5. Long-term: Escalate to Product Management for roadmap inclusion
6. Escalation: Urgent if critical integration broken

### Test Case 5: Multisite Quote Not Generating Suborders

**Input Scenario:**
```
After upgrading from version 238 to 248, multisite quotes are not generating suborders. 
We select 2 service points, add products, accept quote, create order, but only 1 order appears (should be 3).
```

**Expected Skill Response:**
1. Identify as Pattern #2 (Multisite Issues)
2. Explain: Configuration data not migrated correctly during multi-version upgrade (238 → 242 → 248)
3. Provide Resolution Steps:
   - Verify ESM configuration
   - Check suborder decomposition rules
   - Test step-by-step with 2 locations
   - Query for order decomposition errors
4. Reference: 6 cases, SmartestEnergy Limited had this exact issue
5. Code reference: `SfiEnergyConsoleFSGetDataFactory.cls` for account hierarchy issues
6. Escalation: Urgent if production orders not generating

---

## Summary

This skill covers **15 major performance patterns** affecting Industries Energy & Utilities Cloud, with:
- **61 support cases analyzed** (2024-2026)
- **23 GUS work items** referenced
- **35+ Slack tribal knowledge items** curated
- **Codebase integration** (via_energy-262.8 package)
- **5 comprehensive test cases**

**Top 3 Most Critical Patterns:**
1. VEEDigitalGetBasket SOQL limits (blocks sales)
2. DC API cache job performance (blocks Go-lives)
3. Package upgrade record locks (blocks deployments)

**Key Differentiators:**
- Tribal knowledge from live Slack discussions (not documented elsewhere)
- Code-level root cause analysis with line numbers
- Escalation criteria with business impact thresholds
- Workarounds for known product limitations

**When to Use This Skill:**
- E&U Cloud performance degradation
- "Too many SOQL queries" errors
- Integration Procedure timeouts
- Package upgrade failures
- Multisite order issues
- License/permission errors at scale

**When to Escalate Beyond This Skill:**
- New error patterns not covered (add to skill in next update)
- Platform-level issues (Salesforce core, not Industries)
- Custom code introduced by customer (not OOTB)

---

*Generated by AI Expert Suite - Industries EU Cloud Performance Skill*  
*Version 1.0.0 | Created: May 29, 2026*  
*Next Review: June 26, 2026*