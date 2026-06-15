## A. Triage & Classification

When a support engineer brings an Action Plan issue, first classify it:

```
Is the issue about...

├── 1. PERMISSIONS & ACCESS
│   ├── "Insufficient Privileges" error
│   ├── Cannot access Action Plan Template
│   ├── Cannot reassign Action Plans
│   ├── Permission Set not visible / not assignable
│   └── "You do not have the level of access necessary"
│   → Go to: Pattern 3 (Licensing/PSL) or Pattern 7 (Guest User)
│
├── 2. TASK DEPENDENCIES & SUCCESSOR TASKS
│   ├── "Couldn't create successor tasks for task" error
│   │   → Go to: Pattern 1 (Successor Tasks)
│   ├── Tasks stuck in "Waiting" / not transitioning to "In Progress"
│   │   → FIRST check: Pattern 9 (Inactive User blocking dependency engine)
│   │   → THEN check: Pattern 1 (Successor Tasks — template corruption)
│   ├── Dependent tasks not triggering automations
│   │   → Go to: Pattern 1 (Successor Tasks)
│   ├── Tasks not transitioning after predecessor completed
│   │   → Go to: Pattern 9 (Inactive User) — most common cause
│   │   → Also check: Pattern 1 (template corruption)
│   └── Child tasks created even when parent is "Not Required"
│       → Go to: Pattern 1 (Successor Tasks)
│
├── 3. TEMPLATE CONFIGURATION & SETUP
│   ├── Template saving errors ("There is an error while saving")
│   ├── Template items not visible after data load
│   ├── Reorder button not working
│   ├── Tasks not saving in reordered sequence
│   ├── DisplayOrder field issues
│   ├── Obsolete templates showing in lookup
│   └── Template deployed as ReadOnly
│   → Go to: Pattern 4 (Reordering) or Pattern 6 (ReadOnly deployment)
│
├── 4. LICENSING & ENTITLEMENTS
│   ├── Action Plan PSL disabled
│   ├── Negative PSL counts
│   ├── License not provisioned in sandbox
│   ├── Action Plan features not available after license purchase
│   └── Sales Action Plans vs Action Plans confusion
│   → Go to: Pattern 3 (PSL) or Pattern 8 (Sales vs FSC)
│
├── 5. UI / COMPONENT ISSUES
│   ├── Action Plan component not visible on mobile
│   ├── Action Plan Item List not refreshing
│   ├── Tasks show "No records to display"
│   ├── Column customization not available
│   └── Document Checklist Items not appearing
│   → Go to: Pattern 5 (Mobile) or Pattern 10 (Refresh)
│
├── 6. DOCUMENT CHECKLIST IN ACTION PLANS
│   ├── "DocumentChecklistItem an invalid action plan template entity"
│   ├── Document Checklist not supported in Sales Action Plans
│   ├── Document Checklist Items not appearing in Site
│   └── Document Checklist permissions issue
│   → Go to: Pattern 2 (DocumentChecklistItem error)
│
├── 7. DEPLOYMENT & MIGRATION
│   ├── Templates deployed as ReadOnly
│   ├── Templates missing after sandbox refresh
│   ├── Status field errors post-migration
│   └── Cloning errors with inactive users
│   → Go to: Pattern 6 (ReadOnly) or Pattern 9 (Inactive User)
│
└── 8. INACTIVE USER ISSUES (High-Impact — often misdiagnosed)
    ├── Tasks stuck in "Waiting" with no error message
    ├── Dependency engine not advancing task states
    ├── Action Plans assigned to deactivated users
    ├── Cloning/sync errors referencing inactive user
    └── Issue affecting "most users" after user deactivation
    → Go to: Pattern 9 (Inactive User — Dependency Engine Blocked)
```

### Quick Diagnostic Questions

Ask these to narrow down quickly:
1. **What type of Action Plans?** "Sales Action Plans" (Account Plan feature) or "FSC Action Plans"?
2. **What is the exact error message?** Full text, including any ErrorId/Gack ID (format: `XXXXXXXXXX-XXXXX` or `XXXXXXXXXX-XXXXX (negative_number)`). **If they have an ErrorId → immediately use Columbo to look it up (Section J).**
3. **What user profile/permission set?** System Admin or specific profile?
4. **Production or Sandbox?**
5. **When did this start?** After a release/deployment/sandbox refresh? **Also ask: "Were any users deactivated around that time?"** (Inactive users are a top root cause for "tasks stuck in Waiting" — Pattern 9)
6. **What objects involved?** Tasks only, or also Document Checklist Items/Events?
7. **How many users affected?** One user or broadly across the team? (Broad impact + "Waiting" status = likely Pattern 9: Inactive User)
8. **Are there any inactive/deactivated users** who previously owned Action Plans or AP Items? (Critical for Pattern 9)

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

**With CodeSearch (verified — repo: gitcore.soma.salesforce.com/core-2206/core-262-public):**
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

**With GUS (MANDATORY — always search before diagnosing):**

**Step 1: Search for known bugs matching the symptom:**
```bash
sf data query --query "SELECT Id, Name, Subject__c, Status__c, Priority__c FROM ADM_Work__c WHERE Product_Tag__r.Name = 'FSC Action Plans' AND Type__c = 'Bug' AND Status__c NOT IN ('Closed', 'Duplicate', 'Never Fix') ORDER BY Priority__c, CreatedDate DESC LIMIT 10" --target-org gus --json
```

**Step 2: Search by symptom keywords (extract from customer description):**
```bash
sf data query --query "SELECT Name, Subject__c, Status__c, Resolution__c, Root_Cause_Brief__c FROM ADM_Work__c WHERE Product_Tag__r.Name = 'FSC Action Plans' AND (Subject__c LIKE '%waiting%' OR Subject__c LIKE '%stuck%' OR Subject__c LIKE '%transition%' OR Subject__c LIKE '%inactive%') AND Type__c IN ('Bug', 'Investigation') ORDER BY CreatedDate DESC LIMIT 10" --target-org gus --json
```

**Step 3: If you know a specific Work ID, pull full details:**
```bash
sf data query --query "SELECT Name, Subject__c, Details__c, Resolution__c, Root_Cause_Brief__c, Status__c FROM ADM_Work__c WHERE Name = 'W-22451209'" --target-org gus --json
```

→ **ALWAYS search GUS BEFORE providing diagnosis.** If Engineering already investigated this issue, use their findings rather than guessing.
→ **Reference the GUS Work ID in your response** so the support engineer can link it to the customer case.

**With Slack:**
```
Search: "in:#tech-prod-help-financial-services-cloud <error_message_or_keyword>"
```
→ Finds if someone already solved this recently.

---

## B. Known Issue Patterns

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
Repo: gitcore.soma.salesforce.com/core-2206/core-262-public
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

**What CodeSearch tells you (translated for support):**
When you find a permission check in code, translate it like this:
- Code says `ActionPlansPsl` → Tell engineer: Look for **"Financial Services Action Plans"** in Setup → Company Information → Permission Set Licenses
- Code says `FSCSalesPsl` → Tell engineer: Look for **"FSC Sales"** in same location
- Code says `FSCServicePsl` → Tell engineer: Look for **"FSC Service"** in same location

**Important: Action Plans access can come from MULTIPLE licenses.** Always check all of these:
- **Financial Services Action Plans** (dedicated AP license)
- **FSC Sales** (includes AP access for sales use cases)
- **FSC Service** (includes AP access for service use cases)

**Resolution Steps:**

```
User can't assign Action Plans permission?
│
├── Step 1: Check which licenses exist in the org
│   Setup → Company Information → Permission Set Licenses
│   Look for ANY of these (customer may have one or more):
│   • "Financial Services Action Plans"
│   • "FSC Sales"
│   • "FSC Service"
│   │
│   ├── NONE of these listed → License never provisioned
│   │   → Is this Production or Sandbox?
│   │   ├── Sandbox → Refresh sandbox from Production
│   │   └── Production → Contact License Provisioning team
│   │       (Tell them which license the customer purchased)
│   │
│   └── One or more LISTED → Check status of each
│       │
│       ├── Status = "Disabled"
│       │   → Cannot enable via UI
│       │   → Escalate to Support/License team
│       │
│       └── Status = "Active" → Check Available count
│           │
│           ├── Available = 0
│           │   → Check: Are inactive users consuming slots?
│           │   → Setup → Permission Sets → "Action Plans" → Assigned Users
│           │   → Remove license assignment from inactive users
│           │
│           └── Available > 0 → Permission set itself is the issue
│               → Setup → Permission Sets → "Action Plans" → edit
│               → Check "License" field on the Permission Set
│               → If set to "Salesforce" → change to blank or match
│                 the PSL the customer actually has (FSC Sales/FSC Service)
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

### Pattern 9: Inactive User Blocking Dependency Engine / Cloning

**Frequency:** HIGH — often misdiagnosed as release regression or template issue  
**Severity:** Level 1-2 (Critical — blocks entire task progression for multiple users)  
**Average TTR:** 14-30 days (high because root cause is non-obvious)  
**GUS References:** W-22451209 (task transition stuck due to inactive user ownership)

**Symptoms:**

| Symptom | How It Appears |
|---------|---------------|
| Tasks stuck in "Waiting" | Next task shows "Waiting" status even after predecessor is completed |
| No error message visible | Unlike Pattern 1, there may be NO visible error — tasks just don't advance |
| Affects "most users" | Broad impact across the team, not just one user |
| Started after user deactivation | Often correlates with someone leaving/being deactivated |
| `WaitingOnPrevious` in data | ActionPlanItem.DependencyStatus = `WaitingOnPrevious` even when parent = `COMPLETED` |
| Cloning error | `operation performed with inactive user [ID] as owner of actionPlanTemplateItemValue` |
| Records can't sync | Ownership references inactive user, preventing DML operations |

**Root Causes:**

| # | Root Cause | Frequency | How to Identify |
|---|-----------|-----------|-----------------|
| 1 | **Inactive user owns ActionPlanItem records** — dependency engine cannot update items owned by inactive users, so `DependencyStatus` stays `WaitingOnPrevious` permanently | Most common | Query AP items with inactive owner |
| 2 | **Inactive user is assigned as task owner in template** — new APs created from template assign tasks to inactive user, immediately blocking progression | Common | Check template item assignee references |
| 3 | **Inactive user owns the ActionPlan itself** — the AP record's owner is inactive, preventing system-level status updates | Less common | Check AP owner |

**Code Reference (from CodeSearch — verified):**

**What the dependency engine does (translated for support):**
> When you complete a task in an Action Plan, the system checks: "Are ALL predecessor tasks for the next task completed?" If yes, it updates the next task's status from "Waiting" to "In Progress". But — the system needs to UPDATE the ActionPlanItem record to change its status. If that record is owned by an inactive user, Salesforce blocks the DML update silently. The task stays stuck in "Waiting" forever.

```
Primary class: ActionPlanDependentTaskServiceImpl.java
Path: core/industries-actionplan-impl/java/src/actionplan/service/ActionPlanDependentTaskServiceImpl.java

What happens:
1. User completes Task A → triggers dependency check
2. System calls getChildItemIdToUncompletedParentMap() in ActionPlanItemUtil.java
3. Finds that Task B has no uncompleted parents → should transition
4. System attempts to UPDATE ActionPlanItem for Task B:
   - Set DependencyStatus = None (was WaitingOnPrevious)
   - Set ItemState = IN_PROGRESS (was PENDING)
5. ⚠️ IF Task B's ActionPlanItem is OWNED by an INACTIVE user:
   - DML update fails silently (no exception thrown to end user)
   - DependencyStatus stays WaitingOnPrevious
   - Task B remains stuck in "Waiting"

Key class for state transitions: ActionPlanItemUtil.java
Path: core/industries-actionplan-impl/java/src/actionplan/util/ActionPlanItemUtil.java
- getChildItemIdToUncompletedParentMap() — checks parent completion
- isWaitingOnPrevious() — filter predicate for stuck items
- State machine: WaitingOnPrevious → None (when parents complete)
  BUT this transition requires a successful DML UPDATE on the item record
```

**Why this is hard to diagnose:**
- There is NO error message shown to the user
- The task just stays in "Waiting" silently
- It's easily confused with template corruption (Pattern 1) or release regression
- It affects ALL users' tasks if those tasks are assigned to/owned by the inactive user's AP items

**Resolution Steps:**

```
Tasks stuck in "Waiting" after predecessor completed?
│
├── Step 1: Identify stuck ActionPlanItem records
│   Query (run in customer org):
│   SELECT Id, Name, ItemState, DependencyStatus, OwnerId, ActionPlan.Name
│   FROM ActionPlanItem
│   WHERE DependencyStatus = 'WaitingOnPrevious'
│   AND ActionPlan.Status != 'Completed'
│
├── Step 2: Check if owners are inactive
│   SELECT Id, Name, IsActive FROM User WHERE Id IN (<OwnerIds from Step 1>)
│   │
│   ├── If ANY owner IsActive = FALSE → THIS IS THE ROOT CAUSE
│   │   │
│   │   ├── Step 3a: Reassign ownership of stuck AP Items
│   │   │   Option A (preferred): Transfer ownership to an active user
│   │   │   - Data Loader / Workbench → UPDATE ActionPlanItem SET OwnerId = '<active_user_id>'
│   │   │     WHERE OwnerId = '<inactive_user_id>'
│   │   │
│   │   │   Option B (temporary): Reactivate user → let tasks transition → deactivate
│   │   │   - Setup → Users → find user → Edit → Active = TRUE → Save
│   │   │   - Wait for task transitions to complete (may need to re-complete predecessor)
│   │   │   - Then deactivate user again
│   │   │
│   │   └── Step 3b: Fix the template (prevent recurrence)
│   │       - Check template: are any items still assigned to inactive user?
│   │       - UPDATE template item assignees to active users or role-based assignment
│   │
│   └── If ALL owners are active → Not this pattern
│       → Check Pattern 1 (template corruption)
│       → Check for automation interference (flows/triggers on Task)
│
└── Step 4: Verify fix
    - Complete a predecessor task
    - Confirm the next task transitions from "Waiting" to "In Progress"
    - Check: ActionPlanItem.DependencyStatus should now = 'None'
    - Check: ActionPlanItem.ItemState should now = 'InProgress'
```

**Verification:**
After reassigning ownership:
1. Complete a predecessor task in the Action Plan
2. Verify the next task moves to "In Progress" within seconds
3. Query: `SELECT ItemState, DependencyStatus FROM ActionPlanItem WHERE Id = '<fixed_item_id>'`
4. Expected: `ItemState = 'InProgress'`, `DependencyStatus = 'None'`

**Prevention (Tell the Customer):**
> Before deactivating ANY user in your org, always check if they own Action Plan Items:
> ```sql
> SELECT COUNT(Id) FROM ActionPlanItem WHERE OwnerId = '<user_to_deactivate>' AND ActionPlan.Status != 'Completed'
> ```
> If count > 0 → reassign ownership BEFORE deactivating the user.

**GUS Investigation (if connected):**
```bash
sf data query --query "SELECT Name, Subject__c, Resolution__c, Root_Cause_Brief__c FROM ADM_Work__c WHERE Name = 'W-22451209'" --target-org gus --json
```

**Common Misdiagnosis:**
- ❌ "It's a Winter '26 release regression" — timing correlation with Oct 2025 is coincidental; it correlates with when the user was deactivated
- ❌ "Template is corrupted" — template is fine; it's the item ownership
- ❌ "Flow/automation was deactivated" — standard FSC dependency engine handles this, not a Flow

**Workarounds (from Slack):**
- Scheduled Flow that finds `WaitingOnPrevious` items whose ALL parents are `COMPLETED`, then reassigns ownership to an active admin and updates status
- Batch Apex to clean up orphaned AP items owned by inactive users
- Org-wide rule: add Action Plan item reassignment to the "user offboarding" checklist

**Related Documentation:**
- Action Plan Permissions: https://help.salesforce.com/s/articleView?id=sf.fsc_action_plans_permissions.htm
- User Deactivation Best Practices: https://help.salesforce.com/s/articleView?id=sf.deactivating_users.htm

**Escalation Criteria:**
- If reassigning ownership doesn't fix the stuck tasks → Escalate to FSC Engineering with W-22451209 reference
- If the issue persists for items with ACTIVE owners → different root cause, escalate
- Gather: Org ID, stuck ActionPlanItem IDs, owner User IDs, query results showing DependencyStatus

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

### PSL Developer Name ↔ Customer-Facing Label Reference

**USE THIS TABLE for all license/permission references in your responses.**

| Customer-Facing Label (visible in Setup UI) | Developer/API Name | What It Unlocks | Common Objects |
|---|---|---|---|
| **Financial Services Action Plans** | `ActionPlansPsl` | Action Plan Templates, Items, Dependencies | ActionPlan, ActionPlanItem, ActionPlanTemplate |
| **FSC Sales** | `FSCSalesPsl` | Core FSC sales objects + Action Plans (Sales context) | ActionPlan, Account (FSC fields), Financial Account |
| **FSC Service** | `FSCServicePsl` | Core FSC service objects + Action Plans (Service context) | ActionPlan, Case (FSC fields), Financial Account |
| **Financial Services Cloud Lending** | `FSCLendingPsl` | Lending objects, Document Decision Requirements | DocumentDecisionRequirement, LoanApplicant, LoanApplication |
| **Health Cloud Platform** | `HealthCloudPsl` | Health Cloud objects | CarePlan, CareProgram, Patient |
| **Financial Services Cloud Foundation** | `FSCFoundationPsl` | Base FSC objects (Accounts, Relationships) | PersonAccount (FSC), FinancialAccount, FinancialGoal |
| **Industries Common** | `IndustriesCommonPsl` | Shared Industries platform objects | AssessmentTask, DocumentChecklistItem |

### User Permission Labels Reference

| Customer-Facing Label (in Permission Set) | Developer/API Name | Required For |
|---|---|---|
| **Document Checklist User Access** | `DocumentChecklistUserAccess` | Creating/managing Document Checklist Items |
| **Assessment Platform User** | `AssessmentPlatformUser` | Assessment/Decision Table features |
| **Decision Table Execution User Access** | `DecisionTableExecUserAccess` | Running Decision Tables |
| **Customize Application** | `CustomizeApplication` | Admin-level metadata access |
| **Manage Financial Services Standard Objects** | `ManageFSCStdObjects` | Admin CRUD on all FSC standard objects |

### Multi-PSL Resolution Logic

**IMPORTANT: When a user cannot access an object, ALWAYS check ALL PSLs that could grant access.**

Many FSC objects are accessible through MULTIPLE licenses. Suggesting only one PSL is incorrect — the right PSL depends on what the customer purchased.

**Process:**
1. Identify the object/feature the user cannot access
2. Look up ALL PSLs that include that object (use table above)
3. Present ALL options with confidence levels
4. Tell the support engineer how to verify which one the customer has

**Example — User can't access DocumentDecisionRequirement:**
```
This object could be unlocked by any of these licenses:

| License (in Setup UI)                  | Confidence | Reasoning |
|----------------------------------------|------------|-----------|
| Financial Services Cloud Lending       | High       | Primary PSL for lending objects including DocumentDecisionRequirement |
| FSC Sales                              | Medium     | May include access via broader FSC platform entitlement |
| FSC Service                            | Medium     | May include access via broader FSC platform entitlement |

To verify: Setup → Company Information → Permission Set Licenses
→ Check which of these show Status = "Active" with Available > 0
```

**Example — User can't access Action Plan features:**
```
Action Plan access could come from any of these licenses:

| License (in Setup UI)                  | Confidence | Reasoning |
|----------------------------------------|------------|-----------|
| Financial Services Action Plans        | High       | Dedicated Action Plans PSL |
| FSC Sales                              | High       | Includes Action Plans for sales use cases |
| FSC Service                            | High       | Includes Action Plans for service use cases |

To verify: Setup → Company Information → Permission Set Licenses
→ Check which of these show Status = "Active"
→ Then: Setup → Permission Sets → verify user has the corresponding Permission Set assigned
```

### PSL Troubleshooting Flowchart

```
Can't access Action Plans features?
│
├── Check: Setup → Company Information → Permission Set Licenses
│   │
│   ├── "Financial Services Action Plans" NOT listed
│   │   ├── Sandbox? → Refresh from Production
│   │   └── Production? → License not provisioned → Contact License team
│   │
│   ├── Status = "Disabled"
│   │   └── Escalate to Support (cannot self-enable)
│   │
│   └── Status = "Active"
│       ├── Available = 0 → Remove from inactive users
│       └── Available > 0 → Check Permission Set license field
│           └── If "Salesforce" → change to blank or "FSC Sales"/"FSC Service"
│
├── Still can't access?
│   ├── Check: Do they have "FSC Sales" or "FSC Service" license? (either may grant AP access)
│   └── Check: Is "Action Plans" permission set ASSIGNED to user?
│
└── PSL looks fine but still can't use?
    ├── Check: Does user's profile explicitly DENY ActionPlan object access?
    └── Check: Are there any org-level features that need enabling?
        (Setup → Action Plans Settings → verify toggle is ON)
```

---

## E. Public-Facing Documentation (Share with Customers)

**ALWAYS include a relevant Help link when providing resolutions.** Support engineers need links they can share with customers or reference to verify expected behavior.

### Official Salesforce Help Articles

| Topic | Link | When to Reference |
|-------|------|-------------------|
| Action Plans Overview | https://help.salesforce.com/s/articleView?id=sf.fsc_action_plans.htm | Any AP case — start here |
| Action Plan Templates | https://help.salesforce.com/s/articleView?id=sf.fsc_action_plan_templates.htm | Template setup/config issues |
| Action Plan Permissions | https://help.salesforce.com/s/articleView?id=sf.fsc_action_plans_permissions.htm | Permission/access errors |
| FSC Licensing Overview | https://help.salesforce.com/s/articleView?id=sf.fsc_admin_licensing.htm | License/PSL issues |
| Permission Set Licenses (General) | https://help.salesforce.com/s/articleView?id=sf.perm_sets_managing_licenses.htm | How to manage PSLs |
| Document Checklist Items | https://help.salesforce.com/s/articleView?id=sf.fsc_document_checklist.htm | DocChecklist in AP errors |
| Sales Action Plans | https://help.salesforce.com/s/articleView?id=sf.sales_action_plans.htm | Sales vs FSC confusion |
| Action Plan Template Deployment | https://help.salesforce.com/s/articleView?id=sf.fsc_deploy_action_plan_templates.htm | ReadOnly/deployment issues |
| FSC Setup Guide | https://help.salesforce.com/s/articleView?id=sf.fsc_admin_setup.htm | General FSC setup |
| Known Issues (Action Plans) | https://issues.salesforce.com/?keywords=action+plan | Active bugs/known issues |

### Trailhead Modules

| Module | Link | Useful For |
|--------|------|-----------|
| Financial Services Cloud Basics | https://trailhead.salesforce.com/content/learn/modules/financial-services-cloud-basics | New FSC customers |
| Action Plans in FSC | https://trailhead.salesforce.com/content/learn/modules/fsc-action-plans | AP setup training |

**Usage Rule:** When you provide a resolution, ALWAYS append:
> **Reference:** [Relevant Help article title](link)

---

## E2. Key Behavioral Design Rules

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
| W-19402617 | 36 days | 0.41 days | ISE querying ActionPlanTemplate.status field |
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

**If GUS is connected, run this query to get current open bugs:**

```bash
sf data query --query "SELECT Name, Subject__c, Priority__c, Status__c, CreatedDate FROM ADM_Work__c WHERE Product_Tag__r.Name = 'FSC Action Plans' AND Type__c = 'Bug' AND Status__c NOT IN ('Closed', 'Duplicate', 'Never Fix', 'Not a Bug') ORDER BY Priority__c, CreatedDate DESC LIMIT 15" --target-org gus --json
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
Repository: gitcore.soma.salesforce.com/core-2206/core-262-public
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

## I2. Code-to-Support Translation Guide

**PURPOSE:** When you use CodeSearch or Columbo to debug an issue, you MUST translate the raw findings into language a support engineer can understand and act on. This section tells you HOW to translate.

### Translation Process

```
STEP 1: Find the issue in code (CodeSearch/Columbo)
        ↓
STEP 2: Identify what type of finding it is (permission check, config requirement, 
        feature flag, object access, etc.)
        ↓
STEP 3: Map to the customer-facing equivalent using tables in Section D
        ↓
STEP 4: Present as a Setup UI action with exact navigation path
```

### Common Code Patterns → Support-Friendly Translation

| What You Find in Code | What You Tell the Support Engineer |
|---|---|
| `requirePermission("DocumentChecklistUserAccess")` | "The user needs the **Document Checklist User Access** permission. Check: Setup → Permission Sets → [relevant PS] → System Permissions → verify it's enabled" |
| `checkPSL("FSCSalesPsl")` | "This requires the **FSC Sales** license. Verify: Setup → Company Information → Permission Set Licenses → look for 'FSC Sales' with Status = Active" |
| `checkPSL("FSCLendingPsl")` | "This requires the **Financial Services Cloud Lending** license. Verify: Setup → Company Information → Permission Set Licenses → look for 'Financial Services Cloud Lending'" |
| `checkPSL("ActionPlansPsl")` | "This requires the **Financial Services Action Plans** license. Verify: Setup → Company Information → Permission Set Licenses → look for 'Financial Services Action Plans'" |
| `isFeatureEnabled("DocumentChecklist")` | "The **Document Checklist** feature must be enabled. Check: Setup → Document Checklist Settings → verify toggle is ON" |
| `isFeatureEnabled("IndustriesAssessment")` | "The **Industries Assessment** feature must be enabled in the org. Check: Setup → Assessment Settings" |
| `isFeatureEnabled("DecisionTable")` | "The **Decision Table** feature must be enabled. Check: Setup → Decision Table Settings" |
| `hasObjectAccess("ActionPlan", "CREATE")` | "The user needs **Create** permission on the **Action Plan** object. Check: Setup → Profiles/Permission Sets → Object Settings → Action Plans → verify Create is checked" |
| `ActionPlanTemplateTypeEnum.SALES` | "The org is using **Sales Action Plans** (for Account Plans). This type has limited features compared to FSC Action Plans. Check: Setup → Action Plans Settings" |
| `ActionPlanTemplateTypeEnum.INDUSTRIES` | "The org is using **FSC Action Plans** (full feature set). Check: Setup → Action Plans Settings" |
| `NullPointerException in HighlightsPanelController` | "There's an internal error in the Action Plan template editor. This is a platform issue, not a customer configuration problem. Workaround: try publishing via API/Workbench instead of the UI" |
| `SfdcSqlException in populateDerivedStatusField` | "The system encountered a data error when calculating the Action Plan status. This usually means the template has corrupted data. Resolution: Clone the template → delete the old one → publish the clone" |

### When Code Reveals a Missing Permission — Full Response Template

When CodeSearch reveals that a feature requires specific permissions/licenses, present it like this:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FINDING: [Object/Feature Name] requires specific licensing

WHAT'S NEEDED:
This feature requires the following (check ALL, not just one):

| What to Check (Setup UI) | Where to Find It | Expected State |
|---|---|---|
| [License Name - UI Label] | Setup → Company Information → Permission Set Licenses | Status = Active, Available > 0 |
| [Permission Name - UI Label] | Setup → Permission Sets → [PS Name] → System Permissions | Enabled (checked) |
| [Object Access - UI Label] | Setup → Permission Sets → [PS Name] → Object Settings → [Object] | CRUD as needed |
| [Feature Toggle - UI Label] | Setup → [Feature] Settings | Enabled (ON) |

POSSIBLE LICENSES (check which customer has):
| License (Setup UI Name) | Confidence | Notes |
|---|---|---|
| [Primary] | High | Most likely needed |
| [Alternative 1] | Medium | May also grant access |
| [Alternative 2] | Medium | May also grant access |

HOW TO VERIFY:
1. Go to Setup → Company Information → Permission Set Licenses
2. Look for [License Name] — confirm Status = "Active" and Available > 0
3. Go to Setup → Users → [Affected User] → Permission Set Assignments
4. Verify they have a permission set that includes [Permission Name]
5. If missing → Assign the relevant permission set

REFERENCE: [Help Article Link]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

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
