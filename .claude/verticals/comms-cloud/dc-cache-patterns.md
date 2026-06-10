## B. Known Issue Patterns

Based on analysis of 48 support cases and 34 GUS investigations:

---

### Pattern 1: Cache Batch Job Not Running / Skipped

**Frequency:** 26 cases (54% of all cases) — **HIGHEST FREQUENCY**  
**Severity:** 77% Urgent  
**Avg Resolution Time:** 9.0 days  
**Root Cause:** 42% User Configuration

#### Common Symptoms
- "Cache Catalog Product Definition Job not running"
- "Populate API Cache is getting skipped"
- "Batch job shows as 'Skipped' in logs"
- "Cache not populating after running job"
- "Job completes but no data in cache"

#### Affected Batch Jobs
1. **Cache Catalog Product Definition Job**
   - Class: `ProductAttributesCacheBatchJob`
   - Scheduler: `ProductAttributesCacheBatchJobScheduler`
   
2. **Populate API Cache Job**
   - Populates cacheable API responses
   
3. **Product Attributes Cache Batch Job**
   - Caches product attribute data

#### Root Causes (from case resolutions)
1. **Job Trigger Disabled** — Scheduled trigger was deactivated
2. **Missing Prerequisites** — Cache configuration not set up
3. **Governor Limit Reached** — Batch size too large
4. **Insufficient Permissions** — User running job lacks access
5. **Concurrent Job Conflict** — Another job holding locks

#### Resolution Steps

**Step 1: Verify Job Trigger**
```apex
// Query scheduled jobs
List<CronTrigger> triggers = [
    SELECT Id, CronJobDetail.Name, State, NextFireTime
    FROM CronTrigger
    WHERE CronJobDetail.Name LIKE '%ProductAttributesCache%'
       OR CronJobDetail.Name LIKE '%PopulateAPICache%'
];

// Check if any are in Deleted or Error state
for (CronTrigger t : triggers) {
    System.debug('Job: ' + t.CronJobDetail.Name + ', State: ' + t.State);
}
```

**Step 2: Check Cache Configuration**
```apex
// Verify CacheableAPIConstants setup
System.debug('Cacheable API enabled: ' + CacheableAPIConstants.ENABLE_CACHEABLE_APIS);
System.debug('Cache TTL: ' + CacheableAPIConstants.CACHE_TTL);
```

**Step 3: Review Batch Apex Limits**
```apex
// Check batch job limits
System.debug('Batch size: ' + Database.getBatchSize());
System.debug('Current limits: ' + Limits.getQueries() + '/' + Limits.getLimitQueries());
```

**Step 4: Manual Job Execution Test**
```apex
// Manually execute to see full error
ProductAttributesCacheBatchJob batch = new ProductAttributesCacheBatchJob();
Database.executeBatch(batch, 200); // Default batch size

// Check execution
List<AsyncApexJob> jobs = [
    SELECT Id, Status, NumberOfErrors, JobItemsProcessed,
           TotalJobItems, CreatedDate, ExtendedStatus
    FROM AsyncApexJob
    WHERE ApexClass.Name = 'ProductAttributesCacheBatchJob'
    ORDER BY CreatedDate DESC
    LIMIT 1
];
```

#### Verification
- [ ] Job trigger exists and is active
- [ ] Cache configuration is set up (CacheableAPIConstants)
- [ ] Batch job completes without errors
- [ ] Cache tables populated (query cache objects)
- [ ] Test API call returns cached data

#### Known Case Examples
- **Case 44006988** (British Telecommunications): Cache Catalog Product Definition Job not running - trigger was disabled
- Root cause: Scheduled trigger deactivated during deployment

---

### Pattern 2: API Errors and Timeouts

**Frequency:** 21 cases (43% of all cases)  
**Severity:** 52% Urgent, 24% Medium  
**Avg Resolution Time:** 9-14 days  
**Root Cause:** 67% Error message issues

#### Common Symptoms
- "Unable to process basket operation"
- "Attempt to de-reference a null object"
- "Error in Getting Offers from Catalog"
- API timeout after 30 seconds
- Null pointer exceptions in API response

#### Affected APIs
1. **Regenerate Cache APIs**
   - Regenerate product cache
   - Regenerate price list cache
   - Regenerate promotion cache
   
2. **Cacheable APIs**
   - Get offers from catalog
   - Get basket data
   - Get product attributes

#### Root Causes
1. **Null Object Reference** — API expects data not present
2. **Timeout** — Large data volume exceeding time limits
3. **Invalid Request** — Missing required fields in API call
4. **Cache Miss** — Requested data not in cache, sync call fails
5. **Version Mismatch** — API version incompatibility

#### Resolution Steps

**Step 1: Identify the Specific API**
```
Which API endpoint is failing?
- /services/apexrest/vlocity_cmt/v1/catalog/regenerate
- /services/apexrest/vlocity_cmt/v1/cache/product
- Other endpoint?
```

**Step 2: Check API Request Format**
```json
// Correct request structure
{
  "objectType": "Product2",
  "objectId": "01t...",
  "regenerateType": "full" | "incremental",
  "options": {
    "includeAttributes": true,
    "includeRelationships": true
  }
}
```

**Step 3: Test with Minimal Payload**
```
Start with smallest possible request:
1. Single product ID
2. No optional parameters
3. Incremental regenerate (faster)

If this works → Issue is with request complexity
If this fails → Issue is with API/config itself
```

**Step 4: Check for Null Safety**
```apex
// Common null pointer locations
if (request.objectId != null && request.objectType != null) {
    // Proceed
} else {
    throw new CustomException('Required fields missing: objectId, objectType');
}

// Check cache lookup
CacheResult result = CacheManager.get(cacheKey);
if (result != null && result.value != null) {
    return result.value;
} else {
    // Cache miss - handle gracefully
    return fetchFromDatabase();
}
```

#### Verification
- [ ] API request has all required fields
- [ ] API response doesn't contain null in critical fields
- [ ] Timeout limit is reasonable for data volume
- [ ] Cache is populated before API call
- [ ] Error handling catches null references

#### Known Case Examples
- **Case 43848925** (Edison SPA): "Unable to process basket operation" - null object in basket API response
- **Case 43851231** (Brightspeed): "Attempt to de-reference a null object" in GetOffers API

---

### Pattern 3: Cache Performance Issues

**Frequency:** 20 cases (41% mention "cache")  
**Severity:** Mixed (Urgent to Medium)  
**Avg Resolution Time:** 5-14 days

#### Common Symptoms
- Slow API response times
- Cache not refreshing
- Stale data in cache
- Cache expiration issues
- Large cache size causing performance problems

#### Root Causes
1. **Cache TTL Too Long** — Data becomes stale
2. **Cache TTL Too Short** — Too many cache misses, constant regeneration
3. **Cache Size Too Large** — Memory issues
4. **No Cache Invalidation** — Updates not reflected
5. **Cache Warming Not Scheduled** — Cold cache on first access

#### Resolution Steps

**Step 1: Check Cache TTL Configuration**
```apex
// From CacheableAPIConstants
System.debug('Cache TTL: ' + CacheableAPIConstants.CACHE_TTL + ' seconds');
System.debug('Cache enabled: ' + CacheableAPIConstants.ENABLE_CACHEABLE_APIS);

// Typical values:
// - High-frequency data: 300-900 seconds (5-15 min)
// - Medium-frequency: 1800-3600 seconds (30-60 min)
// - Low-frequency: 7200-14400 seconds (2-4 hours)
```

**Step 2: Monitor Cache Hit/Miss Ratio**
```apex
// Implement cache statistics
public class CacheMonitor {
    private static Integer hits = 0;
    private static Integer misses = 0;
    
    public static void recordHit() { hits++; }
    public static void recordMiss() { misses++; }
    
    public static Decimal getHitRatio() {
        Integer total = hits + misses;
        return total > 0 ? (Decimal)hits / total * 100 : 0;
    }
}

// Target: >80% hit ratio for good performance
```

**Step 3: Implement Cache Invalidation**
```apex
// On product update, invalidate cache
trigger ProductTrigger on Product2 (after update) {
    for (Product2 prod : Trigger.new) {
        String cacheKey = 'Product_' + prod.Id;
        CacheManager.remove(cacheKey);
        
        // Optionally trigger async regeneration
        System.enqueueJob(new RegenerateCacheQueueable(prod.Id));
    }
}
```

**Step 4: Schedule Cache Warming**
```apex
// Schedule cache warming job before peak hours
System.schedule(
    'Cache Warming - 7AM Daily',
    '0 0 7 * * ?', // 7 AM daily
    new CacheWarmingSchedulable()
);
```

#### Verification
- [ ] Cache TTL is appropriate for data change frequency
- [ ] Hit ratio is >80%
- [ ] Cache invalidation triggers on data updates
- [ ] Cache warming scheduled before peak hours
- [ ] Monitor cache size (don't exceed platform limits)

---

### Pattern 4: Versioning Issues

**Frequency:** 27 GUS investigations (79% of GUS work)  
**Severity:** N/A (Test automation work)  
**Focus:** Post-regeneration test scenarios for versioning

#### GUS Investigation Themes
From the 34 GUS investigations, 27 (79%) focus on versioning scenarios:

**Versioning Test Automation Work:**
- PriceList versioning regeneration tests
- PriceListEntry versioning tests  
- Promotion versioning tests
- OfferSpecification attribute & attachment tests
- Non-versioning scenario tests

#### Common Versioning Scenarios

1. **Version 1 (Active) + Version 2 (Draft)**
   - Regenerate Version 1 → Should not affect Version 2
   - Regenerate Version 2 → Should not affect Version 1
   
2. **Version Activation**
   - Activate Version 2 → Version 1 becomes inactive
   - Regenerate after activation → Both versions updated correctly?
   
3. **Cross-Version Dependencies**
   - Product A references Product B
   - Both have versions
   - Regenerate Product A → Does it use correct version of Product B?

#### Test Scenarios (from GUS)

**Scenario 1: PriceList Versioning**
```
Given: PriceList v1 (Active) and PriceList v2 (Draft)
When: Regenerate cache for PriceList v1
Then: 
  - v1 cache updated
  - v2 cache unchanged
  - API returns v1 when requesting active version
  - API returns v2 when explicitly requesting draft version
```

**Scenario 2: Promotion Versioning**
```
Given: Promotion v1 (Active) with 10 PromotionItems
When: Create Promotion v2, add 2 new PromotionItems
And: Regenerate cache
Then:
  - v1 cache has 10 items
  - v2 cache has 12 items
  - No cross-contamination between versions
```

**Scenario 3: Offer Specification Attributes**
```
Given: OfferSpecification with attributes and attachments
When: Update attribute in v2
And: Regenerate cache
Then:
  - v1 attributes unchanged
  - v2 attributes reflect update
  - Attachments preserved across versions
```

#### Verification Checklist
- [ ] Each version has separate cache entries
- [ ] Cache keys include version identifier
- [ ] Regenerate respects version boundaries
- [ ] API correctly routes to requested version
- [ ] Cross-version references resolve correctly

---

### Pattern 5: Configuration and User Education Issues

**Frequency:** 20 cases (40% root cause: "User Needs Education")  
**Severity:** Mixed (Urgent to Medium)  
**Avg Resolution Time:** 5-9 days  
**Resolution:** Education + Documentation

#### Common Symptoms
- "How do I set up cache management?"
- "Is it possible to send UpdateBasket with only basic information?"
- "Cache jobs not running" (due to misconfiguration)
- "API returning unexpected results" (due to wrong parameters)

#### Common Misconfigurations

**1. Cache Not Enabled**
```apex
// Check if caching is enabled
CacheableAPIConstants.ENABLE_CACHEABLE_APIS = false; // ❌ Caching disabled

// Should be:
CacheableAPIConstants.ENABLE_CACHEABLE_APIS = true; // ✅
```

**2. Batch Job Not Scheduled**
```apex
// No scheduled job exists
List<CronTrigger> jobs = [SELECT Id FROM CronTrigger WHERE CronJobDetail.Name = 'ProductAttributesCache'];
// jobs.size() == 0 → Job not scheduled!

// Fix: Schedule the job
System.schedule('ProductAttributesCache', '0 0 2 * * ?', new ProductAttributesCacheBatchJobScheduler());
```

**3. Incorrect API Endpoint**
```
Wrong: /services/apexrest/v1/catalog/regenerate
Right: /services/apexrest/vlocity_cmt/v1/catalog/regenerate
                        ^^^^^^^^^^ namespace required!
```

**4. Missing Required Fields in API Call**
```json
// ❌ Missing objectType
{
  "objectId": "01t..."
}

// ✅ Correct
{
  "objectType": "Product2",
  "objectId": "01t..."
}
```

#### Configuration Checklist

**Initial Setup:**
- [ ] Enable cacheable APIs in `CacheableAPIConstants`
- [ ] Set appropriate cache TTL
- [ ] Schedule cache batch jobs
- [ ] Grant permissions to batch job user
- [ ] Run initial cache population

**API Configuration:**
- [ ] Use correct namespace in endpoints (`vlocity_cmt`)
- [ ] Include all required fields in requests
- [ ] Set appropriate timeout limits
- [ ] Handle cache miss scenarios gracefully

**Monitoring:**
- [ ] Set up batch job monitoring
- [ ] Track cache hit/miss ratios
- [ ] Alert on job failures
- [ ] Review logs regularly

#### Resolution Approach
1. **Education** — Share documentation, explain concepts
2. **Configuration Review** — Walk through setup checklist
3. **Testing** — Verify configuration with test scenarios
4. **Documentation Update** — Add missing setup steps to docs

#### Known Case Examples
- **Case 43848925** (Edison SPA): Education needed on basket API usage
- **Case 44006988** (British Telecommunications): Job configuration missing

---

## C. Regenerate Cache API Reference

### Core Regenerate APIs

#### 1. Regenerate Product Cache
```apex
@RestResource(UrlMapping='/vlocity_cmt/v1/catalog/regenerate/product')
global class RegenerateProductCacheAPI {
    @HttpPost
    global static Response regenerateProductCache(Request request) {
        // Regenerates cache for specified product(s)
    }
}
```

**Request:**
```json
{
  "productIds": ["01t...", "01t..."],
  "regenerateType": "full",  // or "incremental"
  "includeChildren": true
}
```

**Response:**
```json
{
  "success": true,
  "recordsProcessed": 25,
  "cacheKeysUpdated": ["Product_01t...", "Product_01t..."],
  "executionTimeMs": 1250
}
```

#### 2. Regenerate Price List Cache
```apex
@RestResource(UrlMapping='/vlocity_cmt/v1/catalog/regenerate/pricelist')
```

**Request:**
```json
{
  "priceListId": "a0X...",
  "versionNumber": 2,  // Optional, regenerates active version if omitted
  "includeEntries": true
}
```

#### 3. Regenerate Promotion Cache
```apex
@RestResource(UrlMapping='/vlocity_cmt/v1/catalog/regenerate/promotion')
```

**Request:**
```json
{
  "promotionId": "a0Y...",
  "regenerateItems": true,
  "regenerateRules": true
}
```

### API Error Codes

| Error Code | Message | Resolution |
|------------|---------|------------|
| `INVALID_REQUEST` | Required field missing | Add objectId and objectType |
| `OBJECT_NOT_FOUND` | Object with ID not found | Verify ID is correct |
| `CACHE_DISABLED` | Caching not enabled | Set ENABLE_CACHEABLE_APIS = true |
| `TIMEOUT_EXCEEDED` | Operation exceeded timeout | Reduce batch size or use incremental |
| `NULL_POINTER` | Null object reference | Check all required data populated |

---

## D. Batch Job Reference

### Cache Management Batch Jobs

#### 1. ProductAttributesCacheBatchJob

**Purpose:** Caches product attribute data for fast retrieval

**Class:** `ProductAttributesCacheBatchJob`  
**Scheduler:** `ProductAttributesCacheBatchJobScheduler`  
**Frequency:** Nightly (default 2 AM)

**Configuration:**
```apex
// Schedule the job
System.schedule(
    'Product Attributes Cache - Nightly',
    '0 0 2 * * ?',  // 2 AM daily
    new ProductAttributesCacheBatchJobScheduler()
);

// Batch size (adjust based on data volume)
Integer batchSize = 200; // Default
Database.executeBatch(new ProductAttributesCacheBatchJob(), batchSize);
```

**Monitoring:**
```apex
// Check last execution
List<AsyncApexJob> jobs = [
    SELECT Id, Status, NumberOfErrors, JobItemsProcessed,
           TotalJobItems, CreatedDate, CompletedDate
    FROM AsyncApexJob
    WHERE ApexClass.Name = 'ProductAttributesCacheBatchJob'
    ORDER BY CreatedDate DESC
    LIMIT 1
];

System.debug('Status: ' + jobs[0].Status);
System.debug('Processed: ' + jobs[0].JobItemsProcessed + '/' + jobs[0].TotalJobItems);
System.debug('Errors: ' + jobs[0].NumberOfErrors);
```

#### 2. Cache Catalog Product Definition Job

**Purpose:** Populates catalog cache with product definitions

**Common Issues:**
- Job shows "Skipped" → Check trigger is active
- Job fails → Check governor limits
- Job completes but no data → Check cache configuration

**Troubleshooting:**
```apex
// 1. Verify job is scheduled
List<CronTrigger> triggers = [
    SELECT Id, CronJobDetail.Name, State, NextFireTime, PreviousFireTime
    FROM CronTrigger
    WHERE CronJobDetail.Name LIKE '%CacheCatalog%'
];

// 2. Check last run
System.debug('Last run: ' + triggers[0].PreviousFireTime);
System.debug('Next run: ' + triggers[0].NextFireTime);
System.debug('State: ' + triggers[0].State);

// 3. If State = 'Deleted', reschedule
if (triggers[0].State == 'DELETED') {
    // Reschedule the job
}
```

#### 3. Populate API Cache Job

**Purpose:** Pre-populates cache with frequently accessed API responses

**Batch Size Recommendations:**
- Small org (<10K products): Batch size 200
- Medium org (10K-50K products): Batch size 100
- Large org (>50K products): Batch size 50, run off-peak hours

**Performance Tuning:**
```apex
// Monitor execution time
Datetime startTime = Datetime.now();
Database.executeBatch(new PopulateAPICacheBatch(), 100);
// Record end time in finish() method

// If taking >1 hour, reduce batch size or split into multiple jobs
```

---

## E. Configuration & Setup Guide

### Initial Setup Checklist

**Step 1: Enable Cacheable APIs**
```apex
// In CacheableAPIConstants or custom metadata
CacheableAPIConstants.ENABLE_CACHEABLE_APIS = true;
CacheableAPIConstants.CACHE_TTL = 3600; // 1 hour in seconds
```

**Step 2: Schedule Batch Jobs**
```apex
// Schedule nightly cache refresh
System.schedule(
    'Cache Refresh - 2AM Daily',
    '0 0 2 * * ?',
    new ProductAttributesCacheBatchJobScheduler()
);

// Schedule cache warming before business hours
System.schedule(
    'Cache Warming - 6AM Daily',
    '0 0 6 * * ?',
    new CacheWarmingSchedulable()
);
```

**Step 3: Grant Permissions**
```
Required permission sets for batch job user:
- Read on all cacheable objects (Product2, PriceList, Promotion, etc.)
- Execute Apex
- View Setup and Configuration (to schedule jobs)
```

**Step 4: Initial Cache Population**
```apex
// Run manually first time
Database.executeBatch(new ProductAttributesCacheBatchJob(), 200);

// Check completion
// Query AsyncApexJob to verify success
```

**Step 5: Configure API Endpoints**
```
Ensure Named Credentials or Connected Apps are set up for:
- Authentication
- API version compatibility
- Timeout limits (increase if needed)
```

### Common Misconfigurations

| Issue | Symptom | Fix |
|-------|---------|-----|
| Cache disabled | API responses slow | Set ENABLE_CACHEABLE_APIS = true |
| TTL too short | Constant regeneration | Increase CACHE_TTL |
| TTL too long | Stale data | Decrease CACHE_TTL, add invalidation |
| Job not scheduled | Cache never populates | Schedule batch jobs |
| Wrong namespace | API not found | Use /vlocity_cmt/ in URLs |
| Missing permissions | Job fails silently | Grant required permissions |

---

## F. Troubleshooting Flowcharts

### Flowchart 1: Cache Job Not Running

```
START: Cache job not running
│
├─ Is job scheduled?
│  ├─ NO → Schedule the job (Section D)
│  └─ YES → Continue
│
├─ Is trigger active?
│  ├─ NO → Trigger was deleted/deactivated → Reschedule
│  └─ YES → Continue
│
├─ Check last AsyncApexJob status
│  ├─ Status = Failed → Check error logs (Section F.2)
│  ├─ Status = Completed, but 0 records → Check query filter
│  └─ No recent job → Trigger not firing → Check cron expression
│
└─ RESOLVED or ESCALATE
```

### Flowchart 2: API Returns Error

```
START: API error
│
├─ What error code?
│  ├─ NULL_POINTER → Check required fields (Section B, Pattern 2)
│  ├─ INVALID_REQUEST → Verify request format (Section C)
│  ├─ TIMEOUT → Reduce payload size (Section B, Pattern 3)
│  ├─ OBJECT_NOT_FOUND → Verify ID exists
│  └─ Other → Check API logs
│
├─ Is cache enabled?
│  ├─ NO → Enable cache (Section E)
│  └─ YES → Continue
│
├─ Is cache populated?
│  ├─ NO → Run batch job (Section D)
│  └─ YES → Continue
│
├─ Test with minimal payload
│  ├─ Works → Issue is request complexity
│  └─ Fails → Issue is configuration/cache
│
└─ RESOLVED or ESCALATE
```

### Flowchart 3: Performance Issues

```
START: Slow performance
│
├─ Check cache hit ratio
│  ├─ <50% → Increase TTL, schedule cache warming
│  ├─ 50-80% → Investigate cache misses (what's not cached?)
│  └─ >80% → Cache is good, issue elsewhere
│
├─ Check API response time
│  ├─ >5 seconds → Investigate query optimization
│  ├─ 1-5 seconds → Acceptable for complex queries
│  └─ <1 second → Cache is working well
│
├─ Check batch job execution time
│  ├─ >2 hours → Reduce batch size
│  ├─ 30min-2hours → Acceptable
│  └─ <30min → Optimal
│
└─ RESOLVED or ESCALATE
```

---

## G. Codebase Reference (via_cpq-260.9)

**Codebase Path:** `/Users/sandesh.kulkarni/MPCode/via_cpq-260.9`  
**Total Classes:** 214,317 lines of code  
**Cache/Regenerate Classes:** 56 classes

### Key Classes

#### Cache Management Core

| Class | Purpose | Key Methods |
|-------|---------|-------------|
| **EcomCacheManagementService** | Core cache management service | `regenerateCache()`, `invalidateCache()`, `getCacheStats()` |
| **CacheManager** | Cache storage abstraction | `get()`, `put()`, `remove()`, `clear()` |
| **CacheableAPIConstants** | Configuration constants | `ENABLE_CACHEABLE_APIS`, `CACHE_TTL`, `MAX_CACHE_SIZE` |
| **CacheQueryStore** | Query result caching | `storeQuery()`, `retrieveQuery()` |
| **CacheStoreWrapper** | Cache storage wrapper | `wrapCache()`, `unwrapCache()` |

#### Batch Jobs

| Class | Purpose | Schedule |
|-------|---------|----------|
| **ProductAttributesCacheBatchJob** | Cache product attributes | Nightly (2 AM) |
| **ProductAttributesCacheBatchJobScheduler** | Scheduler for above | Nightly |
| **ProductAttributesCacheResource** | REST resource for cache | On-demand API |

#### API Resources

| Class | Purpose | Endpoint |
|-------|---------|----------|
| **ProductAttributesCacheResource** | Product cache API | `/vlocity_cmt/v1/cache/product` |
| **CacheableAttributeService** | Attribute caching service | Internal service |
| **CacheableAPIsHandlerError** | Error handling for cache APIs | Error handler |
| **CacheableAPIsHandlerErrors** | Multiple error handling | Error handler |

#### Supporting Classes

| Class | Purpose |
|-------|---------|
| **EcomCacheResultsBase** | Base class for cache results |
| **EcomCacheResultItemBase** | Individual cache result item |
| **CachedRuleSetWrapper** | Wraps cached rule sets |
| **CachedRSAuxData** | Auxiliary data for cached rules |
| **CPQScaleCacheService** | Cache service for CPQ scale |
| **CpqUICacheUtil** | UI caching utilities |
| **APICacheResponseMigrationV3** | Cache migration utility |

### Code Patterns

#### Pattern A: Cache Get/Put
```apex
// From CacheManager
public static Object get(String cacheKey) {
    if (!CacheableAPIConstants.ENABLE_CACHEABLE_APIS) {
        return null; // Caching disabled
    }
    
    Cache.OrgPartition orgPart = Cache.Org.getPartition('local.CachePartition');
    return orgPart.get(cacheKey);
}

public static void put(String cacheKey, Object value) {
    if (!CacheableAPIConstants.ENABLE_CACHEABLE_APIS) {
        return; // Caching disabled
    }
    
    Cache.OrgPartition orgPart = Cache.Org.getPartition('local.CachePartition');
    Integer ttl = CacheableAPIConstants.CACHE_TTL;
    orgPart.put(cacheKey, value, ttl);
}
```

#### Pattern B: Batch Job Structure
```apex
// From ProductAttributesCacheBatchJob
global class ProductAttributesCacheBatchJob implements Database.Batchable<sObject> {
    
    global Database.QueryLocator start(Database.BatchableContext BC) {
        return Database.getQueryLocator([
            SELECT Id, Name, /* fields */
            FROM Product2
            WHERE IsActive = true
        ]);
    }
    
    global void execute(Database.BatchableContext BC, List<Product2> scope) {
        for (Product2 prod : scope) {
            // Cache product attributes
            String cacheKey = 'Product_' + prod.Id;
            Map<String, Object> attributes = getProductAttributes(prod.Id);
            CacheManager.put(cacheKey, attributes);
        }
    }
    
    global void finish(Database.BatchableContext BC) {
        // Log completion
        System.debug('Cache batch job completed');
    }
}
```

#### Pattern C: API Error Handling
```apex
// From CacheableAPIsHandlerError
public class CacheableAPIsHandlerError {
    public static void handleError(Exception e, String context) {
        // Log error
        System.debug(LoggingLevel.ERROR, 'Cache API Error in ' + context + ': ' + e.getMessage());
        
        // Create custom error response
        Map<String, Object> errorResponse = new Map<String, Object>{
            'success' => false,
            'errorCode' => getErrorCode(e),
            'errorMessage' => e.getMessage(),
            'context' => context
        };
        
        // Return error response
        RestContext.response.statusCode = 400;
        RestContext.response.responseBody = Blob.valueOf(JSON.serialize(errorResponse));
    }
    
    private static String getErrorCode(Exception e) {
        if (e instanceof NullPointerException) return 'NULL_POINTER';
        if (e instanceof QueryException) return 'QUERY_ERROR';
        if (e instanceof DmlException) return 'DML_ERROR';
        return 'UNKNOWN_ERROR';
    }
}
```

---

## H. Escalation & Expert Consultation

**When to escalate to CME Core Canyonlands team:**
- Cache batch jobs consistently failing (>3 consecutive failures)
- API errors not resolved by standard troubleshooting
- Performance degradation affecting production
- Versioning issues with cross-version contamination
- Suspected platform bug (not configuration issue)
- Customer escalation to P0/P1 severity

**What to prepare before escalation:**
- Completed troubleshooting steps (Sections B-F)
- Error logs (API logs, batch job logs, debug logs)
- Configuration snapshot (cache settings, batch job schedules)
- Reproduction steps (if reproducible)
- Business impact statement
- GUS/case number if already created

**Escalation Statistics:**
- 14% of cases escalate to GUS investigations
- 6% result in bug creation
- Most escalations: Configuration issues (preventable with proper setup)
- Avg GUS resolution time: N/A (mostly test automation work)

**Slack Channels:**
- **#industries-cpq-dc-standard-apis-help** — First line support
- **#industries-cmt-cpq** — Broader CPQ team discussions

**GUS Product Tag:** Industries CPQ - DC APIs

---

## Z. Test Scenarios (From GUS Test Automation Work)

Based on 79% of GUS investigations focusing on versioning test automation:

### Test Scenario 1: PriceList Versioning - Part 1 (W-12388661)
```gherkin
Feature: PriceList Versioning Post-Regeneration

Scenario: Regenerate PriceList Version 1 without affecting Version 2
  Given PriceList "PLTest" with Version 1 (Active) and Version 2 (Draft)
  And Version 1 has 10 PriceListEntries
  And Version 2 has 12 PriceListEntries (2 new entries)
  When I call Regenerate Cache API for PriceList "PLTest" Version 1
  Then Cache for Version 1 is updated
  And Cache contains 10 PriceListEntries for Version 1
  And Cache for Version 2 is unchanged
  And Cache contains 12 PriceListEntries for Version 2
  And API call for active PriceList returns Version 1 data
```

### Test Scenario 2: PriceListEntry Non-Versioning - Part 2 (W-12404978)
```gherkin
Scenario: Regenerate PriceListEntry in non-versioned scenario
  Given PriceList "PLNonVersioned" with no versioning
  And 15 PriceListEntries
  When I update PriceListEntry "PLE001" price from $10 to $15
  And I call Regenerate Cache API
  Then Cache is updated within 5 seconds
  And API returns new price $15 for "PLE001"
  And Other 14 entries are unchanged
```

### Test Scenario 3: Promotion & PromotionItem Non-Versioning - Part 1 (W-12404952)
```gherkin
Scenario: Regenerate Promotion without versioning
  Given Promotion "Spring Sale" with 8 PromotionItems
  When I add 2 new PromotionItems to "Spring Sale"
  And I call Regenerate Cache API for Promotion
  Then Cache contains 10 PromotionItems
  And All 10 items are returned by API
  And Promotion rules are regenerated
```

### Test Scenario 4: Offer Specification Attributes & Attachments (W-12404984)
```gherkin
Scenario: Regenerate OfferSpecification with attributes and attachments
  Given OfferSpecification "BroadbandOffer" with:
    | Attribute Type | Count |
    | Standard       | 5     |
    | Custom         | 3     |
  And "BroadbandOffer" has 2 attachments (PDF, Image)
  When I update 1 custom attribute value
  And I call Regenerate Cache API for OfferSpecification
  Then Cache is regenerated
  And Updated attribute is reflected in cache
  And 4 other attributes unchanged
  And 2 attachments preserved in cache
  And API returns correct attribute values
```

### Test Scenario 5: Cross-Version Reference
```gherkin
Scenario: Product references PriceList with versioning
  Given Product "Product_A" Version 1
  And Product "Product_A" references PriceList "PL_B" Version 2
  When I regenerate cache for Product "Product_A" Version 1
  Then Cache correctly references PriceList "PL_B" Version 2
  And Not Version 1 or Version 3 of PriceList "PL_B"
```

---

## Updates & Maintenance

### Update Cadence
- **Monthly:** Review case patterns, update with new issues
- **Quarterly:** Refresh statistics, update resolution times
- **After CPQ Release:** Update codebase references (via_cpq-260.9 → 262.x)
- **As Needed:** Add new patterns as they emerge

### Skill Effectiveness Metrics
- Track % of cases resolved with this skill (target: >60%)
- Monitor avg resolution time (target: <7 days for Urgent)
- Track escalation rate (target: <10%)
- Measure user education success (reduce repeat cases)

### Feedback Channels
- **Slack:** #industries-cpq-dc-standard-apis-help
- **Team:** CME Core Canyonlands
- **Product Tags:** 
  - GUS: Industries CPQ - DC APIs
  - OrgCS: Industry-CPQ / Order Management / Digital Commerce

---

## Known Limitations

This skill works best when:
- Issue is clearly related to cache or regenerate APIs
- Error messages are available
- User can provide reproduction steps

This skill may struggle with:
- Non-cache DC issues (use other DC skills)
- Deep platform bugs requiring engineering investigation
- Custom implementations that deviate from standard patterns

In such cases:
- Use troubleshooting flowcharts to isolate the issue
- Escalate to CME Core Canyonlands team with full context
- Create GUS investigation with detailed reproduction steps

---

**Skill Version:** 1.0.0  
**Last Updated:** 2026-05-28  
**Trained On:** 48 cases + 34 GUS investigations + via_cpq-260.9 codebase  
**Maintainer:** CME Core Canyonlands Team