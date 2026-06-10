## A. Triage & Classification

**SOURCE:** Analysis of 665 support cases and 242 GUS investigations for Order Management / Order Decomposition (CME - OM Starlord team).

| Symptom Category | Frequency | Pattern |
|---|---|---|
| Permission/Access Issues | 133 cases (20%) | Pattern 4 |
| Orchestration Errors | 63 cases (9.5%) | Pattern 2 |
| UNABLE_TO_LOCK_ROW | 39 cases (5.9%) | Pattern 3 |
| Status/State Transition | 35 cases (5.3%) | Pattern 6 |
| Decomposition Not Working | 30 cases (4.5%) | Pattern 1 |
| Product Structure/Hierarchy | 14 cases (2.1%) | Pattern 5 |

**Key stats:** 59.4% are Level 2 - Urgent (avg resolution: 12.1 days). 20% of all cases are permission-related.

---

## B. Known Issue Patterns

**SOURCE:** Analysis of 665 support cases and 242 GUS investigations for Order Management-Order Decomposition (CME - OM Starlord team)

This section contains CRITICAL preventive knowledge. When decomposing stories, always check if the feature could trigger these known issues and add preventive tasks accordingly.

---

### Pattern 1: Decomposition Not Working / Not Available

**Frequency:** 30 cases (4.5% of all cases)  
**Severity:** 50% Urgent/Critical (avg resolution: 12 days)  
**Root Cause Category:** 71% "Documentation or User Knowledge Issue"

#### Common Symptoms
- "Orders not getting decomposed"
- "Decomposition is not available as order is not decomposed yet" (error message)
- View Decomposition button shows error or blank screen
- Decomposition not happening for amend/cancel orders
- Clicking "Decompose Order" shows no result

#### Root Causes (from case resolutions)
1. **Missing Decomposition Specification** — Product lacks vlocity_cmt__DecompositionSpecification__c records
2. **Product Not Marked as Decomposable** — vlocity_cmt__IsDecomposable__c = false
3. **Missing Orchestration Plan Assignment** — No vlocity_cmt__OrchestrationPlan__c linked to product
4. **Incomplete Product Hierarchy** — Commercial product missing technical product children (vlocity_cmt__ProductChildItem__c)
5. **Order Status Issues** — Order in wrong status for decomposition (must be "Submitted" or "In Flight")
6. **XOM Administration Not Configured** — vlocity_cmt__XOMAdministration__c settings missing

#### Preventive Tasks to Add
When decomposing any story involving new products or decomposition changes:
- [ ] Validate vlocity_cmt__DecompositionSpecification__c exists for all new products
- [ ] Set vlocity_cmt__IsDecomposable__c = true on commercial products
- [ ] Link vlocity_cmt__OrchestrationPlan__c to products
- [ ] Create complete product hierarchy with vlocity_cmt__ProductChildItem__c records
- [ ] Test decomposition in all order statuses (New, Submitted, In Flight, Amend)
- [ ] Verify XOM Administration settings (CPQ/ORDER MANAGEMENT INTERFACE STATUS)
- [ ] Add negative test: "Order in invalid status should show clear error message"

#### Test Scenarios
```gherkin
Scenario: Decomposition not configured
  Given a commercial product without decomposition specification
  When I submit an order with that product
  And I click "View Decomposition"
  Then I should see error "Decomposition specification not found for Product XYZ"
  And the error message should guide me to configuration steps

Scenario: Order status validation
  Given an order in "Draft" status
  When I attempt decomposition
  Then I should see "Order must be submitted before decomposition"
```

#### Case Examples
- Case 43849470: Orders not getting decomposed (SmartestEnergy) — Missing orchestration plan
- Case 44xxx: Decomposition not working on amend orders — Status transition issue

---

### Pattern 2: Orchestration Errors

**Frequency:** 63 cases (9.5% of all cases)  
**Severity:** 68% Urgent (avg resolution: 12 days)  
**Root Cause Category:** 72% "Error message issue"  
**GUS Escalation:** 35% (86 investigations mention "orchestration")

#### Common Symptoms
- Orchestration items stuck in 'Ready' state never transitioning to 'Running'
- Orchestration items stuck in 'Pending' status indefinitely
- AutoTask steps failing with generic errors
- Orders not completing orchestration ("Successful Order Amendment" step stuck)
- Orchestration dependency never triggering dependent items

#### Root Causes (from 242 GUS investigations)
1. **Orchestration Plan Misconfiguration** — Invalid dependency chains, missing items
2. **Dependent Item Timing** — Parent item completes before child ready, race condition
3. **Batch Job Failures** — AutoTask batch fails silently, no retry mechanism
4. **State Transition Validation** — Custom validation rules blocking vlocity_cmt__OrchestrationItem__c updates
5. **Platform Event Delays** — Events published but not consumed, causing item state mismatch
6. **Missing Prerequisites** — AutoTask expects data/config not present
7. **Concurrency Issues** — Multiple orchestration items updating same parent records

#### Preventive Tasks to Add
When decomposing orchestration-related stories:
- [ ] Design orchestration plan with explicit dependency graph (document dependencies)
- [ ] Implement orchestration item state logging (debug logs at each transition)
- [ ] Add retry logic for AutoTask items (handle transient failures)
- [ ] Validate orchestration prerequisites before plan execution (pre-execution checks)
- [ ] Test orchestration with concurrent orders (bulk test 10+ orders submitted simultaneously)
- [ ] Implement orchestration item timeout monitoring (alert if item stuck > X minutes)
- [ ] Add orchestration failure rollback logic (compensating transactions)
- [ ] Test orchestration plan with missing/invalid data scenarios

#### Test Scenarios
```gherkin
Scenario: Orchestration item stuck in Ready
  Given an orchestration plan with AutoTask item "Provision Service"
  When the item transitions to "Ready"
  And 10 minutes pass without transition to "Running"
  Then system should auto-retry the item
  And log warning "Orchestration item stuck - auto-retry triggered"

Scenario: Dependent item waiting
  Given OrchestrationItem A depends on OrchestrationItem B
  When Item B completes
  Then Item A should transition from "Pending" to "Ready" within 30 seconds
  And if not, escalation alert should fire
```

#### Known GUS Investigations
- W-12291287: Additional permissions required for orchestration plans (Closed - Doc/Usability)
- W-12782775: Orders failing in orchestration even after completion at XOM (Closed - New Bug Logged)
- W-13053459: Queue member deletion deletes wrong queue member (Closed - New Bug Logged)

---

### Pattern 3: UNABLE_TO_LOCK_ROW Errors

**Frequency:** 39 cases (5.9% of all cases)  
**Severity:** 51% Urgent, 13% Critical  
**Root Cause Category:** 100% "Product Bug" or "Error message issue"  
**GUS Escalation:** HIGH (often requires bug fix)

#### Common Symptoms
- `UNABLE_TO_LOCK_ROW` in AutoTask orchestration item (most common)
- `Insert failed. First exception on row 0; first error: UNABLE_TO_LOCK_ROW`
- `Update failed` with lock error during decomposition
- Split and submit causing lock errors
- Batch job failures with lock contention

#### Root Causes (from case resolutions)
1. **Concurrent Batch Jobs** — Multiple AutoTask batches updating same Order/FulfillmentRequest
2. **Parallel Orchestration Items** — Multiple items updating parent vlocity_cmt__Order__c simultaneously
3. **Trigger Recursion** — vlocity_cmt.XOMTaskTrigger causing re-entrant updates
4. **Large Batch Sizes** — Batch size=200 causing prolonged row locks
5. **External Callout + DML** — Mixed DML in same transaction locking records

#### Preventive Tasks to Add
When decomposing stories involving batch processing or orchestration:
- [ ] **Reduce batch sizes** — Use batch size 50-100 instead of 200 for XOM objects
- [ ] **Implement retry with exponential backoff** — Catch UNABLE_TO_LOCK_ROW, wait, retry (3 attempts)
- [ ] **Sequence dependent orchestration items** — Avoid parallel updates to same parent record
- [ ] **Review trigger bulkification** — Ensure vlocity_cmt triggers handle bulk properly
- [ ] **Use Platform Events for async** — Decouple callouts from DML transactions
- [ ] **Add lock management logging** — Log when retries triggered, which records locked
- [ ] **Test with concurrent orders** — Submit 50+ orders simultaneously, monitor for lock errors
- [ ] **Implement deadlock detection** — Monitor for circular lock dependencies

#### Resolution Code Pattern
```apex
// Retry logic for UNABLE_TO_LOCK_ROW
public static void updateWithRetry(List<SObject> records) {
    Integer maxRetries = 3;
    Integer retryCount = 0;
    Boolean success = false;
    
    while (!success && retryCount < maxRetries) {
        try {
            update records;
            success = true;
        } catch (DmlException e) {
            if (e.getMessage().contains('UNABLE_TO_LOCK_ROW') && retryCount < maxRetries - 1) {
                retryCount++;
                // Exponential backoff: 2^retryCount seconds
                Integer waitMs = (Integer)Math.pow(2, retryCount) * 1000;
                System.debug('Lock error, retry ' + retryCount + ' after ' + waitMs + 'ms');
                // Note: Sleep not available in Apex, implement via Queueable chain
            } else {
                throw e;
            }
        }
    }
}
```

#### Test Scenarios
```gherkin
Scenario: Concurrent orchestration item updates
  Given 2 orchestration items updating the same Order record
  When both items execute simultaneously
  Then one should succeed immediately
  And the other should retry after exponential backoff
  And both should eventually succeed without user intervention
```

#### Known GUS Investigations
- W-13069217: Order Item Fulfillment Statuses different for Normal vs Supplemental (Closed - New Bug Logged)

---

### Pattern 4: Permission / Access Issues

**Frequency:** 133 cases (20% of ALL cases — HIGHEST frequency)  
**Severity:** 60% Urgent, 7.5% Critical  
**Root Cause Category:** 100% "User Needs Education" or "Feature activation"  
**Resolution Time:** Avg 10 days (faster than bugs, education issue)

#### Common Symptoms
- `Insufficient permissions: secure query included inaccessible field`
- `vlocity_cmt.XOMTaskTrigger: execution of AfterUpdate caused by: System.QueryException`
- Orchestration plan not visible for user/profile
- "Additional permissions required for orchestration plans" (after package upgrade)
- Decomposition page not loading, error "retrieving data"
- Fields not accessible in XOM AutoTask code

#### Root Causes (from case resolutions)
1. **Missing Permission Sets** — User lacks "XOM User" or "XOM Admin" permission set
2. **Field-Level Security** — FLS missing on vlocity_cmt__* fields (especially after upgrade)
3. **Custom Profile Configuration** — Profile missing object CRUD on vlocity_cmt objects
4. **Package Upgrade FLS Changes** — Vlocity package upgrade resets FLS
5. **Namespace Issues** — Code references vlocity_cmt__Field__c but FLS set on Field__c (no namespace)
6. **Shared Object Access** — User can't see orchestration plans due to sharing rules

#### Preventive Tasks to Add
For ANY story involving XOM/Order Decomposition:
- [ ] **Document required permission sets** — List all permission sets (XOM User, XOM Admin, etc.)
- [ ] **Create permission set assignment script** — Automate assignment for test users
- [ ] **FLS verification script** — Query FieldPermissions for all vlocity_cmt fields used
- [ ] **Profile audit** — List minimum profile permissions required
- [ ] **Post-deployment FLS check** — Automated test to verify FLS after deploy
- [ ] **Permission documentation** — Add to README: "Permissions required for this feature"
- [ ] **Test with minimal-permission user** — Create test user with ONLY required permissions
- [ ] **Package upgrade regression test** — After vlocity upgrade, re-test all FLS

#### Resolution Checklist (for support cases)
```markdown
## Permission Troubleshooting Checklist

1. [ ] User has "XOM User" permission set assigned
2. [ ] User has "XOM Admin" permission set (if admin actions needed)
3. [ ] Profile has READ access to:
   - vlocity_cmt__Order__c
   - vlocity_cmt__OrchestrationPlan__c
   - vlocity_cmt__OrchestrationItem__c
   - vlocity_cmt__FulfillmentRequest__c
   - vlocity_cmt__FulfillmentRequestLine__c
4. [ ] Field-Level Security (FLS) on all vlocity_cmt__*__c fields
5. [ ] Check sharing settings on OrchestrationPlan (OWD Private?)
6. [ ] Recent package upgrade? Check if FLS reset
7. [ ] Custom code references namespace correctly (vlocity_cmt__)
```

#### Test Scenarios
```gherkin
Scenario: Minimal permission user
  Given a user with ONLY "XOM User" permission set
  And NO additional object/field permissions
  When user views an order with decomposition
  Then user should see decomposition hierarchy
  And no "Insufficient permissions" errors in logs

Scenario: Post-upgrade FLS regression
  Given vlocity package upgraded from 242.x to 244.x
  When automated test runs FLS check script
  Then all required vlocity_cmt__*__c fields should have FLS enabled
  And test should fail with clear message if FLS missing
```

#### Known GUS Investigations
- W-12291287: Additional permissions required for orchestration plans (Closed - Doc/Usability) — Added documentation
- W-12359827: Orchestration Plan not visible for particular user/profile (Closed - Known Bug Exists)

---

### Pattern 5: Product Structure / Hierarchy Issues

**Frequency:** 14 cases (2.1% of cases, but HIGH severity)  
**Severity:** 71% Urgent, 14% Critical  
**Root Cause Category:** "Product Bug" or misconfiguration  
**Business Impact:** HIGH (incorrect decomposition = wrong fulfillment)

#### Common Symptoms
- Technical product decomposed independently rather than in hierarchy
- Parent FRL getting Disconnect action despite child FRLs present (orphaned children)
- Child products created as independent FRLs instead of under parent
- Extra technical products appearing in decomposition without connection
- Product relationships not preserved in Fulfillment Request structure

#### Root Causes
1. **Incorrect ProductChildItem Configuration** — vlocity_cmt__ProductChildItem__c missing or wrong hierarchy level
2. **Missing Product Relationship Type** — Relationship type not set correctly (e.g., "Relies On", "Includes")
3. **Decomposition Specification Error** — DecompositionSpecification__c not mapping hierarchy correctly
4. **Child Product Action Mismatch** — Parent has "Disconnect" but child has "None", logic doesn't handle
5. **Multi-level Hierarchy Complexity** — 3+ levels of nesting causing incorrect parent-child assignment

#### Preventive Tasks to Add
When decomposing stories involving product hierarchy:
- [ ] **Document product hierarchy diagram** — Visual tree showing commercial → technical → sub-technical
- [ ] **Validate ProductChildItem records** — Test query to verify all parent-child links exist
- [ ] **Test multi-level hierarchy** — Test with 3-level product nesting (commercial → CFS → component)
- [ ] **Action propagation logic** — Define how actions (Add/Change/Disconnect) flow parent→child
- [ ] **Orphan detection** — Script to find FRLs without parent when parent expected
- [ ] **Hierarchy visualization test** — Ensure decomposition UI shows correct tree structure
- [ ] **Product relationship validation** — Pre-submit validation that hierarchy is intact

#### Test Scenarios
```gherkin
Scenario: Three-level product hierarchy
  Given Commercial Product "Fiber Internet Bundle"
  And Technical Product "Residential Broadband CFS" (child of Bundle)
  And Technical Product "Rate Plan" (child of CFS)
  And Technical Product "Recurring Charge" (child of Rate Plan)
  When I submit order for "Fiber Internet Bundle"
  And view decomposition
  Then I should see hierarchy:
    Bundle
      └─ CFS
          └─ Rate Plan
              └─ Recurring Charge
  And NOT see Rate Plan and Recurring Charge as independent FRLs

Scenario: Parent disconnect with active children
  Given Order with Parent FRL and 3 Child FRLs
  And Parent has action "Disconnect"
  When orchestration processes the order
  Then system should REJECT if children not also "Disconnect"
  Or system should AUTO-DISCONNECT children with Parent
  And log warning "Parent disconnect triggered child disconnect"
```

#### Known Patterns from Cases
- Case 43875336 (Now New Zealand): Residential Analog Telephony Line decomposing CFS, Rate Plan, Charge under SEPARATE FRs instead of hierarchy (Bug: W-12462489)
- Case 44135593 (PLDT): Extra technical products in decomposition without any connection (Bug exists)

---

### Pattern 6: Status / State Transition Issues

**Frequency:** 35 cases (5.3% of cases)  
**Severity:** 63% Urgent  
**Root Cause Category:** Configuration or "Working as Documented"

#### Common Symptoms
- Order status not updating to 'Ready To Submit' on creation
- Orchestration item status not transitioning (stuck in Ready/Pending/Running)
- Order stuck in "Submitted" state, decomposition complete but status not reflecting
- XOM Administration setting "CPQ/ORDER MANAGEMENT INTERFACE STATUS" not configured

#### Root Causes
1. **XOM Administration Not Configured** — vlocity_cmt__XOMAdministration__c custom settings missing
2. **Custom Status Transition Logic** — Custom trigger/flow overriding standard status
3. **Workflow Rules Blocking Updates** — Validation rule failing on status field update
4. **State Machine Misconfiguration** — Invalid status transitions defined in metadata
5. **Async Status Update Delay** — Status updated via Platform Event with delay

#### Preventive Tasks to Add
When decomposing stories involving status/state:
- [ ] **Document status state machine** — Diagram showing all valid status transitions
- [ ] **XOM Administration setup script** — Automated setup of vlocity_cmt__XOMAdministration__c
- [ ] **Status transition tests** — Test all valid transitions, test invalid transitions rejected
- [ ] **Validation rule audit** — List all validation rules that could block status updates
- [ ] **Async status update monitoring** — Alert if status not updated within expected timeframe
- [ ] **Status rollback logic** — Handle scenario where status must revert (e.g., orchestration fails)

---

### Pattern Usage in Decomposition

When decomposing a user story, ALWAYS:

1. **Read the story requirements**
2. **Identify which of the 6 patterns could be triggered**
3. **Add preventive tasks from relevant patterns**
4. **Reference pattern in task description** (e.g., "Prevent Pattern 3: UNABLE_TO_LOCK_ROW")
5. **Add pattern-specific test scenarios**

**Example:**
```
Story: "Add new commercial product 'IoT Connectivity Bundle' with 4 technical products"

Decomposition should include:
✅ Task: Configure product hierarchy (Pattern 5)
  - Validate ProductChildItem records
  - Test 4-level hierarchy in decomposition viewer
  - Verify no orphaned FRLs created

✅ Task: Set up permissions (Pattern 4)
  - Document required permission sets
  - Test with minimal-permission user
  
✅ Task: Configure decomposition (Pattern 1)
  - Create DecompositionSpecification
  - Mark IsDecomposable = true
  - Link OrchestrationPlan
```

---

## L. Known Limitations

This skill works best when:
- Story has clear acceptance criteria
- OM domain area is explicitly stated or inferable
- Technical context is provided (or available via GUS/CodeSearch)

This skill may struggle with:
- Extremely vague stories ("Make orders better")
- Stories spanning multiple products (OM + CPQ + Billing)
- Stories requiring deep product/design decisions not yet made
- Stories in brand-new OM domains without established patterns

In such cases, the skill will:
- Ask clarifying questions to narrow scope
- Highlight open questions requiring product/design input
- Provide decomposition options with trade-offs for user to choose
- Recommend breaking the story into smaller, clearer stories

---

## M. XOM Codebase Reference (via_xom-262.8)

**Codebase Path:** `/Users/sandesh.kulkarni/MPCode/via_xom-262.8`  
**Version:** 262.8  
**Total Classes:** 645 Apex classes  
**Total Objects:** 72 custom objects

This section provides direct code references from the actual XOM codebase to use when decomposing stories.

---

### Core Decomposition Classes

| Class | Purpose | Key Methods | When to Reference |
|-------|---------|-------------|-------------------|
| **SimpleDecompositionController** | Main decomposition entry point | `decompose(Id orderId)`, `validateOrderForOrderDecomposition()`, `isSfdcDecomposedOrder()` | Any story involving order decomposition initiation |
| **SimpleDecompositionManager** | Manages order submission | `submitOrderViaODIN(Id orderId)`, `submitOrderViaODIN(Id orderId, Map<String, Object> options)` | Stories about decomposition execution |
| **XOMDecompV2Constant** | All constants, actions, scopes, error messages | Constants for actions, scopes, error messages | Reference for validation messages, action types |
| **XOMDecompV2TreeDiffManager** | Manages diff between AsIs and ToBe trees | Tree diff calculation | Stories involving delta calculation |
| **XOMDecompV2TreeToBeManager** | Manages ToBe tree creation | ToBe tree building | Stories about target state calculation |

### Core Orchestration Classes

| Class | Purpose | Key Methods | When to Reference |
|-------|---------|-------------|-------------------|
| **OrchestrationManager** | Core orchestration manager | `updateQueuesCounters()`, `ensureAllQueuesStarted()` | Stories about orchestration queue management |
| **OrchestrationItemsManager** | Manages orchestration item lifecycle | Item state transitions, dependency evaluation | Stories about orchestration item processing |
| **OrchestrationItemsExecutor** | Executes orchestration items | Item execution logic | Stories about AutoTask/Manual/Callback execution |
| **OrchestrationEventHandler** | Handles orchestration platform events | Event processing | Stories involving async orchestration updates |
| **XOMOrchestrationPlan** | Orchestration plan domain object | Plan management | Stories about orchestration plan configuration |

### Lock & Concurrency Classes

| Class | Purpose | Key Methods | When to Reference |
|-------|---------|-------------|-------------------|
| **XOMLockManager** | Distributed lock management | `acquireLock(String lockName)` | **Pattern 3** (UNABLE_TO_LOCK_ROW) - Always reference this |
| **XOMIntegrationRetryPolicy** | Retry policy configuration | `getRetryPolicyType()`, `getMaximumNumberOfRetries()` | **Pattern 2** (Orchestration Errors) - Reference for retry logic |

### Key Custom Objects

#### Decomposition Objects

| Object | Fields to Know | Purpose in Tasks |
|--------|----------------|------------------|
| **DecompositionRelationship__c** | `OrderItemId__c`, `FulfillmentRequestLineId__c`, `RelationshipType__c` | Track OrderItem → FRL lineage |
| **FulfilmentRequestDecompRelationship__c** | `FulfillmentRequestId__c`, `DecompositionRelationshipId__c` | Link FR to decomposition |
| **FulfilmentRequestLineDecompRelationship__c** | `FulfillmentRequestLineId__c`, `DecompositionRelationshipId__c` | Link FRL to decomposition |

#### Fulfillment Objects

| Object | Key Fields | Purpose in Tasks |
|--------|------------|------------------|
| **FulfilmentRequest__c** | `OrderId__c`, `Status__c`, `RootItemId__c` | Top-level FR - one per order typically |
| **FulfilmentRequestLine__c** | `FulfilmentRequestId__c`, `Action__c`, `ProductId__c`, `ParentFulfillmentRequestLineId__c` | **Most important object** - represents technical products, hierarchy preserved here |
| **FulfilmentRequestLineRelationship__c** | `ParentFRLId__c`, `ChildFRLId__c`, `RelationshipType__c` | Explicitly track parent-child FRL relationships |

**FulfilmentRequestLine__c.Action__c Values:**
- `Add`, `Modify`, `Disconnect`, `Suspend`, `Resume`, `No Change`
- Reference: `XOMDecompV2Constant.ACTION_*`

#### Orchestration Objects

| Object | Key Fields | Purpose in Tasks |
|--------|------------|------------------|
| **OrchestrationPlan__c** | `OrderId__c`, `State__c`, `IsFrozen__c` | Created after decomposition, drives orchestration |
| **OrchestrationItem__c** | `OrchestrationPlanId__c`, `State__c`, `ItemType__c`, `FulfillmentRequestLineId__c` | Individual orchestration task (AutoTask/Manual/Callback) |
| **OrchestrationDependency__c** | `OrchestrationItemId__c`, `DependsOnOrchestrationItemId__c`, `DependencyType__c` | **Pattern 2** - Define dependencies to prevent stuck items |
| **OrchestrationQueue__c** | `State__c`, `Replicas__c` | Queue for executing orchestration items |
| **OrchestrationEvent__e** | `OrchestrationItemId__c`, `NewState__c` | Platform event for async state updates |
| **OrchestrationItemLog__b** | `OrchestrationItemId__c`, `Message__c`, `Timestamp__c` | Big object for audit trail (no governor limits) |
| **XOMLock__c** | `Name__c` | **Pattern 3** - Distributed lock object |

**OrchestrationItem__c.State__c Values:**
- `Pending`, `Ready`, `Running`, `Completed`, `Failed`, `Suspended`

**OrchestrationItem__c.ItemType__c Values:**
- `AutoTask`, `ManualTask`, `Callback`, `Milestone`

### Constants from XOMDecompV2Constant

Use these exact constants in task descriptions:

#### Decomposition Scopes

```apex
SCOPE_DOWNSTREAM_ORDERITEM = 'Downstream Order Item'  // No merging
SCOPE_ORDERITEM = 'Order Item'                       // Merge at UPOI
SCOPE_TOP_ORDERITEM = 'Top Order Item'              // Merge at STOI, multi-stage
SCOPE_ORDER = 'Order'                                // Merge at SO level
SCOPE_ACCOUNT = 'Account'                            // Account-based assets
SCOPE_RELIES_ON = 'Relies On'                        // Relies-on relationships
```

#### Actions & Weights

```apex
ACTION_ADD = 'Add'                    // Weight: 6
ACTION_MODIFY = 'Modify'              // Weight: 8
ACTION_DISCONNECT = 'Disconnect'      // Weight: 5
ACTION_SUSPEND = 'Suspend'            // Weight: 10 (highest)
ACTION_RESUME = 'Resume'              // Weight: 9
ACTION_NO_CHANGE = 'No Change'        // Weight: 7
```

#### Error Messages (use for validation task descriptions)

```apex
MESSAGE_DECOMPOSITION_NOT_AVAILABLE_DRAFT = 
    "Decomposition is not available as order is not decomposed yet."
    
MESSAGE_DECOMPOSITION_NOT_AVAILABLE_REJECTED = 
    "Decomposition is not available as order is rejected."
    
MESSAGE_SUPPLEMENTALORDER_NOPLAN = 
    "There is no Orchestration Plan."
    
MESSAGE_SUPPLEMENTALORDER_PONRREACHED = 
    "Changes are not allowed anymore because at least one of the impacted 
     bundles has reached a point of no return."
```

### Code Patterns to Reference in Tasks

#### Pattern A: Validate Order Before Decomposition (Pattern 1 Prevention)

```apex
// Reference: SimpleDecompositionController.validateOrderForOrderDecomposition()
public static String validateOrderForOrderDecomposition(Id orderId) {
    if (orderId == null) {
        return XOMDecompV2Constant.MESSAGE_NULL_ORDERID;
    }
    
    List<Order> orders = [SELECT Id, Status FROM Order WHERE Id = :orderId LIMIT 1];
    if (orders.isEmpty()) {
        return XOMDecompV2Constant.MESSAGE_NULL_ORDEROBJECT + orderId;
    }
    
    Order order = orders[0];
    if (order.Status == 'Draft') {
        return XOMDecompV2Constant.MESSAGE_DECOMPOSITION_NOT_AVAILABLE_DRAFT;
    }
    if (order.Status == 'Rejected') {
        return XOMDecompV2Constant.MESSAGE_DECOMPOSITION_NOT_AVAILABLE_REJECTED;
    }
    
    return null; // No errors - order is valid for decomposition
}
```

**Use in tasks:** Any story involving decomposition validation

#### Pattern B: Acquire Lock with Timeout (Pattern 3 Prevention)

```apex
// Reference: XOMLockManager.acquireLock() + OrchestrationItemsManager timeout pattern
public static void acquireLockWithTimeout(String lockName, Integer timeoutSeconds) {
    Datetime lockStartTs = Datetime.now();
    Boolean lockAcquired = false;
    
    while (!lockAcquired) {
        try {
            List<XOMLock__c> locks = [
                SELECT Name__c 
                FROM XOMLock__c 
                WHERE Name__c = :lockName 
                FOR UPDATE
            ];
            
            if (locks.isEmpty()) {
                insert new XOMLock__c(Name__c = lockName, Name = lockName);
            }
            lockAcquired = true;
            
        } catch (QueryException e) {
            // Check timeout
            Long elapsedSeconds = (Datetime.now().getTime() - lockStartTs.getTime()) / 1000;
            if (elapsedSeconds >= timeoutSeconds) {
                throw new LockTimeoutException('Could not acquire lock ' + lockName + ' after ' + timeoutSeconds + ' seconds');
            }
            // Retry (in production, use Queueable for proper retry)
        }
    }
}
```

**Use in tasks:** Orchestration item updates, concurrent batch processing

#### Pattern C: Orchestration Item State Logging (Pattern 2 Prevention)

```apex
// Reference: OrchestrationItemsManager + XOMOrchestrationItemLogHelper
public static void logOrchestrationItemTransition(
    Id itemId, 
    String previousState, 
    String newState
) {
    // Use Big Object for unlimited logging (no governor limits)
    OrchestrationItemLog__b logEntry = new OrchestrationItemLog__b();
    logEntry.OrchestrationItemId__c = itemId;
    logEntry.PreviousState__c = previousState;
    logEntry.NewState__c = newState;
    logEntry.Timestamp__c = Datetime.now();
    logEntry.Message__c = 'State transition: ' + previousState + ' -> ' + newState;
    
    Database.insertImmediate(logEntry);
}
```

**Use in tasks:** Any story involving orchestration state management

#### Pattern D: Check FRL Hierarchy Integrity (Pattern 5 Prevention)

```apex
// Query to find orphaned FRLs (FRLs without expected parent)
public static List<FulfilmentRequestLine__c> findOrphanedFRLs(Id fulfillmentRequestId) {
    // Find all FRLs where ParentFulfillmentRequestLineId__c is null
    // but ProductRelationshipType__c indicates it should have a parent
    return [
        SELECT Id, Name, Action__c, ProductId__c, ParentFulfillmentRequestLineId__c
        FROM FulfilmentRequestLine__c
        WHERE FulfilmentRequestId__c = :fulfillmentRequestId
        AND ParentFulfillmentRequestLineId__c = null
        AND ProductRelationshipType__c IN ('Child', 'Component', 'Included')
    ];
}
```

**Use in tasks:** Product hierarchy validation, post-decomposition verification

#### Pattern E: Retry Policy Configuration (Pattern 2 Prevention)

```apex
// Reference: XOMIntegrationRetryPolicy
// Configure staggered (exponential backoff) retry policy
IntegrationRetryPolicy__c retryPolicy = new IntegrationRetryPolicy__c();
retryPolicy.Name = 'FulfillmentOrchestrationRetry';
retryPolicy.RetryPolicyType__c = 'StaggeredRetryPolicy';  
retryPolicy.MaximumNumberOfRetries__c = 5;
retryPolicy.RetryIntervals__c = '60,120,240,480,960';  // 1m, 2m, 4m, 8m, 16m
insert retryPolicy;
```

**Use in tasks:** Orchestration plan configuration, integration setup

### REST API Endpoints (for Testing Tasks)

```
Base URL: /services/apexrest/Decomposition/

GET /isSfdcDecomposedOrder?orderId={id}
  → Returns: { "result": "true" | "false" }
  
GET /viewDecomposition?orderId={id}
  → Returns: { "result": "OK" | error message }
  
GET /viewDecomposedOrder?orderId={id}
  → Returns: Full decomposed order JSON structure
  
GET /viewDecomposedOrderStructure?orderId={id}
  → Returns: Decomposition hierarchy tree

GET /viewDecomposedRootOrderItem?orderId={id}&rootOrderItemId={id}
  → Returns: Specific root order item decomposition
```

**Use in tasks:** Integration testing, automated test scenarios

### File References for CodeSearch

When using CodeSearch to find similar implementations:

```
Decomposition logic:
  /Users/sandesh.kulkarni/MPCode/via_xom-262.8/classes/SimpleDecompositionController.cls
  /Users/sandesh.kulkarni/MPCode/via_xom-262.8/classes/XOMDecompV2Constant.cls
  /Users/sandesh.kulkarni/MPCode/via_xom-262.8/classes/XOMDecompV2TreeDiffManager.cls

Orchestration logic:
  /Users/sandesh.kulkarni/MPCode/via_xom-262.8/classes/OrchestrationManager.cls
  /Users/sandesh.kulkarni/MPCode/via_xom-262.8/classes/OrchestrationItemsManager.cls
  /Users/sandesh.kulkarni/MPCode/via_xom-262.8/classes/OrchestrationItemsExecutor.cls

Lock/Retry patterns:
  /Users/sandesh.kulkarni/MPCode/via_xom-262.8/classes/XOMLockManager.cls
  /Users/sandesh.kulkarni/MPCode/via_xom-262.8/classes/XOMIntegrationRetryPolicy.cls

Test patterns:
  /Users/sandesh.kulkarni/MPCode/via_xom-262.8/classes/XOMTestStoriesDecompositionMultiLevel.cls
  /Users/sandesh.kulkarni/MPCode/via_xom-262.8/classes/OrchestrationItemsExecutorTest.cls
```

### Architectural Flow References

#### Decomposition Flow
```
User clicks "Decompose Order"
  ↓
SimpleDecompositionController.decompose(orderId)
  ↓  
validateOrderForOrderDecomposition(orderId)  // Pattern 1 prevention
  ↓
SimpleDecompositionManager.submitOrderViaODIN(orderId)
  ↓
XOMDecompV2TreeToBeManager builds ToBe tree
XOMDecompV2TreeDiffManager calculates diff
  ↓
Create FulfilmentRequest__c + FulfilmentRequestLine__c records
Create DecompositionRelationship__c records
  ↓
Trigger OrchestrationPlan__c creation (if OM Plus)
```

#### Orchestration Flow
```
OrchestrationPlan__c created
  ↓
OrchestrationItemsManager evaluates dependencies
  ↓
Ready items enqueued to OrchestrationQueue__c
  ↓
OrchestrationManager.ensureAllQueuesStarted()
  ↓
OrchestrationItemsExecutor executes items
  ↓
State changes publish OrchestrationEvent__e  // Pattern 2 prevention
  ↓
OrchestrationEventHandler processes events
  ↓
Update dependent items, repeat until completion
```

**Use these flows in:** Architecture diagrams, technical design documents, task dependencies

---
