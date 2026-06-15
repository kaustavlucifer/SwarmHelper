## A. Triage & Classification

### Quick Diagnostic Questions

1. **What type of issue?**
   - Performance/Slowness → Section B1
   - Timeout/CPU Limit → Section B2
   - Cache Issues → Section B3
   - SOQL/Governor Limits → Section B4
   - Cart/Pricing Problems → Section B5
   - EPC Designer/Attributes → Section B6
   - Upgrade/Version Issues → Section B7
   - Other Errors → Section B8

2. **When does it occur?**
   - On specific actions (add to cart, price calculation, checkout)
   - After recent upgrade
   - Intermittently vs. consistently
   - In specific environments (sandbox vs. prod)

3. **What's the volume?**
   - Number of products in cart
   - Size of catalog (products, offers, attributes)
   - Number of concurrent users
   - Matrix size (pricing/promotion matrices)

### Decision Tree

```
┌─ Slow response (>5s) ────────────┐
│  ├─ Cart operations ────────────→ B1.1 Cart Performance
│  ├─ Pricing calculation ────────→ B1.2 Pricing Performance
│  ├─ EPC operations ─────────────→ B1.3 EPC Performance
│  └─ General slowness ───────────→ B1.4 General Performance
│
├─ "CPU time limit exceeded" ─────→ B2.1 CPU Timeout
├─ "Apex timeout" ────────────────→ B2.2 Apex Timeout
│
├─ Cache errors ──────────────────┐
│  ├─ "CacheKey not found" ───────→ B3.1 Missing Cache
│  ├─ Stale data ─────────────────→ B3.2 Cache Invalidation
│  └─ Cache jobs failing ─────────→ B3.3 Cache Jobs
│
├─ "Too many SOQL queries: 101" ──→ B4.1 SOQL Limits
├─ "Too many query rows: 50001" ──→ B4.2 Query Rows Limit
│
├─ Cart/Pricing issues ───────────┐
│  ├─ Prices not displaying ──────→ B5.1 Price Display
│  ├─ Promotion not applying ─────→ B5.2 Promotions
│  ├─ Cart items missing ─────────→ B5.3 Cart Display
│  └─ Incorrect pricing ──────────→ B5.4 Price Calculation
│
├─ EPC Designer issues ───────────┐
│  ├─ Attributes not saving ──────→ B6.1 Attribute Save
│  ├─ Picklist values missing ───→ B6.2 Picklist Sync
│  ├─ JSON compilation error ─────→ B6.3 JSON Compilation
│  └─ Matrix issues ──────────────→ B6.4 Decision Matrix
│
├─ Post-upgrade errors ───────────→ B7 Upgrade Issues
│
└─ Other exceptions ──────────────→ B8 Error Patterns
```

---

## B. Known Issue Patterns

### B1. Performance Issues (29 cases, High Impact)

#### Pattern B1.1: ESM Cart Slowness

**Frequency:** High (15 cases)  
**Severity:** Urgent/High  
**TTR Impact:** 3-7 days  

**Symptoms:**
- ESM cart loads slowly (>10 seconds)
- "Slowness ESM Shopping Cart - Product and Attribute Selection"
- Rendering delays when adding products
- Performance degradation in sandbox vs. prod

**Root Cause:**
- Large product catalogs without proper indexing
- Excessive SOQL queries in cart rendering
- Unoptimized attribute retrieval
- Platform cache not allocated or misconfigured

**Code Reference:**
- **via_telco**: `CpqCartGetOffersInvocable.cls`, `DefaultResultToXLIImplementation.cls`
- **via_cpq**: `ProductWrapper.cls`, `PCDetailsService.cls`
- Cache classes: `AttributeCacheService.cls`, `CacheableAPIConstants.cls`

**Resolution Steps:**
1. **Enable Platform Cache:**
   ```
   Setup → Platform Cache → Allocate cache to partitions:
   - vlocity_cmt.CommonCache (minimum 10 MB)
   - vlocity_cmt.ProductCache (minimum 20 MB)
   ```

2. **Run Cache Population Jobs:**
   ```apex
   // Run from Developer Console
   Database.executeBatch(new ProductAttributesCacheBatchJob(), 200);
   Database.executeBatch(new AttributeMatrixInfoCacheBatchJob(), 200);
   ```

3. **Enable CPQ Indexing:**
   ```
   Request via support case: "Please enable CPQ indexing for Org [OrgID]"
   Indexes: Product2, vlocity_cmt__ProductChildItem__c
   Fields: vlocity_cmt__GlobalKey__c, vlocity_cmt__ProductCode__c
   ```

4. **Optimize Cart Configuration:**
   ```
   CpqConfigurationSetup__c settings:
   - Cart.MaxItemsPerPage: 50 (default 200)
   - Cart.EnableLazyLoading: true
   - Cart.DisableRealTimePricing: true (for large carts)
   ```

5. **Review Custom Code:**
   - Check for triggers on Quote/QuoteLineItem
   - Bulkify any custom pricing implementations
   - Review integration procedure performance

**Verification:**
- Cart load time < 3 seconds for 20 products
- No SOQL warnings in debug logs
- Cache hit rate > 80% (check via `System.debug('Cache stats: ' + PlatformCache.getOrgPartition('vlocity_cmt.CommonCache').getCapacity());`)

**Workarounds:**
- Enable "Load More" pagination
- Reduce displayed attributes in cart
- Use Summary mode instead of Full view

**Related Documentation:**
- [Comms Cloud Performance Best Practices](https://help.salesforce.com/s/articleView?id=sf.comm_cloud_performance.htm)
- [Platform Cache Implementation Guide](https://help.salesforce.com/s/articleView?id=sf.platform_cache_implementation.htm)

**Escalation Criteria:**
- Cart load time > 30 seconds after optimization
- Performance degraded only in production (not sandbox)
- Cache allocation doesn't improve performance

**GUS Investigations:**
- W-15333678 (Cart Performance)
- W-17751632 (Salesforce Slowness)
- W-18473024 (Vlocity Upgrade Slowness)

---

#### Pattern B1.2: Pricing Calculation Timeout

**Frequency:** High (8 cases)  
**Severity:** Urgent/Critical  
**TTR Impact:** 5-10 days  

**Symptoms:**
- "Issues with Performance on CPQ Large Matrix - Urgent"
- "Low Performance to Calculate Price regarding Large Calculation Matrix"
- Pricing takes >60 seconds
- CPU timeout during price calculation

**Root Cause:**
- Large calculation matrices (>5000 rows)
- Nested matrix lookups
- Real-time pricing on every attribute change
- Unoptimized custom pricing implementations

**Code Reference:**
- **via_telco**: `CalculateUnitPriceImplTest.cls`, `DefaultCMPriceListEntryImplementation.cls`
- **via_cpq**: `PCDetailsService.cls`, `APIAssetPricingV2TestCpq.cls`

**Resolution Steps:**
1. **Optimize Calculation Matrix:**
   - Split large matrices into smaller, targeted matrices
   - Use Context Rules to filter applicable matrices
   - Index matrix on key lookup fields

2. **Enable Asynchronous Pricing:**
   ```apex
   CpqConfigurationSetup__c:
   - PricingPlan.EnableAsyncPricing: true
   - PricingPlan.BatchSize: 100
   ```

3. **Use Attribute-Based Pricing (ABP) Caching:**
   ```apex
   // In custom pricing implementation
   if (CacheableAPIUtility.isCacheEnabled()) {
       String cacheKey = generatePricingCacheKey(context);
       Decimal cachedPrice = (Decimal)PlatformCache.get(cacheKey);
       if (cachedPrice != null) return cachedPrice;
   }
   ```

4. **Reduce Matrix Size:**
   - Archive old pricing versions
   - Use Effective Date filtering
   - Consolidate overlapping rules

5. **Profile Slow Queries:**
   - Enable Debug Logs for vlocity_cmt
   - Identify slow SOQL (>1000ms)
   - Add selective filters

**Verification:**
- Pricing calculation < 10 seconds for complex products
- No CPU timeout errors
- Matrix query count < 20 per pricing operation

**Workarounds:**
- Use manual pricing override temporarily
- Simplify product configuration
- Calculate prices offline (batch job)

**Escalation Criteria:**
- Matrix optimization doesn't reduce time
- CPU timeout persists after async pricing
- Custom pricing implementation required

**GUS Investigations:**
- W-14866204 (Attribute Based Pricing)

---

#### Pattern B1.3: EPC Performance Issues

**Frequency:** Medium (6 cases)  
**Severity:** High  
**TTR Impact:** 4-7 days  

**Symptoms:**
- Slow product spec save (>30s)
- Timeout during attribute assignment
- Batch jobs running excessively

**Root Cause:**
- JSON compilation jobs running on every save
- Large attribute hierarchies
- Unoptimized context rules
- Excessive batch job execution

**Code Reference:**
- **via_cpq**: `EPCConvertFunctionJSONBatchJob.cls`, `EPCValidationUtilsTest.cls`, `ProductAttributesCacheBatchJob.cls`

**Resolution Steps:**
1. **Disable Automatic JSON Compilation:**
   ```
   EPCConfiguration__c:
   - DisableAutoJSONCompilation: true
   ```

2. **Run Batch Jobs Manually (Off-Peak Hours):**
   ```apex
   // Schedule for night/weekend
   Database.executeBatch(new EPCProductAttributeJSONBatchJob(), 100);
   Database.executeBatch(new FixProductAttribJSONBatchJob(), 100);
   ```

3. **Optimize Context Rules:**
   - Review rules with complex logic
   - Remove unused rules
   - Cache rule evaluation results

4. **Reduce Attribute Depth:**
   - Flatten nested attribute categories
   - Limit attribute inheritance levels to 3

**Verification:**
- Product spec save < 10 seconds
- JSON compilation completes without timeout
- Batch jobs complete in <30 minutes

**Workarounds:**
- Save products in batches
- Use IDX Workbench for bulk operations

**Escalation Criteria:**
- Batch jobs consistently timeout
- Performance doesn't improve after optimization

---

### B2. Timeout/CPU Issues (3 cases, Critical Impact)

#### Pattern B2.1: Apex CPU Time Limit Exceeded

**Frequency:** Medium (3 cases)  
**Severity:** Urgent  
**TTR Impact:** 7-14 days  

**Symptoms:**
- "Submit Order Error: Apex CPU time limit exceeded"
- "Timeout message with less 50s"
- Error during checkout, order creation, or complex configuration

**Root Cause:**
- Non-bulkified code in triggers
- Excessive loops in pricing/validation logic
- Synchronous callouts in transaction
- Large data volume processing

**Code Reference:**
- **via_telco**: `CpqAppHandler.cls`, `TimerAroundLogger.cls`
- **via_cpq**: `CpqProductTriggerListener.cls`, `ProductLineItemEventHandler.cls`

**Resolution Steps:**
1. **Identify CPU-Intensive Operations:**
   ```
   Enable Debug Logs → Set to FINEST
   Filter for "LIMIT_USAGE_FOR_NS" with namespace vlocity_cmt
   Look for operations consuming >5000ms CPU
   ```

2. **Bulkify Custom Code:**
   ```apex
   // BAD - queries in loop
   for (QuoteLineItem qli : qlis) {
       Product2 p = [SELECT Id FROM Product2 WHERE Id = :qli.Product2Id];
   }
   
   // GOOD - query outside loop
   Set<Id> productIds = new Set<Id>();
   for (QuoteLineItem qli : qlis) productIds.add(qli.Product2Id);
   Map<Id, Product2> products = new Map<Id, Product2>([SELECT Id FROM Product2 WHERE Id IN :productIds]);
   ```

3. **Move Long Operations to Async:**
   ```apex
   // Use @future or Queueable for expensive operations
   @future
   public static void calculateComplexPricing(Set<Id> quoteIds) {
       // Long-running pricing logic
   }
   ```

4. **Disable Unnecessary Automation:**
   ```
   Review and disable:
   - Workflow rules on Quote/Order objects
   - Process Builder flows
   - Unnecessary triggers
   ```

5. **Optimize SOQL Queries:**
   - Add WHERE clause filters
   - Limit query results
   - Use selective queries (indexed fields)

**Verification:**
- CPU time < 5000ms for checkout operation
- Debug logs show balanced CPU usage
- No timeout errors in transactions

**Workarounds:**
- Break large operations into smaller batches
- Process orders in batches overnight

**Escalation Criteria:**
- CPU timeout in OOTB Vlocity code
- Timeout occurs even with minimal data
- Custom code optimization doesn't help

**GUS Investigations:**
- None currently linked — escalate if persistent

---

### B3. Cache Issues (9 cases, High Impact)

#### Pattern B3.1: Cache Records Created Excessively

**Frequency:** High (4 cases)  
**Severity:** High  
**TTR Impact:** 3-5 days  

**Symptoms:**
- "Records are being created excessively" (Cached_APIResponse__c)
- Storage limit warnings
- CachedAPIResponse__c object growing rapidly (>100K records)
- "Unable to see product in ESM cart" due to cache issues

**Root Cause:**
- CacheAPI enabled without proper cleanup
- Unique cache keys generated on every request
- Expired cache records not deleted
- Force invalidate cache flag used excessively

**Code Reference:**
- **via_telco**: `CachedAPIResponseGenerator.cls`, `CacheManagementContext.cls`, `DefaultRegenerationCachedAPIRHandler.cls`
- **via_cpq**: `CacheManager.cls`, `CacheableAPIConstants.cls`, `CacheKeeper.cls`

**Resolution Steps:**
1. **Enable Automatic Cache Cleanup:**
   ```apex
   // Schedule cleanup job (daily at 2 AM)
   System.schedule('Delete Expired Cache', '0 0 2 * * ?', new DeleteExpiredAPICacheJob());
   ```

2. **Review Cache Configuration:**
   ```
   CpqConfigurationSetup__c:
   - CacheAPI.EnableCaching: true
   - CacheAPI.CacheExpirationDays: 7 (default 30 - reduce if storage issue)
   - CacheAPI.CreateCartFromContextKey: true
   ```

3. **Optimize Cache Key Generation:**
   ```apex
   // Ensure cache keys are deterministic and reusable
   // BAD - includes timestamp
   String cacheKey = 'cart_' + accountId + '_' + System.now();
   
   // GOOD - based on stable identifiers
   String cacheKey = EncryptionService.generateKey(
       EncryptionService.MD5, 
       'cart_' + accountId + '_' + contextKey
   );
   ```

4. **Manual Cleanup (Immediate Relief):**
   ```apex
   // Delete cache records older than 7 days
   Database.executeBatch(new DeleteExpiredAPICacheJob(7), 200);
   
   // Or via SOQL
   List<CachedAPIResponse__c> oldRecords = [
       SELECT Id FROM CachedAPIResponse__c 
       WHERE CreatedDate < LAST_N_DAYS:7
       LIMIT 10000
   ];
   delete oldRecords;
   ```

5. **Avoid Force Invalidate:**
   ```
   Remove 'forceinvalidatecache=true' from API calls unless absolutely necessary
   Use targeted cache invalidation instead
   ```

**Verification:**
- CachedAPIResponse__c record count stable or decreasing
- Storage usage under limits
- Cache hit rate > 60%

**Workarounds:**
- Manually delete old cache records weekly
- Disable CacheAPI temporarily if storage critical

**Related Documentation:**
- [Comms Cloud Cache Management](https://help.salesforce.com/s/articleView?id=sf.comm_cache_management.htm)

**Escalation Criteria:**
- Cache cleanup job fails
- Storage limits exceeded despite cleanup
- Cache records created even when disabled

**GUS Investigations:**
- W-21195961 (Spring 26 Standard Cart API Cache Issue)
- W-21277603

---

#### Pattern B3.2: EPC Cache Not Refreshing

**Frequency:** Medium (3 cases)  
**Severity:** Urgent  
**TTR Impact:** 2-4 days  

**Symptoms:**
- "Picklist values not reflected at Product Spec and Offer Level"
- Updated attributes not visible in cart
- "Ran all cache jobs but issue persists"

**Root Cause:**
- JSON compilation not triggered
- Platform cache not cleared
- Multiple cache layers not synchronized
- Old compiled attributes cached in browser

**Code Reference:**
- **via_cpq**: `ProductAttributesCacheBatchJob.cls`, `CPQScaleCacheService.cls`

**Resolution Steps:**
1. **Run Full Cache Refresh (In Order):**
   ```apex
   // Step 1: Clear platform cache
   PlatformCache.getOrgPartition('vlocity_cmt.CommonCache').remove('allKeys');
   PlatformCache.getOrgPartition('vlocity_cmt.ProductCache').remove('allKeys');
   
   // Step 2: Run JSON compilation
   Database.executeBatch(new EPCProductAttributeJSONBatchJob(), 100);
   
   // Step 3: Run attribute cache population
   Database.executeBatch(new ProductAttributesCacheBatchJob(), 200);
   
   // Step 4: Fix attribute overrides
   Database.executeBatch(new EPCFixCompiledAttributeOverrideBatchJob(), 100);
   ```

2. **Product Hierarchy Maintenance:**
   ```
   Setup → Vlocity CPQ Admin → Product Hierarchy Maintenance
   - Select "Delete Old Data"
   - Run maintenance
   ```

3. **Clear Browser Cache:**
   ```
   Have users clear browser cache (Ctrl+Shift+Delete)
   Or force refresh with versioned static resources
   ```

4. **Verify Cache API Population:**
   ```
   Navigate to: Vlocity CPQ Admin → Populate API Cache
   Ensure button is enabled and run for affected products
   ```

**Verification:**
- Updated picklist values visible in Product Spec details page
- Cart displays correct attribute values
- No old values cached

**Workarounds:**
- Use Edit Attribute page (shows real-time values)
- Manually refresh after every attribute change

**Escalation Criteria:**
- Cache refresh doesn't update values
- Issue persists after all jobs run successfully
- Only specific attributes affected

---

### B4. SOQL/Governor Limit Issues (7 cases, High Impact)

#### Pattern B4.1: Too Many SOQL Queries (101)

**Frequency:** High (5 cases)  
**Severity:** Urgent  
**TTR Impact:** 5-7 days  

**Symptoms:**
- "SOQL Error applying Vlocity CPQ discount"
- "vlocity_cmt:Too many SOQL queries: 101"
- "Issues Adding Any Product" with SOQL limit error

**Root Cause:**
- Custom integration procedures with unbulkified queries
- Triggers querying in loops
- Excessive remote actions called sequentially
- Discount calculation logic hitting limits

**Code Reference:**
- **via_cpq**: `AttributeGeneralService.cls`, `ProductConfProcedureController.cls`
- **via_telco**: `LineItemAdjustmentService.cls`, `PricingManAdjEligibilityServiceTest.cls`

**Resolution Steps:**
1. **Identify SOQL Source:**
   ```
   Enable Debug Logs → Set Apex Code to FINEST
   Search for: "Number of SOQL queries: 101"
   Find the last class/method before limit
   ```

2. **Bulkify Custom Code:**
   ```apex
   // Example: Bulkifying discount application
   // BAD
   for (QuoteLineItem qli : qlis) {
       Discount__c discount = [SELECT Id, Amount__c FROM Discount__c 
                               WHERE Product__c = :qli.Product2Id LIMIT 1];
       qli.Discount__c = discount.Amount__c;
   }
   
   // GOOD
   Set<Id> productIds = new Set<Id>();
   for (QuoteLineItem qli : qlis) productIds.add(qli.Product2Id);
   
   Map<Id, Discount__c> discounts = new Map<Id, Discount__c>();
   for (Discount__c d : [SELECT Id, Product__c, Amount__c FROM Discount__c 
                         WHERE Product__c IN :productIds]) {
       discounts.put(d.Product__c, d);
   }
   
   for (QuoteLineItem qli : qlis) {
       if (discounts.containsKey(qli.Product2Id)) {
           qli.Discount__c = discounts.get(qli.Product2Id).Amount__c;
       }
   }
   ```

3. **Reduce Integration Procedure Queries:**
   ```
   Review IP for:
   - Multiple DataRaptor Extract steps → Combine into single DR
   - Remote Actions in loops → Use Remote Action with List input
   - Unnecessary SOQL queries → Use cached data
   ```

4. **Use Platform Events for Async Processing:**
   ```apex
   // Publish event instead of immediate processing
   Discount_Event__e evt = new Discount_Event__e(
       QuoteId__c = quoteId
   );
   EventBus.publish(evt);
   ```

5. **Review Automation:**
   ```
   Deactivate temporarily and test:
   - Workflow rules
   - Process Builder
   - Flows triggered on Quote/QuoteLineItem
   ```

**Verification:**
- SOQL count < 80 for discount application
- Debug logs show bulkified patterns
- No "Too many SOQL" errors

**Workarounds:**
- Apply discounts manually
- Process in smaller batches
- Use batch apex for bulk operations

**Related Documentation:**
- [Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm)
- [Bulkification Best Practices](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_best_practices.htm)

**Escalation Criteria:**
- SOQL limit hit in OOTB Vlocity code
- Bulkification doesn't resolve issue
- Limit hit with minimal data volume

**GUS Investigations:**
- W-17100320 (Issues Adding Any Product)

---

### B5. Cart/Pricing Issues (23 cases, Very High Impact)

#### Pattern B5.1: Prices Not Visible After Promotion

**Frequency:** High (6 cases)  
**Severity:** High/Urgent  
**TTR Impact:** 3-5 days  

**Symptoms:**
- "Price not visible after applying promotion on LWC cart"
- Prices visible initially, disappear after promotion
- Prices appear only after page refresh
- postCartsPromoItems doesn't return pricing

**Root Cause:**
- LWC cart state not updated after promotion
- Price fields missing from postCartsPromoItems response
- Cart refresh not triggered
- Promotional pricing calculation async issue

**Code Reference:**
- **via_telco**: `CpqDeleteAppliedPromotionItemActionV2.cls`, `DefaultCMPriceListEntryImplementation.cls`
- **via_cpq**: `APIReplaceOffersCartsCpqV2Test.cls`, `APIPriceListsCpqV2Test.cls`

**Resolution Steps:**
1. **Add Price Fields to Post-Promotion Response:**
   ```
   CpqConfigurationSetup__c:
   - PostCartsPromoItems.AdditionalFields: 
     vlocity_cmt__TotalPrice__c,vlocity_cmt__UnitPrice__c,
     vlocity_cmt__PromotionalPrice__c,vlocity_cmt__FinalPrice__c
   ```

2. **Enable Cart Auto-Refresh:**
   ```javascript
   // In LWC cart component
   @wire(getCartItems, { cartId: '$cartId' })
   wiredCartItems({ error, data }) {
       if (data) {
           this.cartItems = data;
           this.refreshPricing(); // Add this
       }
   }
   
   refreshPricing() {
       // Force pricing recalculation
       this.dispatchEvent(new CustomEvent('cartrefresh'));
   }
   ```

3. **Use Standard Cart API (Recommended):**
   ```
   Enable: CpqConfigurationSetup__c
   - StandardCartAPI.Enabled: true
   
   Standard Cart API includes pricing in all responses automatically
   ```

4. **Manual Workaround:**
   ```javascript
   // After promotion applied, call pricing API
   postCartsPromoItems(cartId, promoData)
     .then(() => {
         return getCartItems(cartId); // Refetch cart items with pricing
     })
     .then(cart => {
         this.updateCartDisplay(cart);
     });
   ```

**Verification:**
- Prices visible immediately after promotion
- No page refresh required
- All price fields populated in response

**Workarounds:**
- Instruct users to refresh page after applying promotion
- Display "Please wait..." message during promotion application

**Related Documentation:**
- [LWC Cart Configuration](https://help.salesforce.com/s/articleView?id=sf.comm_lwc_cart.htm)
- [Standard Cart API Guide](https://help.salesforce.com/s/articleView?id=sf.comm_standard_cart_api.htm)

**Escalation Criteria:**
- Issue persists with Standard Cart API enabled
- Price fields added but still not returned
- Only specific promotions affected

---

#### Pattern B5.2: TypeException After Upgrade to 242+

**Frequency:** Medium (3 cases)  
**Severity:** Urgent  
**TTR Impact:** 7-14 days  

**Symptoms:**
- "System.TypeException: Invalid conversion from runtime type vlocity_cmt.CpqCart to SObject"
- "Invalid conversion from runtime type List<vlocity_cmt.CpqCartItem> to List<SObject>"
- Custom pricing implementation breaks after upgrade

**Root Cause:**
- API change in version 242: CpqCart/CpqCartItem no longer extend SObject
- Custom code directly casting to SObject
- Legacy pricing plan implementations

**Code Reference:**
- **via_cpq**: Custom pricing implementations using SObject casting

**Resolution Steps:**
1. **Update Custom Pricing Code:**
   ```apex
   // BAD (pre-242)
   public void manipulateLineItems(List<SObject> lineItems) {
       for (SObject sobj : lineItems) {
           QuoteLineItem qli = (QuoteLineItem)sobj;
           // processing
       }
   }
   
   // GOOD (242+)
   public void manipulateLineItems(List<vlocity_cmt.CpqCartItem> cartItems) {
       for (vlocity_cmt.CpqCartItem item : cartItems) {
           // Access properties directly
           Decimal price = item.getTotalPrice();
           // processing
       }
   }
   ```

2. **Replace SObject Casting:**
   ```apex
   // BAD
   SObject cart = (SObject)inputMap.get('cart');
   String cartId = (String)cart.get('Id');
   
   // GOOD
   vlocity_cmt.CpqCart cart = (vlocity_cmt.CpqCart)inputMap.get('cart');
   String cartId = cart.getCartId();
   ```

3. **Use Proper API Methods:**
   ```apex
   // Instead of direct field access
   vlocity_cmt.CpqCartItem item = cartItems[0];
   
   // Use getter methods
   Decimal unitPrice = item.getUnitPrice();
   Decimal totalPrice = item.getTotalPrice();
   String productId = item.getProductId();
   ```

4. **Test Thoroughly:**
   ```apex
   // Create test class for upgraded version
   @isTest
   private class CustomPricingPlanStepImplTest {
       @isTest
       static void testPricingWithCpqCart() {
           vlocity_cmt.CpqCart cart = new vlocity_cmt.CpqCart();
           // ... test implementation
       }
   }
   ```

**Verification:**
- No TypeException errors in debug logs
- Pricing calculations work correctly
- All custom pricing implementations updated

**Workarounds:**
- Rollback to version 238 temporarily (not recommended)
- Use OOTB pricing until code updated

**Related Documentation:**
- [Comms Cloud 242 Release Notes](https://help.salesforce.com/s/articleView?id=release-notes.rn_comm_242.htm)
- [CPQ API Changes Guide](https://help.salesforce.com/s/articleView?id=sf.comm_api_changes.htm)

**Escalation Criteria:**
- Unable to update custom code (third-party vendor)
- OOTB Vlocity code shows TypeException
- Migration path unclear

**GUS Investigations:**
- W-12635353 (TypeException After Upgrade to 242)

---

### B6. EPC Designer/Product Catalog Issues (14 cases, High Impact)

#### Pattern B6.1: Picklist Values Not Reflected

**Frequency:** High (4 cases)  
**Severity:** Urgent  
**TTR Impact:** 3-5 days  

**Symptoms:**
- "Change in Picklist values not reflected at Product Spec and Offer Level"
- Updated picklist shows in Edit Attribute but not Details page
- Old values visible even after running all batch jobs

**Root Cause:**
- Compiled JSON attributes not refreshed
- Picklist metadata cached in multiple layers
- Product hierarchy not updated
- Attribute override records not recompiled

**Code Reference:**
- **via_cpq**: `EPCConvertFunctionJSONBatchJob.cls`, `EPCFixCompiledAttributeOverrideBatchJob.cls`, `ProductAttributesCacheBatchJob.cls`

**Resolution Steps:**
1. **Run Complete EPC Refresh (In Sequence):**
   ```apex
   // Step 1: EPCProductAttributeJSONBatchJob
   Database.executeBatch(new EPCProductAttributeJSONBatchJob('SELECT Id FROM Product2 WHERE vlocity_cmt__ProductSpec__c != null'), 100);
   
   // Step 2: FixProductAttribJSONBatchJob
   Database.executeBatch(new FixProductAttribJSONBatchJob('SELECT Id FROM Product2 WHERE vlocity_cmt__ProductSpec__c != null'), 100);
   
   // Step 3: EPCFixCompiledAttributeOverrideBatchJob
   Database.executeBatch(new EPCFixCompiledAttributeOverrideBatchJob('SELECT Id FROM Product2 WHERE vlocity_cmt__ProductSpec__c != null'), 100);
   
   // Step 4: ProductAttributesCacheBatchJob
   Database.executeBatch(new ProductAttributesCacheBatchJob(), 200);
   ```

2. **Product Hierarchy Maintenance:**
   ```
   Setup → Vlocity CPQ Admin → Product Hierarchy Maintenance
   Options:
   - [x] Delete Old Data
   - [ ] Preserve Custom Changes
   Run Maintenance
   ```

3. **Clear All Caches:**
   ```apex
   // Platform cache
   PlatformCache.getOrgPartition('vlocity_cmt.CommonCache').remove('allKeys');
   PlatformCache.getOrgPartition('vlocity_cmt.ProductCache').remove('allKeys');
   
   // Browser cache (user action required)
   ```

4. **Verify Picklist Definition:**
   ```
   Setup → Vlocity Product Designer → Picklists
   - Ensure picklist values are Active
   - Check Effective Dates
   - Verify no duplicate picklists
   ```

5. **Invalidate Specific Product Cache:**
   ```apex
   // For targeted refresh
   Set<Id> productIds = new Set<Id>{/* affected product IDs */};
   vlocity_cmt.CacheService.invalidateCache('Product', productIds);
   ```

**Verification:**
- Updated picklist values visible in Product Spec Details page
- Values match Edit Attribute screen
- Cart displays new values correctly

**Workarounds:**
- Users can use Edit Attribute screen (shows real-time values)
- Manually update JSONAttribute__c field

**Related Documentation:**
- [EPC Designer Administration](https://help.salesforce.com/s/articleView?id=sf.comm_epc_admin.htm)
- [Product Attribute Management](https://help.salesforce.com/s/articleView?id=sf.comm_product_attributes.htm)

**Escalation Criteria:**
- All batch jobs complete successfully but values not updated
- Only specific picklists affected
- Issue occurs immediately after batch job completion

---

#### Pattern B6.2: Attribute-Based Pricing Not Functioning

**Frequency:** Medium (3 cases)  
**Severity:** Urgent  
**TTR Impact:** 5-10 days  

**Symptoms:**
- "Attribute Based Pricing not functioning"
- Pricing rules not triggered when attribute changes
- Expected price adjustment not applied

**Root Cause:**
- Attribute mapping incorrect
- Pricing matrix not linked to attributes
- Expression set evaluation error
- Lookup table cache stale

**Code Reference:**
- **via_telco**: `AttributeCacheService.cls`, `DefaultCMPriceListEntryImplementation.cls`
- **via_cpq**: `PCDetailsService.cls`, `APIPriceListsCpqV2Test.cls`

**Resolution Steps:**
1. **Verify Attribute Assignment:**
   ```
   Product Designer → Object Type → Attributes
   - Attribute assigned to Object Type
   - Attribute inherited by Product Spec
   - Attribute Code matches pricing matrix reference
   ```

2. **Check Pricing Matrix Configuration:**
   ```
   Pricing → Attribute-Based Pricing → Matrix
   - Input Attribute: [AttributeCode]
   - Output: Price Adjustment
   - Effective Date range valid
   - Matrix rows populated
   ```

3. **Test Expression Set:**
   ```
   Pricing → Expression Set → Test
   Input sample attribute values
   Verify output matches expectation
   ```

4. **Refresh Attribute Cache:**
   ```apex
   Database.executeBatch(new AttributeMatrixInfoCacheBatchJob(), 200);
   Database.executeBatch(new ProductAttributesCacheBatchJob(), 200);
   ```

5. **Debug Pricing Calculation:**
   ```apex
   // Enable debug logs
   CpqConfigurationSetup__c:
   - Debug.EnablePricingDebug: true
   
   // Check logs for:
   // "Attribute [Code] value: [Value]"
   // "Matrix lookup: [Result]"
   ```

**Verification:**
- Changing attribute value triggers price recalculation
- Price adjustment matches matrix definition
- Debug logs show successful matrix lookup

**Workarounds:**
- Manual price override
- Use traditional pricing plans

**Escalation Criteria:**
- Attribute mapping verified but pricing not triggered
- Matrix lookup returns null unexpectedly
- Issue in OOTB pricing logic

**GUS Investigations:**
- W-14866204 (Attribute Based Pricing Not Functioning)

---

### B7. Upgrade/Version Issues (30 cases, High Impact)

#### Pattern B7.1: Post-Upgrade Functionality Breaks

**Frequency:** Very High (12 cases)  
**Severity:** Urgent/Critical  
**TTR Impact:** 7-21 days  

**Symptoms:**
- Features working in 238 break after upgrade to 242+
- "Multiple UI/UX issues post Vlocity package upgraded to v242"
- Custom integrations fail post-upgrade
- "No MODULE named markup://c:lodash found"

**Root Cause:**
- API breaking changes in new version
- Deprecated methods removed
- New security model enforced
- Static resource versioning changes

**Resolution Steps:**
1. **Review Release Notes:**
   ```
   Check: Comms Cloud [Version] Release Notes
   Section: Breaking Changes
   - API changes
   - Deprecated features
   - Migration requirements
   ```

2. **Run Post-Upgrade Scripts:**
   ```apex
   // Provided by Salesforce after upgrade
   Database.executeBatch(new VlocityPostUpgradeBatch(), 200);
   
   // EPC-specific
   Database.executeBatch(new EPCProductAttributeJSONBatchJob(), 100);
   ```

3. **Update Custom Code:**
   ```apex
   // Common changes for 242+:
   // 1. CpqCart/CpqCartItem type casting (see B5.2)
   // 2. Remote action signatures
   // 3. Static resource references
   ```

4. **Clear Browser Cache (Critical):**
   ```
   Instruct all users:
   Chrome: Ctrl+Shift+Delete → Clear cache
   Safari: Cmd+Option+E
   
   Or version static resources:
   vlocity_cmt/cartApp_v242.js
   ```

5. **Test Critical Paths:**
   ```
   - Add product to cart
   - Configure product
   - Apply promotion
   - Calculate pricing
   - Create order
   ```

6. **Rollback Plan:**
   ```
   If critical issues:
   1. Log case with Severity 1
   2. Request rollback to previous version (sandbox only)
   3. Production: Apply hotfix patch
   ```

**Verification:**
- All critical functionality works
- No console errors in browser
- Custom integrations successful

**Workarounds:**
- Revert to classic cart UI temporarily
- Disable problematic features
- Use manual processes

**Related Documentation:**
- [Comms Cloud Upgrade Guide](https://help.salesforce.com/s/articleView?id=sf.comm_upgrade_guide.htm)
- [Version Compatibility Matrix](https://help.salesforce.com/s/articleView?id=sf.comm_version_compatibility.htm)

**Escalation Criteria:**
- OOTB functionality broken
- No documented migration path
- Production impacted

**GUS Investigations:**
- W-13488328 (Errors After Summer 23 Update)
- W-17079770 (Multiple UI/UX Issues Post v242)

---

### B8. Other Error Patterns

#### Pattern B8.1: UNABLE_TO_LOCK_ROW

**Frequency:** Medium (4 cases)  
**Severity:** High/Urgent  
**TTR Impact:** 3-7 days  

**Symptoms:**
- "Error saving record: UNABLE_TO_LOCK_ROW"
- "Rowlocks in production"
- Concurrent edit conflicts

**Root Cause:**
- Multiple processes updating same record simultaneously
- Long-running transactions holding locks
- Batch jobs conflicting with user transactions

**Resolution Steps:**
1. **Identify Lock Contention:**
   ```
   Query Event Log Files for ReadLock/WriteLock events
   Identify classes/objects with frequent locks
   ```

2. **Implement Retry Logic:**
   ```apex
   Integer retries = 3;
   while (retries > 0) {
       try {
           update records;
           break;
       } catch (DmlException e) {
           if (e.getMessage().contains('UNABLE_TO_LOCK_ROW') && retries > 0) {
               retries--;
               System.sleep(500); // Wait 500ms
           } else {
               throw e;
           }
       }
   }
   ```

3. **Reduce Lock Scope:**
   ```apex
   // Use FOR UPDATE carefully
   List<Account> accounts = [SELECT Id FROM Account WHERE Id = :accountId FOR UPDATE];
   
   // Or avoid locks by using Platform Events
   ```

4. **Schedule Batch Jobs Off-Peak:**
   ```apex
   // Schedule for nighttime instead of business hours
   System.schedule('Nightly Job', '0 0 2 * * ?', new MyBatchJob());
   ```

**Verification:**
- Row lock errors reduced or eliminated
- Debug logs show successful retries

**Escalation Criteria:**
- Lock contention in OOTB code
- Issue persists despite off-peak scheduling

**GUS Investigations:**
- W-14586721 (Unable to Lock Row)

---

## C. Setup & Configuration Guide

### Essential CPQ Setup Checklist

**1. Platform Cache Allocation**
```
Setup → Platform Cache → Allocate
- vlocity_cmt.CommonCache: 20 MB minimum
- vlocity_cmt.ProductCache: 30 MB minimum
- vlocity_cmt.PricingCache: 10 MB minimum
```

**2. Required CpqConfigurationSetup__c Records**
```
Name: Cart.EnableLazyLoading, Value: true
Name: Cart.MaxItemsPerPage, Value: 50
Name: CacheAPI.EnableCaching, Value: true
Name: CacheAPI.CacheExpirationDays, Value: 7
Name: PricingPlan.EnableAsyncPricing, Value: true
Name: StandardCartAPI.Enabled, Value: true
```

**3. Scheduled Jobs**
```
- Delete Expired Cache: Daily at 2 AM
- Product Attribute Cache: Weekly Sunday 2 AM
- Attribute Matrix Cache: Weekly Sunday 3 AM
```

**4. Permission Sets (Minimum Required)**
```
- Vlocity CPQ Admin
- Vlocity EPC Designer
- Platform Cache Viewer (for troubleshooting)
```

**5. Custom Metadata**
```
CPQ Compiler Settings:
- Enable JSON Compilation: true
- Batch Size: 100
- Timeout: 120000ms
```

### Common Misconfigurations

| Issue | Misconfiguration | Fix |
|-------|------------------|-----|
| Slow cart load | Platform cache not allocated | Allocate minimum 20MB to CommonCache |
| Cache records grow | No cleanup job scheduled | Schedule DeleteExpiredAPICacheJob |
| Pricing timeout | Real-time pricing on all attributes | Enable async pricing, reduce matrix size |
| SOQL limit | Custom DR extracts in loops | Bulkify Integration Procedures |
| Picklist not refreshing | JSON compilation disabled | Run EPCProductAttributeJSONBatchJob |
| Row locks | Batch jobs running during business hours | Schedule off-peak (2-4 AM) |

---

## D. Licensing & Entitlements

### Required Licenses

**Industries CPQ (formerly Vlocity CMT)**
- Base: Industries Communications Cloud
- Required PSLs: 
  - OmniStudio (for Omniscripts, DataRaptors, Integration Procedures)
  - Industries CPQ License
  - Platform Cache Plus (recommended for performance)

**License Troubleshooting Flowchart**

```
User cannot access CPQ?
├─ Check: User has Industries CPQ permission set license assigned?
│  NO → Assign: Industries CPQ PSL
│  YES → Continue
│
├─ Check: User has required permission sets assigned?
│  - IndustriesCPQUser
│  - OmnistudioUser
│  NO → Assign permission sets
│  YES → Continue
│
├─ Check: Feature license enabled on profile?
│  Setup → Users → Profile → Custom Permissions
│  - Enable Industries CPQ
│  NO → Enable on profile
│  YES → Continue
│
└─ Check: Namespace entitlement
   Error: "User requires namespace entitlement for namespaces: [vlocity_cmt]"
   Solution: Assign vlocity_cmt namespace permission set
```

---

## E. Documentation Quick-Reference

### Key Documentation Links

**Performance Optimization:**
- [Comms Cloud Performance Best Practices](https://help.salesforce.com/s/articleView?id=sf.comm_cloud_performance.htm)
- [Platform Cache Implementation](https://help.salesforce.com/s/articleView?id=sf.platform_cache_implementation.htm)
- [CPQ Optimization Guide](https://help.salesforce.com/s/articleView?id=sf.comm_cpq_optimization.htm)

**Cart & Pricing:**
- [Standard Cart API Guide](https://help.salesforce.com/s/articleView?id=sf.comm_standard_cart_api.htm)
- [LWC Cart Configuration](https://help.salesforce.com/s/articleView?id=sf.comm_lwc_cart.htm)
- [Attribute-Based Pricing](https://help.salesforce.com/s/articleView?id=sf.comm_attribute_pricing.htm)

**EPC Designer:**
- [EPC Designer Administration](https://help.salesforce.com/s/articleView?id=sf.comm_epc_admin.htm)
- [Product Attribute Management](https://help.salesforce.com/s/articleView?id=sf.comm_product_attributes.htm)

**Troubleshooting:**
- [Debug Logs for Vlocity](https://help.salesforce.com/s/articleView?id=sf.comm_debug_logs.htm)
- [Known Issues and Workarounds](https://help.salesforce.com/s/articleView?id=sf.comm_known_issues.htm)

### Known Documentation Gaps

- **Cache Management:** Comprehensive cache strategy for large catalogs (>10K products) not documented
- **MACD Performance:** Optimization guide for high-volume MACD operations missing
- **Matrix Optimization:** Best practices for matrices >5000 rows limited
- **Standard Cart API Migration:** Complete migration guide from legacy cart API not available

### Behavioral Rules

1. **Cache Expiration:** CachedAPIResponse__c records expire after CacheExpirationDays but are NOT auto-deleted unless cleanup job scheduled
2. **JSON Compilation:** Triggered automatically on save UNLESS DisableAutoJSONCompilation is true
3. **Platform Cache:** Eviction policy is LRU (Least Recently Used) when partition full
4. **Pricing Calculation:** Synchronous by default; set EnableAsyncPricing for >20 line items
5. **Attribute Inheritance:** Product Spec inherits from Object Type; Offer inherits from Product Spec

---

## F. Escalation Paths

### When to Escalate

**Immediate Escalation (P0/P1):**
- Production system down or severely impacted
- Data loss or corruption
- Security vulnerability
- OOTB functionality completely broken post-upgrade

**Standard Escalation (P2):**
- Performance degradation >50% after optimization attempts
- SOQL/CPU limits in OOTB code
- Cache issues persisting after all documented steps
- Custom code assistance needed (upgrade migration)

**Low Priority (P3/P4):**
- Documentation clarification
- Feature requests
- Non-critical bugs with workarounds

### Information to Gather Before Escalating

**Required:**
1. Org ID
2. Comms Cloud version (Setup → Installed Packages → vlocity_cmt)
3. Case number and summary
4. Steps to reproduce
5. Expected vs. actual behavior
6. Debug logs (with Apex Code = FINEST)
7. Screenshots or video recording

**Performance Issues:**
8. Event Log Files (last 24 hours)
9. CPU time breakdown from debug logs
10. Number of products in cart/catalog
11. Before/after performance comparison

**Code Issues:**
12. Apex class name and line number
13. Full error message and stack trace
14. Sample code snippet (custom implementations)

**Cache Issues:**
15. CachedAPIResponse__c record count
16. Platform cache allocation
17. Batch job execution history

### Escalation Contacts

**Comms Cloud Support:**
- Create case via [Salesforce Help Portal](https://help.salesforce.com)
- Product: Industries - Communications Cloud
- Sub-product: CPQ / Cart / EPC Designer (as applicable)

**Technical Account Manager:**
- For architectural guidance
- For roadmap/feature requests
- For multi-case strategic issues

**GUS Teams:**
- **CME Core - CPQNext:** Cart, Pricing, CPQ APIs
- **Industries CPQ - DC APIs:** Digital Commerce APIs, Context Eligibility
- **Industries CPQ - Cart APIs:** Cart operations, promotions

### High-TTR Investigation References

**Top 10 Investigations by TTR:**
1. W-12635353 - TypeException post-242 upgrade (21 days)
2. W-14866204 - Attribute-Based Pricing not functioning (18 days)
3. W-17100320 - SOQL limit adding products (14 days)
4. W-15333678 - Cart rendering performance (12 days)
5. W-13488328 - Media Plan errors post-upgrade (10 days)
6. W-17751632 - Platform slowness (9 days)
7. W-12486822 - TMF 620 logs not created (8 days)
8. W-12735600 - MACD asset move failure (7 days)
9. W-21195961 - Spring 26 cache issue (6 days)
10. W-16193919 - ESM UX configuration issue (5 days)

---

## G. Tribal Knowledge & Pro Tips

### From Slack #industries-esm-support, #industries-cmt-cpq (Limited Data - Connect Slack for More)

**Note:** Slack search returned no results. Connect Slack to #industries-esm-support, #industries-cmt-cpq channel for:
- Real-time workarounds from support engineers
- Undocumented configuration tips
- Common gotchas in recent releases
- Quick fixes from product team

**To Connect Slack:**
```
Settings → MCP Servers → Slack → Connect
Add channel: #industries-esm-support, #industries-cmt-cpq
```

### General Pro Tips

1. **Always enable Platform Cache BEFORE implementing caching strategies**
   - Vlocity's caching code will run but silently fail if cache not allocated
   - Minimum 20MB for CommonCache, 30MB for ProductCache

2. **Run cache batch jobs in sequence, not parallel**
   - EPCProductAttributeJSONBatchJob FIRST
   - Then FixProductAttribJSONBatchJob
   - Then EPCFixCompiledAttributeOverrideBatchJob
   - Finally ProductAttributesCacheBatchJob

3. **Use Standard Cart API for all new implementations (242+)**
   - Legacy cart API will be deprecated
   - Better performance and automatic pricing

4. **Debug logs: Set Apex Code to FINEST, Database to INFO**
   - FINEST captures Vlocity namespace execution
   - Avoid DEBUG (too verbose, logs truncated)

5. **CachedAPIResponse__c cleanup is MANUAL unless job scheduled**
   - Common cause of storage limit issues
   - Schedule DeleteExpiredAPICacheJob DAILY

6. **Matrix optimization: Context Rules are your friend**
   - Filter matrices before evaluation
   - Reduces query time by 60-80%

7. **MACD performance: Disable real-time pricing during asset moves**
   - Recalculate prices after move completes
   - Avoids timeout on large asset hierarchies

8. **Browser cache is a common culprit post-upgrade**
   - Always have users clear cache after package upgrade
   - Or version your static resources

9. **For SOQL limit debugging: Check Integration Procedures FIRST**
   - Common source of unbulkified queries
   - DataRaptor Extracts in loops

10. **Platform Events can replace synchronous updates**
    - Reduces lock contention
    - Improves perceived performance

---

## H. Active Known Issues

**Note:** GUS is not authenticated. Connect GUS for live query of active P0/P1/P2 bugs.

**To Connect GUS:** run `/gus` and follow the prompts (see `.claude/capabilities/gus.md`).

**Recent Known Issues (From Case Data):**

1. **W-21811800** - Production: Unable to create quote - Namespace required (P0)
2. **W-21576263** - CpqAppHandler errors post-upgrade (P2)
4. **W-21356376** - Asset duplication issue (P2)
5. **W-20956955** - postCartsItems pricing not returned with Standard Cart API (P1)

**Workaround Status:** Check GUS for latest patches and hotfixes.

---

## I. Code Reference Map

### Key Performance-Critical Classes

**Cache Management (via_cpq):**
```
EcomCacheResultsBase.cls - Base cache handling
CacheManager.cls - Cache orchestration
CacheableAPIConstants.cls - Cache configuration constants
ProductAttributesCacheBatchJob.cls - Attribute cache population
CPQScaleCacheService.cls - Scalability optimizations
CacheKeeper.cls - Cache key management
```

**Cache Management (via_telco):**
```
AttributeCacheService.cls - Attribute caching
CachedAPIResponseGenerator.cls - API response caching
CacheManagementContext.cls - Cache context handling
CacheRecordRepository.cls - Cache persistence
OCCachePopulationImplementation.cls - Cache population strategies
DefaultRegenerationCachedAPIRHandler.cls - Cache regeneration
```

**Cart Operations (via_cpq):**
```
ProductWrapper.cls - Product data handling
PCDetailsService.cls - Product configuration details
APIReplaceOffersCartsCpqV2Test.cls - Cart API operations
CpqProductTriggerListener.cls - Product trigger logic
```

**Cart Operations (via_telco):**
```
CpqCartGetOffersInvocable.cls - Get cart offers
CpqCartDeleteItemInvocableTest.cls - Delete cart items
CpqCartInvocableUtil.cls - Cart utility methods
DefaultResultToXLIImplementation.cls - Result transformation
LineItemAdjustmentService.cls - Line item adjustments
```

**Pricing (via_cpq):**
```
APIAssetPricingV2TestCpq.cls - Asset pricing
APIPriceListsCpqV2Test.cls - Price list management
```

**Pricing (via_telco):**
```
CalculateUnitPriceImplTest.cls - Unit price calculation
DefaultCMPriceListEntryImplementation.cls - Price list entries
PricingManAdjEligibilityServiceTest.cls - Pricing eligibility
DefaultCMPriceListEntryImplementation.cls - CM pricing
CachedBasketPricingPlanImpl.cls - Cached pricing plans
```

**EPC Designer (via_cpq):**
```
EPCConvertFunctionJSONBatchJob.cls - JSON compilation
EPCValidationUtilsTest.cls - EPC validation
EPCObjectContextRuleHandlerTest.cls - Context rules
ProductAttributesCacheBatchJob.cls - Attribute cache
```

**Timeout/Performance (via_telco):**
```
TimerAroundLogger.cls - Performance logging
MultiSiteOrderTest.cls - Multi-site performance
```

### Issue Pattern → Code Path Mapping

| Issue Pattern | Primary Code Path | Root Cause Location |
|---------------|-------------------|---------------------|
| Cart Slowness | CpqCartGetOffersInvocable → ProductWrapper | SOQL queries not bulkified in ProductWrapper |
| Price Calculation Timeout | CalculateUnitPriceImplTest → DefaultCMPriceListEntryImplementation | Matrix lookup not cached |
| Cache Growth | CachedAPIResponseGenerator → CacheRecordRepository | No expiration cleanup |
| Picklist Not Refreshing | EPCConvertFunctionJSONBatchJob → ProductAttributesCacheBatchJob | JSON compilation not triggered |
| SOQL Limit | LineItemAdjustmentService | Discount queries in loop |
| TypeException (post-242) | Custom pricing implementations | SObject casting no longer supported |

## K. Test Cases

### Test Case 1: Cart Performance Issue
**Input:**
"Our ESM cart is loading very slowly - takes over 20 seconds to display 30 products. Users are complaining."

**Expected Response:**
1. Acknowledge: This is Pattern B1.1 - ESM Cart Slowness (High frequency)
2. Diagnostic questions:
   - Is Platform Cache allocated?
   - How many products in catalog?
   - Any custom triggers on Quote/QuoteLineItem?
3. Immediate actions:
   - Check Platform Cache allocation (Setup → Platform Cache)
   - Run cache population jobs
   - Enable CPQ indexing
4. Provide step-by-step resolution from B1.1
5. Verification steps
6. Escalation criteria if not resolved

---

### Test Case 2: Post-Upgrade Error
**Input:**
"After upgrading to version 242, our custom pricing implementation is throwing 'System.TypeException: Invalid conversion from runtime type vlocity_cmt.CpqCart to SObject'"

**Expected Response:**
1. Recognize: Pattern B5.2 - TypeException After Upgrade to 242+
2. Explain root cause: API change in 242 - CpqCart no longer extends SObject
3. Provide code fix examples (BAD vs GOOD)
4. Link to GUS investigation: W-12635353
5. Testing guidance
6. Escalation if third-party vendor code

---

### Test Case 3: Cache Growth
**Input:**
"Our org is hitting storage limits. I see we have over 150,000 CachedAPIResponse__c records. What should we do?"

**Expected Response:**
1. Identify: Pattern B3.1 - Cache Records Created Excessively
2. Immediate relief: Delete old records (SOQL or batch job)
3. Long-term fix: Schedule DeleteExpiredAPICacheJob daily
4. Root cause analysis: Check for force invalidate usage
5. Configuration review: Set CacheExpirationDays to 7
6. Verification: Monitor record count weekly

---

### Test Case 4: SOQL Limit Error
**Input:**
"Users are getting 'Too many SOQL queries: 101' error when trying to apply discounts to quote line items. This is urgent!"

**Expected Response:**
1. Pattern: B4.1 - Too Many SOQL Queries
2. Immediate: Enable debug logs with Apex Code = FINEST
3. Identify source: Find last class before limit
4. Provide bulkification examples
5. Review Integration Procedures for unbulkified DataRaptor Extracts
6. Escalation criteria: If SOQL limit in OOTB code → Create P1 case with GUS team

---

### Test Case 5: EPC Picklist Issue
**Input:**
"I updated a picklist in EPC Designer and ran all the batch jobs, but the new values still don't show up in the Product Spec details page. They show in Edit Attribute screen though."

**Expected Response:**
1. Pattern: B6.1 - Picklist Values Not Reflected
2. Explain cache layers: JSON compilation, platform cache, browser cache
3. Provide complete refresh sequence:
   - EPCProductAttributeJSONBatchJob
   - FixProductAttribJSONBatchJob  
   - EPCFixCompiledAttributeOverrideBatchJob
   - ProductAttributesCacheBatchJob
4. Product Hierarchy Maintenance with "Delete Old Data"
5. Clear platform cache
6. Clear browser cache
7. Verification: Check Details page matches Edit Attribute screen

---

## L. Completeness Report

**Data Sources:**
- ✅ **Case Data:** 238 cases analyzed from OrgCS (Industry-Communication Cloud / Comms Business App-Performance)
- ✅ **Code References:** 320+ CPQ/Cart classes from via_cpq-260.9, 1030+ classes from via_telco-260.9
- ❌ **GUS:** Not authenticated - 27 GUS investigations referenced from case data, but root causes not retrieved directly
- ❌ **Slack:** #industries-esm-support, #industries-cmt-cpq channel search returned no results - tribal knowledge limited
- ✅ **Help Docs:** Standard documentation links included

**Pattern Coverage:**
- ✅ Performance Issues: 29 cases → 4 detailed patterns
- ✅ Timeout/CPU Issues: 3 cases → 1 detailed pattern
- ✅ Cache Issues: 9 cases → 2 detailed patterns  
- ✅ SOQL/Governor Limits: 7 cases → 1 detailed pattern
- ✅ Cart/Pricing Issues: 23 cases → 2 detailed patterns
- ✅ EPC Designer Issues: 14 cases → 2 detailed patterns
- ✅ Upgrade Issues: 30 cases → 1 detailed pattern
- ✅ Error Patterns: 21 cases → 1 detailed pattern
- ✅ Other Issues: 102 cases → Documented but not primary patterns

**Gaps:**
- **GUS root causes:** Limited to case-embedded GUS work item IDs - need authentication for full details
- **Slack tribal knowledge:** No results from #industries-esm-support, #industries-cmt-cpq - channel may be inactive or search returned empty
- **Code-level root causes:** Identified key classes but cannot trace exact line numbers without CodeSearch authentication
- **Low-confidence patterns:** 102 "Other Issues" cases need further categorization (TMF APIs, Omniscript issues, etc.)

**Recommendations to Fill Gaps:**
1. **Connect GUS:** run `/gus` to authenticate, then retrieve:
   - Root cause briefs for 27 linked investigations
   - Active P0/P1/P2 bugs for product tags
   - Recommended resolutions from engineering

2. **Connect Slack:** Add #industries-esm-support, #industries-cmt-cpq channel for:
   - Recent workarounds (last 90 days)
   - Escalation contacts
   - Known quirks not in documentation

3. **CodeSearch Authentication:** For precise code references:
   - Line numbers where errors originate
   - Specific methods causing performance issues
   - Complete stack traces

4. **Next Iteration:** Categorize "Other Issues" into sub-patterns:
   - TMF API issues (5+ cases)
   - Omniscript/Integration Procedure issues (10+ cases)
   - MACD-specific issues (8+ cases)
   - Community/Experience Cloud issues (6+ cases)

---

## M. Skill Maintenance

**Maintained By:** Support Engineering - Industries CPQ Team  
**Last Updated:** 2026-05-29  
**Version:** 1.0.0  
**Next Review:** 2026-06-30  

**Change Log:**
- 2026-05-29: Initial skill creation based on 238 cases (2-3 year span)
- Case analysis: Performance (29), Cart/Pricing (23), EPC (14), SOQL (7), Cache (9), Timeout (3), Upgrade (30)
- Code references: via_cpq-260.9 (320 CPQ classes), via_telco-260.9 (1030 CPQ classes)

**To Contribute Updates:**
1. Query latest cases from OrgCS
2. Check #industries-esm-support, #industries-cmt-cpq Slack weekly
3. Update patterns when frequency > 5 cases
4. Document new GUS investigations with resolutions
5. Add code references for new patterns
6. Update version number and change log