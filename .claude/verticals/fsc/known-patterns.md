## A. Triage & Classification

When a support engineer brings an FSC issue, first identify the product area:

```
Is the issue about...

├── ACTION PLANS
│   ├── "Couldn't create successor tasks for task" error
│   ├── Task dependencies not progressing
│   ├── Template reordering / DisplayOrder issues
│   ├── Action Plan PSL disabled or unavailable
│   ├── DocumentChecklistItem entity error
│   ├── Template deployed as ReadOnly
│   ├── Guest user cannot create Action Plans
│   ├── Mobile component not visible
│   └── Sales Action Plans vs FSC Action Plans confusion
│   → Go to: Section B (Action Plan Patterns 1-10)
│
├── REFERRALS
│   ├── Referral not showing on record page
│   ├── Referral routing not working (to correct user/queue)
│   ├── Referral status not updating
│   ├── Duplicate referrals being created
│   ├── Referral permission / sharing issue
│   └── Referral reports not showing data
│   → Go to: Section K (Referral Patterns)
│
├── ROLLUP SUMMARY RULES (RSR)
│   ├── RSR not calculating / stale values
│   ├── RSR not visible in Setup
│   ├── RSR hitting limits (max 20 per object)
│   ├── Field not available for RSR mapping
│   ├── RSR runs but value incorrect
│   └── RSR performance / async calculation delays
│   → Go to: Section L (RSR Patterns)
│
├── FINANCIAL ACCOUNTS & HOLDINGS
│   ├── Financial Account not visible on client record
│   ├── Financial Account roles / ownership issues
│   ├── Account hierarchy / household rollup missing
│   ├── Holdings not showing or updating
│   ├── FinancialAccount object permissions
│   └── Financial Account sharing rules
│   → Go to: Section M (Financial Account Patterns)
│
├── FSC TIMELINE
│   ├── Timeline component not showing records
│   ├── Specific object not appearing in Timeline
│   ├── Timeline configuration errors
│   └── Timeline performance issues
│   → Go to: Section N (Timeline Patterns)
│
├── HOUSEHOLD & GROUP MANAGEMENT
│   ├── Person Account vs Household confusion
│   ├── Group member roles missing or incorrect
│   ├── Relationship map not rendering
│   ├── Household rollup not calculating
│   └── Contact sharing within household
│   → Go to: Section O (Household Patterns)
│
├── INTERACTION SUMMARIES
│   ├── Interaction Summary not saving
│   ├── Related records not linking to summary
│   ├── Interaction Summary permissions
│   └── Summary not appearing on timeline
│   → Go to: Section P (Interaction Summary Patterns)
│
├── GOALS & LIFE EVENTS
│   ├── Goal not showing on client record
│   ├── Life Event templates not available
│   ├── Life Event not triggering automation
│   └── Goal tracking / progress calculation
│   → Go to: Section Q (Goals & Life Events Patterns)
│
└── FSC DATA MODEL & SHARING
    ├── OWD / sharing rule misconfiguration
    ├── Guest user access issues
    ├── Custom sharing conflicts
    └── FSC object permissions matrix
    → Go to: Section R (FSC Data Model & Sharing)
```

### Quick Diagnostic Questions

Ask these to narrow down quickly:
1. **What FSC feature area?** (Action Plans, Referrals, RSR, Financial Accounts, Timeline, Household, etc.)
2. **What is the exact error message?** Full text, including any ErrorId/Gack ID (format: `XXXXXXXXXX-XXXXX` or `XXXXXXXXXX-XXXXX (negative_number)`). **If they have an ErrorId → immediately use Columbo to look it up (Section J).**
3. **What user profile/permission set?** System Admin or specific profile?
4. **Production or Sandbox?**
5. **When did this start?** After a release/deployment/sandbox refresh?
6. **What objects are involved?** (specific FSC objects, custom fields, automations)

### Deep Investigation Steps (Use Connected Tools)

If tools are connected, perform these before diagnosing:

**With Columbo (Gack/Error Lookup) — USE WHEN CUSTOMER PROVIDES AN ERROR ID:**

If the customer provides a Gack/ErrorId (format: `XXXXXXXXXX-XXXXX (negative_number)` or just `XXXXXXXXXX-XXXXX`):

```
Step 1: Look up the specific gack
Tool: mcp__plugin_columbo_columbo__fetch_gack_details
Input: gack_id = "<the_gack_id>" (e.g., "1770957752-49415")

Step 2: If specific lookup returns empty (gack expired), search by subject keyword
Tool: mcp__plugin_columbo_columbo__search_gacks
Input: subject = "ActionPlan" (or specific error keyword)
       limit = 50

Step 3: For deeper analysis of a stack trace pattern
Tool: mcp__plugin_columbo_columbo__fetch_bulk_gacks_by_stacktrace_ids
Input: stacktrace_ids = [<ids from search results>]
```

**What Columbo tells you:**
- **Stack trace source** → exact Java class and method where the error originated
- **Exception type** → SfdcSqlException, NullPointerException, etc.
- **Frequency** → how many production orgs are hitting this
- **Recency** → when the error last occurred
- **Stack trace ID** → links related gacks across orgs (same root cause)

**Known Action Plan Gack Patterns (from Columbo):**

| Pattern | Source Class | Method | Exception | Stack Trace IDs |
|---------|------------|--------|-----------|----------------|
| SQL error during AP creation | `actionplan.entity.ActionPlanTemplateObject` | `populateDerivedStatusField` | SfdcSqlException | -398836090, -1810462813, 1474149669, -188873150 |
| NullPointer on template publish/deactivate | `ui.industries.actionplan.components.editor.HighlightsPanelController` | `publishTemplate`, `deactivateTemplate` | NullPointerException ("actionPlanTemplateItemService is null") | 51869049, 1987205671 |

→ If the gack matches a known pattern, skip to the resolution. If it's new, escalate with the full stack trace.

**With CodeSearch (verified — repo: gitcore.soma.salesforce.com/core-2206/core-260-public):**
```
mcp__plugin_deep-research_codesearch__search
query: content:"<exact_error_message_from_customer>" repo:"gitcore.soma.salesforce.com"

Key module: core/industries-actionplan-impl/java/src/actionplan/
Key classes:
  - ActionPlanDependentTaskServiceImpl.java → successor task logic
  - ActionPlanItemUtil.java → item state transitions
  - IndustriesActionPlanServiceImpl.java → plan/item creation
  - ActionPlanTemplateObject.java → template validation
```
→ Traces the error to the exact Java class that throws it.

**With GUS:**
```bash
sf data query --query "SELECT Id, Name, Subject__c, Status__c, Priority__c FROM ADM_Work__c WHERE Product_Tag__r.Name = 'FSC Action Plans' AND Type__c = 'Bug' AND Status__c NOT IN ('Closed', 'Duplicate', 'Never Fix') ORDER BY Priority__c, CreatedDate DESC LIMIT 10" --target-org gus --json
```
→ Checks if this is a known open bug.

**With Slack:**
```
Search: "in:#tech-prod-help-financial-services-cloud <error_message_or_keyword>"
```
→ Finds if someone already solved this recently.

---

## B. Action Plan Known Issue Patterns

---

### Pattern 1: "Couldn't create successor tasks for task"

**Frequency:** High — multiple P1/P2 cases  
**Severity:** Level 1-2 (Critical/Urgent)  
**Average TTR:** 54 days (worst case), 21 days (typical)  
**GUS References:** W-21482069 (TTR: 54d), W-21228390 (TTR: 21d)

**Symptoms:**
- Error: `Couldn't create successor tasks for the action plan`
- Error: `Insert failed. First exception on row 0; first error: REQUIRED_FIELD_MISSING, Required fields are missing: [Name]`
- Action Plan shows as "stuck" — no tasks progress
- Advisors missing key tasks

**Root Causes (from GUS investigations + case resolutions):**

| # | Root Cause | Frequency |
|---|-----------|-----------|
| 1 | Corrupted ActionPlanTemplateItem — orphaned/corrupted template item records | Most common |
| 2 | Workflow rule interfering with Sales Action Plans functionality | Common |
| 3 | FinServ.TaskTrigger exceeding SOQL limits in bulk operations | Medium |
| 4 | Account Team Member not mapped with "Account Owner" role | Medium |
| 5 | Custom process inserting items with missing dependencies | Less common |

**Code Reference (from CodeSearch — verified):**

```
Primary class: ActionPlanDependentTaskServiceImpl.java
Repo: gitcore.soma.salesforce.com/core-2206/core-260-public
Path: core/industries-actionplan-impl/java/src/actionplan/service/ActionPlanDependentTaskServiceImpl.java
Scrum Team: FSC-Darksaber | Author: b.sugiarto | Since: release 232

This class:
- Resolves ActionPlanTemplateItemDependencies to create successor tasks
- Queries dependency graph via getActionPlanTemplateItemDependencies(versionId, soap)
- FAILS when template items are corrupted/orphaned (cannot resolve dependency chain)

Supporting classes:
- ActionPlanItemUtil.java → getChildItemIdToUncompletedParentMap() — checks parent completion
- ActionPlnTmplItmDependencyFunctions.java → cycle detection in dependencies
- IndustriesActionPlanServiceImpl.java → createActionPlanItems() — initial item creation

State machine (ActionPlanItemDependencyStatusEnum):
- None → item has no dependencies, can proceed
- WaitingOnPrevious → waiting for parent(s) to complete

Search to trace further:
  content:"ActionPlanDependentTaskServiceImpl" lang:java repo:"gitcore.soma.salesforce.com"
```

**Resolution Steps:**

1. **Check for corrupted template items:**
   ```sql
   SELECT Id, Name, ActionPlanTemplateVersionId, IsRequired, DisplayOrder 
   FROM ActionPlanTemplateItem 
   WHERE ActionPlanTemplateVersionId = '<template_version_id>'
   ```
   Look for: items with NULL Name, orphaned references, duplicate DisplayOrder values.
   → **Fix:** Delete corrupted records.

2. **Check for interfering automation:**
   - Setup → Workflow Rules → filter on Task object
   - Setup → Process Builder → look for processes on Task/ActionPlanItem
   - Setup → Flows → look for record-triggered flows on Task
   → **Fix:** Temporarily deactivate and test. If resolves, refactor the automation.

3. **Check SOQL limits (if bulk operation):**
   - Is a flow creating multiple Action Plans in one transaction?
   - Each Action Plan orchestrates child records independently — not bulk
   → **Fix:** Add bulkification or serialize Action Plan creation.

4. **Check Account Team mapping:**
   - Is the task assigned to "Account Owner" role?
   - Does the Account have a Team Member with that role?
   → **Fix:** Add Account Team Member with correct role.

**Verification:** Create a new Action Plan from the same template. Complete parent task. Confirm successor tasks appear.

**Workarounds (from Slack #tech-prod-help-financial-services-cloud):**
- If template is corrupted beyond repair: Clone the template → delete old one → publish clone
- If workflow interference: Use "Run in System Context without sharing" for the interfering flow
- FinServ.TaskTrigger cannot be modified (managed package) — reduce what triggers it

**Escalation Criteria:**
- Corrupted records cannot be identified or deleted → Escalate to FSC Engineering
- FinServ.TaskTrigger SOQL issue with no workaround → Escalate with debug logs
- Gather: Org ID, Template ID, debug logs from failing operation, SOQL query results

---

### Pattern 2: "DocumentChecklistItem an invalid action plan template entity"

**Frequency:** High — recurring across many orgs  
**Severity:** Level 2 (Urgent)  
**Average TTR:** 10 days

**Symptoms:**
- Error: `Review the errors on this page. DocumentChecklistItem an invalid action plan template entity.`
- Users unable to create Action Plans
- Appears even with proper sharing rules and permissions

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Using Sales Action Plans but template has Document Checklist Items | Check Setup → Action Plans Settings |
| 2 | User lacks CRUD on DocumentChecklistItem object | Check Profile/Permission Set |
| 3 | Template has Document Checklist Items in FSC context but org using Sales context | Compare template type vs org setting |

**Code Reference (from CodeSearch — verified):**
```
Type determination: ActionPlanTemplateObject.java
Path: core/actionplan/java/src/actionplan/entity/ActionPlanTemplateObject.java
- Default ActionPlanType = ActionPlanTemplateTypeEnum.INDUSTRIES
- Template type determines which item types (Task/Event/DocChecklist) are supported

Type enum: ActionPlanTemplateTypeEnum
Values: INDUSTRIES (full FSC), ADMIN_CREATED, SALES (Account Plans only)
- SALES type does NOT support DocumentChecklistItem as an item entity type
- Only INDUSTRIES type includes DocumentChecklistItem support

Validation: ActionPlanTemplateItemTestingUtil.java
- setDisplayOrder(), setIsRequired(), setItemEntityType() — item types enforced at creation

Search to trace further:
  content:"ActionPlanTemplateTypeEnum" lang:java repo:"gitcore.soma.salesforce.com"
  content:"DocumentChecklistItem" content:"ActionPlan" lang:java repo:"gitcore.soma.salesforce.com"
```

**Critical Design Knowledge:**
> "The 'Sales' Action Plan type was originally developed specifically to support the Account Plan feature. Because the initial requirements for Account Plans only necessitated Task and Event orchestration, the 'Sales' engine was not built to include Document Checklist functionality."

| Feature | Sales Action Plans | FSC Action Plans |
|---------|-------------------|-----------------|
| Tasks | ✅ | ✅ |
| Events | ✅ | ✅ |
| Document Checklist Items | ❌ NOT SUPPORTED | ✅ |
| Auto-status update | ❌ Manual only | ✅ |
| Standard flow support | Limited | Full |

**Resolution Steps:**

1. **Determine Action Plan type in org:**
   - Setup → Action Plans Settings
   - Is "Enable Sales Action Plans" or "Enable Action Plans" toggled?

2. **If Sales Action Plans (most common cause):**
   - Document Checklist Items are NOT supported — period.
   - Remove Document Checklist Items from the template
   - OR switch to FSC Action Plans (requires FSC license)

3. **If FSC Action Plans but still failing:**
   - Create/assign permission set with CRUD on `DocumentChecklistItem`
   - Verify template version is Published (not Draft/Obsolete)
   - Check user's profile for explicit denials on the object

4. **Refresh user session after permission changes** (logout/login)

**Verification:** Create Action Plan from template → Document Checklist Items appear without error.

**Escalation Criteria:** If org needs Document Checklist + is on Sales Action Plans and cannot switch → Consult Product Management on roadmap.

---

### Pattern 3: Action Plan Permission Set License (PSL) Disabled

**Frequency:** High — most common licensing issue  
**Severity:** Level 2-3  
**Average TTR:** 5-15 days  
**GUS References:** W-22248225, W-22308298

**Symptoms:**
- Action Plan features not available in org
- "Action Plans" permission set not visible to assign
- PSL showing as "disabled" in Company Information
- Negative PSL counts
- Features vanish after license renewal/transition

**Root Causes:**
1. PSL not provisioned to sandbox (sandboxes don't auto-inherit on renewal)
2. PSL disabled after FSC managed package → core transition
3. License value on permission set = "Salesforce" instead of blank/"FSC Sales"
4. Incomplete org provisioning during FSC setup
5. Inactive users consuming all available PSL slots

**Code Reference (if CodeSearch connected):**
```
Search: content:"ActionPlansPsl" content:"PermissionSetLicense"
Search: content:"ActionPlanAccess" content:"Permission"
```

**Resolution Steps:**

```
User can't assign Action Plans permission?
│
├── Step 1: Check PSL exists
│   Setup → Company Information → Permission Set Licenses
│   Look for "ActionPlansPsl" or "Financial Services Action Plans"
│   │
│   ├── NOT LISTED → License never provisioned
│   │   → Is this Production or Sandbox?
│   │   ├── Sandbox → Refresh sandbox from Production
│   │   └── Production → Contact License Provisioning team
│   │
│   └── LISTED → Check status
│       │
│       ├── Status = "Disabled"
│       │   → Cannot enable via UI
│       │   → Escalate to Support/License team
│       │
│       └── Status = "Active" → Check Available count
│           │
│           ├── Available = 0
│           │   → Check: Are inactive users consuming slots?
│           │   → sf data query: "SELECT AssigneeId FROM PermissionSetAssignment 
│           │     WHERE PermissionSet.Name = 'ActionPlans'"
│           │   → Remove from inactive users
│           │
│           └── Available > 0 → Permission set itself is the issue
│               → Check License field on the Permission Set
│               → If "Salesforce" → change to blank or "FSC Sales"
│               → The permission will then appear assignable
```

**Verification:**
- Setup → Company Information → PSL Active with count > 0
- Can assign "Action Plans" permission set to user
- User can create/view Action Plans

**GUS Investigation (if connected):**
```bash
sf data query --query "SELECT Name, Subject__c, Resolution__c FROM ADM_Work__c WHERE Name IN ('W-22248225', 'W-22308298')" --target-org gus --json
```

**Pro Tip (from Slack):** After ANY license renewal or transition, always verify PSL status in Company Information. Don't just check the permission set — the PSL itself may be disabled at the org level even though the permission set exists.

---

### Pattern 4: Task Reordering Issues (DisplayOrder)

**Frequency:** Medium-High  
**Severity:** Level 3-4  
**Average TTR:** 5-8 days

**Symptoms:**
- Tasks change order after publishing template
- Tasks not saving in reordered sequence
- "Reorder" button does nothing
- Random order after Action Plan creation
- Subject sequence doesn't match template

**Root Cause:** The `DisplayOrder` field on `ActionPlanTemplateItem` is the authoritative sequence field. If not explicitly set or has gaps/duplicates, order is unpredictable.

**Code Reference (if CodeSearch connected):**
```
Search: content:"DisplayOrder" content:"ActionPlanTemplateItem"
Search: content:"ActionPlanTemplateItem" content:"ORDER BY"
```

**Resolution Steps:**

1. Query current DisplayOrder values:
   ```sql
   SELECT Id, Name, DisplayOrder, ItemSubject 
   FROM ActionPlanTemplateItem 
   WHERE ActionPlanTemplateVersionId = '<version_id>' 
   ORDER BY DisplayOrder
   ```

2. Check for problems:
   - NULL values? → Items will appear at random position
   - Duplicate values? → Order within duplicates is undefined
   - Gaps (1, 2, 5, 10)? → Works but can cause confusion during reorder

3. Fix:
   - Update all items to sequential integers (1, 2, 3, 4...)
   - No gaps, no duplicates, no NULLs
   - Use Data Loader or Workbench to update

4. Re-publish the template after fixing

**For Data Load scenarios:** ALWAYS include `DisplayOrder` in your data load CSV. Set sequential integer values starting from 1.

**Verification:** Create new Action Plan from template. Tasks appear in correct order.

---

### Pattern 5: Action Plan Not Visible on Mobile

**Frequency:** Medium  
**Severity:** Level 2-3  
**Average TTR:** 1-5 days

**Symptoms:**
- Component visible on desktop, invisible in Salesforce Mobile
- Search for "Action Plan" in mobile returns nothing
- Action Plan List component missing entirely

**Root Cause:** Standard Action Plan List component has limited mobile support. Mobile rendering uses different component visibility rules.

**Resolution Steps:**
1. Check Lightning Record Page → Form Factor settings:
   - Edit page → select component → check "Mobile" form factor
2. Mobile Navigation: Setup → Mobile Navigation → include Action Plans
3. Search: Setup → Search Layouts → ensure ActionPlan is searchable
4. Alternative: Use "Related List - Single" component pointing to Action Plans (better mobile support)

**Known Limitation:** Action Plan List standard component does NOT support column customization (unlike Action Plan Item List). This is by design — confirmed with engineering.

---

### Pattern 6: Templates Deployed as ReadOnly

**Frequency:** Medium  
**Severity:** Level 2-3  
**Average TTR:** 5-8 days

**Symptoms:**
- Templates migrated from sandbox → production arrive as "ReadOnly"
- Cannot use, publish, or delete ReadOnly templates
- Must clone every template manually

**Root Cause:** Deployment behavior by design — Action Plan Templates deployed via change sets or metadata API arrive as ReadOnly. Published status doesn't transfer.

**Resolution Steps:**
1. Post-deployment: Navigate to each template → "Create Version" → Publish
2. For bulk: Use API/Data Loader to update status
3. To delete ReadOnly templates: API delete (may require disabling validation rules)
4. **Prevention:** Include "Publish" step in deployment runbook

**Workaround:** Deploy templates → run post-deployment script to auto-publish all templates in target org.

---

### Pattern 7: Guest User / Experience Cloud Cannot Create Action Plans

**Frequency:** Medium  
**Severity:** Level 1-2 (Critical — blocks customer-facing processes)  
**Average TTR:** 14 days

**Symptoms:**
- Action Plan not created when guest user submits form
- Flow-triggered AP creation fails with permission error
- "insufficient access rights" on ActionPlan creation

**Root Cause:** EC Guest Service Account user runs on Salesforce Integration User license which doesn't support ActionPlan CRUD.

**Code Reference (from CodeSearch — verified):**
```
Creation entry point: IndustriesActionPlanServiceImpl.java
Path: core/industries-actionplan-impl/java/src/actionplan/service/IndustriesActionPlanServiceImpl.java
- createActionPlanItems() requires CRUD on ActionPlanItem entity
- Uses UddDb.createEntityObject(ActionPlanItemUddConstants.EntityId)
- Guest/Integration User license lacks entity-level permissions for ActionPlan objects

Flow-based creation: ActionPlanBuilder.java
Path: core/actionplan/java/src/actionplan/ActionPlanBuilder.java  
- Generates FlowRecordCreate nodes — flow execution context determines permissions
- If flow runs as guest user → creation fails at DML level

Search to trace further:
  content:"ActionPlan" content:"Permission" lang:java repo:"gitcore.soma.salesforce.com"
```

**Resolution Steps:**
1. Identify flow running user → if guest context, it lacks ActionPlan access
2. Solutions:
   - Set flow to "Run in System Context (without sharing)" — Setup → Flows → Edit → Run Mode
   - Use Platform Event: guest fires event → admin-context flow creates AP
   - Use Apex Invocable with `without sharing`
3. Verify permissions on running user: ActionPlan (CRUD), ActionPlanItem (CRUD), Task (CRUD)

---

### Pattern 8: Sales Action Plans vs FSC Action Plans Confusion

**Frequency:** HIGH — root cause of many mis-diagnosed issues  
**Severity:** Varies

**Symptoms:**
- Expected features not available
- Status not auto-updating after task completion
- Document Checklist Items not working
- "Click Update Action Plan Status" flow not working

**Root Cause:** Customer has "Sales Action Plans" enabled but expects "FSC Action Plans" behavior.

**Key Differences:**

| Capability | Sales Action Plans | FSC Action Plans |
|-----------|-------------------|-----------------|
| Designed for | Account Plan feature | Full process orchestration |
| Tasks | ✅ | ✅ |
| Events | ✅ | ✅ |
| Document Checklist Items | ❌ | ✅ |
| Auto-status update on task completion | ❌ (manual/custom only) | ✅ (flow-based) |
| "Click Update Action Plan Status" flow | ❌ Not supported | ✅ |
| License | Sales Cloud | FSC + PSL |

**Resolution:**
1. Setup → Action Plans Settings → check which is enabled
2. If needs FSC capabilities → verify FSC license → enable FSC Action Plans
3. If must stay on Sales → educate on limitations, offer custom automation alternatives

---

### Pattern 9: Inactive User Causing Cloning/Sync Errors

**Frequency:** Medium  
**Severity:** Level 2-3

**Symptoms:**
- Error: `operation performed with inactive user [ID] as owner of actionPlanTemplateItemValue`
- Records cannot sync when owner deactivated
- Cloning fails referencing inactive user

**Resolution:**
1. Identify inactive user from error (User ID)
2. **Workaround:** Temporarily reactivate user → reassign ownership → deactivate again
3. **Prevention:** Before deactivating users, reassign their Action Plan items

---

### Pattern 10: Action Plan Item List Component Not Auto-Refreshing

**Frequency:** Medium  
**Severity:** Level 3-4

**Symptoms:**
- After updating a task, FSC "Action Plan Item List" component doesn't refresh
- Users must manually reload page

**Resolution:**
1. Verify it's the standard FSC component
2. Check Known Issues for "Action Plan Item List refresh"
3. Workaround: manual page refresh after updates
4. If regression after release → File GUS bug against FSC Action Plans team

---

## C. Setup & Configuration Guide

### Complete Setup Checklist

```
1. LICENSE VERIFICATION
   □ FSC license provisioned (Setup → Company Information)
   □ Action Plan PSL status = Active, count > 0
   □ "Action Plans" permission set exists

2. PERMISSION SETS (assign to all Action Plan users)
   □ ActionPlan: Create, Read, Edit, Delete
   □ ActionPlanItem: Create, Read, Edit, Delete
   □ ActionPlanTemplate: Read (end users) / CRUD (admins)
   □ ActionPlanTemplateVersion: Read
   □ ActionPlanTemplateItem: Read
   □ Task: Create, Read, Edit
   □ Event: Create, Read, Edit (if using Events)
   □ DocumentChecklistItem: CRUD (if using Document Checklists)
   □ "Manage Financial Services Standard Objects" permission (admins)

3. SETTINGS
   □ Setup → Action Plans Settings → Enable correct type
   □ Only ONE type can be active (Sales OR FSC, not both)

4. PAGE LAYOUTS
   □ "Action Plans" related list on record pages
   □ "Action Plan" Lightning component on record pages
   □ "Unique Name" field ON the Action Plan Template page layout
     (CRITICAL — template fails without this)

5. TEMPLATE CREATION
   □ Create template → Add items → Set DisplayOrder (1,2,3...)
   □ Configure dependencies (parent-child)
   □ Mark required items → Publish

6. SHARING
   □ OWD for ActionPlan set appropriately
   □ Templates accessible to users who create plans
```

### Common Misconfigurations

| Misconfiguration | Error/Symptom | Fix |
|-----------------|---------------|-----|
| "Unique Name" not on layout | Template blank/error on load | Add to page layout |
| DisplayOrder not set | Random task order | Set sequential values |
| DocumentChecklistItem CRUD missing | "invalid action plan template entity" | Permission set with CRUD |
| PSL disabled | Features unavailable | Contact support to enable |
| Sales AP enabled instead of FSC | Missing features | Switch to FSC (needs license) |
| Template not Published | Can't create plans from it | Publish template version |
| Account Team Member missing | Tasks assign to wrong person | Map team member with role |

---

## D. Licensing & Entitlements

### PSL Troubleshooting Flowchart

```
Can't access Action Plans features?
│
├── Check: Setup → Company Information → Permission Set Licenses
│   │
│   ├── "ActionPlansPsl" NOT listed
│   │   ├── Sandbox? → Refresh from Production
│   │   └── Production? → License not provisioned → Contact License team
│   │
│   ├── Status = "Disabled"
│   │   └── Escalate to Support (cannot self-enable)
│   │
│   └── Status = "Active"
│       ├── Available = 0 → Remove from inactive users
│       └── Available > 0 → Check Permission Set license field
│           └── If "Salesforce" → change to blank or "FSC Sales"
│
└── PSL looks fine but still can't use?
    ├── Check: Is "Action Plans" permission set ASSIGNED to user?
    └── Check: Does user's profile explicitly DENY ActionPlan object access?
```

---

## E. Key Behavioral Design Rules

These are **by design** — not bugs. Educate the customer:

1. Dependent tasks are created ONLY when **ALL** parent tasks are completed
2. A parent task MUST be marked "Required" in template — cannot be deleted if has dependents
3. Tasks are NOT created in bulk across Action Plans — each AP orchestrates independently
4. "Not Required" status CANNOT cascade to dependent child tasks
5. Sales Action Plans do NOT auto-update status — manual or custom automation only
6. Action Plan List component does NOT support column customization
7. Templates deployed via change sets arrive as ReadOnly — must publish post-deploy
8. FinServ.TaskTrigger is managed package — cannot be edited by customers

---

## F. Escalation Paths

### When to Escalate

| Scenario | Team | Priority |
|----------|------|----------|
| Corrupted template items, can't delete | FSC Engineering (Dhaval-Thakker) | P2 |
| PSL disabled, can't enable | License Provisioning | P2 |
| FinServ.TaskTrigger SOQL limits | FSC Engineering | P2 |
| Regression after release | FSC Engineering | P1 |
| Feature request | Product Management | P4 |
| Security concern (permissions) | Security team | P1 |

### Gather Before Escalating

```
□ Org ID (18-char)
□ User ID(s) affected
□ Exact error message (full text + ErrorId)
□ Steps to reproduce (numbered)
□ Template ID + Version ID
□ Debug logs (Apex + System) from failing operation
□ Screenshots: Template config, error state, permission sets
□ Custom automation on Task/ActionPlan/ActionPlanItem objects
□ Last known working date (if regression)
```

### High-TTR Investigation References

| Work ID | TTR | Investigation | Issue Area |
|---------|-----|---------------|-----------|
| W-21482069 | 54 days | 26.82 days | Complex successor task failure |
| W-19402617 | 36 days | 0.41 days | Template dependency corruption |
| W-21185271 | 27 days | 0.98 days | Action Plan configuration |
| W-19835821 | 24 days | 5.11 days | Deep troubleshooting |
| W-21228390 | 21 days | 1.37 days | Task behavior |

**To pull details (if GUS connected):**
```bash
sf data query --query "SELECT Name, Subject__c, Details__c, Resolution__c, Root_Cause_Brief__c FROM ADM_Work__c WHERE Name = 'W-21482069'" --target-org gus --json
```

---

## G. Tribal Knowledge & Pro Tips

*Source: #tech-prod-help-financial-services-cloud, #help-sell-financial-services-cloud*

**To get fresh tips (if Slack connected), search:**
```
in:#tech-prod-help-financial-services-cloud action plan workaround
in:#tech-prod-help-financial-services-cloud action plan fix
in:#help-sell-financial-services-cloud action plan
```

**Embedded tips (from past Slack discussions):**

1. **Always check Sales vs FSC first** — #1 misdiagnosis. Different features, different behaviors.
2. **"Unique Name" field is sneaky** — Not on layout = template appears to save but fails on load.
3. **Sandbox PSL never auto-inherits** — After production license renewal, refresh sandbox.
4. **Post-deploy = post-publish** — Deployed templates are ReadOnly. ALWAYS publish after deploy.
5. **DisplayOrder is king** — Wrong task order? Check this field first. Always.
6. **Deactivating users = reassign first** — Check AP item ownership before deactivating anyone.
7. **Reporting workaround** — Can't report on Template Version? Create custom lookup field on ActionPlan → ActionPlanTemplateVersion.
8. **Account Team = task assignment** — Wrong assignee? Check Account Team Member mapping.
9. **Bulk = governor limits** — Each AP orchestrates independently. Creating 50 APs in one flow = pain.
10. **FinServ.TaskTrigger = untouchable** — Managed package. Reduce what triggers it instead.

---

## H. Active Known Issues (Live Lookup)

**If GUS is connected, run this query to get current open bugs for a specific FSC area:**

```bash
# Action Plans
sf data query --query "SELECT Name, Subject__c, Priority__c, Status__c, CreatedDate FROM ADM_Work__c WHERE Product_Tag__r.Name = 'FSC Action Plans' AND Type__c = 'Bug' AND Status__c NOT IN ('Closed', 'Duplicate', 'Never Fix', 'Not a Bug') ORDER BY Priority__c, CreatedDate DESC LIMIT 15" --target-org gus --json

# Referrals
sf data query --query "SELECT Name, Subject__c, Priority__c, Status__c, CreatedDate FROM ADM_Work__c WHERE Product_Tag__r.Name LIKE '%FSC%Referral%' AND Type__c = 'Bug' AND Status__c NOT IN ('Closed','Duplicate','Never Fix','Not a Bug') ORDER BY Priority__c DESC LIMIT 10" --target-org gus --json

# RSR / Rollup
sf data query --query "SELECT Name, Subject__c, Priority__c, Status__c, CreatedDate FROM ADM_Work__c WHERE (Subject__c LIKE '%Rollup Summary%' OR Subject__c LIKE '%RSR%') AND Product_Tag__r.Name LIKE '%FSC%' AND Type__c = 'Bug' AND Status__c NOT IN ('Closed','Duplicate','Never Fix') ORDER BY Priority__c DESC LIMIT 10" --target-org gus --json

# Financial Accounts / Holdings
sf data query --query "SELECT Name, Subject__c, Priority__c, Status__c, CreatedDate FROM ADM_Work__c WHERE (Subject__c LIKE '%FinancialAccount%' OR Subject__c LIKE '%Financial Account%') AND Type__c = 'Bug' AND Status__c NOT IN ('Closed','Duplicate','Never Fix') ORDER BY Priority__c DESC LIMIT 10" --target-org gus --json

# All open FSC bugs (broad sweep)
sf data query --query "SELECT Name, Subject__c, Priority__c, Status__c, Product_Tag__r.Name FROM ADM_Work__c WHERE Product_Tag__r.Name LIKE '%FSC%' AND Type__c = 'Bug' AND Priority__c IN ('P1','P2') AND Status__c NOT IN ('Closed','Duplicate','Never Fix','Not a Bug') ORDER BY Priority__c, CreatedDate DESC LIMIT 20" --target-org gus --json
```

If GUS is not connected, direct user to check:
- Known Issues site: https://issues.salesforce.com (search "Action Plan")
- Or ask the user to run the query manually in their GUS-connected terminal

---

## I. Code Reference Map

**Source:** CodeSearch (gitcore.soma.salesforce.com) — verified connected.

### Core Module Structure

```
core/
├── actionplan/                          ← Base Action Plan entity definitions
│   └── java/src/actionplan/
│       ├── entity/
│       │   └── ActionPlanTemplateObject.java        ← Template entity logic, field validation, defaults
│       ├── field/
│       │   ├── ActionPlanItemDependencyStatusEnum.java  ← Dependency states (None, WaitingOnPrevious)
│       │   └── ActionPlanTemplateTypeFactory.java      ← Sales vs Industries type handling
│       ├── constants/
│       │   └── ActionPlanTemplateTypeEnum.java        ← INDUSTRIES, ADMIN_CREATED, SALES types
│       └── service/
│           └── ActionPlanServiceImpl.java             ← Plan creation, flow generation
│
├── industries-actionplan-impl/          ← FSC/Industries Action Plan implementation
│   └── java/src/
│       ├── actionplan/
│       │   ├── service/
│       │   │   ├── ActionPlanDependentTaskServiceImpl.java  ← ★ SUCCESSOR TASK LOGIC
│       │   │   └── IndustriesActionPlanServiceImpl.java     ← Action Plan item creation
│       │   ├── util/
│       │   │   └── ActionPlanItemUtil.java                  ← Item state transitions, dependency checks
│       │   └── transactionobserver/
│       │       └── SalesActionPlanEventRecordStatusTransactionObserver.java  ← Event-based status updates
│       ├── industries/actionplan/
│       │   ├── ActionPlanItemFunctions.java              ← Item lifecycle management
│       │   └── impl/actionplntmplitmdependency/
│       │       └── ActionPlnTmplItmDependencyFunctions.java  ← Dependency cycle detection
│       └── sales/actionplan/service/impl/
│           └── SalesActionPlanEventServiceImpl.java      ← Sales AP event handling
│
└── industries-actionplan/               ← Shared utilities
    └── java/src/actionplan/util/
        └── ActionPlanSlackNotificationUtil.java  ← Notification on item completion
```

### Key Classes by Issue Pattern

#### Pattern 1: Successor Task Failures

**Primary class:** `ActionPlanDependentTaskServiceImpl.java`
```
Repository: gitcore.soma.salesforce.com/core-2206/core-260-public
Path: core/industries-actionplan-impl/java/src/actionplan/service/ActionPlanDependentTaskServiceImpl.java
Scrum Team: FSC-Darksaber
Author: b.sugiarto (since release 232)
```

Key internals:
- Implements `ActionPlanDependentTaskService` interface
- Uses `ActionPlanTemplateItemDependencies` and `ActionPlanTemplateItemDependenciesPrevious` fields
- Calls `getActionPlanTemplateItemDependencies(versionId, soap)` to query dependency graph
- **Root cause of "Couldn't create successor tasks"**: When this service cannot resolve dependencies (corrupted items, missing references), it fails

**Dependency validation:** `ActionPlnTmplItmDependencyFunctions.java`
```
Path: core/industries-actionplan-impl/java/src/industries/actionplan/impl/actionplntmplitmdependency/ActionPlnTmplItmDependencyFunctions.java
```
- Detects cycles in template item dependencies
- Uses `GraphBuilder` class for cycle detection
- If cycle exists → prevents publishing

#### Pattern 2: Item State Machine

**Dependency Status Enum:** `ActionPlanItemDependencyStatusEnum.java`
```
Path: core/actionplan/java/src/actionplan/field/ActionPlanItemDependencyStatusEnum.java
```
States:
- `None` — No dependencies, item can proceed
- `WaitingOnPrevious` — Has uncompleted parent items, waiting

**Item State Transitions:** `ActionPlanItemUtil.java`
```
Path: core/industries-actionplan-impl/java/src/actionplan/util/ActionPlanItemUtil.java
```
Key methods:
- `getChildItemIdToUncompletedParentMap()` — Maps child items to their uncompleted parents
- `calculateActionPlanItemStatusBasedOnEventSchedule()` — Computes status from event dates
- Updates dependency status when item reaches `COMPLETED` state and was `WaitingOnPrevious`
- **Filter predicate:** `isWaitingOnPrevious()` — selects items still waiting

**State values:** `ActionPlanItemStateEnum`
- `PENDING` → `IN_PROGRESS` → `COMPLETED`
- `NONE` (when dependencies not yet resolved)

#### Pattern 3: Action Plan Creation Flow

**Service:** `IndustriesActionPlanServiceImpl.java`
```
Path: core/industries-actionplan-impl/java/src/actionplan/service/IndustriesActionPlanServiceImpl.java
```
Key method: `createActionPlanItems()`
- Creates `ActionPlanItem` records via `UddDb.createEntityObject(ActionPlanItemUddConstants.EntityId)`
- Sets initial `dependencyStatus` = `ActionPlanItemDependencyStatusEnum.None`
- Sets initial `itemState` = `ActionPlanItemStateEnum.NONE` (for items with dependencies)
- Links items to ActionPlan, Target, and TemplateItem

**Flow-based creation:** `ActionPlanBuilder.java`
```
Path: core/actionplan/java/src/actionplan/ActionPlanBuilder.java
```
- Generates Flow metadata for Action Plan execution
- Creates `FlowRecordCreate` nodes for each ActionPlanItem
- Sets fields: ActionPlan, Target, Item, ItemState (IN_PROGRESS), LastActionDateTime

#### Pattern 4: Sales vs Industries Action Plan Types

**Type Enum:** `ActionPlanTemplateTypeEnum`
```
Values: INDUSTRIES, ADMIN_CREATED, SALES
```
- `INDUSTRIES` → Full FSC Action Plans (supports doc checklist, auto-status)
- `SALES` → Account Plan Action Plans (tasks/events only, manual status)

**Template Object:** `ActionPlanTemplateObject.java`
```
Path: core/actionplan/java/src/actionplan/entity/ActionPlanTemplateObject.java
```
- Default `ActionPlanType` = `ActionPlanTemplateTypeEnum.INDUSTRIES`
- Default `TargetEntityType` = `AccountUddConstants.Name`
- Contains field validation logic and default value computation

#### Pattern 5: Sales Action Plan Event Status

**Observer:** `SalesActionPlanEventRecordStatusTransactionObserver.java`
```
Path: core/industries-actionplan-impl/java/src/actionplan/transactionobserver/SalesActionPlanEventRecordStatusTransactionObserver.java
```
- Updates item status to calculated value ("Completed", "InProgress", "Pending")
- Uses `getActionPlanItemUpdatesStatusMap(eventsWithScheduleChanges)` to compute new states
- Only for Sales Action Plans with Events — NOT for FSC task-based plans

**Sales Event Service:** `SalesActionPlanEventServiceImpl.java`
```
Path: core/industries-actionplan-impl/java/src/sales/actionplan/service/impl/SalesActionPlanEventServiceImpl.java
```
- Updates action plan items to `COMPLETED` state
- Filters items in states: `PENDING`, `IN_PROGRESS`

#### Pattern 6: Notifications on Completion

**Notification Utility:** `ActionPlanSlackNotificationUtil.java`
```
Path: core/industries-actionplan/java/src/actionplan/util/ActionPlanSlackNotificationUtil.java
```
- Sends notification when item is completed (NotificationType.ACTION_PLAN_ITEM_COMPLETED)
- View: `finserv__actionplanitem_completed`
- Logic: If item state == COMPLETED → notify AP owner
- If creating adhoc item with status COMPLETED → also notify

### CodeSearch Queries for Live Investigation

When investigating a specific case, use these queries:

| Scenario | Query |
|----------|-------|
| Trace successor task failure | `content:"ActionPlanDependentTaskServiceImpl" lang:java repo:"gitcore.soma.salesforce.com"` |
| Find dependency cycle logic | `content:"ActionPlnTmplItmDependency" content:"cycle" lang:java` |
| Trace item state transitions | `content:"ActionPlanItemStateEnum" content:"COMPLETED" lang:java repo:"gitcore.soma.salesforce.com"` |
| Find template type handling | `content:"ActionPlanTemplateTypeEnum" lang:java repo:"gitcore.soma.salesforce.com"` |
| Trace DisplayOrder usage | `content:"DisplayOrder" content:"ActionPlanTemplateItem" lang:java` |
| Find permission checks | `content:"ActionPlan" content:"Permission" lang:java repo:"gitcore.soma.salesforce.com"` |
| Trace item creation | `content:"ActionPlanItem" content:"create" lang:java repo:"gitcore.soma.salesforce.com"` |
| Sales AP vs Industries AP | `content:"SalesActionPlan" lang:java repo:"gitcore.soma.salesforce.com"` |
| Dependency status enum | `content:"WaitingOnPrevious" content:"ActionPlanItem" lang:java repo:"gitcore.soma.salesforce.com"` |
| Error handling | `content:"ActionPlanCannotCreateException" lang:java` |

---

## J. Gack/Error Trace Lookup (Columbo)

**When to use:** Customer provides an ErrorId/Gack ID (format: `XXXXXXXXXX-XXXXX` with optional `(negative_number)` suffix), or you see an internal error that needs production-level tracing.

### How to Investigate a Gack

```
1. AUTHENTICATE (if not already):
   mcp__plugin_columbo_columbo__refresh_auth → service: "delphi"
   mcp__plugin_columbo_columbo__refresh_auth → service: "kodama"

2. DIRECT LOOKUP (if you have the exact gack ID):
   mcp__plugin_columbo_columbo__fetch_gack_details
   Input: gack_id = "1770957752-49415"
   
   Returns: Exception type, source class, method, stack trace, org ID, timestamp

3. IF DIRECT LOOKUP EMPTY (gack expired/purged — common for older errors):
   mcp__plugin_columbo_columbo__search_gacks
   Input: subject = "ActionPlan" (or error keyword)
          limit = 50
   
   Returns: All recent production gacks matching the keyword — with stack trace IDs

4. BULK ANALYSIS (find all orgs hitting the same pattern):
   mcp__plugin_columbo_columbo__fetch_bulk_gacks_by_stacktrace_ids
   Input: stacktrace_ids = [-398836090, -1810462813]  (from step 2/3)
   
   Returns: All gack instances across orgs for those stack traces — shows frequency/impact

5. DEEP DIVE ON STACK TRACE:
   mcp__plugin_columbo_columbo__fetch_gacks_by_stacktrace
   Input: stacktrace_id = -398836090
   
   Returns: Full stack trace text, all orgs affected, timeline
```

### Known Action Plan Gack Patterns

These are **active production gack patterns** found via Columbo search:

#### Gack Pattern A: SQL Exception on Action Plan Creation

| Field | Value |
|-------|-------|
| **Exception** | `SfdcSqlException` — "SQL Exception for entity ActionPlan" |
| **Source** | `actionplan.entity.ActionPlanTemplateObject` |
| **Method** | `populateDerivedStatusField` |
| **Frequency** | High — multiple stack trace IDs |
| **Stack Traces** | -398836090, -1810462813, 1474149669, -188873150 |
| **Likely Trigger** | Creating a new Action Plan from a template with corrupted/invalid derived status field data |

**Resolution Path:**
1. Check template version integrity (are all items valid?)
2. Check for custom triggers that fire during AP creation and modify status
3. If data corruption → clone template → delete old → publish clone
4. Escalate with stack trace ID if no customer-side fix possible

#### Gack Pattern B: NullPointerException on Template Publish/Deactivate

| Field | Value |
|-------|-------|
| **Exception** | `NullPointerException` — "Cannot invoke ActionPlanTemplateItemService.getItems() because actionPlanTemplateItemService is null" |
| **Source** | `ui.industries.actionplan.components.editor.HighlightsPanelController` |
| **Methods** | `publishTemplate`, `deactivateTemplate` |
| **Stack Traces** | 51869049, 1987205671 |
| **Likely Trigger** | Service injection failure in the Action Plan template editor UI component |

**Resolution Path:**
1. This is likely a platform-level service injection issue (not customer-configurable)
2. Try: Clear browser cache → hard refresh → attempt publish again
3. Try: Different browser or incognito mode
4. If persists: File GUS bug with stack trace ID referencing this pattern
5. Workaround: Publish via API/Workbench instead of UI

### Gack Format Reference

Customers may provide error IDs in different formats:
- `1770957752-49415 (-1250433517)` → Gack ID = `1770957752-49415`, Stack Trace ID = `-1250433517`
- `1770957752-49415` → Just the Gack ID (look up directly)
- `ErrorId: 12345678-12345` → Same format, just labeled differently
- Just the negative number `(-1250433517)` → This is a Stack Trace ID — use `fetch_gacks_by_stacktrace`

**Always ask the customer for the full error ID** — it's the fastest path to root cause.

---

## K. Referral Patterns

### Pattern K1: Referral Not Visible on Record Page

**Symptoms:**
- Referral component not rendering on Account/Contact/Lead page
- "No records to display" in Referral related list
- Referrals created but invisible to certain users

**Root Causes & Resolution:**

| Root Cause | How to Identify | Fix |
|-----------|-----------------|-----|
| Referral component not added to page layout | Check Lightning App Builder for the record page | Add "Referrals" component to page |
| OWD for Referral object is too restrictive | Setup → Sharing Settings → Referral | Set OWD to "Public Read Only" or add sharing rules |
| User lacks Read on Referral object | Check profile/permission set | Add Referral Read permission |
| Referral linked to wrong record | Query `SELECT Id, ReferredById, ReferredToId FROM Referral` | Correct the lookup fields |

**Key Objects:**
- `Referral` — main object (`FinServ__Referral__c` in managed, `Referral` in core)
- `ReferredBy` — lookup to the referring user/contact
- `ReferredTo` — lookup to the referred contact/lead

**GUS Query:**
```bash
sf data query --query "SELECT Name, Subject__c, Status__c FROM ADM_Work__c WHERE Product_Tag__r.Name = 'FSC Referrals' AND Type__c = 'Bug' AND Status__c NOT IN ('Closed','Duplicate','Never Fix') LIMIT 10" --target-org gus --json
```

---

### Pattern K2: Referral Routing Not Working

**Symptoms:**
- Referrals not assigned to correct user or queue
- Referral stays in "New" status without routing
- Routing rules appear configured but have no effect

**Root Causes:**
1. Referral routing rules not activated or improperly configured
2. Assignment rules on Referral object not enabled
3. Omni-Channel routing misconfiguration (if using Omni)
4. User/queue referenced in rule is inactive

**Resolution Steps:**
1. Setup → Referral Settings → verify routing is enabled
2. Setup → Assignment Rules → Referral → check rules are active and ordered correctly
3. Verify the target user/queue is active and has Referral object access
4. Check if Omni-Channel is in use: Setup → Omni-Channel → Routing Configurations → Referral

**Design Note:** Referral routing uses standard Salesforce Assignment Rules. All rules for Assignment Rules apply: only one rule set active at a time, criteria evaluated top-to-bottom.

---

### Pattern K3: Duplicate Referrals

**Symptoms:**
- Same referral created multiple times
- Automation triggering referral creation on every save

**Resolution:**
1. Check for Flow or Process Builder triggered on Referral-related object (Account, Lead) — is it firing on update as well as insert?
2. Add entry criteria to Flow: `{!$Record.IsNew} = true` (insert only)
3. Check for missing duplicate rules on Referral object: Setup → Duplicate Rules → Referral

---

## L. Rollup Summary Rules (RSR) Patterns

### Pattern L1: RSR Not Calculating / Stale Values

**Symptoms:**
- RSR-configured field showing 0 or old value
- RSR ran but source records are not reflected
- Value updates only after manual recalculation

**Root Causes:**

| Root Cause | Check | Fix |
|-----------|-------|-----|
| RSR not active | Setup → Rollup Summary Rules | Activate the rule |
| RSR target field not mapped correctly | Check RSR configuration — source object and field | Verify source object, relationship, aggregate field |
| Async recalculation delay | Check last run timestamp in RSR setup | Trigger manual recalc or wait for async |
| Batch recalculation needed after bulk load | After data load, RSR may lag | Run: Setup → Rollup Summary Rules → Recalculate |
| Filter criteria excluding records | RSR has filter that doesn't match source records | Review filter criteria |

**Resolution Steps:**
1. Setup → Financial Services Cloud → Rollup Summary Rules
2. Find the rule → check Status = Active
3. Check "Source Object" and "Aggregate Field" are correct
4. Check "Filter Criteria" — is it too restrictive?
5. Click "Recalculate" to force a fresh run
6. After recalc, query target records to confirm values updated

**Key Design Rules for RSR:**
- Maximum **20 RSR rules per object**
- RSR runs **asynchronously** — not real-time on every save
- RSR supports: SUM, COUNT, MIN, MAX, AVG aggregates
- RSR only supports **numeric, currency, and date** fields for SUM/AVG/MIN/MAX

---

### Pattern L2: RSR Hitting Limit (Max 20 per Object)

**Symptoms:**
- "You have reached the maximum number of rollup summary rules" error
- Cannot create new RSR

**Resolution:**
- Audit existing RSRs — delete unused/obsolete rules
- Combine similar RSRs using calculated fields where possible
- Consider custom Apex batch if more than 20 rollups needed on one object

---

### Pattern L3: Field Not Available for RSR

**Symptoms:**
- Target field doesn't appear in RSR field dropdown
- Only certain field types visible

**Design Note:** RSR only supports fields of type: **Number, Currency, Percent, Date, DateTime**. Text, Checkbox, Lookup, and Formula fields cannot be rollup targets.

**Resolution:** Create a helper numeric/currency field as the RSR target, then display via formula field if needed.

---

## M. Financial Accounts & Holdings Patterns

### Pattern M1: Financial Account Not Visible on Client Record

**Symptoms:**
- FinancialAccount related list empty despite records existing in org
- Financial Account component missing from page
- User can see account but not financial account

**Root Causes:**

| Root Cause | Fix |
|-----------|-----|
| "Financial Accounts" related list not on page layout | Add to layout via App Builder |
| User lacks Read on FinancialAccount object | Add permission in Profile or Permission Set |
| OWD for FinancialAccount too restrictive | Check Sharing Settings — add sharing rule |
| Financial Account not associated to correct Person Account | Check `PrimaryOwner` lookup field on FinancialAccount |

**Key Relationships:**
- `FinancialAccount.PrimaryOwner` → links to Contact (Person Account)
- `FinancialAccount.FinancialAccountRole` → defines roles (Owner, Beneficiary, etc.)
- Multiple contacts can have roles on one Financial Account

**Useful Query:**
```sql
SELECT Id, Name, FinancialAccountType, PrimaryOwner.Name 
FROM FinancialAccount 
WHERE PrimaryOwner.Name = '<client_name>'
```

---

### Pattern M2: Financial Account Roles / Ownership Issues

**Symptoms:**
- Client not showing as "Owner" of their Financial Account
- Wrong role displayed for household member
- Financial Account not rolling up to household

**Root Causes:**
1. `FinancialAccountRole` record missing or has wrong `Role` picklist value
2. `PrimaryOwner` field points to wrong Contact
3. Household rollup misconfigured (Group Member record missing)

**Resolution:**
1. Query `FinancialAccountRole` records for the FinancialAccount
2. Verify at least one role = "Owner" linked to the primary contact
3. Check `FinancialAccount.PrimaryOwner` — should match the client Contact
4. For household rollup: verify Account-Contact relationship (GroupMember or AccountContactRelation)

---

### Pattern M3: Holdings Not Showing or Updating

**Symptoms:**
- `FinancialHolding` records exist but don't appear on Financial Account
- Holdings data stale after trade/update

**Resolution:**
1. Verify `FinancialHolding.FinancialAccount` lookup is populated correctly
2. Check page layout includes "Financial Holdings" related list
3. Verify user has Read on `FinancialHolding` object
4. If integration-driven: check integration user has Write on FinancialHolding

---

## N. FSC Timeline Patterns

### Pattern N1: Timeline Component Not Showing Records

**Symptoms:**
- Timeline component renders but is empty
- Only some object types appear, others missing
- Timeline shows spinner indefinitely

**Root Causes:**

| Root Cause | Fix |
|-----------|-----|
| Object not added to Timeline configuration | Setup → Timeline → Add object to "Record Types" |
| User lacks Read on the tracked object | Add object Read permission |
| "Enable Activity Timeline" not turned on | Setup → Activity Settings → Enable |
| Custom object not configured for Timeline | Add object via Timeline Setup, check "Display on Timeline" |
| Date field mismatch | Verify the date field selected for ordering is populated on records |

**Resolution Steps:**
1. Setup → Financial Services Cloud → Timeline
2. Click the Timeline configuration for the record type
3. Verify target objects are listed and enabled
4. For each object: confirm the "Date Field" and "Title Field" are correctly mapped
5. Check user permissions on every object type shown in Timeline config

**Design Note:** Timeline supports both standard activity objects (Task, Event, Email) and custom FSC objects. Each object type must be explicitly added — it is not automatic.

---

### Pattern N2: Specific Object Not in Timeline

**Symptoms:**
- Interaction Summary, Referral, or other FSC object missing from Timeline

**Resolution:**
1. Setup → Timeline → Edit configuration
2. Click "Add Item" → select the object
3. Map: Date Field (must be a Date/DateTime on that object), Title Field, Description Field
4. Save and verify on record page

---

## O. Household & Group Management Patterns

### Pattern O1: Person Account vs Household Confusion

**Symptoms:**
- Expected household group behavior not working
- Related contacts not appearing under household
- Rollup to household not happening

**Key Design:**
- FSC uses **Person Accounts** for individuals — each client is both an Account and a Contact
- **Household** is a separate Account record of RecordType = "Household" (or "IndustrialHousehold")
- Members linked via `AccountContactRelation` (or `GroupMember` in some configurations)
- Financial rollups go from individual Financial Accounts up to Household via RSR or custom logic

**Common Misconfiguration:**
- Customer creates a regular "Account" instead of a "Household" account — rollups won't work
- `AccountContactRelation` not created between Household and Person Account

**Resolution:**
1. Verify RecordType of the "household" account = Household (not a standard B2B account)
2. Verify `AccountContactRelation` records exist linking the contacts to the household
3. Check Household RSRs are active (Section L)

---

### Pattern O2: Relationship Map Not Rendering

**Symptoms:**
- "Relationship Map" component blank or shows error
- Only some relationships visible

**Resolution:**
1. Verify component is "Relationship Map" (standard FSC, not a custom one)
2. Check `AccountContactRelation` records — each linked contact/account needs a relationship record
3. Verify `Roles` picklist on `AccountContactRelation` is populated (controls relationship labels)
4. Check OWD for `AccountContactRelation` — must be accessible to user

---

## P. Interaction Summary Patterns

### Pattern P1: Interaction Summary Not Saving

**Symptoms:**
- Error when saving Interaction Summary
- Required fields not obvious
- Interaction Summary saves but shows no related records

**Root Causes:**

| Root Cause | Fix |
|-----------|-----|
| Required fields missing (Subject, Date) | Ensure Subject and ActivityDate are filled |
| User lacks Create on InteractionSummary object | Add Create permission |
| Related records (Contacts, Accounts) not linked | Use "Add Attendees" / "Related Records" section on the Summary |
| Flow validation rule firing | Check custom validation rules on the object |

**Key Object:** `InteractionSummary` — bridges Activity (Task/Event) with multiple related records (Contacts, Accounts, FinancialAccounts).

---

### Pattern P2: Interaction Summary Not Appearing on Timeline

See Pattern N2 — add `InteractionSummary` to Timeline configuration with correct date field (`ActivityDate`).

---

## Q. Goals & Life Events Patterns

### Pattern Q1: Goal Not Showing on Client Record

**Symptoms:**
- Goal created but not visible on client page
- Goal component empty

**Resolution:**
1. Add "Goals" related list or "Goals" component to page layout
2. Verify user has Read on `FinancialGoal` object
3. Check `FinancialGoal.FinancialAccount` or `FinancialGoal.Contact` lookup is populated to link it to the client
4. Verify OWD for `FinancialGoal` is not too restrictive

---

### Pattern Q2: Life Event Not Triggering Automation

**Symptoms:**
- Life Event created but Flow/Process not firing
- Life Event template not available in dropdown

**Resolution:**
1. Verify "Life Events" feature enabled: Setup → Financial Services Cloud → Life Events
2. Check Life Event Type picklist — is the expected type present?
3. For automation: check Flow entry criteria uses `{!$Record.RecordType.Name}` or Life Event Type field correctly
4. Verify the Flow is active and triggered on `LifeEvent` object (insert)

**Key Object:** `LifeEvent` — stores the life event record linked to a Contact/Account.

---

## R. FSC Data Model & Sharing

### Key FSC Object Permissions Matrix

| Object | Typical Permissions Needed | Notes |
|--------|---------------------------|-------|
| `FinancialAccount` | CRUD (advisors), R (clients) | OWD usually Private + sharing rules |
| `FinancialAccountRole` | CR (advisors), R (clients) | Links contacts to financial accounts |
| `FinancialHolding` | CRUD (integration), R (users) | — |
| `Referral` | CRUD | OWD Public Read/Write or sharing rules |
| `InteractionSummary` | CRUD | — |
| `FinancialGoal` | CRUD | — |
| `LifeEvent` | CRUD | — |
| `ActionPlan` | CRUD | Requires PSL |
| `ActionPlanTemplate` | R (users), CRUD (admins) | — |
| `DocumentChecklistItem` | CRUD (if using doc checklist) | — |

### Common FSC Sharing Patterns

**Guest User Access:**
- Guest users cannot access FSC objects with OWD = Private — must use Platform Events + admin-context flows
- See Action Plan Pattern 7 for the canonical guest user workaround pattern

**Household Sharing:**
- Typically: individual Financial Accounts are private; household members access via sharing rule based on AccountContactRelation
- Verify: Setup → Sharing Settings → FinancialAccount → Sharing Rules reference the correct criteria

**GUS Query for FSC Sharing Bugs:**
```bash
sf data query --query "SELECT Name, Subject__c, Status__c FROM ADM_Work__c WHERE Product_Tag__r.Name LIKE 'FSC%' AND Subject__c LIKE '%sharing%' AND Type__c = 'Bug' AND Status__c NOT IN ('Closed','Duplicate','Never Fix') LIMIT 10" --target-org gus --json
```
