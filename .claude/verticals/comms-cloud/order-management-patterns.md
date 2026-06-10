## DEPRECATED SECTIONS NOTICE

> Sections A-F and G-K in this file contain **developer story decomposition methodology** (not troubleshooting patterns). These were misplaced here by a previous author. They will be removed in a future cleanup. 
>
> **For troubleshooting, skip directly to:**
> - **Section Z** (line ~778): Known Issue Patterns from Production Data (665 cases analyzed)
> - **Section after Z**: Permission Troubleshooting Checklist
> - **Section M**: XOM Codebase Reference (class/object mapping)

---

## A. Story Analysis & Context Gathering (DEPRECATED — dev methodology, not troubleshooting)

When a user provides an Order Management user story, follow this systematic approach:

### Step 1: Extract Story Information

From the user's input or GUS work item, extract:
- **Story ID** (if provided)
- **Title/Summary**
- **Description** — full requirements and context
- **Acceptance Criteria** — what defines "done"
- **Business Context** — why this matters, customer impact
- **Technical Context** — existing system constraints, dependencies
- **Priority & Target Release**

If story is in GUS, fetch using:
```bash
sf data query --query "SELECT Id, Name, Subject__c, Details__c, Acceptance_Criteria__c, Priority__c, Status__c, Product_Tag__r.Name FROM ADM_Work__c WHERE Name = '<STORY_ID>'" --target-org gus --json
```

### Step 2: Identify Order Decomposition Domain Areas

**PRIMARY FOCUS:** Order Decomposition specifically handles the breakdown of commercial products into technical products (Fulfillment Request Lines - FRLs) and orchestration.

Map the story to relevant Order Decomposition domain areas:

| Domain Area | Key Concepts | Common Vlocity Objects | Known Issue Frequency |
|------------|--------------|------------------------|----------------------|
| **Product Decomposition** | Commercial → Technical product breakdown, Product hierarchy, Decomposition specifications | vlocity_cmt__Product2__c, vlocity_cmt__ProductChildItem__c, vlocity_cmt__DecompositionSpecification__c | **HIGH** (30 cases: decomp not working) |
| **Orchestration Plans** | XOM orchestration, AutoTask items, Manual tasks, Orchestration item dependencies | vlocity_cmt__OrchestrationPlan__c, vlocity_cmt__OrchestrationItem__c, vlocity_cmt__OrchestrationDependency__c | **VERY HIGH** (63 cases: orchestration errors) |
| **Fulfillment Requests** | FRL creation, FRL hierarchy, Parent-child relationships | vlocity_cmt__FulfillmentRequest__c, vlocity_cmt__FulfillmentRequestLine__c | **MEDIUM** (14 cases: hierarchy issues) |
| **Order Status & State** | Order lifecycle, XOM status transitions, State management | vlocity_cmt__Order__c, vlocity_cmt__OrderStatus__c, vlocity_cmt__XOMAdministration__c | **MEDIUM** (35 cases: status issues) |
| **Decomposition UI** | Decomposition viewer, Product hierarchy display, Fulfillment diagram | vlocity_cmt:decompositionViewer, vlocity_cmt:fulfillmentDesigner | **LOW** (UI rendering issues) |
| **Permissions & Access** | XOM User permissions, Field-level security, Profile access | PermissionSet (XOM User, XOM Admin), Field-level security on vlocity_cmt objects | **CRITICAL** (133 cases: 20% of all issues) |
| **Batch & Lock Management** | Concurrency, Row locking, Batch job conflicts | BatchApex, Queueable jobs, Platform Events | **HIGH** (39 cases: UNABLE_TO_LOCK_ROW) |
| **Integration Procedures** | Order submission via IP, Decomposition triggers from external systems | vlocity_cmt__IntegrationProcedure__c | **MEDIUM** (Related to orchestration) |

**KEY INSIGHT FROM CASE DATA:** 
- 59.4% of cases are Level 2 - Urgent with avg resolution time of 12.1 days
- 20% of all cases are permission-related (most common root cause)
- 17.3% of cases escalate to GUS investigations
- 8.9% result in bug creation

### Step 3: Analyze Codebase Impact (if CodeSearch available)

Search for relevant code areas:
```
Tool: mcp__plugin_deep-research_codesearch__search
Queries:
1. Core domain classes: content:"OrderSummary" OR content:"FulfillmentOrder" lang:java
2. Related services: content:"<DomainArea>Service" lang:java
3. API endpoints: content:"OrderManagement" content:"@RestResource" OR content:"@AuraEnabled"
4. Triggers: content:"OrderTrigger" OR content:"OrderSummaryTrigger"
5. Test classes: content:"<Feature>Test" lang:java
```

Identify:
- Affected modules/classes
- Existing patterns to follow
- Related features that may have dependencies
- Test coverage patterns

### Step 4: Research Historical Context

Query GUS for similar Order Decomposition work:
```bash
sf data query --query "SELECT Name, Subject__c, Details__c, Resolution__c, Status__c FROM ADM_Work__c WHERE Product_Tag__r.Name = 'Industry-CPQ / Order Management / Digital Commerce' AND Team__r.Name = 'CME - OM Starlord' AND Subject__c LIKE '%<similar_feature>%' ORDER BY CreatedDate DESC LIMIT 10" --target-org gus --json
```

Query OrgCS for support cases (if OrgCS available):
```
Tool: mcp__orgcs__soqlQuery
Query: SELECT CaseNumber, Subject, Description, Case_Cause__c, Sub_Cause__c, Severity_Level__c 
       FROM Case 
       WHERE Product_Topic__r.Name = 'Industry-CPQ / Order Management / Digital Commerce' 
       AND Subject LIKE '%<similar_feature>%'
       ORDER BY CreatedDate DESC 
       LIMIT 10
```

Look for:
- Similar stories and how they were decomposed
- Known pitfalls from 242 GUS investigations (top statuses: "New Bug Logged" 73, "Resolved Without Code Change" 41)
- Common case causes: "Error message issue" (198), "User Needs Education" (196), "Product Bug" (31)
- Patterns in task breakdown
- Historical estimates vs actuals

### Step 5: Check Against Known Issue Patterns

**CRITICAL:** Before proceeding with decomposition, check if the story relates to any of the 6 top issue patterns identified from 665 support cases. If so, incorporate preventive tasks.

See **Section Z: Known Issue Patterns from Production Data** for:
- Pattern 1: Decomposition Not Working (30 cases, 4.5%)
- Pattern 2: Orchestration Errors (63 cases, 9.5%)
- Pattern 3: UNABLE_TO_LOCK_ROW Errors (39 cases, 5.9%)
- Pattern 4: Permission/Access Issues (133 cases, 20%)
- Pattern 5: Product Structure/Hierarchy Issues (14 cases, 2.1%)
- Pattern 6: Status/State Transition Issues (35 cases, 5.3%)

---

## B. Decomposition Framework

Use this systematic framework to break down any OM story:

### Layer 1: Technical Architecture Layers

Decompose across standard architectural layers:

1. **Data Model Layer**
   - Schema changes (new objects, fields, relationships)
   - Custom metadata types
   - Sharing model changes
   - Data migration considerations

2. **Business Logic Layer**
   - Service classes (new services or updates to existing)
   - Trigger handlers
   - Validation rules
   - Business process flows
   - Calculation engines (pricing, tax, discounts)

3. **Integration Layer**
   - API endpoints (REST/SOAP)
   - External service integrations
   - Platform events
   - Connect API usage
   - Asynchronous processing (Queueable, Batch, Future)

4. **User Interface Layer**
   - Lightning Web Components (LWC)
   - Aura components (if legacy)
   - Lightning pages/apps
   - Quick actions and flows
   - Experience Cloud pages

5. **Testing & Quality Layer**
   - Unit tests (Apex)
   - Integration tests
   - Component tests (Jest/LWC)
   - E2E tests
   - Performance tests

6. **Documentation & Enablement Layer**
   - Technical documentation
   - API documentation
   - Release notes
   - User guides/help articles

### Layer 2: Cross-Cutting Concerns

For each technical layer task, consider:

- **Security & Access Control**
  - Permission sets/profiles
  - Field-level security
  - Object-level security
  - Sharing rules
  - Multi-org considerations

- **Data Governance**
  - Data retention policies
  - PII handling
  - GDPR compliance
  - Data archival

- **Performance**
  - Governor limits (SOQL, DML, CPU, heap)
  - Bulkification
  - Query optimization
  - Caching strategies

- **Multi-Tenancy**
  - Org-specific configurations
  - Package considerations (managed/unmanaged)
  - Namespace handling

- **Observability**
  - Logging and debug traces
  - Error handling and recovery
  - Monitoring and alerts
  - Instrumentation for analytics

### Layer 3: Order Management-Specific Patterns

Apply OM-specific decomposition patterns:

#### Pattern A: New Order Action
For stories adding new order actions (e.g., new fulfillment action, cancel action):
1. Define action metadata (OrderAction type, status transitions)
2. Implement action service class
3. Create action validation rules
4. Update OrderSummary lifecycle
5. Add UI action buttons/flows
6. Implement audit logging
7. Add tests for all status transitions
8. Update API documentation

#### Pattern B: Fulfillment Flow Enhancement
For fulfillment-related changes:
1. Analyze FulfillmentOrder and related objects
2. Update orchestration plan if needed
3. Modify inventory allocation logic
4. Update shipping/delivery method handling
5. Adjust fulfillment status transitions
6. Implement rollback/compensation logic
7. Test multi-location scenarios
8. Test partial fulfillment cases

#### Pattern C: Payment Integration
For payment-related features:
1. Define PaymentGateway adapter interface
2. Implement authorization logic
3. Add capture/settlement handling
4. Implement refund/void scenarios
5. Add retry and error recovery
6. Handle async payment responses
7. Implement webhook handlers
8. Test PCI compliance requirements

#### Pattern D: Order Amendment/Change
For order modification features:
1. Analyze ChangeOrder object usage
2. Calculate delta (price, quantity, items)
3. Implement amendment validation
4. Handle inventory adjustments
5. Update payment adjustments
6. Manage order item versioning
7. Test complex amendment scenarios
8. Handle amendment cancellation

#### Pattern E: Returns & Refunds
For return order processing:
1. Create ReturnOrder and line items
2. Calculate refund amounts (with rules)
3. Implement return reason validation
4. Update inventory (restock logic)
5. Process refund through payment gateway
6. Handle partial returns
7. Test restocking scenarios
8. Update customer account (credits)

---

## C. Task Template Structure

For each decomposed task, use this template:

```
### Task: [LAYER] - [SPECIFIC WORK]

**Domain Area:** [Order Capture / Fulfillment / Inventory / etc.]
**Type:** [Data Model / Business Logic / Integration / UI / Test / Documentation]
**Priority:** [P0 - Blocker / P1 - Critical / P2 - High / P3 - Medium / P4 - Low]
**Estimate:** [Story Points or Days]

**Description:**
[Clear, concise description of what needs to be built/changed]

**Acceptance Criteria:**
- [ ] [Specific, testable criterion 1]
- [ ] [Specific, testable criterion 2]
- [ ] [Specific, testable criterion 3]

**Technical Details:**
- Objects/Fields: [List affected standard/custom objects and fields]
- Classes/Components: [List classes/LWCs to create or modify]
- APIs: [List API endpoints if applicable]
- Permissions: [List required permission sets/profiles]

**Dependencies:**
- [Task ID or description] must be completed first
- [External dependency, e.g., "Requires Payment Gateway sandbox access"]

**Test Scenarios:**
1. [Happy path scenario]
2. [Edge case scenario]
3. [Error/exception scenario]
4. [Performance scenario if applicable]

**Code References:** (if CodeSearch available)
- [Existing similar implementation to reference]
- [Related classes to review]
- [Test patterns to follow]

**Risks & Considerations:**
- [Governor limit concerns]
- [Performance considerations]
- [Security/access control concerns]
- [Backward compatibility concerns]

**Definition of Done:**
- [ ] Code implemented and peer reviewed
- [ ] Unit tests written (>85% coverage)
- [ ] Integration tests added
- [ ] Security review completed
- [ ] Performance tested (governor limits check)
- [ ] Documentation updated
- [ ] Code merged to main branch
```

---

## D. Execution Steps

When a user asks to decompose an OM story, follow this process:

### Step 1: Clarify Requirements (if needed)

Ask clarifying questions if story lacks detail:
```
I'm analyzing your Order Management story. To provide accurate decomposition, I need clarity on:

1. **Scope:** Does this affect existing orders or only new orders?
2. **Integration:** Are there external systems involved (payment gateway, ERP, shipping)?
3. **UI:** Is a UI component needed, or is this backend-only?
4. **Data Volume:** Expected order volume and performance requirements?
5. **Rollout:** Phased rollout or all-at-once? Any feature flags needed?
6. **Backward Compatibility:** Must support existing order data?
```

### Step 2: Gather Context (Parallel)

Run these in parallel:
1. Fetch GUS story details (if story ID provided)
2. Search CodeSearch for relevant existing code
3. Query historical similar work items
4. Search Slack for team context (if channel provided)

### Step 3: Apply Decomposition Framework

1. Map story to OM domain areas (Section B, Step 2)
2. Identify all affected architectural layers (Section C, Layer 1)
3. Apply relevant OM-specific pattern (Section C, Layer 3)
4. Consider cross-cutting concerns (Section C, Layer 2)

### Step 4: Generate Task List

For each identified work unit:
1. Create a task using the template (Section D)
2. Assign priority based on:
   - Dependency chain (foundational work = higher priority)
   - Business risk (customer-facing = higher priority)
   - Technical risk (complex integration = higher priority)
3. Add effort estimate based on:
   - Complexity (simple CRUD vs complex orchestration)
   - Uncertainty (well-known pattern vs new territory)
   - Testing requirements
4. Map dependencies explicitly

### Step 5: Validate Completeness

Check that decomposition covers:
- [ ] All acceptance criteria from original story
- [ ] Data model changes (if any)
- [ ] Business logic implementation
- [ ] UI changes (if customer-facing)
- [ ] API changes (if integration point)
- [ ] Security and permissions
- [ ] Testing at all levels
- [ ] Documentation
- [ ] Performance considerations
- [ ] Error handling and rollback

### Step 6: Output Structured Decomposition

Present the decomposition in this format:

```markdown
# Story Decomposition: [STORY TITLE]

**Story ID:** [ID if available]
**Domain Areas:** [List of affected OM domains]
**Estimated Total Effort:** [Sum of task estimates]
**Critical Path:** [Identify longest dependency chain]

---

## Summary

[High-level overview of the approach — what patterns were applied, key architectural decisions]

---

## Task Breakdown

### Phase 1: Foundation (Data Model & Core Services)
[Tasks that must be done first — data schema, core services]

#### Task 1.1: [Task Title]
[Full task template as defined in Section D]

#### Task 1.2: [Task Title]
[Full task template]

---

### Phase 2: Business Logic & Integration
[Tasks that build on foundation]

#### Task 2.1: [Task Title]
[Full task template]

---

### Phase 3: User Interface & Experience
[UI tasks, flows, actions]

#### Task 3.1: [Task Title]
[Full task template]

---

### Phase 4: Testing & Quality Assurance
[Comprehensive testing tasks]

#### Task 4.1: [Task Title]
[Full task template]

---

### Phase 5: Documentation & Enablement
[Documentation, release notes, enablement]

#### Task 5.1: [Task Title]
[Full task template]

---

## Dependency Graph

```
[Visual representation using text or mermaid diagram showing task dependencies]

Task 1.1 (Data Model)
  ↓
Task 2.1 (Service Layer) ← Task 1.2 (Metadata)
  ↓
Task 3.1 (UI Component)
  ↓
Task 4.1 (Integration Tests)
```

---

## Risk Assessment

| Risk | Impact | Mitigation | Owner |
|------|--------|------------|-------|
| [Identified risk 1] | High/Med/Low | [Mitigation strategy] | [Team/Role] |
| [Identified risk 2] | High/Med/Low | [Mitigation strategy] | [Team/Role] |

---

## Open Questions

- [ ] [Question 1 that needs clarification]
- [ ] [Question 2 requiring product/design decision]

---

## Recommended Next Steps

1. [Immediate next action, e.g., "Review decomposition with tech lead"]
2. [Follow-up action, e.g., "Create GUS work items for each task"]
3. [Long-term action, e.g., "Schedule architecture review"]
```

---

## E. Common Order Management Decomposition Patterns

Reference these proven patterns when decomposing OM stories:

### Pattern 1: Add New Order Status

**Trigger:** Story adds new status to OrderSummary or FulfillmentOrder

**Standard Decomposition:**
1. **Data Model:** Add picklist value, update status flow diagram
2. **Business Logic:** Update status transition validation service
3. **Status Calculation:** Modify status rollup/aggregation logic
4. **Audit:** Add status change to audit log
5. **UI:** Update status badges/indicators in LWC
6. **API:** Update API response schemas to include new status
7. **Tests:** Test all valid/invalid status transitions
8. **Docs:** Update status lifecycle documentation

**Estimated Effort:** 3-5 days

### Pattern 2: New Calculated Field on Order

**Trigger:** Story requires new calculated/derived field on OrderSummary

**Standard Decomposition:**
1. **Data Model:** Add custom field (decide: formula vs calculated vs stored)
2. **Calculation Logic:** Implement calculation service/trigger
3. **Recalculation:** Handle recalc on order amendments/changes
4. **Performance:** Ensure calculation doesn't hit governor limits
5. **API Exposure:** Add field to REST/GraphQL schemas
6. **UI Display:** Show field in order summary components
7. **Tests:** Test calculation accuracy, bulk scenarios, edge cases
8. **Docs:** Document calculation formula and business rules

**Estimated Effort:** 2-4 days

### Pattern 3: External System Integration

**Trigger:** Story requires integration with external system (payment, shipping, ERP)

**Standard Decomposition:**
1. **Integration Design:** Define contract (API spec, error handling, retry)
2. **Named Credential:** Set up authentication
3. **External Service:** Register external service (if applicable)
4. **Callout Service:** Implement HTTP callout wrapper
5. **Async Processing:** Implement Queueable for callouts (avoid mixed DML)
6. **Response Handler:** Parse response, update Salesforce records
7. **Error Recovery:** Implement retry logic, dead-letter queue
8. **Platform Events:** Publish events for async processing
9. **Monitoring:** Add logging, alerts for failures
10. **Tests:** Mock callouts, test timeout/error scenarios
11. **Docs:** Document integration architecture, credentials setup

**Estimated Effort:** 5-10 days

### Pattern 4: Order Amendment Flow

**Trigger:** Story allows modifying existing orders (add/remove/change items)

**Standard Decomposition:**
1. **Amendment Rules:** Define what can be amended in which statuses
2. **ChangeOrder Creation:** Generate ChangeOrder record with deltas
3. **Price Recalculation:** Recalc pricing for amended items
4. **Inventory Adjustment:** Release old reservation, create new
5. **Payment Adjustment:** Calculate payment delta, process authorization
6. **Status Management:** Update order status during amendment
7. **Rollback Logic:** Handle amendment cancellation
8. **Audit Trail:** Log all amendment actions
9. **UI Flow:** Build amendment wizard (multi-step flow)
10. **Validation:** Validate amendment doesn't violate business rules
11. **Tests:** Test complex scenarios (partial amendments, cancellations)
12. **Docs:** Document amendment process, limitations

**Estimated Effort:** 10-15 days

### Pattern 5: New Order Action Type

**Trigger:** Story adds new action button/flow on order (e.g., "Mark as Gift")

**Standard Decomposition:**
1. **Action Metadata:** Define OrderAction type in metadata
2. **Action Service:** Implement action execution logic
3. **Validation:** Pre-action validation (can this order be gift-wrapped?)
4. **State Update:** Update order/fulfillment state after action
5. **Side Effects:** Trigger downstream processes (e.g., special packaging)
6. **UI Action:** Add action button to Order Summary page
7. **Permissions:** Define who can execute this action
8. **Audit:** Log action execution with timestamp and user
9. **Undo/Rollback:** Implement reverse action if applicable
10. **Tests:** Test action in various order states
11. **Docs:** Document action behavior, prerequisites

**Estimated Effort:** 4-7 days

### Pattern 6: Fulfillment Allocation Strategy

**Trigger:** Story changes how inventory is allocated to fulfillment orders

**Standard Decomposition:**
1. **Strategy Design:** Define allocation algorithm (closest location, cheapest, fastest)
2. **Location Evaluation:** Implement location scoring service
3. **Inventory Check:** Real-time inventory availability check
4. **Reservation Logic:** Reserve inventory at selected location
5. **Multi-Location Handling:** Split fulfillment across locations if needed
6. **Fallback Strategy:** Handle out-of-stock scenarios
7. **Configuration:** Make strategy configurable (custom metadata)
8. **Performance:** Optimize for bulk orders (avoid N+1 queries)
9. **Tests:** Test various inventory scenarios, multi-location splits
10. **Docs:** Document allocation logic, configuration options

**Estimated Effort:** 7-12 days

---

## F. Effort Estimation Guidelines

Use these guidelines to estimate each task:

### Complexity Factors

| Factor | Simple (1x) | Medium (2x) | Complex (3x) |
|--------|------------|-------------|-------------|
| **Code Complexity** | Single class, straightforward logic | Multiple classes, moderate branching | Complex orchestration, many edge cases |
| **Data Volume** | <1000 records | 1K-100K records | >100K records, bulk processing |
| **Integration** | No external systems | Simple REST callout | Complex integration, error handling, retry |
| **Testing Effort** | Basic unit tests | Integration tests needed | E2E tests, performance tests, complex mocking |
| **UI Complexity** | Simple form/display | Interactive component, events | Complex wizard, real-time updates, validation |
| **Uncertainty** | Well-known pattern | Some unknowns | Uncharted territory, POC needed |

### Base Estimates (for a mid-level engineer)

- **Simple Data Model Change:** 0.5-1 day
- **Business Logic (Service Class):** 1-3 days
- **Integration (External Callout):** 2-5 days
- **UI Component (LWC):** 2-4 days
- **Complex Orchestration:** 5-10 days
- **Unit Tests:** 20-30% of implementation time
- **Integration Tests:** 30-50% of implementation time
- **Documentation:** 10-15% of implementation time

### Adjustment Multipliers

- **Junior Engineer:** 1.5-2x base estimate
- **Senior Engineer:** 0.7-0.8x base estimate
- **Unfamiliar Domain:** 1.3-1.5x base estimate
- **High Coupling:** 1.2-1.4x base estimate (many dependencies)
- **Strict Performance Req:** 1.3-1.6x base estimate (requires optimization)

---

## G. Quality Checklist (Self-Review Before Output)

Before delivering the decomposition, verify:

- [ ] Every original acceptance criterion is covered by at least one task
- [ ] All affected OM domain areas are identified
- [ ] Tasks follow dependency order (foundation → logic → UI → tests)
- [ ] Each task has clear, measurable acceptance criteria
- [ ] Estimates are grounded in complexity factors, not guesses
- [ ] Security and permissions are explicitly addressed
- [ ] Performance/governor limits are considered for bulk operations
- [ ] Error handling and rollback scenarios are included
- [ ] Testing tasks cover unit, integration, and E2E levels
- [ ] Documentation tasks are included
- [ ] Code references point to real patterns (if CodeSearch available)
- [ ] Risks are identified with mitigation strategies
- [ ] Open questions are explicitly called out
- [ ] The decomposition is technology-appropriate (no over-engineering)

---

## H. Tribal Knowledge & Pro Tips (Order Decomposition Specific)

**SOURCE:** Extracted from 665 support cases and #tmp-help-cme-order-management Slack discussions

### Tip 1: Always Assign Permission Sets First (Pattern 4)
**20% of all cases are permission issues.** Before ANY decomposition testing, assign "XOM User" and "XOM Admin" permission sets. Verify FLS on all vlocity_cmt__*__c fields. This prevents the #1 most common issue.

### Tip 2: Decomposition Requires "Submitted" Status (Pattern 1)
Orders must be in "Submitted" or "In Flight" status before decomposition. 4.5% of cases are "decomposition not working" because order in wrong status. Always validate status before calling decomposition.

### Tip 3: Batch Size = 50 for XOM to Avoid Locks (Pattern 3)
Reduce batch sizes from 200 to 50-100 for vlocity_cmt objects to prevent UNABLE_TO_LOCK_ROW. 5.9% of cases are lock errors from concurrent batch jobs. Always implement retry logic with exponential backoff.

### Tip 4: Orchestration Items Stuck = Check Dependencies (Pattern 2)
If orchestration item stuck in "Ready" for >5 minutes, check vlocity_cmt__OrchestrationDependency__c. 9.5% of cases are orchestration errors from misconfigured dependencies or timing issues.

### Tip 5: Product Hierarchy Must Be Explicit (Pattern 5)
Use vlocity_cmt__ProductChildItem__c to define parent-child relationships. Never rely on implicit relationships. 2.1% of cases are hierarchy issues causing orphaned FRLs or incorrect decomposition structure.

### Tip 6: XOM Administration is Required (Pattern 6)
Configure vlocity_cmt__XOMAdministration__c custom settings, specifically "CPQ/ORDER MANAGEMENT INTERFACE STATUS". 5.3% of cases are status issues from missing XOM Administration config.

### Tip 7: Test with Amend and Cancel Orders
30% of "decomposition not working" cases happen on Amend/Cancel orders, not initial order submission. Always test full order lifecycle (New → Submit → Amend → Cancel → Resubmit).

### Tip 8: Monitor Orchestration Item Timeouts
Set up monitoring for orchestration items stuck >30 minutes. Avg case resolution time is 12 days when orchestration silently fails. Proactive monitoring cuts this to hours.

### Tip 9: After Vlocity Package Upgrade, Re-verify FLS
Package upgrades from vlocity reset field-level security. Run automated FLS verification script after EVERY package upgrade (238.x → 240.x, etc.). This prevents regression.

### Tip 10: Use Platform Events to Decouple Lock-Prone Operations
For operations that frequently lock (AutoTask batch updates, external callouts), use Platform Events to decouple and reduce lock contention. This is the #1 resolution for UNABLE_TO_LOCK_ROW patterns.

### Tip 11: Document Orchestration Plans as Diagrams
For complex orchestration plans with >5 items, create visual dependency graph. This prevents 35% of "orchestration stuck" issues caused by circular dependencies or missing prerequisites.

### Tip 12: Pre-Validation Before Decomposition
Implement pre-decomposition validation:
- Check vlocity_cmt__IsDecomposable__c = true
- Verify vlocity_cmt__DecompositionSpecification__c exists
- Validate vlocity_cmt__OrchestrationPlan__c assignment
- Check order status = "Submitted"
This prevents 4.5% of "decomposition not available" cases.

---

## I. Escalation & Expert Consultation

**When to escalate to CME - OM Starlord team:**
- Story impacts core XOM/vlocity_cmt decomposition objects
- Changes affect orchestration plan execution or state machine
- New decomposition pattern not covered by existing specifications
- Performance concerns with high-volume order decomposition (>1000 orders/hour)
- Multi-package or namespace boundary concerns (vlocity_cmt/vlocity_ins)
- Permission issues not resolved by standard XOM permission sets
- UNABLE_TO_LOCK_ROW errors requiring platform changes
- Product hierarchy issues requiring schema changes

**What to prepare before escalation:**
- Completed story analysis (Section A) with Step 5 (Check Against Known Issue Patterns)
- Attempted decomposition with specific questions highlighted
- Query historical GUS investigations from CME - OM Starlord (Section A, Step 4)
- Code references showing existing patterns (if CodeSearch available)
- Specific technical questions referencing known patterns (Pattern 1-6)
- Support case data if this is addressing customer issue (OrgCS query)

**When to involve broader OM team:**
- Changes span Order Decomposition + Fulfillment + Inventory
- Integration with other OM products (CPQ, Revenue Cloud)
- Enterprise-wide decomposition strategy changes

**Team Contacts:**
- **Primary Slack:** #tmp-help-cme-order-management (CME - OM Starlord)
- **Escalation Data:** 17.3% of decomposition cases escalate to GUS, 8.9% result in bugs
- **Avg GUS Resolution Approaches:** 30% New Bug Logged, 17% Resolved Without Code Change, 16% Resolved With Internal Tools

---

## J. Update Cadence

Refresh this skill:
- **After each OM release** (3x/year — Spring, Summer, Winter) to incorporate new platform features
- **When new OM patterns emerge** from team retrospectives
- **After major architecture changes** to OM core objects
- **Quarterly review** of estimation accuracy (compare estimates vs actuals)

---

## K. Test Cases

### Test Case 1: Simple Order Status Addition
**Input:** "Add a 'Ready for Pickup' status to OrderSummary for in-store pickup orders"
**Expected Output:**
- Tasks for: picklist value, validation logic, UI update, API schema, tests, docs
- Dependencies: Data model → Logic → UI → Tests
- Estimate: ~3-4 days
- Pattern applied: Pattern 1 (Add New Order Status)

### Test Case 2: External Shipping Integration
**Input:** "Integrate with ShipStation API to push fulfillment orders and track shipments"
**Expected Output:**
- Tasks for: integration design, named credential, callout service, async handler, error recovery, platform events, monitoring, tests
- Dependencies: Auth setup → Integration → Async → Error handling → Tests
- Estimate: ~8-12 days
- Pattern applied: Pattern 3 (External System Integration)
- Risks identified: API rate limits, timeout handling, credential expiry

### Test Case 3: Order Amendment Feature
**Input:** "Allow CSRs to add new items to existing orders that are not yet shipped"
**Expected Output:**
- Tasks for: amendment rules, ChangeOrder logic, pricing recalc, inventory adjust, payment auth, rollback, audit, UI flow, tests
- Dependencies: Foundation (rules, ChangeOrder) → Pricing → Inventory → Payment → UI → Tests
- Estimate: ~12-15 days
- Pattern applied: Pattern 4 (Order Amendment Flow)
- Risks identified: Payment re-authorization failure, inventory unavailable, complex rollback scenarios

### Test Case 4: Performance Optimization
**Input:** "Optimize order summary page load time — currently taking 8+ seconds for orders with 100+ line items"
**Expected Output:**
- Tasks for: profiling/diagnostics, query optimization, lazy loading, caching, pagination, tests
- Not a feature decomposition but performance investigation + fixes
- Estimate: ~5-8 days (includes measurement, optimization, validation)
- Special considerations: Measure before/after, set SLA target

### Test Case 5: Multi-Location Fulfillment Strategy
**Input:** "Implement smart fulfillment allocation — fulfill from closest warehouse with inventory, split if needed"
**Expected Output:**
- Tasks for: strategy design, location scoring, inventory check, reservation, split logic, fallback, config, tests
- Dependencies: Strategy algo → Location scoring → Inventory check → Reservation → Tests
- Estimate: ~10-14 days
- Pattern applied: Pattern 6 (Fulfillment Allocation Strategy)
- Risks identified: Performance with many locations, split order complexity, partial inventory scenarios

---

## Z. Known Issue Patterns from Production Data

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

## N. Feedback Loop

To improve this skill over time:

1. **Track Decomposition Accuracy:** Compare estimated vs actual effort after story completion
2. **Capture New Patterns:** When a novel OM pattern emerges, document it for future decompositions
3. **Refine Estimates:** Adjust estimation guidelines based on historical data
4. **Update for Platform Changes:** Incorporate new OM platform features/objects after each release (currently 262.8)
5. **Update Codebase References:** When XOM version changes, re-analyze codebase and update Section M
6. **Gather Team Feedback:** Ask OM engineers which decompositions were helpful vs which missed critical tasks

**Feedback channels:**
- Slack: #tmp-help-cme-order-management (CME - OM Starlord team channel)
- Team: CME - OM Starlord
- Product Tag: Industry-CPQ / Order Management / Digital Commerce
- Product Feature: Order Management-Order Decomposition
- Codebase: `/Users/sandesh.kulkarni/MPCode/via_xom-262.8`