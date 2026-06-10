## A. Triage & Classification

| Symptom | Pattern |
|---|---|
| Revenue arrangement not creating / stuck "Pending" | Pattern 1 |
| Revenue schedules have wrong dates | Pattern 2 |
| Multi-element allocation incorrect | Pattern 3 |
| Deferred revenue balance wrong | Pattern 4 |
| ASC 606 / IFRS 15 compliance config issues | Pattern 5 |
| Revenue waterfall/distribution errors | Pattern 6 |
| DRO integration with Orders/Billing failing | Pattern 7 |
| Multi-currency revenue errors | Pattern 8 |
| Restatement needed / historical corrections | Pattern 9 |
| Performance obligation not tracking | Pattern 10 |

---

## B. Known Issue Patterns

### Pattern 1: Revenue Recognition Not Calculating

**Symptoms:**
- No revenue schedules generated after order activation
- Revenue arrangement status stuck in "Pending"
- Zero revenue amounts in recognition reports
- Error: "Unable to calculate revenue recognition"

**Common Error Messages:**
```
- "Revenue calculation failed: Missing required configuration"
- "Performance obligation not defined for product"
- "Revenue schedule generation error: Invalid date range"
- "Revenue recognition rule not found"
```

**Root Causes:**
1. Missing revenue recognition rules on products
2. Revenue schedule not triggered (workflow/flow issue)
3. Incomplete performance obligation setup
4. Missing required fields on Order or Contract
5. Revenue arrangement not created or activated
6. User permissions insufficient for revenue objects

**Resolution Steps:**

1. **Verify Product Configuration**
   ```
   - Check Product record has Revenue Recognition Rule assigned
   - Verify Revenue Recognition Rule is active
   - Confirm Revenue Schedule Type is set (e.g., Daily, Monthly, Quarterly)
   - Validate Revenue Distribution Method (Even, First Period, Last Period)
   ```

2. **Check Revenue Arrangement Creation**
   ```
   - Query: SELECT Id, Status, RevenueAmount FROM RevenueArrangement WHERE OrderId = '[OrderId]'
   - Verify Status = 'Active' or 'Pending Recognition'
   - Confirm RevenueAmount matches expected value
   - Check CreatedDate is after Order activation
   ```

3. **Validate Performance Obligations**
   ```
   - Query: SELECT Id, Status, Product2Id FROM PerformanceObligation WHERE RevenueArrangementId = '[RAId]'
   - Ensure all products have corresponding performance obligations
   - Verify Status = 'Active'
   - Check StartDate and EndDate are populated
   ```

4. **Review Revenue Schedule Generation**
   ```
   - Query: SELECT Id, CreatedDate, RevenueAmount FROM RevenueSchedule WHERE RevenueArrangementId = '[RAId]'
   - If empty, check if Revenue Schedule job is running
   - Review Setup > Jobs > Scheduled Jobs for "Revenue Schedule Generation"
   - Check Debug Logs for revenue calculation errors
   ```

5. **Verify User Permissions**
   ```
   Required Permission Sets:
   - Revenue Cloud User
   - Revenue Recognition User (if available)
   
   Required Object Permissions:
   - Read/Create/Edit on RevenueArrangement
   - Read/Create/Edit on PerformanceObligation
   - Read/Create/Edit on RevenueSchedule
   - Read/Create/Edit on RevenueScheduleLine
   ```

6. **Check Automation Configuration**
   ```
   - Verify Flow/Process Builder triggers revenue arrangement creation
   - Check Flow: "Create Revenue Arrangement" is active
   - Validate automation entry criteria matches your use case
   - Review Flow error logs if arrangement not created
   ```

**Verification Steps:**
1. Create a test order with known products
2. Activate the order and wait for processing
3. Check that Revenue Arrangement is created within expected timeframe
4. Verify Revenue Schedule lines are generated
5. Confirm revenue amounts match expected calculations
6. Review revenue waterfall report for accuracy

**Best Practices:**
- Always assign Revenue Recognition Rules at product creation
- Test revenue calculation in sandbox before production deployment
- Document custom revenue logic and exceptions
- Set up monitoring for failed revenue calculations
- Maintain audit trail for all revenue adjustments

---

### Pattern 2: Revenue Schedule Generation Timing Issues

**Symptoms:**
- Revenue schedules created but with wrong dates
- Recognition periods don't match contract terms
- Revenue recognized too early or too late
- Schedule dates don't align with billing cycles

**Common Error Messages:**
```
- "Revenue schedule dates outside contract period"
- "Invalid revenue start date"
- "Schedule period mismatch with performance obligation"
- "Revenue schedule overlaps existing schedule"
```

**Root Causes:**
1. Incorrect Service Period Start/End dates on Order Product
2. Revenue Recognition Rule schedule type mismatch
3. Contract term dates not aligned with revenue periods
4. Time zone issues affecting date calculations
5. Custom revenue schedule logic errors
6. Manual schedule overrides causing conflicts

**Resolution Steps:**

1. **Validate Service Period Dates**
   ```
   - Check OrderItem.ServiceDate (Start Date)
   - Verify OrderItem.EndDate (End Date)
   - Ensure dates span the full service delivery period
   - Confirm dates don't overlap with existing schedules
   ```

2. **Review Revenue Recognition Rule**
   ```
   Navigate to: Product > Revenue Recognition Rule
   - Schedule Type: Daily, Monthly, Quarterly, Annually, or Custom
   - Recognition Type: Over Time, Point in Time, or Milestone
   - Period Alignment: Start, Middle, or End of period
   - Day of Period: Specific day for monthly/quarterly recognition
   ```

3. **Check Contract/Order Dates**
   ```
   - Contract Start Date should match earliest service date
   - Contract End Date should match latest service date
   - Order EffectiveDate impacts revenue start
   - Order EndDate (if present) impacts revenue end
   ```

4. **Validate Revenue Schedule Records**
   ```
   Query: SELECT Id, ScheduleStartDate, ScheduleEndDate, RecognitionDate, 
                 Amount, Status 
          FROM RevenueScheduleLine 
          WHERE RevenueScheduleId = '[ScheduleId]'
          ORDER BY ScheduleStartDate
   
   Verify:
   - No gaps between schedule periods
   - Dates fall within contract/service period
   - Recognition dates align with business requirements
   ```

5. **Review Time Zone Settings**
   ```
   - Company Time Zone: Setup > Company Information
   - User Time Zone: User record > Time Zone
   - Revenue Processing Time Zone: DRO Settings
   - Ensure consistency across all settings
   ```

6. **Check Custom Logic**
   ```
   - Review any Apex triggers on RevenueSchedule objects
   - Check Flow/Process Builder date field updates
   - Validate formula fields used in date calculations
   - Test custom revenue allocation logic
   ```

**Verification Steps:**
1. Create test order with clear service period (e.g., 12 months)
2. Activate order and verify revenue schedule generation
3. Export revenue schedule lines to Excel
4. Validate date ranges cover entire service period
5. Confirm recognition dates align with accounting periods
6. Check sum of schedule amounts equals total order value

**Best Practices:**
- Standardize service period date entry (start of month, end of month)
- Use validation rules to ensure date consistency
- Document date calculation logic in revenue rules
- Set up alerts for schedule date anomalies
- Regular audits of revenue schedule date accuracy

---

### Pattern 3: Revenue Allocation Across Multiple Performance Obligations

**Symptoms:**
- Revenue not properly split across products/services
- Allocation percentages don't match expected values
- Some performance obligations show zero allocation
- Standalone Selling Price (SSP) calculations incorrect

**Common Error Messages:**
```
- "Unable to allocate revenue: Missing standalone selling price"
- "Allocation exceeds 100% of arrangement value"
- "Performance obligation allocation mismatch"
- "SSP not defined for product in multi-element arrangement"
```

**Root Causes:**
1. Missing or incorrect Standalone Selling Price (SSP) on products
2. Allocation method not configured correctly
3. Performance obligations not properly identified
4. Contract modifications affecting allocation
5. Variable consideration not factored in
6. Discount allocation logic issues

**Resolution Steps:**

1. **Verify Standalone Selling Price (SSP) Configuration**
   ```
   Navigate to: Product > Related > Standalone Selling Prices
   
   Check:
   - SSP defined for each product in the arrangement
   - SSP effective dates cover the transaction date
   - SSP amount is reasonable compared to list price
   - SSP percentages sum correctly for bundles
   
   Common SSP Methods:
   - Adjusted Market Assessment
   - Expected Cost Plus Margin
   - Residual Approach
   ```

2. **Review Performance Obligation Setup**
   ```
   Query: SELECT Id, Product2Id, AllocatedAmount, AllocationPercent,
                 StandaloneSellPrice, ActualPrice
          FROM PerformanceObligation
          WHERE RevenueArrangementId = '[RAId]'
   
   Validate:
   - One PO per distinct product/service
   - AllocatedAmount calculates correctly
   - AllocationPercent sums to 100%
   - SSP values are populated
   ```

3. **Check Revenue Allocation Method**
   ```
   Setup > Revenue Cloud Settings > Allocation Method
   
   Options:
   - Proportional: Allocates based on SSP ratios
   - Residual: Allocates excess to specific items
   - Relative: Uses relative fair value
   - Custom: Uses Apex class for allocation
   
   Verify method matches ASC 606/IFRS 15 requirements
   ```

4. **Validate Bundle and Discount Handling**
   ```
   For bundled products:
   - Check bundle configuration includes all components
   - Verify discount is allocated across all obligations
   - Confirm discount allocation method (proportional/specific)
   
   Query: SELECT TotalPrice, Discount, NetAmount
          FROM OrderItem
          WHERE OrderId = '[OrderId]' AND Product2.IsBundle = true
   ```

5. **Review Variable Consideration**
   ```
   If contract includes variable consideration (rebates, credits, refunds):
   - Check Variable Consideration Amount field
   - Verify constraint applied per accounting standards
   - Confirm allocation includes/excludes variable amount
   - Review probability weighting if using expected value method
   ```

6. **Test Allocation Calculation**
   ```
   Manual Calculation:
   1. Sum all SSPs: SSP_Total = SSP1 + SSP2 + SSP3...
   2. Calculate allocation percent: (SSP_Item / SSP_Total) × 100
   3. Apply percent to transaction price: TransPrice × AllocationPercent
   4. Verify calculated amounts match DRO allocated amounts
   
   Example:
   Product A SSP: $1,000 (SSP % = 40%)
   Product B SSP: $1,500 (SSP % = 60%)
   Transaction Price: $2,000
   
   Allocation A: $2,000 × 40% = $800
   Allocation B: $2,000 × 60% = $1,200
   ```

**Verification Steps:**
1. Create multi-product test order with known SSPs
2. Apply discount to test allocation logic
3. Verify allocated amounts per performance obligation
4. Compare allocation percentages to manual calculations
5. Check that total allocated amount equals order total
6. Generate allocation report and review for accuracy

**Best Practices:**
- Maintain current SSP data for all products
- Document SSP determination methodology
- Regular SSP reviews and updates (annually minimum)
- Use SSP ranges for similar products
- Audit allocation logic quarterly
- Train sales on ASC 606 implications of discounting

---

### Pattern 4: Deferred Revenue Discrepancies

**Symptoms:**
- Deferred revenue balance doesn't match expectations
- Revenue recognized immediately when should be deferred
- Unearned revenue not decreasing over time
- Month-end deferred revenue reports show inconsistencies

**Common Error Messages:**
```
- "Deferred revenue calculation error"
- "Revenue recognition date before contract start"
- "Unearned revenue balance mismatch"
- "Revenue schedule not found for deferred calculation"
```

**Root Causes:**
1. Revenue recognition rule set to "Point in Time" instead of "Over Time"
2. Service dates not properly configured
3. Revenue schedule generation not automated
4. Manual adjustments not properly recorded
5. Contract modifications not updating deferred balances
6. Integration issues with GL accounts

**Resolution Steps:**

1. **Verify Recognition Method**
   ```
   Navigate to: Revenue Recognition Rule
   
   For deferred revenue, must be:
   - Recognition Type: Over Time
   - Schedule Type: Daily, Monthly, or Quarterly (not Point in Time)
   
   Check Product configuration:
   - Revenue Recognition Rule assigned
   - Rule is active
   - Rule matches contract terms
   ```

2. **Check Deferred Revenue Account Mapping**
   ```
   Setup > Revenue Cloud Settings > Account Mapping
   
   Verify:
   - Deferred Revenue Account assigned
   - Revenue Recognition Account assigned
   - Accounts are active in Chart of Accounts
   - GL account codes are correct
   ```

3. **Review Revenue Schedule Status**
   ```
   Query: SELECT Id, Status, TotalRevenueAmount, RecognizedAmount,
                 DeferredAmount, RemainingAmount
          FROM RevenueSchedule
          WHERE RevenueArrangementId = '[RAId]'
   
   Calculate:
   - Deferred Amount = Total Revenue - Recognized Amount
   - Verify DeferredAmount field matches calculation
   - Check RemainingAmount decreases each period
   ```

4. **Validate Revenue Schedule Lines**
   ```
   Query: SELECT ScheduleDate, Amount, Status, RecognitionDate
          FROM RevenueScheduleLine
          WHERE RevenueScheduleId = '[ScheduleId]'
          AND Status IN ('Pending', 'Deferred')
          ORDER BY ScheduleDate
   
   Review:
   - Future-dated lines should be 'Deferred'
   - Past-dated lines should be 'Recognized'
   - Current period handling (recognized vs deferred)
   ```

5. **Check Revenue Recognition Job**
   ```
   Setup > Jobs > Scheduled Jobs > "Revenue Recognition Job"
   
   Verify:
   - Job is active and scheduled
   - Last run time is recent (daily expected)
   - No errors in job history
   - Job processes all eligible revenue schedules
   ```

6. **Reconcile Deferred Revenue Report**
   ```
   Run Report: Deferred Revenue Summary
   
   Reconciliation:
   1. Beginning Deferred Balance
   2. + New Deferrals (from new contracts)
   3. - Revenue Recognized (current period)
   4. +/- Adjustments (amendments, cancellations)
   5. = Ending Deferred Balance
   
   Compare to:
   - Revenue Schedule DeferredAmount sum
   - GL Deferred Revenue account balance
   ```

7. **Review Contract Modifications**
   ```
   For amended contracts:
   - Check if amendment created new performance obligations
   - Verify deferred revenue adjusted for contract changes
   - Confirm revenue reallocation if required
   - Review cumulative catch-up adjustment
   ```

**Verification Steps:**
1. Create test order with 12-month service period
2. Verify initial deferred revenue equals full order value
3. Run revenue recognition job
4. Check that recognized revenue equals 1/12 of total (monthly)
5. Verify deferred balance decreased by recognized amount
6. Repeat for multiple periods and confirm linear decrease

**Best Practices:**
- Automate revenue recognition job (daily recommended)
- Monthly reconciliation of deferred revenue to GL
- Document all manual deferred revenue adjustments
- Set up alerts for unusual deferred balance changes
- Regular review of aging deferred revenue
- Establish cutoff procedures for month-end close

---

### Pattern 5: ASC 606 / IFRS 15 Compliance Configuration

**Symptoms:**
- Revenue recognition not following 5-step model
- Contract modifications not handled per standard
- Performance obligation identification issues
- Transaction price allocation concerns
- Uncertainty about compliance implementation

**Common Error Messages:**
```
- "Contract modification type not specified"
- "Performance obligation not satisfied"
- "Transaction price adjustment required"
- "Constraint on variable consideration not applied"
```

**Root Causes:**
1. Incomplete understanding of ASC 606 requirements
2. System not configured for 5-step revenue model
3. Contract modification logic not implemented
4. Variable consideration not properly constrained
5. Lack of documentation for compliance
6. Audit trail gaps

**Resolution Steps:**

**Step 1: Identify the Contract with Customer**
```
Configuration:
- Define what constitutes a contract (Order, Agreement, Quote)
- Set contract approval requirements
- Configure contract combination rules for multiple documents
- Establish contract modification identification

In Salesforce:
- Use Contract or Order object
- Implement contract approval process
- Create fields: Contract Type, Modification Type, Parent Contract
- Build reports for contract analysis
```

**Step 2: Identify Performance Obligations**
```
Configuration:
- Define criteria for distinct goods/services
- Configure performance obligation object
- Set up obligation identification rules
- Establish series guidance for similar obligations

In Salesforce:
- Performance Obligation object per distinct promise
- Link to Product/Service delivered
- Define satisfaction criteria (point in time vs over time)
- Configure obligation status tracking

Distinct Goods/Services Test:
1. Capable of being distinct (customer can benefit)
2. Distinct within context of contract (separately identifiable)
```

**Step 3: Determine Transaction Price**
```
Configuration:
- Include: Fixed consideration + Variable consideration estimate
- Variable consideration: Rebates, refunds, credits, performance bonuses
- Constrain variable consideration (only include if probable)
- Consider financing component (time value of money)
- Non-cash consideration handling

In Salesforce:
- Transaction Price field on Order/Contract
- Variable Consideration field (estimated)
- Constraint Method: Expected Value or Most Likely Amount
- Financing Component calculation (if material)
- Non-cash consideration valuation

Variable Consideration Constraint:
- Include only if "highly probable" that significant reversal won't occur
- Reassess each reporting period
- Document constraint assessment
```

**Step 4: Allocate Transaction Price to Performance Obligations**
```
Configuration:
- Standalone Selling Price (SSP) determination
- Allocation method: Proportional to SSP
- Discount allocation (proportional or specific)
- Variable consideration allocation (if applies to specific PO)

In Salesforce:
- Configure SSP on Products
- Set Allocation Method in DRO Settings
- Populate AllocatedAmount on Performance Obligations
- Track allocation percentages
- Document allocation adjustments

SSP Hierarchy:
1. Observable price (standalone sale exists)
2. Adjusted market assessment approach
3. Expected cost plus margin approach
4. Residual approach (only if 1-3 not available)
```

**Step 5: Recognize Revenue When/As Obligations Satisfied**
```
Configuration:
- Point in time: Control transfers at specific date
- Over time: One of three criteria met
  a) Customer receives and consumes benefits as entity performs
  b) Entity's performance creates/enhances asset customer controls
  c) No alternative use + enforceable right to payment

In Salesforce:
- Revenue Recognition Rule: Point in Time or Over Time
- Schedule Type: Event-based, Time-based, or Milestone
- Recognition Pattern: Straight-line, Output method, Input method
- Configure Revenue Schedules
- Automate recognition job

Over Time Methods:
- Input methods: Costs incurred, labor hours, time elapsed
- Output methods: Units produced, milestones achieved
```

**Contract Modification Handling:**
```
Determine Modification Type:

Type 1: Separate Contract
- Distinct goods/services at SSP
- Account as new contract
- No impact to existing contract

Type 2: Termination of Old + Creation of New
- Not distinct goods/services, OR
- Distinct but not priced at SSP
- Reallocate remaining transaction price
- Adjust existing performance obligations

Type 3: Cumulative Catch-up
- Part of single contract
- Adjust current period for change
- No restatement of prior periods

In Salesforce:
- Amendment creates new Performance Obligations or adjusts existing
- Recalculate allocation across all POs
- Generate catch-up adjustment if needed
- Update revenue schedules prospectively
```

**Documentation Requirements:**
```
Maintain for Audit:
1. Contract identification policy
2. Performance obligation identification methodology
3. SSP determination methods and evidence
4. Transaction price calculation including variable consideration
5. Allocation methodology
6. Revenue recognition policy (point in time vs over time)
7. Contract modification assessment
8. Significant judgments and estimates

In Salesforce:
- Use Notes & Attachments on Revenue Arrangements
- Custom fields for compliance tracking
- Approval history for key decisions
- Change tracking on revenue objects
```

**Verification Steps:**
1. Select sample contracts and walk through 5-step model
2. Verify performance obligations identified correctly
3. Validate SSP and allocation calculations
4. Check recognition timing matches obligation satisfaction
5. Review contract modifications for proper accounting
6. Confirm documentation is complete and accessible

**Best Practices:**
- Document accounting policies aligned with ASC 606/IFRS 15
- Train sales and finance on revenue recognition impacts
- Implement contract review process for complex deals
- Regular compliance audits (quarterly minimum)
- External audit preparation checklist
- Stay current with FASB/IASB guidance updates
- Implement controls over key judgments and estimates

---

### Pattern 6: Revenue Waterfall and Distribution Logic Errors

**Symptoms:**
- Revenue amounts don't flow correctly across related records
- Parent-child revenue relationships broken
- Consolidated revenue totals incorrect
- Revenue not rolling up to contract level
- Product-level revenue doesn't sum to order total

**Common Error Messages:**
```
- "Revenue waterfall calculation failed"
- "Parent revenue arrangement not found"
- "Revenue distribution percentages exceed 100%"
- "Circular reference in revenue hierarchy"
```

**Root Causes:**
1. Revenue hierarchy not configured correctly
2. Parent-child relationships missing or broken
3. Roll-up summary fields not calculating
4. Distribution percentages don't total 100%
5. Custom revenue logic conflicts with standard waterfall
6. Data corruption in revenue arrangement records

**Resolution Steps:**

1. **Verify Revenue Hierarchy Structure**
   ```
   Revenue Waterfall Levels:
   Contract (Top)
     └─ Order
        └─ Order Product (Line Item)
           └─ Performance Obligation
              └─ Revenue Schedule
                 └─ Revenue Schedule Line
   
   Check parent-child relationships:
   - Order.ContractId
   - OrderItem.OrderId
   - PerformanceObligation.OrderProductId
   - RevenueSchedule.PerformanceObligationId
   - RevenueScheduleLine.RevenueScheduleId
   ```

2. **Validate Revenue Amount Flow**
   ```
   Query validation:
   
   -- Contract level
   SELECT Id, TotalAmount FROM Contract WHERE Id = '[ContractId]'
   
   -- Order level
   SELECT ContractId, SUM(TotalAmount) 
   FROM Order 
   WHERE ContractId = '[ContractId]'
   GROUP BY ContractId
   
   -- Order Product level
   SELECT OrderId, SUM(TotalPrice)
   FROM OrderItem
   WHERE OrderId = '[OrderId]'
   GROUP BY OrderId
   
   -- Performance Obligation level
   SELECT OrderProductId, SUM(AllocatedAmount)
   FROM PerformanceObligation
   WHERE RevenueArrangementId = '[RAId]'
   GROUP BY OrderProductId
   
   Verify totals match at each level
   ```

3. **Check Roll-up Summary Fields**
   ```
   Common roll-up fields:
   - Order.TotalAmount (sum of OrderItem.TotalPrice)
   - RevenueSchedule.TotalAmount (sum of lines)
   - Contract.TotalContractValue (sum of related orders)
   
   Troubleshooting:
   - Check if roll-up field exists
   - Verify calculation criteria
   - Look for "Calculating..." status
   - Check for record locking issues
   - Review roll-up field filters
   ```

4. **Validate Distribution Percentages**
   ```
   Query: SELECT Product2.Name, DistributionPercent,
                 AllocatedAmount, TotalArrangementAmount
          FROM PerformanceObligation
          WHERE RevenueArrangementId = '[RAId]'
   
   Verify:
   - SUM(DistributionPercent) = 100%
   - AllocatedAmount = TotalArrangementAmount × (DistributionPercent/100)
   - No null distribution percentages
   - No negative amounts
   ```

5. **Check for Circular References**
   ```
   Identify potential circular references:
   - Parent PO references child PO
   - Revenue arrangement references itself
   - Order product references another order product
   
   Query for loops:
   SELECT Id, Name, ParentPerformanceObligationId
   FROM PerformanceObligation
   WHERE ParentPerformanceObligationId != null
   
   Trace hierarchy to ensure no loops
   ```

6. **Review Custom Apex/Flows**
   ```
   Check for customizations affecting revenue flow:
   - Apex triggers on revenue objects
   - Flow/Process Builder updating amounts
   - Formula fields in calculation chain
   - Workflow rules modifying revenue
   
   Test by:
   - Temporarily disabling customizations
   - Creating test order without custom logic
   - Comparing results
   ```

**Verification Steps:**
1. Create simple test case (1 contract, 1 order, 2 products)
2. Trace revenue amounts from bottom to top:
   - Sum revenue schedule lines
   - Compare to revenue schedule total
   - Compare to performance obligation allocated amount
   - Compare to order product total price
   - Compare to order total
   - Compare to contract total
3. Verify all levels match
4. Repeat with complex scenario (multiple orders, products)

**Best Practices:**
- Maintain clean parent-child relationships
- Avoid manual revenue amount overrides
- Document any custom distribution logic
- Regular data integrity checks on revenue hierarchy
- Use validation rules to enforce distribution = 100%
- Implement record locking strategy during recognition
- Monitor and alert on orphaned revenue records

---

### Pattern 7: Integration with Orders, Contracts, and Billing

**Symptoms:**
- Revenue arrangements not created from orders
- Contract data not syncing to revenue objects
- Billing cycle misalignment with revenue recognition
- Invoice amounts don't match revenue amounts
- Data loss during integration handoffs

**Common Error Messages:**
```
- "Unable to create revenue arrangement from order"
- "Contract data sync failed"
- "Billing schedule does not match revenue schedule"
- "Integration user lacks revenue permissions"
- "API call limit exceeded during revenue sync"
```

**Root Causes:**
1. Integration flow not triggering correctly
2. Field mappings between objects incomplete
3. API user permissions insufficient
4. Timing issues between billing and revenue recognition
5. Data transformation errors
6. External system connectivity problems

**Resolution Steps:**

1. **Verify Order-to-Revenue Flow**
   ```
   Standard Flow: Order → Revenue Arrangement → Performance Obligations → Revenue Schedules
   
   Check activation:
   Setup > Process Automation > Flows > "Create Revenue Arrangement from Order"
   
   Verify:
   - Flow is Active
   - Entry Criteria: Order Status = 'Activated'
   - Actions create Revenue Arrangement
   - Error handling is configured
   
   Debug:
   - Check Debug Logs for flow execution
   - Review Flow error emails
   - Test with simple order
   ```

2. **Validate Field Mappings**
   ```
   Key field mappings:
   
   Order → Revenue Arrangement:
   - Order.TotalAmount → RevenueArrangement.Amount
   - Order.AccountId → RevenueArrangement.AccountId
   - Order.EffectiveDate → RevenueArrangement.StartDate
   - Order.EndDate → RevenueArrangement.EndDate
   
   Order Product → Performance Obligation:
   - OrderItem.Product2Id → PerformanceObligation.Product2Id
   - OrderItem.TotalPrice → PerformanceObligation.AllocatedAmount
   - OrderItem.ServiceDate → PerformanceObligation.StartDate
   - OrderItem.EndDate → PerformanceObligation.EndDate
   
   Check for:
   - Null values in source fields
   - Data type mismatches
   - Picklist value incompatibility
   - Required fields not mapped
   ```

3. **Review Integration User Permissions**
   ```
   Required permissions for integration user:
   
   Object Access (CRUD):
   - Order (Read)
   - OrderItem (Read)
   - Contract (Read)
   - RevenueArrangement (Read, Create, Edit)
   - PerformanceObligation (Read, Create, Edit)
   - RevenueSchedule (Read, Create, Edit)
   - RevenueScheduleLine (Read, Create, Edit)
   
   Permission Sets:
   - Revenue Cloud Integration User
   - API Enabled
   
   Field-Level Security:
   - Read access to all mapped fields
   - Edit access to target fields
   ```

4. **Align Billing and Revenue Schedules**
   ```
   Issue: Billing happens before/after revenue recognition
   
   Best Practice Alignment:
   - Billing Schedule Type matches Revenue Schedule Type
   - Billing Period = Revenue Recognition Period (monthly, quarterly)
   - Invoice Date ≈ Revenue Recognition Date (same day/period)
   
   Configuration:
   Navigate to: Billing > Billing Schedule
   - Check Billing Frequency
   - Review Billing Period Alignment
   - Verify Billing Start Date = Service Start Date
   
   Navigate to: Revenue Recognition Rule
   - Check Schedule Type
   - Review Recognition Frequency
   - Verify Recognition Start Date = Service Start Date
   
   Acceptable Patterns:
   1. Upfront Billing + Deferred Revenue (common for SaaS)
   2. Aligned Billing + Revenue (monthly in arrears)
   3. Milestone Billing + Milestone Revenue (project-based)
   ```

5. **Check External System Integration**
   ```
   For ERP/Accounting system integration:
   
   Verify connectivity:
   - Test connection to external system
   - Check API credentials are valid
   - Verify firewall/network access
   - Review SSL certificate validity
   
   Data sync validation:
   - Check Integration Log object for errors
   - Review sync frequency (real-time vs batch)
   - Validate data transformation rules
   - Confirm GL account code mappings
   
   Common integrations:
   - NetSuite
   - QuickBooks
   - SAP
   - Oracle Financials
   - Microsoft Dynamics
   ```

6. **Monitor API Usage**
   ```
   Issue: Integration hitting API limits
   
   Check limits:
   Setup > System Overview > API Usage
   
   Optimization strategies:
   - Bulk API for large data volumes
   - Composite API for related records
   - Reduce query complexity
   - Implement caching where possible
   - Schedule batch jobs during off-peak hours
   - Request API limit increase if justified
   ```

**Verification Steps:**
1. Create test order with all required fields
2. Activate order and monitor integration
3. Verify revenue arrangement created within SLA
4. Check all fields mapped correctly
5. Confirm billing schedule generated
6. Verify revenue schedule aligns with billing
7. Test end-to-end flow from order to GL

**Best Practices:**
- Document complete integration architecture
- Implement comprehensive error handling
- Set up monitoring and alerting for integration failures
- Regular testing of integration flows
- Maintain separate integration user account
- Keep API credentials secure and rotated
- Log all integration transactions for audit
- Implement retry logic for transient failures

---

### Pattern 8: Multi-Currency Revenue Recognition

**Symptoms:**
- Revenue amounts incorrect after currency conversion
- Exchange rate not applied correctly
- Multi-currency reports don't balance
- Dated exchange rates used instead of current rates
- Currency conversion timing issues

**Common Error Messages:**
```
- "Exchange rate not found for currency on date"
- "Corporate currency conversion failed"
- "Dated exchange rate not available"
- "Multi-currency calculation error"
```

**Root Causes:**
1. Multi-currency not enabled or configured
2. Exchange rates not defined for all currencies
3. Dated exchange rates missing for historical dates
4. Conversion rate type incorrect (spot vs average)
5. Currency on order doesn't match product currency
6. Corporate currency not set

**Resolution Steps:**

1. **Enable and Configure Multi-Currency**
   ```
   Setup > Company Information > Multi-Currency
   
   Verify:
   - Multi-Currency is enabled (cannot be disabled once enabled)
   - Corporate Currency is set (typically USD, EUR, GBP)
   - Active Currencies include all transactional currencies
   - Currency ISO Code is correct
   
   Note: Enabling multi-currency is permanent decision
   ```

2. **Maintain Exchange Rates**
   ```
   Setup > Manage Currencies > Manage Currency > Advanced Setup
   
   For each active currency:
   - Enter current exchange rate
   - Set Effective Date (start date for rate)
   - Use Next Start Date for rate changes
   - Maintain dated exchange rates for historical periods
   
   Best Practice:
   - Update rates regularly (daily for volatile currencies)
   - Maintain historical rates for past transactions
   - Document rate sources (Central Bank, Bloomberg, etc.)
   ```

3. **Configure Revenue Recognition Currency Handling**
   ```
   Setup > Revenue Cloud Settings > Multi-Currency
   
   Options:
   1. Transaction Currency: Recognize in original currency
      - Pros: Preserves original amounts
      - Cons: Requires conversion for reporting
   
   2. Corporate Currency: Convert at creation
      - Pros: Simplified reporting
      - Cons: Exchange gain/loss at creation
   
   3. Dual Currency: Maintain both
      - Pros: Best of both worlds
      - Cons: Additional data storage
   
   Recommendation: Dual currency for ASC 606 compliance
   ```

4. **Validate Currency on Revenue Objects**
   ```
   Check currency consistency:
   
   Query: SELECT Id, CurrencyIsoCode, Amount, 
                 ConvertedAmount, ConversionRate
          FROM RevenueArrangement
          WHERE Id = '[RAId]'
   
   Verify:
   - CurrencyIsoCode matches Order currency
   - ConversionRate is populated and reasonable
   - ConvertedAmount = Amount × ConversionRate
   
   For multi-currency orders:
   - All line items should have same currency as order
   - Performance obligations inherit currency
   - Revenue schedules use same currency
   ```

5. **Check Exchange Rate Application Timing**
   ```
   Determine when exchange rate is applied:
   
   Options:
   a) Transaction Date (order date)
      - Rate on date order was created
      - Fixed rate for life of arrangement
   
   b) Recognition Date (revenue recognition date)
      - Rate on date revenue is recognized
      - Rate changes each period (gain/loss recognized)
   
   c) Month-End Rate (accounting period close)
      - Rate on last day of accounting period
      - Used for financial reporting
   
   Configure in: Revenue Recognition Rule > Currency Settings
   ```

6. **Review Multi-Currency Reports**
   ```
   Key Reports:
   - Revenue by Currency Report
   - Exchange Gain/Loss Report
   - Multi-Currency Revenue Schedule
   - Corporate Currency Conversion Report
   
   Validation:
   - Sum of all currency revenues (converted) = Corporate currency total
   - Exchange rate matches defined rates
   - No null currency values
   - Historical data uses correct dated rates
   ```

7. **Handle Exchange Gain/Loss**
   ```
   For revenue recognized over time in foreign currency:
   
   Calculate monthly:
   1. Revenue in transaction currency (fixed)
   2. Convert using current month's rate
   3. Compare to prior month's converted amount
   4. Difference = Exchange gain/loss
   
   Accounting treatment:
   - Record exchange gain/loss separately
   - Don't adjust original revenue amount
   - Classify as financial income/expense
   
   In Salesforce:
   - ExchangeGainLoss field on RevenueScheduleLine
   - Separate GL account for FX gain/loss
   ```

**Verification Steps:**
1. Create test order in foreign currency (e.g., EUR)
2. Verify exchange rate applied correctly
3. Check revenue arrangement shows both currencies
4. Generate revenue schedule and verify currency
5. Convert to corporate currency and validate
6. Update exchange rate and check impact
7. Run multi-currency report and confirm totals

**Best Practices:**
- Document currency conversion policies
- Automate exchange rate updates via API
- Monitor exchange rate volatility
- Hedge significant foreign currency exposure
- Regular reconciliation of multi-currency revenue
- Train users on currency selection at order entry
- Set up alerts for missing exchange rates
- Maintain audit trail of rate changes

---

### Pattern 9: Revenue Restatement and Historical Corrections

**Symptoms:**
- Need to correct revenue recognized in prior periods
- Historical data errors discovered during audit
- Contract modifications require retrospective adjustment
- Revenue previously recognized needs to be deferred
- Prior period revenue allocation was incorrect

**Common Error Messages:**
```
- "Cannot modify closed revenue period"
- "Revenue schedule is locked"
- "Historical adjustment not permitted"
- "Restatement approval required"
```

**Root Causes:**
1. Errors in original revenue recognition
2. Contract terms misinterpreted
3. Performance obligations incorrectly identified
4. Allocation errors discovered later
5. Accounting policy changes
6. System implementation issues
7. Contract modifications requiring retrospective treatment

**Resolution Steps:**

1. **Assess Restatement Type and Scope**
   ```
   Types of Restatements:
   
   Type 1: Error Correction (ASC 250)
   - Mathematical errors
   - Misapplication of accounting standards
   - Oversight or misuse of facts
   → Requires prior period adjustment
   
   Type 2: Change in Estimate (ASC 250)
   - New information about uncertain events
   - Change in transaction price estimate
   - Variable consideration adjustment
   → Prospective or cumulative catch-up
   
   Type 3: Change in Accounting Policy (ASC 250)
   - Voluntary policy change
   - New accounting standard adoption
   - Change in recognition method
   → Retrospective or cumulative effect
   
   Determine materiality:
   - Impact on revenue > 5% = Material
   - Impact on net income > 10% = Material
   - Qualitative factors (fraud, misstatement)
   ```

2. **Document Restatement Requirements**
   ```
   Required documentation:
   
   1. Restatement Memo
      - Description of error/change
      - Periods affected
      - Quantification of impact
      - Root cause analysis
      - Corrective actions
   
   2. Accounting Analysis
      - Applicable accounting guidance (ASC 606, ASC 250)
      - Treatment determination (retrospective/prospective)
      - Disclosure requirements
      - Financial statement impact
   
   3. Approvals
      - CFO approval
      - Controller approval
      - Audit Committee (if material)
      - External auditor notification
   
   In Salesforce:
   - Create Restatement Case record
   - Attach documentation
   - Route for approvals
   - Track status
   ```

3. **Calculate Restatement Amounts**
   ```
   Retrospective Restatement Calculation:
   
   For each affected period:
   1. Original revenue recognized
   2. Correct revenue (recalculated)
   3. Difference = Adjustment amount
   4. Cumulative impact on retained earnings
   
   Example:
   Q1 2025: Originally $100K, Should be $80K → Reduce by $20K
   Q2 2025: Originally $100K, Should be $80K → Reduce by $20K
   Q3 2025: Originally $100K, Should be $80K → Reduce by $20K
   Q4 2025: Originally $100K, Should be $120K → Increase by $20K
   
   Cumulative catch-up in Q1 2026: -$40K
   
   Query for affected records:
   SELECT Id, RecognitionDate, Amount, Status
   FROM RevenueScheduleLine
   WHERE RevenueScheduleId IN (SELECT Id FROM RevenueSchedule 
                                WHERE RevenueArrangementId = '[RAId]')
   AND RecognitionDate < [RestatementDate]
   ```

4. **Unlock Revenue Periods for Adjustment**
   ```
   Standard process:
   
   Setup > Revenue Cloud Settings > Period Management
   
   1. Identify closed periods requiring adjustment
   2. Submit unlock request (approval required)
   3. Unlock period temporarily
   4. Make corrections
   5. Re-close period
   6. Lock period again
   
   Alternative (preferred):
   - Don't unlock old periods
   - Create adjustment entry in current period
   - Book cumulative catch-up adjustment
   - Maintain audit trail of adjustment
   ```

5. **Execute Restatement in Salesforce**
   ```
   Option A: Recreate Revenue Schedules
   1. Inactivate incorrect revenue arrangement
   2. Create new revenue arrangement with correct data
   3. Allocate correctly from inception
   4. Generate corrected revenue schedules
   5. Book cumulative adjustment for past periods
   
   Option B: Adjust Existing Schedules
   1. Update performance obligation allocations
   2. Recalculate revenue schedule lines
   3. Create adjustment lines for differences
   4. Update recognized amounts
   5. Maintain link to original records
   
   Option C: Journal Entry Adjustment (External)
   1. Don't modify Salesforce revenue records
   2. Create manual journal entry in GL
   3. Record as "Revenue Restatement Adjustment"
   4. Document link to Salesforce records
   5. Update reports to reflect adjustment
   
   Recommendation: Option C for closed periods, Option B for current period
   ```

6. **Update Financial Statements**
   ```
   For material restatements:
   
   1. Restate Prior Period Financials
      - Income Statement: Adjust revenue lines
      - Balance Sheet: Adjust deferred revenue, receivables
      - Cash Flow: Typically no change
      - Statement of Changes in Equity: Adjust retained earnings
   
   2. Disclosure Requirements
      - Nature of error/change
      - Impact on each financial statement line
      - Impact on each prior period presented
      - Cumulative effect on retained earnings
   
   3. In Salesforce Reports
      - Update custom reports with adjusted amounts
      - Add filter for "Excluding Restated" vs "Including Restated"
      - Create restatement impact report
   ```

7. **Prevent Future Restatements**
   ```
   Implement controls:
   
   1. Enhanced Review Process
      - Management review of all revenue arrangements >$X
      - Accounting review of complex deals
      - Legal review of contract terms
      - Multi-step approval for non-standard terms
   
   2. System Controls
      - Validation rules on key fields
      - Workflow alerts for unusual scenarios
      - Automated compliance checks
      - Prevent backdating without approval
   
   3. Training and Communication
      - Train sales on revenue implications
      - Finance training on ASC 606
      - Regular contract review meetings
      - Documentation of complex scenarios
   
   4. Monitoring and Detection
      - Monthly revenue analytics
      - Unusual pattern detection
      - Quarterly revenue audits
      - External audit readiness checks
   ```

**Verification Steps:**
1. Recalculate revenue for affected periods manually
2. Compare to system-calculated amounts
3. Verify adjustment amounts are correct
4. Check that cumulative adjustment balances
5. Confirm all affected records updated
6. Review reports show corrected amounts
7. Validate audit trail is complete

**Best Practices:**
- Document all judgments and estimates contemporaneously
- Implement robust contract review process
- Regular revenue accounting training
- Quarterly self-audits of revenue recognition
- Maintain detailed workpapers for complex deals
- Proactive communication with auditors
- Learn from restatements - update policies/controls
- Consider revenue recognition software for complex scenarios

---

### Pattern 10: Performance Obligation Configuration and Tracking

**Symptoms:**
- Performance obligations not auto-generated
- Obligation status not updating correctly
- Can't determine when obligation is satisfied
- Multiple obligations for single product
- Obligation fulfillment tracking issues

**Common Error Messages:**
```
- "Performance obligation not satisfied"
- "Unable to determine obligation status"
- "Performance obligation missing required fields"
- "Duplicate performance obligation detected"
```

**Root Causes:**
1. Performance obligation criteria not defined
2. Satisfaction method not configured
3. Status update automation not working
4. Distinct goods/services not properly identified
5. Series guidance not applied correctly
6. Manual tracking incomplete

**Resolution Steps:**

1. **Define Performance Obligation Identification Criteria**
   ```
   ASC 606 Distinct Criteria:
   
   A good/service is distinct if BOTH:
   1. Capable of being distinct
      - Customer can benefit from it separately
      - Available separately in the market
      - Customer could use with own resources
   
   2. Distinct within context of contract
      - Separately identifiable from other promises
      - Not highly integrated with other promises
      - Not significantly modified by other promises
   
   In Salesforce Configuration:
   Product Object Fields:
   - Is_Distinct__c (checkbox)
   - Performance_Obligation_Type__c (picklist)
   - Satisfaction_Criteria__c (text)
   
   Automations:
   - Flow: Auto-create PO when OrderItem created
   - Entry Criteria: Product2.Is_Distinct__c = TRUE
   - Action: Create PerformanceObligation record
   ```

2. **Configure Satisfaction Method**
   ```
   Satisfaction Methods:
   
   Point in Time:
   - Delivery/shipment of goods
   - Completion of service
   - Transfer of control indicators:
     * Customer has physical possession
     * Customer has legal title
     * Customer has risks/rewards of ownership
     * Customer has accepted the asset
   
   Over Time (one of three):
   - Customer receives/consumes benefit as performed
   - Customer controls asset as created/enhanced
   - No alternative use + right to payment for progress
   
   In Salesforce:
   Performance Obligation Fields:
   - Satisfaction_Method__c: 'Point in Time' or 'Over Time'
   - Satisfaction_Criteria__c: Specific criteria
   - Satisfaction_Evidence__c: How to verify
   - Satisfaction_Date__c: When satisfied (point in time)
   - Percent_Complete__c: Progress (over time)
   ```

3. **Set Up Status Tracking**
   ```
   Performance Obligation Status Values:
   - Not Started: Created but service not begun
   - In Progress: Service delivery ongoing
   - Satisfied: Obligation complete
   - Cancelled: No longer required
   - On Hold: Temporarily paused
   
   Status Update Triggers:
   
   Not Started → In Progress:
   - Service Start Date reached
   - First milestone achieved
   - Customer notification sent
   
   In Progress → Satisfied:
   - Service End Date reached
   - All milestones complete
   - Customer acceptance received
   - 100% complete for over-time recognition
   
   Automation:
   - Scheduled Job: Daily status check
   - Update status based on dates and milestones
   - Generate alerts for stuck obligations
   ```

4. **Implement Series Guidance**
   ```
   Series Guidance (ASC 606-10-25-15):
   
   Treat multiple distinct goods/services as single PO if:
   1. Each distinct good/service is substantially the same
   2. Same pattern of transfer to customer
   
   Examples:
   - Monthly SaaS subscription (12 months) = 1 PO
   - Weekly cleaning services (52 weeks) = 1 PO
   - Daily data feeds (365 days) = 1 PO
   
   In Salesforce:
   - Don't create separate PO for each period
   - Create one PO for entire series
   - Use Revenue Schedule for period breakdown
   - Single satisfaction tracking for series
   
   Product Configuration:
   - Apply_Series_Guidance__c (checkbox)
   - If checked: Create 1 PO regardless of quantity/periods
   ```

5. **Track Obligation Fulfillment**
   ```
   For over-time performance obligations:
   
   Input Methods:
   - Costs incurred
   - Labor hours expended
   - Time elapsed
   - Resources consumed
   
   Output Methods:
   - Units produced/delivered
   - Milestones achieved
   - Value delivered to customer
   
   In Salesforce:
   Custom Objects:
   - Milestone__c: Tracks project milestones
   - Deliverable__c: Tracks deliverables
   - Time_Entry__c: Tracks labor hours
   
   Calculation:
   Percent_Complete__c = 
     (Actual_Input_or_Output / Total_Estimated_Input_or_Output) × 100
   
   Update Obligation Status based on Percent_Complete
   ```

6. **Handle Multiple Obligations Per Product**
   ```
   Scenarios requiring multiple POs:
   
   1. Bundled Products with Distinct Components
      Example: Hardware + Software + Support
      → Create 3 separate performance obligations
   
   2. Warranty Obligations
      Example: Product + Extended Warranty
      → Create 2 POs (product + warranty)
   
   3. Implementation + Subscription
      Example: Setup fee + Monthly SaaS
      → Create 2 POs (different satisfaction timing)
   
   Configuration:
   - Bundle Product with Components
   - Each component = distinct PO
   - Allocate transaction price across all POs
   - Different revenue recognition rules per PO
   ```

**Verification Steps:**
1. Create test order with various product types
2. Verify correct number of POs created
3. Check satisfaction method is appropriate
4. Update dates/milestones and verify status changes
5. Confirm series guidance applied where appropriate
6. Validate revenue recognition follows PO status
7. Test complete obligation lifecycle

**Best Practices:**
- Document performance obligation policy
- Train sales on distinct goods/services identification
- Regular review of PO identification for new products
- Automate status updates where possible
- Maintain evidence of obligation satisfaction
- Use milestones for complex projects
- Regular reconciliation of POs to revenue schedules
- Clear handoff process from sales to delivery teams

---

## C. Key Objects & Diagnostic Queries

### DRO Object Model

### RevenueArrangement
- Purpose: Top-level container for revenue recognition
- Key Fields: AccountId, Amount, Status, StartDate, EndDate
- Relationships: Related to Order, Contract

### PerformanceObligation
- Purpose: Represents distinct good/service promise
- Key Fields: Product2Id, AllocatedAmount, Status, StartDate, EndDate
- Relationships: Child of RevenueArrangement

### RevenueSchedule
- Purpose: Holds revenue recognition schedule
- Key Fields: TotalAmount, RecognizedAmount, DeferredAmount, Status
- Relationships: Child of PerformanceObligation

### RevenueScheduleLine
- Purpose: Individual revenue recognition entry
- Key Fields: ScheduleDate, Amount, Status, RecognitionDate
- Relationships: Child of RevenueSchedule

### RevenueRecognitionRule
- Purpose: Defines how/when revenue is recognized
- Key Fields: ScheduleType, RecognitionType, Period Alignment
- Relationships: Assigned to Products

### StandaloneSellingPrice (SSP)
- Purpose: Fair value for allocation purposes
- Key Fields: Product2Id, Amount, EffectiveDate, EndDate
- Relationships: Associated with Products

---

## Diagnostic Queries

### Check Revenue Arrangement Status
```sql
SELECT Id, Name, Status, Amount, RecognizedAmount, 
       DeferredAmount, OrderId, AccountId
FROM RevenueArrangement
WHERE OrderId = '[OrderId]'
```

### Review Performance Obligations
```sql
SELECT Id, Product2.Name, AllocatedAmount, AllocationPercent,
       Status, StartDate, EndDate
FROM PerformanceObligation
WHERE RevenueArrangementId = '[RAId]'
```

### Analyze Revenue Schedules
```sql
SELECT Id, TotalRevenueAmount, RecognizedAmount, 
       RemainingAmount, DeferredAmount, Status
FROM RevenueSchedule
WHERE PerformanceObligationId IN 
  (SELECT Id FROM PerformanceObligation 
   WHERE RevenueArrangementId = '[RAId]')
```

### Examine Schedule Lines
```sql
SELECT ScheduleDate, RecognitionDate, Amount, Status
FROM RevenueScheduleLine
WHERE RevenueScheduleId = '[ScheduleId]'
ORDER BY ScheduleDate
```

### Find Missing SSP
```sql
SELECT Id, Name, (SELECT Id FROM StandaloneSellingPrices)
FROM Product2
WHERE IsActive = true
AND Id IN (SELECT Product2Id FROM OrderItem WHERE OrderId = '[OrderId]')
```

### Check Revenue Recognition Jobs
```sql
SELECT Id, ApexClass.Name, Status, JobType, 
       CompletedDate, NumberOfErrors
FROM AsyncApexJob
WHERE JobType IN ('ScheduledApex', 'BatchApex')
AND ApexClass.Name LIKE '%Revenue%'
ORDER BY CompletedDate DESC
LIMIT 10
```

---

## D. Escalation

- Slack: `#support-rev-dev-amer`, `#support-rev-rlm-global-swarm-help`, `#rlm-office-hours`
- GUS product tag: `Revenue Cloud`

Escalate when:
- Confirmed product bug in revenue calculation engine
- Data corruption in revenue arrangements
- Performance problems with revenue recognition batch jobs
- Complex multi-currency/multi-element allocation beyond standard config