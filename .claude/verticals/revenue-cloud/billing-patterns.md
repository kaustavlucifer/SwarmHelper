## A. Triage & Classification

### Quick Diagnostic Questions

Ask these first to narrow the issue category:

1. **What is the error message?** (exact text, error ID if available)
2. **Where does it fail?** (UI button, Billing Batch Scheduler, API call, Flow)
3. **What is the Invoice/Billing Schedule status?** (Draft in Progress, Error, Queueing In Progress)
4. **Is this a new setup or was it working before?**
5. **Are there Revenue Transaction Error Logs (RTELs)?** (check Invoice Batch Run related list)
6. **What permission sets does the running user have?**
7. **Is this a sandbox or production org?**

### Decision Tree

```
Customer reports a Billing & Invoicing issue
│
├── Billing Schedules not created after order activation
│   ├── NullPointerException / UNKNOWN_EXCEPTION → Pattern 1
│   ├── Missing permissions → Pattern 5
│   └── Order created directly without Quote → Pattern 5 (Platform design)
│
├── Invoice Batch Run / Billing Batch Scheduler failing
│   ├── Invoices stuck in "Draft in Progress" → Pattern 2
│   ├── "No billing schedules met matching criteria" → Pattern 2
│   ├── Charge type filter (Recurring+Usage) → Pattern 2 (Active Bug)
│   └── HTTPS exception in scheduler → Pattern 2
│
├── Tax engine / tax calculation failures
│   ├── AvaTax documentCode mismatch → Pattern 3
│   ├── Vertex invoice stuck Draft in Progress → Pattern 3
│   └── Standard tax engine not working → Pattern 3
│
├── Invoice PDF / document generation errors
│   ├── Insufficient_Access on generation → Pattern 4
│   └── UNKNOWN_EXCEPTION in DocumentGenerationProcess → Pattern 4
│
├── Permission / access issues
│   ├── Fields not visible → Pattern 5
│   ├── DML not allowed on Invoice/BillingBatchScheduler → Pattern 5
│   └── Billing Admin via PSG not working → Pattern 5
│
├── Credit Memo / Debit Memo issues → Pattern 6
├── Payment issues → Pattern 7
├── Milestone Billing issues → Pattern 8
├── Usage Billing / DPE issues → Pattern 9
└── Proration / date calculation issues → Pattern 10
```

---

## B. Known Issue Patterns

---

### Pattern 1: Billing Schedules Not Generated After Order Activation

**Frequency:** High (50+ cases)
**Severity:** P13–P27
**TTR Impact:** 3–7 days

**Symptoms:**
- Billing schedules not created when order is activated
- Error on `createBillingSchedulesFromBillingTransaction` or `createBillingSchedulesFromTrxn` API:
  - `UNKNOWN_EXCEPTION: Cannot invoke "String.length()" because "id" is null`
  - `We can't create billing schedules for the specified order products because they're the same as the initial sale order products, which already have billing schedules.`
  - `Create a billing schedule for the original transaction, and then create billing schedules for the related amended, renewed, or canceled transactions.`
- No billing schedules created on renewal/amendment orders
- Usage billing schedules not generated for VCSP anchor products

**Root Cause:**
- NullPointerException in billing schedule creation service when billing transaction has null ID (platform bug — check for active GUS investigation)
- Amendment/renewal orders: reference ID on Billing Schedule is blank (missing Order Item reference) — typically caused by data migration without original order line items
- Custom flows interfering with OOTB billing schedule creation logic
- Order created directly without Quote — pricing context not established, assets/billing schedules don't generate

**Resolution Steps:**
1. Check if the order was created via Quote → Order path (required for full orchestration)
2. Check if billing transaction ID is populated on the order
3. Run "Create Billing Schedules From Billing Transaction" action manually via API:
   ```
   POST /services/data/vXX.0/commerce/invoicing/billing-schedules/actions/create
   { "billingTransactionIds": ["<OrderId>"] }
   ```
4. If NullPointerException: check if this is a known platform bug — search GUS for active investigation
5. For amendment/renewal failures: verify asset data is not migrated without original order line items; ensure Reference ID on Billing Schedule points to Order Item
6. If custom flows are involved: test with OOTB flow (Order to Billing Schedule Flow) to isolate the issue; recommend decoupling BS creation from custom flows
7. For renewal BS conflicts: error "same as initial sale" means BS already exist — verify order type and existing BS state

**Verification:**
- Query: `SELECT Id, Status FROM BillingSchedule WHERE OrderId = '<OrderId>'`
- Confirm billing schedules are in "Active" or "Ready for Invoicing" status

**Workarounds (from Slack):**
- If OOTB flow works but custom flow fails: decouple BS creation from custom flows; use standard flow template with conditional logic for special scenarios (Pratibha Rajpoot, #support-rev-rlm-global-swarm-help, 2026-05-05)
- Use API directly: `createBillingSchedulesFromBillingTransaction` or `createBillingSchedulesFromTrxn` as fallback when flow fails
- For data migration scenarios: ensure original order records exist before creating billing schedules; document best practices to customer

**Related Documentation:**
- https://developer.salesforce.com/docs/atlas.en-us.revenue_lifecycle_management_dev_guide.meta/revenue_lifecycle_management_dev_guide/actions_obj_create_billing_schedule_from_billing_transaction.htm
- https://developer.salesforce.com/docs/atlas.en-us.revenue_lifecycle_management_dev_guide.meta/revenue_lifecycle_management_dev_guide/actions_obj_create_billing_schedules_from_transaction.htm

**Escalation Criteria:**
- NullPointerException with no custom code involved → escalate to engineering with org ID and billing transaction ID
- Confirmed platform bug — file GUS investigation

---

### Pattern 2: Invoice Batch Run / Billing Batch Scheduler Failures

**Frequency:** High (60+ cases)
**Severity:** P13–P27
**TTR Impact:** 5–10 days

**Symptoms:**
- Invoices stuck in **"Draft in Progress"** status permanently
- Error in Revenue Transaction Error Logs (RTEL):
  - `Invoice is not created for BS. Contact Salesforce Support.`
  - `Found no records while summarizing billing schedules. Contact Salesforce Support with error ID: XXXXXXXX`
  - `The InvoiceBatchRun could not create any invoices because there were no billing schedules that met the matching criteria.`
  - `INVOICE_GENERATION_SERVICE_DPE_CREATION_FAILED: Your request has errors or warnings.`
  - `Error occurred while updating status on invoice post tax calculation. Not Updated.`
- Billing Batch Scheduler stuck in "Queueing In Progress"
- Invoice run picks up wrong billing schedules (usage only instead of recurring+usage)

**Root Cause:**
- **ACTIVE BUG (2026-05-27):** When charge type filter is set to 'Recurring' + 'Usage' together, the invoice run only filters in Usage billing schedules — Engineering is working on fix
- HTTPS required exception establishing user context (org security settings changed after scheduler setup)
- DPE (Data Processing Engine) not enabled or Data Pipelines Base User permission not assigned
- Invoices stuck after RTEL errors from previous run — need recovery
- Formula error in currency-handling CASE() logic in DPE (multicurrency orgs)
- `Filter_Billing_Schedules_for_Generating_Invoices` DPE job completes but `Generate_Invoices_for_Billing_Schedules` never starts — missing Data Pipeline setup

**Resolution Steps:**
1. Open the Invoice Batch Run → check Revenue Transaction Error Logs for specific error
2. **If invoices stuck in "Draft in Progress":**
   a. Click the **Recovery** button on the Invoice Batch Run — this cancels stuck invoices and recovers associated billing schedules
   b. After recovery, generate a single invoice via API to validate: `POST /services/data/vXX.0/commerce/invoicing/billing-schedules/actions/create-invoices`
   c. If single invoice succeeds, run the Invoice Batch Run again
   d. Note: recovered invoices cannot be re-synced to external systems (e.g., MuleSoft) in the same billing period
3. **If charge type bug (Recurring + Usage):** Change charge type filter to **'All'** as workaround until engineering fix ships
4. **If DPE failure:**
   - Enable Data Pipelines from Setup
   - Assign **Billing Admin** + **Data Pipelines Base User** permission sets to the running user
   - Check Monitoring Workflow Services for DPE job status
5. **If HTTPS exception:**
   - Check if admin user permissions were modified after scheduler was created
   - Run full data resync for Billing Schedule object in Data Manager
   - If issue persists: disconnect and reconnect the Billing Schedule object (do NOT reconnect user object)
   - Try running the scheduler again
6. **If "no matching billing schedules":**
   - Verify billing schedules are in "Ready for Invoicing" status
   - Check billing schedule group EffectiveNextBillingDate — target date must be ≥ this date
   - Confirm billing policy/treatment is correctly configured

**Verification:**
- Confirm IBR status changes to "Completed" or "Completed with Errors"
- Check that billing schedules move from "Ready for Invoicing" to "Invoiced"
- Verify new Invoice records are created in "Draft" or "Posted" status

**Workarounds (from Slack):**
- **Active bug workaround:** Use 'All' as charge type filter on Billing Batch Scheduler instead of 'Recurring'+'Usage' combination (Sakshi Sharma, #rlm-office-hours, 2026-05-27)
- Recovery process for stuck invoices: Recovery button on IBR → single invoice API test → full batch run (Gulla Bharath Kumar, #support-rev-rlm-global-swarm-help, 2025-10-30)
- For invoices > 200 lines stuck in error: Use Draft-to-Posted IBR batch run instead of manually posting large invoices (Gulla Bharath Kumar, #rlm-office-hours)
- **Recovery limit:** Max 200 billing schedules can be recovered at a time; for invoices with >200 lines in error, coordinate with engineering
- If invoice in test class is needed: use `ConnectApi.Billing.generateInvoices()` with autoproc user — DML INSERT not allowed on Invoice or BillingBatchScheduler objects in test classes

**Related Documentation:**
- https://developer.salesforce.com/docs/atlas.en-us.revenue_lifecycle_management_dev_guide.meta/revenue_lifecycle_management_dev_guide/connect_resources_create_invoices_from_billing_schedules.htm

**Escalation Criteria:**
- Invoices cannot be recovered after Recovery button + engineering confirms no fix ETA
- DPE never starts despite correct permissions and setup — escalate with Monitoring Workflow Services logs
- More than 200 billing schedules in error with no recovery path

---

### Pattern 3: Tax Engine / Tax Calculation Failures

**Frequency:** Medium-High (30+ cases)
**Severity:** P13–P26
**TTR Impact:** 5–14 days

**Symptoms:**
- Invoice stuck in "Draft in Progress" — tax not calculated (amount = 0.00)
- RTEL error: `Error occurred while updating status on invoice post tax calculation. This error occurred when the flow tried to create records: INVALID_INPUT: When the Document Input Type is Document Template, the Document Template field must be populated.`
- AvaTax error: `TAX_CALL_FAILURE: Response field documentCode doesn't match with the value provided in input`
- Vertex error: `Tax Engine couldn't process your request. Adapter validation failure`
- `Tax Calculation fails due to duplicate Doc Code`
- Standard Tax Engine not returning calculations
- Issue occurs in batch but not on individual invoice generation

**Root Cause:**
- **Vertex:** Incorrect App Usage Assignment input mapping under Sales Transaction Node in Context Definition. Should only exist under Asset Node.
- **AvaTax documentCode mismatch:** documentCode sent to AvaTax does not match what AvaTax returns — typically a race condition or duplicate transaction code issue in batch processing
- **Document Template missing:** Invoice template deactivated/reconfigured without updating flow reference
- Missing permission sets: Billing Admin + Data Pipeline Base User needed for tax calculation to work
- Dual tax calculation triggered by custom configuration

**Resolution Steps:**
1. Check if the issue is batch-only or also fails on individual invoice generation
2. **For Vertex (invoice stuck Draft in Progress, tax = 0.00):**
   a. Navigate to Context Definition → Sales Transaction Node
   b. Check App Usage Assignment input mapping — it must NOT exist under Sales Transaction Node; it should only be under Asset Node
   c. Remove incorrect mapping → test order → activate → generate invoice
   d. Verify user has **Billing Admin** + **Data Pipeline Base User** permission sets
3. **For AvaTax documentCode mismatch:**
   a. Check if issue is batch-only — if individual invoice generation works, this is a batch concurrency issue
   b. Review AvaTax transaction logs for duplicate documentCode
   c. Engage AvaTax adapter team or file GUS investigation
4. **For Document Template error:**
   a. Check `RLM On Demand Invoice Document Generation` flow is active
   b. Deactivate and reactivate the flow
   c. Ensure Document Template field is populated in flow configuration
5. **For standard tax engine:**
   - Verify Tax Engine Provider, Tax Engine, Tax Policy, Tax Treatment, Billing Policy, Billing Treatment are all correctly configured
   - Check tax estimation setting is enabled for quote-level tax calculations
   - Review RTE logs for specific failure point

**Verification:**
- Generate single invoice and verify tax amount is populated
- Run Invoice Batch Run and confirm invoices reach "Draft" status (not "Draft in Progress")

**Workarounds (from Slack):**
- Vertex: Fix the Context Definition mapping (Sales Transaction Node cleanup) before any other troubleshooting (Gulla Bharath Kumar, #support-rev-rlm-global-swarm-help, 2025-04-04)
- AvaTax batch failures: Retry individual invoices via API while batch issue is investigated
- Log in from Org62 if BT (Black Tab) login is not working for certain orgs

**Related Documentation:**
- https://help.salesforce.com/s/articleView?id=ind.billing.htm&type=5

**Escalation Criteria:**
- AvaTax documentCode mismatch persists after retry — escalate with Splunk/RTE logs
- Vertex mapping is already correct but tax still fails — escalate to RCB engineering

---

### Pattern 4: Invoice PDF / Document Generation Failures

**Frequency:** Medium (20+ cases)
**Severity:** P45
**TTR Impact:** 2–5 days

**Symptoms:**
- `Insufficient_Access: Looks like you don't have access to the invoice document generation feature.`
- `UNKNOWN_EXCEPTION: Something went wrong while retrieving records for DocumentGenerationProcess.`
- `Error element Invoice_Docgen (FlowRecordCreate). INVALID_INPUT: When the Document Input Type is Document Template, the Document Template field must be populated.`
- Job status shows "in progress" indefinitely and never completes
- Error in `RC On Demand Invoice Document Generation` flow

**Root Cause:**
- Non-admin users with only `Billing Operations User` permission set lack additional permissions needed for invoice document generation
- Document template deactivated or incorrectly configured after changes
- Race condition in DocumentGenerationProcess retrieval

**Resolution Steps:**
1. Verify the user's permission sets — Billing Operations User alone is NOT sufficient for invoice document generation
2. Check if additional permission sets are needed (verify in org setup against documentation)
3. Deactivate and reactivate the `RLM On Demand Invoice Document Generation` flow
4. Verify Document Template is assigned in the flow configuration (Document Template field must not be blank)
5. If using custom invoice templates: confirm template is active and properly configured

**Verification:**
- Non-admin user can generate invoice document without error
- Generated document appears in Files related list on Invoice record

**Workarounds (from Slack):**
- If non-admin cannot generate: temporarily test with admin user to confirm it's a permissions issue, then identify exact missing permission (Shaun Ray, #rlm-office-hours, 2025-06-24)
- Flow deactivate/reactivate often resolves transient UNKNOWN_EXCEPTION errors (Hendra Tio, #rlm-office-hours)

**Escalation Criteria:**
- Correct permissions assigned, flow is active, template is configured — still fails → escalate with org ID and exact flow error

---

### Pattern 5: Permissions & Access Issues

**Frequency:** High (40+ cases)
**Severity:** P45
**TTR Impact:** 1–3 days

**Symptoms:**
- `We couldn't create the invoice batch run. Ask your Salesforce admin to assign the Billing Admin permission set and the Data Pipelines Base User permission set...`
- `DML operation INSERT not allowed on Invoice`
- `DML operation INSERT not allowed on BillingBatchScheduler`
- `Entity: InvoiceLine is not api accessible`
- Billing Account / Term Unit fields not visible for certain users
- `Enable Product selling model Permission` needed
- Fields locked on Billing Schedule and Invoice objects (Mulesoft integration user)

**Root Cause:**
- **Critical:** Billing Admin permission set must be assigned **directly** to the user — assigning via Permission Set Group (PSG) does NOT work for Billing Batch Scheduler creation
- Missing Data Pipelines Base User permission set
- Post-sandbox-refresh permission sets not reassigned
- Invoice and BillingBatchScheduler are system-managed objects — DML INSERT not allowed in Apex/test classes; must use ConnectAPI
- InvoiceLine not API-accessible — permissions not applied after sandbox refresh or version upgrade

**Resolution Steps:**
1. Verify the running user has ALL of the following:
   - **Billing Admin** (assigned DIRECTLY, not via PSG)
   - **Billing Operations User**
   - **Data Pipelines Base User**
   - **Create Billing Schedules From Billing Transactions API** (for Order to Billing Schedule flow)
2. If using Permission Set Group: extract Billing Admin as a direct assignment in addition to PSG
3. For InvoiceLine not accessible: re-assign permission sets, check if sandbox needs metadata deployment
4. For Apex test class issues: use `ConnectApi.Billing.generateInvoices()` — do not try to insert Invoice or BillingBatchScheduler records directly
5. For Mulesoft integration user with locked fields: check LAP (License Access Provisioning) — may need LAP request for field write access

**Verification:**
- User can create Billing Batch Scheduler without error
- InvoiceLine is queryable and accessible via API

**Workarounds (from Slack):**
- Billing Admin directly on user vs. PSG: must always assign directly for Billing Batch Scheduler creation (Swapnil Rawat, #support-rev-rlm-global-swarm-help, 2025-02-11)
- Order directly without Quote: Creating Orders without Quote context = pricing, assets, and billing schedules may not generate correctly. Supported path: Quote → Order → Activate (Sai Varun Siddamsetty, #support-rev-rlm-global-swarm-help, 2026-03-11)

**Required Permission Sets Summary:**

| Capability | Billing Admin | Billing Ops User | Data Pipeline Base User | Create BS From Trxn API |
|---|---|---|---|---|
| Create Billing Batch Scheduler | ✅ (direct) | | ✅ | |
| Order to Billing Schedule Flow | ✅ | ✅ | | ✅ |
| Generate Invoice (UI/API) | ✅ | ✅ | ✅ | |
| Invoice Document Generation | ✅ | + (verify) | | |

**Escalation Criteria:**
- All permission sets correctly assigned directly — still getting errors → escalate with exact error and permission set screenshot

---

### Pattern 6: Credit Memo & Debit Memo Issues

**Frequency:** Medium (15+ cases)
**Severity:** P13–P27
**TTR Impact:** 3–7 days

**Symptoms:**
- `Error while voiding a credit memo`
- `INVALID_RECORD_ATTRIBUTE_VALUE: We couldn't post the specified invoice because the associated billing schedule statuses aren't Ready for Invoicing.`
- Tax treatment null on debit memo line — debit memo posting blocked
- Debit memo stuck — `invoice generation status` not updating
- Challenge creating draft credit memo via Billing API
- `ConnectApi Credit Memo Application fails — Invoice AppType not set to Revenue Cloud in test class`

**Root Cause:**
- Tax treatment not populated on debit memo line — required before posting
- Debit memo conversion to invoice line fails when no matching invoice is found (matching key mismatch)
- Credit memo API requires Invoice AppType = Revenue Cloud (often not set in test orgs)
- Evergreen PSM invoices with usage products — billing schedule goes to "Error" after invoice creation due to status validation

**Resolution Steps:**
1. **For debit memo posting blocked (tax treatment null):**
   - Populate Tax Treatment on the debit memo line record
   - Verify Billing Policy and Tax Treatment configuration
2. **For debit memo not converting to invoice line:**
   - Check Revenue Transaction Error Log on the Invoice Batch Run
   - Modify matching invoice reference key or matching reference name to find the correct invoice
   - Or manually convert using Invoice Ingestion API and select "Manually Processed" checkbox
3. **For voiding credit memo errors:**
   - Verify invoice status — cannot void if invoice is already in certain states
   - Check if related billing schedules are in correct state
4. **For test class ConnectApi Credit Memo failure:**
   - Set Invoice AppType = `Revenue Cloud` on the invoice record in test setup

**Workarounds (from Slack):**
- Debit memo conversion: if no matching invoice found in batch run, use Invoice Ingestion API with Manually Processed = true (Prateek Sethi, #rlm-office-hours, 2026-03-16)

**Escalation Criteria:**
- Platform-level error voiding credit memo with no customization involved → escalate with GUS investigation

---

### Pattern 7: Payment Issues

**Frequency:** Medium (15+ cases)
**Severity:** P12–P45
**TTR Impact:** 3–7 days

**Symptoms:**
- `Unable to apply a payment to an invoice with a different currency.`
- `UNKNOWN_EXCEPTION: An unexpected error occurred` while filtering Payment Records based on saved Payment Method
- `Submit Cash and Check Payment` issues
- Payment Standard objects missing in RLM org
- Payment Method add issues
- Ayden/Stripe gateway configuration issues

**Root Cause:**
- Multi-currency mismatch: payment and invoice must be in same currency
- Payment gateway connector not configured (Adyen, Stripe)
- Payment standard objects not visible — missing RCB license or permission setup
- `PayNow Payment Scheduler` configuration issue

**Resolution Steps:**
1. **Currency mismatch:** Verify payment currency matches invoice currency exactly
2. **Payment objects missing:** Confirm RCB license is active; verify Payments feature is enabled in Setup
3. **Gateway setup (Adyen/Stripe):**
   - Adyen: check connector is visible under Payment Gateway setup
   - Stripe: configure Merchant Account per documentation
   - Verify payment gateway provider, payment gateway, and payment method records are configured
4. **Saved payment method UNKNOWN_EXCEPTION:** Check if this is a known platform bug — search GUS; provide error ID to escalation team

**Escalation Criteria:**
- UNKNOWN_EXCEPTION with error ID on payment filtering → escalate with error ID

---

### Pattern 8: Milestone Billing Issues

**Frequency:** Medium (10+ cases)
**Severity:** P13–P45
**TTR Impact:** 5–10 days

**Symptoms:**
- `Milestone Billing is not getting enabled`
- `Billing Milestone Plans not getting created, so the Milestone Billing is not working.`
- `Unable to create more than 20 Billing Treatment Items/Billing Milestone Plan Items`
- `Billing Milestone Plans not behaving as expected`
- `Can Contract with Milestone Billing Plan be Amended?`
- `Platform event flow error updating billing milestone plan`

**Root Cause:**
- Hard limit: maximum 20 Billing Treatment Items / Milestone Plan Items per record
- Milestone billing requires specific billing treatment configuration
- Amendment behavior on milestone billing plans has known limitations

**Resolution Steps:**
1. Verify milestone billing is enabled in Setup (Billing Settings)
2. Confirm Billing Treatment is configured for Milestone type
3. For "not getting created": check platform event flow for errors; verify Billing Milestone Plan Item records are within the 20-record limit
4. For amendment questions: document current behavior — advise customer on supported amendment scenarios for milestone billing; if not supported, file VOC/feature request

**Escalation Criteria:**
- Milestone billing enabled, treatment configured correctly, still not creating → escalate to engineering

---

### Pattern 9: Usage Billing & DPE Issues

**Frequency:** Growing (15+ cases)
**Severity:** P45
**TTR Impact:** 5–14 days

**Symptoms:**
- `Revenue cloud DPE Create Usage Summary issue`
- `Usage BillingSchedule records not generated for VCSP anchor product order items (Summer 26)`
- `Rating failure` on usage summaries
- DPE not available
- `MaxDataProcessingTimePerMonth has been exceeded. Current value is 1804, and the limit is 1800`
- `Transaction Journal stuck in Pending status`
- Liable Summary not calculated correctly
- Fields not populating in General Entries

**Root Cause:**
- DPE not enabled or Data Pipeline Base User permission not assigned
- MaxDataProcessingTimePerMonth org limit exceeded (1800 min/month)
- Usage Billing setup sequence not followed correctly (TJ → Summarize → Rate → Liable Summary → Invoice)
- VCSP anchor product billing schedule not generating — Summer '26 known issue

**Resolution Steps:**
1. **DPE not available:**
   - Enable Data Pipelines from Setup
   - Assign Data Pipelines Base User permission set
2. **MaxDataProcessingTime exceeded:**
   - Contact Salesforce Support to request limit increase or review data processing volume
3. **Usage billing sequence:** Verify all steps are completed in order:
   1. Create TJ (Transaction Journal) records
   2. Summarize TJ records
   3. Rate Usage Summary records
   4. Liable Summary generation
   5. Invoice generation
4. **Transaction Journal stuck in Pending:** Check usage decision tables and context definitions; refresh usage decision tables
5. **VCSP anchor product (Summer '26):** Check if active GUS bug — search for known issue

**Escalation Criteria:**
- VCSP anchor product BS not generating after Summer '26 → confirm known issue, file GUS investigation
- MaxDataProcessingTime limit — customer needs increase → submit LAP/limit increase request

---

### Pattern 10: Billing Schedule Proration & Date Calculation Issues

**Frequency:** Medium (20+ cases)
**Severity:** P13–P27
**TTR Impact:** 3–7 days

**Symptoms:**
- `Incorrect proration of billing schedule groups, billing schedules, and billing period items`
- `New Billing Schedule did not calculate dates correctly`
- `Next Billing Date issue for Past Start Date Order`
- `Billing Schedules not created for past start date orders`
- `Invoice Generation Does Not Reflect Updated Billing Frequency on Order Product`
- `Billing frequency update issue during renewal`
- Negative billing schedule on amendment

**Root Cause:**
- Past start date orders: billing schedules not retroactively created by default
- Billing frequency changes on active orders don't automatically update existing billing schedules
- Amendment creates negative billing schedules when amount decreases — expected behavior for credit generation
- Date alignment issues: invoice target date must be ≥ BillingScheduleGroup EffectiveNextBillingDate

**Resolution Steps:**
1. **Past start date orders:** Use `createBillingSchedulesFromBillingTransaction` API with correct billing transaction ID; billing schedules for past periods may need manual backdating or separate handling
2. **Billing frequency not updating:** After changing billing frequency on order product, existing billing schedules may need to be updated or recreated; document behavior to customer
3. **Negative billing schedules on amendment:** Expected — negative BS generates credit memo; confirm this is correct behavior to customer
4. **Date alignment:** Verify target date on Billing Batch Scheduler is ≥ EffectiveNextBillingDate of all target BillingScheduleGroups

**Escalation Criteria:**
- Proration calculation is verifiably incorrect (wrong amount) with no customization → escalate with calculation detail

---

## C. Setup & Configuration Guide

### Complete Setup Checklist (New RCB Implementation)

- [ ] RCB license active on org
- [ ] Enable Data Pipelines from Setup → Data Pipelines
- [ ] Configure Legal Entity
- [ ] Configure Billing Policy
- [ ] Configure Billing Treatment + Billing Treatment Items
- [ ] Configure Tax Engine Provider (Salesforce standard or third-party: AvaTax/Vertex)
- [ ] Configure Tax Engine
- [ ] Configure Tax Policy + Tax Treatment
- [ ] Configure Payment Gateway (if using payments)
- [ ] Assign permission sets to all relevant users (see Pattern 5 table)
- [ ] Verify Quote → Order path is used (not Order direct)
- [ ] Configure Billing Batch Scheduler with correct charge type filter (use 'All' for now — active bug with combined filters)
- [ ] Test invoice generation with a single billing schedule before bulk run

### Common Misconfigurations

| Misconfiguration | Symptom | Fix |
|---|---|---|
| Billing Admin in PSG only | Cannot create Billing Batch Scheduler | Assign Billing Admin directly to user |
| Data Pipelines Base User missing | DPE failures, invoice stuck | Assign permission set |
| Context Definition mapping wrong (Vertex) | Invoice Draft in Progress, tax=0 | Fix App Usage Assignment — Asset Node only |
| Order created without Quote | Assets/BS not generated | Use Quote → Order workflow |
| Charge type = Recurring+Usage | Only Usage BS picked up | Use 'All' charge type (active bug) |
| Target date < EffectiveNextBillingDate | No matching BS in batch | Set target date ≥ EffectiveNextBillingDate |
| Max 20 Milestone Plan Items | Cannot create more items | Platform limit — file VOC for increase |
| Invoice Ingestion API with non-US address | State/country field errors | Provide state for non-US countries |

---

## D. Licensing & Entitlements

### RCB vs. RCA Capabilities

**Revenue Cloud Billing (RCB):**
- Invoicing & Recurring Billing
- Payment Schedules
- Collections
- Usage Billing

**Revenue Cloud Advanced (RCA) — required for:**
- Product Catalog Management
- Product Configurator / Browse Catalog
- Salesforce Pricing
- Advanced Approvals
- Subscription & Asset Lifecycle
- Dynamic Revenue Orchestrator (DRO)

### License Troubleshooting

```
Customer reports missing features/objects
│
├── Objects not visible → Check RCB license is active
├── Browse Catalog not available → RCA license required
├── Billing schedules not generating → Check feature enablement + permissions
└── Payment objects missing → Confirm Payments feature enabled in Setup
```

**Key Permission Sets:**
- `Billing Admin` — full billing administration (must be direct, not via PSG)
- `Billing Operations User` — day-to-day billing operations
- `Data Pipelines Base User` — required for DPE/invoicing engine
- `Create Billing Schedules From Billing Transactions API` — required for Order to Billing Schedule flow
- `Billing Experience Cloud User` — for Experience Cloud portal users

---

## E. Documentation Quick-Reference

| Topic | Link |
|---|---|
| Billing Overview | https://help.salesforce.com/s/articleView?id=ind.billing.htm&type=5 |
| Invoice Generation API | https://developer.salesforce.com/docs/atlas.en-us.revenue_lifecycle_management_dev_guide.meta/revenue_lifecycle_management_dev_guide/connect_resources_create_invoices_from_billing_schedules.htm |
| createBillingSchedulesFromBillingTransaction | https://developer.salesforce.com/docs/atlas.en-us.revenue_lifecycle_management_dev_guide.meta/revenue_lifecycle_management_dev_guide/actions_obj_create_billing_schedule_from_billing_transaction.htm |
| Invoice Ingestion API | https://developer.salesforce.com/docs/atlas.en-us.revenue_lifecycle_management_dev_guide.meta/revenue_lifecycle_management_dev_guide/connect_resources_invoices_ingestion.htm |
| Billing Apex ConnectAPI | https://developer.salesforce.com/docs/atlas.en-us.revenue_lifecycle_management_dev_guide.meta/revenue_lifecycle_management_dev_guide/apex_ConnectAPI_Billing_static_methods.htm |
| Order to Billing Schedule Flow Setup | https://help.salesforce.com/s/articleView?id=ind.billing_setup_clone_order_to_schedule_flow.htm&type=5 |
| Invoice Document Generation | https://help.salesforce.com/s/articleView?id=ind.billing_invoice_document_generation_batch.htm&type=5 |
| Apex Governor Limits | https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm |

### Known Design Decisions / Behavioral Rules
- Invoice and BillingBatchScheduler objects are system-managed — DML INSERT not supported in Apex; use ConnectAPI
- Billing Admin must be assigned directly (not via PSG) for Billing Batch Scheduler creation
- Maximum 2,000 invoice lines per invoice — exceeding this causes invoice generation failure
- Maximum 200 billing schedules can be recovered at a time via Recovery button
- Maximum 20 Billing Treatment Items / Milestone Plan Items per record
- Orders created directly without a Quote do not have full pricing context — use Quote → Order path
- Charge type filter bug (Recurring + Usage together) is an active platform bug as of 2026-05-27 — use 'All'
- Split invoicing (deposit + remaining balance independently) has no OOB solution — workaround via Invoice Ingestion API

---

## F. Escalation Paths

### When to Escalate
- Platform-level NullPointerException with no customization involved
- Invoices cannot be recovered after following Recovery process
- Tax engine integration (AvaTax/Vertex) fails after correct configuration verified
- Active known bugs with no available workaround
- MaxDataProcessingTime limit needs increase (requires Salesforce backend)
- "Billing: Create billing schedule groups and billing schedules" permission needs to be enabled/disabled at org level — requires LAP

### Information to Gather Before Escalating
1. Org ID + sandbox/production
2. Exact error message + Error ID from RTEL
3. Invoice Batch Run ID (5IR...)
4. Splunk/RTE log output (if accessible)
5. Steps to reproduce in a clean org
6. Screenshot of permission sets assigned to running user
7. Whether issue is new or regression (what changed recently)

### Key Escalation Contacts (from Slack)
- **#rlm-office-hours** — Engineering office hours for RLM billing questions (Sakshi Sharma, Prateek Sethi, Frank Shapiro)
- **#support-rev-rlm-global-swarm-help** — Swarm channel for active customer issues
- GUS investigation: file via #support-rev-rlm-global-swarm-help with org ID and error details

---

## G. Tribal Knowledge & Pro Tips

1. **Active Bug — Charge Type Filter (2026-05-27):** Using 'Recurring' + 'Usage' together in Billing Batch Scheduler only picks up Usage billing schedules. Use **'All'** as the charge type until engineering ships the fix. (Sakshi Sharma, #rlm-office-hours)

2. **Recovery Button First:** When invoices are stuck in "Draft in Progress," always try the Recovery button on the Invoice Batch Run before any other troubleshooting. It cleanly cancels stuck invoices and recovers billing schedules. (Gulla Bharath Kumar, #support-rev-rlm-global-swarm-help)

3. **Billing Admin via PSG Doesn't Work:** Many permission issues stem from Billing Admin being in a Permission Set Group but not assigned directly. Always verify the user has Billing Admin as a **direct** assignment. (Swapnil Rawat, #support-rev-rlm-global-swarm-help)

4. **Quote → Order is Required:** Customers who create Orders directly (without Quote) will face issues with pricing, asset creation, and billing schedule generation. This is a platform design requirement, not a bug. (Sai Varun Siddamsetty, #support-rev-rlm-global-swarm-help)

5. **Test Classes — No Direct Insert:** Billing objects (Invoice, BillingBatchScheduler) cannot be inserted directly in test classes. Use `ConnectApi.Billing.generateInvoices()` with the autoproc user and avoid `@seeAllData=true`. (Pradeepa P, #rlm-office-hours)

6. **Vertex Context Definition:** For Vertex tax integration failures, the first thing to check is Context Definition → Sales Transaction Node. App Usage Assignment mapping must only exist under the Asset Node. This single misconfiguration causes invoice-stuck-in-Draft-In-Progress issues. (Gulla Bharath Kumar, #support-rev-rlm-global-swarm-help)

7. **200-Line / 200-Schedule Limits:** Invoice generation fails if a single invoice would have >2,000 lines. Recovery is limited to 200 billing schedules at once. Use Draft-to-Posted IBR batch run (not manual Post button) for large invoices. (Gulla Bharath Kumar, #rlm-office-hours)

8. **Target Date Must Be ≥ EffectiveNextBillingDate:** If the Billing Batch Scheduler target date is before the BillingScheduleGroup EffectiveNextBillingDate, no billing schedules will match — no error will be thrown, just zero invoices. (Frank Shapiro, #rlm-office-hours)

9. **Split Invoicing Has No OOB Solution:** If a customer needs deposit invoice + remaining balance invoice generated independently (not tied together via Billing Arrangements), the only option is Invoice Ingestion API customization. Log a VOC. (Sakshi Sharma, #rlm-office-hours)

10. **Data Resync for HTTPS Exception:** If Billing Batch Scheduler fails with HTTPS exception, run a full data resync for the Billing Schedule object in Data Manager. Do not reconnect the user object. (Pratibha Rajpoot, #support-rev-rlm-global-swarm-help)

11. **NullPointerException in BS Creation (Renewal+Amendment):** Billing schedule creation NullPointerException typically occurs in narrow edge cases involving renewal + amendment combinations. Testing with OOTB flow (disabling custom flows) in sandbox isolates whether custom flows are the cause. ~99% of cases work correctly. (Pratibha Rajpoot, #support-rev-rlm-global-swarm-help)

12. **AvaTax Batch vs. Individual:** If tax calculation works individually but fails in batch (TAX_CALL_FAILURE with documentCode mismatch), this is a batch concurrency issue with AvaTax — retry individual invoices via API while investigating the batch. (Prasad Rao, #rlm-office-hours)

---

## H. Active Known Issues

*Connect to GUS for real-time P0/P1/P2 bugs. Query:*
```sql
SELECT Id, Name, Subject__c, Status__c, Priority__c 
FROM ADM_Work__c 
WHERE Product_Tag__r.Name = 'Revenue Cloud (Core)-Billing & Invoicing' 
AND Type__c = 'Bug' 
AND Status__c IN ('New', 'Triaged', 'In Progress') 
AND Priority__c IN ('P0', 'P1', 'P2') 
ORDER BY Priority__c, CreatedDate DESC LIMIT 20
```
*(Run via GUS CLI: `sf data query --query "..." --target-org gus`)*

**Known active as of 2026-05-27:**
- **Charge Type Filter Bug:** When 'Recurring' + 'Usage' selected together in Invoice Scheduler, only Usage billing schedules are picked up. Engineering working on fix. Workaround: use 'All'. (Case 473528619)
- **Usage BillingSchedule for VCSP anchor products (Summer '26):** Billing schedules not generated for VCSP anchor product order items after Summer '26 release. (Case 473523664)

---

## I. Code Reference Map

*CodeSearch is not currently connected. To get code-level root cause references for issues like NullPointerException traces, error message origins, and DPE pipeline class names, connect CodeSearch in AI Suite settings (Settings → MCP Servers → CodeSearch).*

**Key classes/components to search for when CodeSearch is available:**
- `InvoicingStep` — invoice orchestration pipeline (appears in Splunk logs)
- `PlaceSalesTransactionStepsExecutorSync` — sales transaction execution (quote/order context)
- `createBillingSchedulesFromBillingTransaction` — billing schedule creation action
- `InvoiceBatchRun` — batch run orchestration
- `BillingBatchScheduler` — scheduler logic
- `ConnectAPI.Billing` — Apex billing methods

---

## Update Cadence

- Refresh every 4 weeks with new case data from orgcs
- After every Salesforce release (Summer/Winter/Spring)
- When new Known Issues are published in GUS
- When Slack surfaces new recurring patterns in #rlm-office-hours or #support-rev-rlm-global-swarm-help

---

## Test Cases

1. **Billing schedules not created after order activation** → Triage: check if order created via Quote → Order path; check permissions; try API; test OOTB flow in sandbox
2. **Invoice stuck in "Draft in Progress"** → Triage: check RTEL → identify if tax error or DPE error → Recovery button → single invoice API test → full batch run
3. **Billing Batch Scheduler picks up no billing schedules** → Triage: check charge type filter (bug — use 'All'); verify target date ≥ EffectiveNextBillingDate; verify billing schedules are in "Ready for Invoicing" status
4. **Cannot create Billing Batch Scheduler (permission error)** → Triage: verify Billing Admin is assigned DIRECTLY (not via PSG) + Data Pipelines Base User
5. **Tax calculation shows 0.00 with Vertex** → Triage: check Context Definition → Sales Transaction Node for App Usage Assignment misconfiguration; verify Billing Admin + Data Pipeline Base User permission sets

---

## Completeness Report

```
=== SKILL COMPLETENESS REPORT ===

✅ Case data: 500 cases analyzed, 10 patterns identified
❌ CodeSearch: not connected — no code references available
   → Connect CodeSearch in AI Suite settings for code-level root cause tracing
❌ GUS active bugs: query failed (object type validation error in orgcs)
   → Run GUS query directly: sf data query --target-org gus
✅ Slack: 60+ tribal knowledge items from #rlm-office-hours + #support-rev-rlm-global-swarm-help
✅ Help Docs: https://help.salesforce.com/s/articleView?id=ind.billing.htm&type=5

GAPS:
- No code references (CodeSearch not connected)
- GUS active P0/P1/P2 bugs not automatically fetched
- Patterns without code-level root cause: all patterns

RECOMMENDATION:
- Connect CodeSearch in AI Suite settings to add code references for NullPointerException traces and DPE error origins
- Run the GUS query manually to populate Section H with current active bugs
```