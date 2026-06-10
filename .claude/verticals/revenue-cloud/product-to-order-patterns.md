## A. Triage & Classification

When a support engineer brings a Product to Order issue, first classify it:

```
Is the issue about...

├── 1. ORDER PRODUCT CREATION ERRORS
│   ├── "Order Product Group ID: id value of incorrect type"
│   ├── Order Product not created from Quote Line Item
│   ├── Order creation fails during quote acceptance
│   ├── Duplicate Order Products created
│   └── Order Product missing required fields
│   → Go to: Pattern 1 (Order Product Creation)
│
├── 2. PRODUCT BUNDLE & HIERARCHY
│   ├── Nested bundle configuration errors
│   ├── Parent-child product relationships broken
│   ├── Bundle components not appearing
│   ├── Product hierarchy not respected in orders
│   └── Sub-bundles not expanding correctly
│   → Go to: Pattern 2 (Product Bundles)
│
├── 3. PRODUCT CONFIGURATION RULES
│   ├── Configuration rules not firing
│   ├── Product validation rules blocking selection
│   ├── Incompatible product selection allowed
│   ├── Required product not enforced
│   └── Configuration attribute issues
│   → Go to: Pattern 3 (Configuration Rules)
│
├── 4. QUOTE TO ORDER CONVERSION
│   ├── "Create Order" button fails
│   ├── Order created but incomplete
│   ├── Quote Line Items not mapping to Order Products
│   ├── Pricing not transferring to Order
│   └── Order activation fails
│   → Go to: Pattern 4 (Quote to Order)
│
├── 5. PRODUCT ATTRIBUTES & SPECIFICATIONS
│   ├── Product attributes not populating
│   ├── Custom attributes missing on Order Products
│   ├── Attribute values not mapping correctly
│   ├── Specification field errors
│   └── Attribute inheritance issues
│   → Go to: Pattern 5 (Product Attributes)
│
├── 6. PRICING & PRICE BOOKS
│   ├── Price book not applied to product
│   ├── Product not in price book
│   ├── Price not found for product
│   ├── Incorrect price pulled for product
│   └── Multi-currency pricing issues
│   → Go to: Pattern 6 (Pricing/Price Books)
│
├── 7. PRODUCT SELECTION IN CART/TLE
│   ├── Products not appearing in catalog
│   ├── Transaction Line Editor (TLE) errors
│   ├── Product search not returning results
│   ├── Cart not loading products
│   └── Product sorting/ordering issues
│   → Go to: Pattern 7 (Product Selection)
│
├── 8. PRODUCT AVAILABILITY & INVENTORY
│   ├── Product marked as unavailable
│   ├── Out of stock errors
│   ├── Product visibility rules
│   ├── Product effective dating issues
│   └── Inventory sync problems
│   → Go to: Pattern 8 (Availability)
│
└── 9. ORDER ACTIVATION ISSUES
    ├── Order stuck in Draft status
    ├── Activation errors
    ├── Fulfillment not triggered
    ├── Asset creation fails during activation
    └── Billing not triggered after activation
    → Go to: Pattern 9 (Order Activation)
```

### Quick Diagnostic Questions

1. **What is the exact error message?** Full text, including error codes
2. **Which step fails?** Product selection, quote creation, order creation, activation?
3. **Product type?** Simple product, bundle, configurable product?
4. **When did this start?** After a release/deployment/configuration change?
5. **Which user/profile?** System Admin or specific user?
6. **Production or Sandbox?**
7. **Can you reproduce with a different product?** Helps isolate product-specific issues

---

## B. Known Issue Patterns

### Pattern 1: Order Product Creation Errors

**Frequency:** High (based on OrgCS patterns)  
**Severity:** Priority 26-45 (Critical to High)

**Symptoms:**
- Error: "Order Product Group ID: id value of incorrect type"
- Order Products not created when order is created from quote
- Some Quote Line Items missing from Order Products
- Duplicate Order Products created
- Order Product required fields blank

**Common Error Messages:**
```
"Order Product Group ID: id value of incorrect type"
"REQUIRED_FIELD_MISSING: Required fields are missing: [OrderId]"
"Insert failed. First exception on row 0; first error: INVALID_ID_FIELD"
"Order Product creation failed: Field validation error"
"INSUFFICIENT_ACCESS_ON_CROSS_REFERENCE_ENTITY: insufficient access rights on Order"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Nested bundle configuration error causing invalid Order Product Group reference | Check if product is nested bundle (bundle within bundle) |
| 2 | Order Product mapping missing required fields from Quote Line Item | Query OrderProduct and QuoteLineItem to compare fields |
| 3 | Workflow/trigger modifying Order Product during creation | Check Workflow Rules, Process Builder, Flows on OrderProduct |
| 4 | Custom Order creation logic not setting required fields | Review custom Apex classes that create orders |
| 5 | Permission issue - user cannot create Order Products | Check user's profile/permission set on OrderProduct object |

**Resolution Steps:**

1. **Check for Nested Bundle Issues:**
   ```sql
   -- Query the product structure
   SELECT Id, Name, IsBundle__c, ParentProductId__c, ProductLevel__c
   FROM Product2
   WHERE Id IN (SELECT Product2Id FROM QuoteLineItem WHERE QuoteId = '<quote_id>')
   
   -- Check if any products are nested bundles (Level > 1)
   -- Nested bundles: Bundle inside another bundle
   ```
   - **Issue:** Ramp schedules with nested bundles create invalid Order Product Group references
   - **Workaround:** Flatten bundle structure or avoid nested bundles with ramp schedules
   - **Fix:** Reconfigure product hierarchy to avoid nesting

2. **Validate Order Product Mapping:**
   ```sql
   -- Check Quote Line Items
   SELECT Id, Product2Id, Quantity, TotalPrice, ServiceDate, EndDate,
          OrderProductId__c
   FROM QuoteLineItem
   WHERE QuoteId = '<quote_id>'
   
   -- Check corresponding Order Products (if created)
   SELECT Id, Product2Id, Quantity, TotalPrice, ServiceDate, EndDate,
          QuoteLineItemId
   FROM OrderProduct
   WHERE OrderId = '<order_id>'
   ```
   - Compare counts: QLI count should = Order Product count
   - Verify Product2Id matches
   - Check required fields populated on both

3. **Check Order Creation Configuration:**
   - Navigate to: Setup → Order Settings
   - Verify "Enable Orders" is checked
   - Verify "Enable Reduction Orders" if using amendments
   - Check if custom Order creation flow is active

4. **Review Automation:**
   ```
   Setup → Flows → Filter by "Order" or "OrderProduct"
   - Check for record-triggered flows on OrderProduct
   - Review entry criteria and actions
   - Look for field updates that might cause validation errors
   
   Setup → Process Builder → Filter by OrderProduct object
   - Check for processes that fire on creation
   - Verify criteria don't conflict
   ```

5. **Verify Permissions:**
   ```sql
   SELECT Parent.Profile.Name, SobjectType, PermissionsCreate
   FROM ObjectPermissions
   WHERE SobjectType = 'OrderProduct' AND PermissionsCreate = true
   ```
   - Ensure user has Create permission on OrderProduct
   - Check field-level security on required fields

6. **Test with Simple Product:**
   - Create test quote with single simple product (not bundle)
   - Accept quote → Create order
   - If works: Issue is product-specific (likely bundle configuration)
   - If fails: Broader configuration or permission issue

**Verification:**
- Order created successfully from quote
- All Quote Line Items have corresponding Order Products
- Order Product fields match Quote Line Item fields
- No duplicate Order Products
- Order Product Group references valid (for bundles)

**Known Workarounds:**
- For nested bundles: Flatten structure to single-level bundles
- For ramp schedules: Create manually instead of using "Create Ramp Schedule" with nested bundles
- For custom flows: Deactivate temporarily to isolate if causing issue

**Escalation Criteria:**
- Nested bundle error persists after flattening → Escalate to Revenue Cloud Engineering
- Order creation works in sandbox but not production → Escalate with org comparison
- Permission set assigned but still access error → Escalate with debug logs

**Sample Case Numbers:** 473505624

---

### Pattern 2: Product Bundle & Hierarchy Issues

**Frequency:** Medium-High  
**Severity:** Priority 13-45

**Symptoms:**
- Bundle components not appearing in cart/order
- Parent product shows but child products missing
- Product hierarchy not respected during configuration
- Bundle pricing incorrect (components priced individually)
- Unable to expand bundle to see components

**Common Error Messages:**
```
"Bundle components not found"
"Product hierarchy configuration error"
"Parent product required for bundle component"
"Bundle configuration incomplete"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Product Bundle Option records missing or inactive | Query ProductBundleOption for the bundle |
| 2 | Parent-child relationship not configured correctly | Check Product2.ParentProduct2Id field |
| 3 | Bundle pricing method misconfigured | Review Product2.BundlePricingType__c |
| 4 | Component products inactive or out of date range | Check Product2.IsActive and EffectiveDate/ExpirationDate |

**Resolution Steps:**

1. **Verify Bundle Configuration:**
   ```sql
   -- Check if product is configured as bundle
   SELECT Id, Name, IsBundle__c, BundlePricingType__c
   FROM Product2
   WHERE Id = '<product_id>'
   
   -- Check bundle components
   SELECT Id, Product2Id, BundleProduct2Id, Quantity, IsRequired,
          SortOrder, IsActive
   FROM ProductBundleOption
   WHERE BundleProduct2Id = '<bundle_product_id>'
   ORDER BY SortOrder
   ```

2. **Validate Component Products:**
   ```sql
   -- Get all component products and check status
   SELECT Id, Name, IsActive, EffectiveDate, ExpirationDate, IsDeleted
   FROM Product2
   WHERE Id IN (
     SELECT Product2Id FROM ProductBundleOption 
     WHERE BundleProduct2Id = '<bundle_id>'
   )
   ```
   - All components should be Active
   - Check effective/expiration dates are valid for current date
   - Verify none are deleted

3. **Check Bundle Pricing Configuration:**
   - **Fixed Pricing:** Bundle has single price, components inherit
   - **Component Pricing:** Each component priced separately, sum = bundle price
   - **Both:** Bundle base price + component prices
   
   Verify Product2.BundlePricingType__c matches intended method

4. **Review Product Hierarchy:**
   ```sql
   -- Map full product hierarchy
   SELECT Id, Name, ParentProduct2Id, Level__c
   FROM Product2
   WHERE Id = '<product_id>' 
   OR ParentProduct2Id = '<product_id>'
   ```
   - Maximum recommended depth: 2-3 levels
   - Nested bundles (bundle inside bundle) can cause issues

5. **Test Bundle Selection:**
   - In Transaction Line Editor: Add bundle product
   - Verify "Expand Bundle" option appears
   - Click expand → Components should appear
   - Verify quantities and pricing correct

**Verification:**
- Bundle displays all components
- Pricing calculates correctly based on bundle pricing type
- Component quantities respect configuration
- Required components cannot be removed

**Best Practices:**
- Keep bundle hierarchy shallow (max 2-3 levels)
- Always set SortOrder on ProductBundleOption for consistent display
- Mark required components as IsRequired = true
- Test bundle in sandbox before deploying to production

---

### Pattern 3: Product Configuration Rules Issues

**Frequency:** Medium  
**Severity:** Priority 31-45

**Symptoms:**
- Configuration rules not enforcing constraints
- Incompatible products can be selected together
- Required product not auto-added
- Validation rule not blocking invalid selection
- Error: "Revenue Cloud Product Configuration Rules issue"

**Common Error Messages:**
```
"Product configuration rule validation failed"
"Required product dependency not satisfied"
"Incompatible product combination"
"Configuration constraint violated"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Configuration rule not activated | Check rule status in Setup |
| 2 | Rule criteria not matching product selection | Review rule conditions |
| 3 | Rule execution order incorrect | Check rule priority/sequence |
| 4 | Custom product selection logic overriding rules | Review Apex classes with product logic |

**Resolution Steps:**

1. **Locate Configuration Rules:**
   - Navigate to: Setup → Product Configuration Rules
   - Or: Setup → Industries → Product Configuration
   - Verify rules exist for the product family/category

2. **Check Rule Status:**
   ```sql
   -- Query configuration rules (object name varies by implementation)
   SELECT Id, Name, Status, IsActive, RuleType, Priority
   FROM ProductConfigurationRule__c
   WHERE Product2Id = '<product_id>' OR ProductFamily = '<family>'
   ```
   - Verify Status = Active
   - Check Priority (lower number = higher priority = executes first)

3. **Validate Rule Logic:**
   - Open each rule → Review conditions
   - **Validation Rule:** Blocks invalid combinations
   - **Selection Rule:** Auto-selects related products
   - **Filter Rule:** Filters product catalog based on context
   - Verify conditions match your use case

4. **Test Rule Execution:**
   - Create test cart/quote
   - Add product that should trigger rule
   - Verify rule fires (check debug logs if needed)
   - Expected outcomes:
     - Validation rule → Error message displayed
     - Selection rule → Related product auto-added
     - Filter rule → Product list filtered

5. **Debug Rule Failures:**
   ```
   - Enable debug logs for user
   - Reproduce product selection
   - Search logs for: "ProductConfiguration", "Rule", "Validation"
   - Common issues:
     * Rule criteria don't match (e.g., ProductFamily mismatch)
     * Field API names wrong in rule conditions
     * Custom code bypassing rules
   ```

**Verification:**
- Validation rules block invalid product combinations
- Selection rules auto-add required products
- Filter rules show only applicable products
- Rules execute in correct order

**Sample Case Numbers:** 473381112

---

### Pattern 4: Quote to Order Conversion Failures

**Frequency:** High  
**Severity:** Priority 0-45 (Varies)

**Symptoms:**
- "Create Order" button does nothing
- Order created but empty (no Order Products)
- Quote Line Items not mapping to Order Products
- Order created in Draft but cannot activate
- Pricing lost during conversion

**Common Error Messages:**
```
"Failed to create order from quote"
"Order creation service unavailable"
"Quote must be accepted before creating order"
"Unable to map quote line items to order products"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Quote not in "Accepted" status | Check Quote.Status field |
| 2 | Missing required fields on Quote or Quote Line Items | Review required field validation |
| 3 | Order creation flow/trigger failing | Check Flow error logs |
| 4 | Custom order creation logic error | Review Apex logs for exceptions |
| 5 | Account on Quote missing required order fields | Verify Account has billing/shipping address |

**Resolution Steps:**

1. **Verify Quote Status:**
   ```sql
   SELECT Id, Name, Status, IsSyncing, TotalPrice, OpportunityId, AccountId
   FROM Quote
   WHERE Id = '<quote_id>'
   ```
   - Status must be "Accepted" to create order
   - IsSyncing should be false
   - Account and Opportunity should be populated

2. **Check Quote Line Items:**
   ```sql
   SELECT Id, Product2Id, Quantity, TotalPrice, ServiceDate, EndDate,
          PricebookEntryId
   FROM QuoteLineItem
   WHERE QuoteId = '<quote_id>'
   ```
   - Verify all required fields populated
   - Check ServiceDate and EndDate exist (required for subscriptions)
   - Confirm PricebookEntryId populated

3. **Review Order Creation Configuration:**
   - Setup → Order Settings → Enable Orders checkbox
   - Setup → Quote Settings → Enable automatic order creation (if applicable)
   - Check if custom order button/action configured

4. **Check for Failed Flows:**
   - Setup → Process Automation → Flows
   - Filter by "Order" or "Quote to Order"
   - Check failed flow interviews: Setup → Environments → Flows → Failed Interviews
   - Review error message and failed element

5. **Verify Account Configuration:**
   ```sql
   SELECT Id, Name, BillingStreet, BillingCity, BillingState, BillingPostalCode,
          ShippingStreet, ShippingCity, ShippingState, ShippingPostalCode
   FROM Account
   WHERE Id = '<account_id>'
   ```
   - Billing address required for order creation
   - Shipping address may be required depending on configuration

6. **Test Manual Order Creation:**
   - Navigate to Quote detail page
   - Click "Create Order" button (or custom action)
   - If immediate error: Configuration or permission issue
   - If succeeds but order incomplete: Mapping or automation issue

7. **Check Order Mapping Configuration:**
   - Review field mappings: Quote fields → Order fields
   - Verify custom mappings (if implemented)
   - Check for field-level security blocking field population

**Verification:**
- Order created successfully from quote
- Order contains all Quote Line Items as Order Products
- Order pricing matches Quote pricing
- Order Status = Draft (ready for activation)
- Order Account, Opportunity, and Contact fields populated

**Known Issues:**
- Quote syncing with Opportunity can block order creation (wait for sync to complete)
- Large quotes (100+ line items) may timeout (consider batch processing)
- Custom validation rules on Order can block creation (review and adjust)

---

### Pattern 5: Product Attributes & Specifications

**Frequency:** Medium  
**Severity:** Priority 13-31

**Symptoms:**
- Product attributes not appearing on Order Product
- Custom attribute values lost during quote-to-order
- Specification fields blank on Order Product
- Attribute inheritance not working
- Configuration attributes not transferring

**Common Error Messages:**
```
"Product attribute not found"
"Invalid attribute value for product"
"Attribute mapping failed"
"Configuration attribute required"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Product Attribute records missing | Query ProductAttribute for product |
| 2 | Attribute value mapping not configured | Check attribute value mapping setup |
| 3 | Field-level security blocking attribute fields | Check FLS on attribute fields |
| 4 | Custom attribute logic not copying to Order Product | Review Apex/Flow logic |

**Resolution Steps:**

1. **Verify Product Attributes Exist:**
   ```sql
   -- Check if product has attributes defined
   SELECT Id, Name, Product2Id, AttributeName, DataType, IsRequired
   FROM ProductAttribute
   WHERE Product2Id = '<product_id>'
   
   -- Check attribute values
   SELECT Id, ProductAttributeId, Value, IsDefault
   FROM ProductAttributeValue
   WHERE ProductAttributeId IN (SELECT Id FROM ProductAttribute WHERE Product2Id = '<product_id>')
   ```

2. **Check Quote Line Item Attributes:**
   ```sql
   SELECT Id, QuoteLineItemId, AttributeName, Value
   FROM QuoteLineItemAttribute__c
   WHERE QuoteLineItemId IN (SELECT Id FROM QuoteLineItem WHERE QuoteId = '<quote_id>')
   ```

3. **Verify Order Product Attributes Created:**
   ```sql
   SELECT Id, OrderProductId, AttributeName, Value
   FROM OrderProductAttribute__c
   WHERE OrderProductId IN (SELECT Id FROM OrderProduct WHERE OrderId = '<order_id>')
   ```
   - Compare with Quote Line Item Attributes
   - Verify values match

4. **Check Attribute Mapping Configuration:**
   - Navigate to: Setup → Product Attribute Mapping
   - Verify mappings exist: QuoteLineItem attribute → OrderProduct attribute
   - Confirm field API names correct

5. **Review Automation:**
   - Check for Flow/Process Builder that copies attributes
   - Verify trigger on OrderProduct that copies from QuoteLineItem
   - Common pattern: After OrderProduct Insert → Copy attributes from related QLI

**Verification:**
- All product attributes from Quote Line Item appear on Order Product
- Attribute values match exactly
- Required attributes populated
- Custom attributes preserved

---

### Pattern 6: Pricing & Price Book Issues

**Frequency:** High  
**Severity:** Priority 0-45

**Symptoms:**
- Product price showing as $0.00
- "No price found for product" error
- Wrong price pulled for product
- Price book not applied to quote/order
- Multi-currency pricing incorrect

**Common Error Messages:**
```
"No price found for this product in the price book"
"Product not in standard price book"
"Price book entry not found"
"Currency mismatch between price book and quote"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Product not in Price Book | Query PricebookEntry for product |
| 2 | Quote/Order not associated with Price Book | Check Quote.Pricebook2Id field |
| 3 | Price Book Entry inactive or expired | Check PricebookEntry.IsActive and dates |
| 4 | Currency mismatch | Compare Quote.CurrencyIsoCode with PricebookEntry currency |

**Resolution Steps:**

1. **Verify Product in Price Book:**
   ```sql
   -- Check if product has price book entry
   SELECT Id, Product2Id, Pricebook2Id, UnitPrice, IsActive, CurrencyIsoCode
   FROM PricebookEntry
   WHERE Product2Id = '<product_id>' AND IsActive = true
   
   -- Check standard price book entry (required)
   SELECT Id, Product2Id, UnitPrice, IsActive
   FROM PricebookEntry
   WHERE Product2Id = '<product_id>' AND Pricebook2Id = '[StandardPricebookId]'
   ```

2. **Check Quote/Order Price Book:**
   ```sql
   SELECT Id, Name, Pricebook2Id, CurrencyIsoCode
   FROM Quote
   WHERE Id = '<quote_id>'
   ```
   - Pricebook2Id must be populated
   - If null: Quote will use Standard Price Book by default

3. **Verify Price Book Entry Active:**
   ```sql
   SELECT Id, Product2Id, UnitPrice, IsActive, UseStandardPrice
   FROM PricebookEntry
   WHERE Product2Id = '<product_id>' AND Pricebook2Id = '<pricebook_id>'
   ```
   - IsActive = true required
   - UnitPrice > 0 (unless using standard price)

4. **Check Currency Match:**
   ```sql
   -- Multi-currency org
   SELECT Id, CurrencyIsoCode FROM Quote WHERE Id = '<quote_id>'
   SELECT Id, CurrencyIsoCode FROM PricebookEntry WHERE Product2Id = '<product_id>'
   ```
   - Currencies must match for price to be found
   - If mismatch: Create price book entry for correct currency

5. **Add Product to Price Book (if missing):**
   - Navigate to: Price Book → Products
   - Click "Add Products"
   - Select product → Enter price → Save
   - **Note:** Must add to Standard Price Book first, then custom price books

**Verification:**
- Product has active price book entry
- Price displays correctly when product added to quote
- Currency matches between quote and price book
- Price can be edited if needed (unless locked)

**Best Practices:**
- Always add products to Standard Price Book first
- Create price book entries for all required currencies
- Set expiration dates on seasonal pricing
- Use Price Rules for dynamic pricing (instead of manual entries)

---

### Pattern 7: Product Selection in Cart/Transaction Line Editor

**Frequency:** Medium  
**Severity:** Priority 31-45

**Symptoms:**
- Products not appearing in catalog/product selector
- Transaction Line Editor (TLE) not loading
- Product search returns no results
- Cart doesn't display added products
- Sort order issues in product list
- Error: "Clarification on Sort Order Customization in Revenue Cloud"

**Common Error Messages:**
```
"No products available"
"Product catalog load failed"
"TLE initialization error"
"Product search service unavailable"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Product not active or outside effective date range | Check Product2.IsActive, EffectiveDate, ExpirationDate |
| 2 | Product not visible to user's price book | Verify price book entry exists |
| 3 | Product filter/search criteria excluding product | Review catalog filter settings |
| 4 | TLE configuration issue | Check TLE component settings |
| 5 | Product category/family not included in catalog | Review product categorization |

**Resolution Steps:**

1. **Verify Product Active and Available:**
   ```sql
   SELECT Id, Name, IsActive, EffectiveDate, ExpirationDate, Family, ProductCode
   FROM Product2
   WHERE Name LIKE '%<product_name>%' OR ProductCode = '<code>'
   ```
   - IsActive = true required
   - Current date should be between EffectiveDate and ExpirationDate
   - If dates null: Product always available

2. **Check Price Book Entry:**
   ```sql
   SELECT Id, Product2Id, Pricebook2Id, IsActive
   FROM PricebookEntry
   WHERE Product2Id = '<product_id>' AND Pricebook2Id = '<quote_pricebook_id>'
   ```
   - If no entry: Product won't appear in catalog for this price book

3. **Review Product Catalog Configuration:**
   - Navigate to: Setup → Product Catalog Settings (or Industries CPQ Configuration)
   - Check catalog filter rules
   - Verify product family/category included
   - Check if product visibility rules applied

4. **Check Transaction Line Editor Settings:**
   - Open Quote/Cart page → Edit Page (Lightning App Builder)
   - Locate TLE component → Review properties:
     - Product Selector: Enabled
     - Catalog Filter: Review criteria
     - Search Fields: Verify correct fields enabled
     - Display Fields: Confirm columns to show

5. **Test Product Search:**
   - In TLE, use search box
   - Try different search terms: Product Name, Product Code, Description
   - If still no results: Check if searchable fields configured correctly

6. **Sort Order Customization:**
   - Sort Order field on Quote Line Item: System assigns 5-digit numbers by default
   - **Cannot customize default numbering** (product design)
   - **Workaround:** Manually update Sort Order after adding products
   - **Best Practice:** Use increments of 10 (10, 20, 30) for easy insertion later

**Verification:**
- Product appears in catalog when adding to quote
- Search returns product when searching by name/code
- Product sort order displays as expected
- All product details visible in TLE

**Sample Case Numbers:** 473494088 (Sort Order customization)

---

### Pattern 8: Product Availability & Inventory

**Frequency:** Low-Medium  
**Severity:** Priority 13-26

**Symptoms:**
- Product marked as unavailable/out of stock
- Inventory levels not updating
- Product visibility rules not working
- Effective date issues
- Product shown but cannot be added

**Common Error Messages:**
```
"Product not available for selection"
"Insufficient inventory for product"
"Product outside effective date range"
"Product visibility constraint violated"
```

**Resolution Steps:**

1. **Check Product Availability:**
   ```sql
   SELECT Id, Name, IsActive, IsAvailable__c, QuantityOnHand__c,
          EffectiveDate, ExpirationDate
   FROM Product2
   WHERE Id = '<product_id>'
   ```

2. **Verify Effective Dating:**
   - Current date must be >= EffectiveDate (if set)
   - Current date must be <= ExpirationDate (if set)
   - If both null: No date restrictions

3. **Check Inventory (if applicable):**
   ```sql
   -- If using inventory management
   SELECT Id, Product2Id, QuantityOnHand, QuantityAvailable
   FROM ProductInventory__c
   WHERE Product2Id = '<product_id>'
   ```

4. **Review Visibility Rules:**
   - Check product visibility criteria (Account type, Industry, Region, etc.)
   - Verify current context matches visibility rules

**Verification:**
- Product available for selection
- Inventory sufficient (if tracked)
- Effective dates valid
- Visibility rules pass

---

### Pattern 9: Order Activation Issues

**Frequency:** Medium  
**Severity:** Priority 13-45

**Symptoms:**
- Order stuck in Draft status
- "Activate" button fails
- Order activates but assets not created
- Billing not triggered after activation
- Fulfillment process not starting

**Common Error Messages:**
```
"Order activation failed"
"Cannot activate order: validation error"
"Asset creation failed during activation"
"Order fulfillment service unavailable"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Validation rule blocking activation | Check validation rules on Order |
| 2 | Required fields missing on Order or Order Products | Verify all required fields populated |
| 3 | Asset creation automation failing | Check Flow/Process Builder for asset creation |
| 4 | Order status field incorrect | Verify Status picklist value |
| 5 | User lacks permission to activate orders | Check permissions on Order object |

**Resolution Steps:**

1. **Verify Order Ready for Activation:**
   ```sql
   SELECT Id, Status, AccountId, EffectiveDate, BillingStreet, BillingCity,
          ShippingStreet, ShippingCity, TotalAmount
   FROM Order
   WHERE Id = '<order_id>'
   ```
   - Status should be "Draft"
   - Account, billing address, effective date required
   - TotalAmount > 0

2. **Check Order Products:**
   ```sql
   SELECT Id, OrderId, Product2Id, Quantity, TotalPrice, ServiceDate, EndDate
   FROM OrderProduct
   WHERE OrderId = '<order_id>'
   ```
   - All Order Products should have required fields
   - For subscriptions: ServiceDate and EndDate required

3. **Review Validation Rules:**
   - Setup → Object Manager → Order → Validation Rules
   - Check which rules fire on Status change to "Activated"
   - Common blockers: Missing required custom fields, invalid date ranges

4. **Test Activation:**
   - Open Order → Click "Activate" button
   - If error displayed: Note exact message
   - If succeeds: Verify Status = "Activated"

5. **Check Asset Creation:**
   ```sql
   -- After activation, verify assets created
   SELECT Id, Name, Product2Id, Quantity, Status, AccountId
   FROM Asset
   WHERE OrderId = '<order_id>'
   ```
   - Asset count should match Order Product count (for asset-based products)

6. **Verify Fulfillment Triggered:**
   - Check if fulfillment flow/process started
   - Review fulfillment object records (if custom)
   - Verify billing integration triggered (if applicable)

**Verification:**
- Order Status = "Activated"
- Assets created (if asset-based products)
- Fulfillment process started
- Billing triggered (if configured)

---

### Pattern 10: Order Product Group Errors

**Frequency:** Medium (especially with bundles)  
**Severity:** Priority 26-45

**Symptoms:**
- "Order Product Group ID: id value of incorrect type"
- Bundle Order Products missing group assignment
- Order Product hierarchy broken
- Parent-child relationships incorrect on Order Products

**Common Error Messages:**
```
"Order Product Group ID: id value of incorrect type"
"Invalid Order Product Group reference"
"Order Product Group mapping failed"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Nested bundle configuration creating invalid group reference | Check for nested bundles in product structure |
| 2 | Ramp schedule with nested bundles | Check if quote has ramp schedule + nested bundles |
| 3 | Order Product Group record missing or invalid | Query OrderProductGroup object |
| 4 | Custom order creation logic not creating groups correctly | Review Apex classes creating Order Products |

**Resolution Steps:**

1. **Identify Nested Bundle Issue:**
   ```sql
   -- Check product hierarchy depth
   SELECT Id, Name, ParentProduct2Id, (SELECT Id FROM ChildProducts) AS Children
   FROM Product2
   WHERE Id IN (SELECT Product2Id FROM OrderProduct WHERE OrderId = '<order_id>')
   ```
   - If any products have ParentProduct2Id AND have ChildProducts: Nested bundle

2. **Check Order Product Groups:**
   ```sql
   SELECT Id, Name, OrderId
   FROM OrderProductGroup
   WHERE OrderId = '<order_id>'
   
   -- Check Order Products assigned to groups
   SELECT Id, Product2Id, OrderProductGroupId, ParentOrderProductId
   FROM OrderProduct
   WHERE OrderId = '<order_id>'
   ```

3. **Workaround for Nested Bundles:**
   - **Option 1:** Flatten bundle structure
     - Make all components direct children of top-level bundle
     - Avoid bundle-within-bundle configuration
   
   - **Option 2:** Don't use Ramp Schedule with nested bundles
     - Create ramp manually
     - Or: Restructure to single-level bundles

4. **Fix Existing Orders:**
   - If error already occurred and order created incomplete:
     - Delete the order
     - Reconfigure product bundle (flatten)
     - Re-create order from quote

**Verification:**
- Order Products created successfully
- Order Product Groups created for bundles
- Parent-child relationships correct
- No "id value of incorrect type" errors

**Sample Case Numbers:** 473505624

---

## C. Setup & Configuration Guide

### Complete Product to Order Setup Checklist

```
1. PRODUCT CATALOG SETUP
   □ Products created with required fields (Name, Product Code, Family)
   □ Products set to Active
   □ Effective Date and Expiration Date set (if applicable)
   □ Product descriptions and images added
   
2. PRODUCT BUNDLES (if using)
   □ Parent product marked as bundle (IsBundle = true)
   □ ProductBundleOption records created for each component
   □ SortOrder set on all bundle options
   □ Required components marked (IsRequired = true)
   □ Bundle pricing method configured (Fixed, Component, or Both)
   □ Avoid nested bundles (bundle within bundle) - causes Order Product Group errors
   
3. PRICE BOOKS
   □ Products added to Standard Price Book (required)
   □ Products added to custom Price Books (if using)
   □ UnitPrice set for each price book entry
   □ Price Book Entries set to Active
   □ Multi-currency entries created (if multi-currency org)
   
4. PRODUCT ATTRIBUTES (if using)
   □ ProductAttribute records created
   □ Attribute data types configured correctly
   □ Required attributes marked
   □ ProductAttributeValue records created
   □ Attribute mapping configured (QLI → OrderProduct)
   
5. PRODUCT CONFIGURATION RULES (if using)
   □ Validation rules created for incompatible combinations
   □ Selection rules created for auto-inclusion
   □ Filter rules created for catalog filtering
   □ Rules activated and tested
   □ Rule priority/sequence set correctly
   
6. QUOTE CONFIGURATION
   □ Setup → Quote Settings → Enable Quotes
   □ Price Book assigned to Quote (or use Standard)
   □ Quote template configured (if using)
   □ Quote approval process configured (if using)
   
7. ORDER CONFIGURATION
   □ Setup → Order Settings → Enable Orders
   □ Order status picklist values configured
   □ Order activation automation configured
   □ Order-to-Asset mapping configured (if asset-based)
   □ Fulfillment process configured (if using)
   
8. QUOTE TO ORDER MAPPING
   □ Field mappings configured: Quote → Order
   □ Field mappings configured: QuoteLineItem → OrderProduct
   □ Custom mapping logic implemented (if needed)
   □ Flow/Process Builder for order creation (if custom)
   
9. PERMISSIONS
   □ Product2: Read access for all users
   □ PricebookEntry: Read access
   □ Quote: Create, Read, Edit
   □ QuoteLineItem: Create, Read, Edit, Delete
   □ Order: Create, Read, Edit
   □ OrderProduct: Create, Read, Edit
   □ Permission sets assigned to users
   
10. TESTING
    □ Test simple product selection and pricing
    □ Test bundle selection and expansion
    □ Test quote creation with multiple products
    □ Test quote to order conversion
    □ Test order activation
    □ Test asset creation (if applicable)
```

### Common Misconfigurations

| Misconfiguration | Error/Symptom | Fix |
|-----------------|---------------|-----|
| Product not in Standard Price Book | "No price found" | Add to Standard PB first, then custom PBs |
| Nested bundles with ramp schedule | "Order Product Group ID incorrect type" | Flatten bundle structure to single level |
| Product inactive | Product doesn't appear in catalog | Set IsActive = true |
| Effective date in future | Product not available | Adjust EffectiveDate to today or earlier |
| Missing Order Product field mapping | Fields blank on Order Product | Configure Quote Line Item → Order Product mapping |
| Configuration rule not activated | Rule doesn't fire | Activate rule in Setup |
| Price Book Entry inactive | $0.00 price | Set IsActive = true on Price Book Entry |
| Bundle component inactive | Component missing from bundle | Activate component product |

---

## D. Licensing & Entitlements

### Required Licenses

| Feature | Required License | Notes |
|---------|-----------------|-------|
| Product Catalog | Sales Cloud or Revenue Cloud | Basic product management |
| Price Books | Sales Cloud or Revenue Cloud | Advanced pricing |
| Product Bundles | Revenue Cloud (recommended) | Or custom development in Sales Cloud |
| Product Configuration Rules | Revenue Cloud | Industries CPQ functionality |
| Quote to Order | Sales Cloud or Revenue Cloud | Standard functionality |
| Advanced Product Config | Industries CPQ License | Required for complex configuration |

---

## E. Key Behavioral Design Rules

These are **by design** — not bugs. Educate the customer:

1. **Nested Bundles with Ramps:** Nested bundles (bundle within bundle) combined with ramp schedules cause "Order Product Group ID" errors. Workaround: Flatten bundle structure.

2. **Standard Price Book Required:** Products must exist in Standard Price Book before adding to custom price books (Salesforce requirement).

3. **Quote Acceptance Required:** Quote must be in "Accepted" status before creating order (cannot create order from draft quote).

4. **Sort Order Numbering:** System-assigned Sort Order uses 5-digit increments (10000, 10001, etc.). Cannot customize default numbering pattern. Manually update Sort Order after adding products if specific sequence needed.

5. **Price Book Entry Currency:** Price Book Entry currency must match Quote/Order currency for price to be found (multi-currency orgs).

6. **Product Availability:** Products only appear in catalog if: (a) Active, (b) Within effective date range, (c) In quote's price book, (d) Pass visibility rules.

7. **Bundle Pricing Calculation:** Bundle pricing depends on BundlePricingType: Fixed = bundle price only, Component = sum of components, Both = bundle + components.

8. **Order Activation:** Order Status must = "Draft" to activate. Once activated, cannot change back to Draft (only forward status progression).

9. **Asset Creation:** Assets only created during activation if product is asset-based (HasAsset__c = true or configured as asset product).

10. **Product Attribute Mapping:** Custom attributes don't auto-copy from Quote Line Item to Order Product. Requires custom mapping/automation.

---

## F. Escalation Paths

### When to Escalate

| Scenario | Team | Priority |
|----------|------|----------|
| Nested bundle error persists after flattening | Revenue Cloud Engineering | P2 |
| Order Product Group error in production | Revenue Cloud Engineering | P1 |
| Product configuration rules not firing | Industries CPQ Team | P2 |
| Quote to Order conversion fails (standard flow) | Revenue Cloud Engineering | P2 |
| Price Book Entry issues after release | Platform Engineering | P2 |
| TLE not loading / performance issue | Revenue Cloud UI Team | P2 |
| Bundle pricing calculation wrong | Revenue Cloud Pricing Team | P2 |

### Gather Before Escalating

```
□ Org ID (18-char)
□ User ID(s) affected
□ Exact error message (full text)
□ Steps to reproduce (numbered)
□ Quote ID (if applicable)
□ Order ID (if applicable)
□ Product IDs involved
□ Debug logs from failure
□ Screenshots: Error state, product configuration
□ Last known working date (if regression)
□ Salesforce release version (if post-release issue)
```

---

## G. Tribal Knowledge & Pro Tips

**Pro Tips from Support Experience:**

1. **Always test bundles in sandbox first** — Bundle configuration errors can corrupt orders
2. **Flatten bundle structures** — Max 2 levels deep (parent + children), avoid nested bundles
3. **Set Sort Order on bundle options** — Controls display order in TLE
4. **Standard Price Book is mandatory** — Add products there first, always
5. **Check effective dates first** — #1 cause of "product not found" issues
6. **Ramp schedules + nested bundles = trouble** — Known limitation, avoid combination
7. **Quote must be Accepted** — Cannot create order from draft quote (by design)
8. **Field mapping is critical** — Missing Quote → Order field mappings = blank Order Products
9. **Test with simple product first** — Isolate if issue is product-specific or system-wide
10. **Price Book Entry activation matters** — Inactive entries = $0 price

**Common "Gotchas":**
- Product not appearing? Check: Active, Effective Dates, Price Book Entry, Visibility Rules
- Order Product missing fields? Check: Quote Line Item → Order Product field mapping
- Bundle components missing? Check: ProductBundleOption records exist and active
- Configuration rule not working? Check: Rule active, criteria match, priority/sequence correct
- Order activation fails? Check: Validation rules, required fields, user permissions

---

## H. Active Known Issues (Live Lookup)

**If GUS is connected, run this query to get current open bugs:**

```bash
sf data query --query "SELECT Name, Subject__c, Priority__c, Status__c, CreatedDate FROM ADM_Work__c WHERE (Product_Tag__r.Name LIKE '%Revenue Cloud%' OR Product_Tag__r.Name LIKE '%Product%' OR Product_Tag__r.Name LIKE '%Order%') AND Type__c = 'Bug' AND Status__c NOT IN ('Closed', 'Duplicate', 'Never Fix', 'Not a Bug') ORDER BY Priority__c, CreatedDate DESC LIMIT 20" --target-org gus --json
```

**Known Issues (based on case analysis):**
- **Nested Bundles + Ramp Schedules:** "Order Product Group ID: id value of incorrect type" error
- **Sort Order Customization:** Cannot customize default 5-digit numbering pattern (product design)
- **Product Configuration Rules:** Rules may not fire if priority/sequence not set correctly

---

## I. Key Objects Reference

### Product to Order Object Model

```
Product2 (Product)
├── IsActive, EffectiveDate, ExpirationDate (Availability)
├── Family, ProductCode, Description (Categorization)
├── IsBundle__c, BundlePricingType__c (Bundle Config)
└── ParentProduct2Id (Hierarchy)

PricebookEntry
├── Product2Id (Link to Product)
├── Pricebook2Id (Link to Price Book)
├── UnitPrice, IsActive (Pricing)
└── CurrencyIsoCode (Multi-currency)

ProductBundleOption
├── BundleProduct2Id (Parent Bundle)
├── Product2Id (Component Product)
├── Quantity, IsRequired, SortOrder (Configuration)
└── IsActive (Status)

Quote
├── Status, IsSyncing (State)
├── Pricebook2Id, CurrencyIsoCode (Pricing Context)
└── AccountId, OpportunityId (Relationships)

QuoteLineItem
├── QuoteId, Product2Id (Relationships)
├── Quantity, UnitPrice, TotalPrice (Pricing)
├── ServiceDate, EndDate (Subscription Dates)
└── SortOrder (Display Order)

Order
├── Status, EffectiveDate (State)
├── AccountId, OpportunityId, QuoteId (Relationships)
└── BillingAddress, ShippingAddress (Fulfillment)

OrderProduct
├── OrderId, Product2Id (Relationships)
├── Quantity, UnitPrice, TotalPrice (Pricing)
├── ServiceDate, EndDate (Subscription Dates)
├── OrderProductGroupId (Bundle Grouping)
└── ParentOrderProductId (Bundle Hierarchy)

OrderProductGroup
├── OrderId (Relationship)
└── Name (Group Name)
```

### Diagnostic SOQL Queries

```sql
-- 1. Check Product Configuration
SELECT Id, Name, IsActive, IsBundle__c, EffectiveDate, ExpirationDate, 
       Family, ProductCode
FROM Product2
WHERE Id = '<product_id>'

-- 2. Check Price Book Entries for Product
SELECT Id, Product2Id, Pricebook2Id, UnitPrice, IsActive, CurrencyIsoCode
FROM PricebookEntry
WHERE Product2Id = '<product_id>'

-- 3. Check Bundle Components
SELECT Id, BundleProduct2Id, Product2Id, Quantity, IsRequired, 
       SortOrder, IsActive
FROM ProductBundleOption
WHERE BundleProduct2Id = '<bundle_id>'
ORDER BY SortOrder

-- 4. Check Quote Line Items
SELECT Id, Product2Id, Quantity, TotalPrice, ServiceDate, EndDate,
       SortOrder, PricebookEntryId
FROM QuoteLineItem
WHERE QuoteId = '<quote_id>'
ORDER BY SortOrder

-- 5. Check Order Products
SELECT Id, Product2Id, Quantity, TotalPrice, ServiceDate, EndDate,
       OrderProductGroupId, ParentOrderProductId
FROM OrderProduct
WHERE OrderId = '<order_id>'

-- 6. Check Order Product Groups
SELECT Id, Name, OrderId
FROM OrderProductGroup
WHERE OrderId = '<order_id>'

-- 7. Find Products Not in Standard Price Book
SELECT Id, Name, ProductCode
FROM Product2
WHERE Id NOT IN (
  SELECT Product2Id FROM PricebookEntry 
  WHERE Pricebook2Id = '<standard_pricebook_id>'
)
AND IsActive = true

-- 8. Find Inactive Price Book Entries
SELECT Id, Product2Id, Pricebook2Id, UnitPrice
FROM PricebookEntry
WHERE IsActive = false AND Product2Id = '<product_id>'

-- 9. Check Product Effective Dates
SELECT Id, Name, EffectiveDate, ExpirationDate
FROM Product2
WHERE (EffectiveDate > TODAY OR ExpirationDate < TODAY)
AND IsActive = true

-- 10. Find Quote Line Items Without Order Products
SELECT Id, QuoteId, Product2Id
FROM QuoteLineItem
WHERE QuoteId = '<quote_id>'
AND Id NOT IN (
  SELECT QuoteLineItemId FROM OrderProduct 
  WHERE OrderId = '<order_id>'
)
```

---

## J. Troubleshooting Decision Tree

```
Product to Order Issue Reported
│
├── 1. CLASSIFY ISSUE (Section A)
│   ├── Order Product creation? → Pattern 1
│   ├── Product bundle? → Pattern 2
│   ├── Configuration rules? → Pattern 3
│   ├── Quote to Order? → Pattern 4
│   ├── Product attributes? → Pattern 5
│   ├── Pricing/Price Book? → Pattern 6
│   ├── Product selection/TLE? → Pattern 7
│   ├── Availability? → Pattern 8
│   ├── Order activation? → Pattern 9
│   └── Order Product Group? → Pattern 10
│
├── 2. ASK QUICK DIAGNOSTIC QUESTIONS (Section A)
│   ├── What's the exact error?
│   ├── Which step fails?
│   ├── Product type (simple/bundle)?
│   ├── When did it start?
│   └── Reproducible with different product?
│
├── 3. RUN DIAGNOSTIC QUERIES (Section I)
│   ├── Check product configuration
│   ├── Verify price book entries
│   ├── Review bundle structure
│   ├── Compare QLI to OrderProduct
│   └── Check order/quote status
│
├── 4. FOLLOW PATTERN RESOLUTION STEPS
│   ├── Verify configuration
│   ├── Check permissions
│   ├── Test with simple product
│   ├── Review automation
│   └── Apply workaround if available
│
├── 5. VERIFY FIX
│   ├── Test with original scenario
│   ├── Test with System Admin
│   ├── Test quote to order flow end-to-end
│   └── Document resolution
│
└── 6. ESCALATE IF NEEDED (Section F)
    ├── Gather diagnostic info
    ├── Identify correct team
    ├── Provide context
    └── Reference similar cases/known issues
```
