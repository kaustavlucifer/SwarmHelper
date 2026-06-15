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
│   → Go to: action-plans-patterns.md (Patterns 1-10)
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

**With CodeSearch (verified — repo: gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public):**
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
sf data query --query "SELECT Id, Name, Subject__c, Status__c, Priority__c FROM ADM_Work__c WHERE Product_Tag__r.Name LIKE '%FSC Action Plans%' AND Type__c = 'Bug' AND Status__c NOT IN ('Closed', 'Duplicate', 'Never Fix') ORDER BY Priority__c, CreatedDate DESC LIMIT 10" --target-org gus --json
```
→ Checks if this is a known open bug.

**With Slack:**
```
Search: "in:#tech-prod-help-financial-services-cloud <error_message_or_keyword>"
```
→ Finds if someone already solved this recently.

---

## B–J. Action Plan Patterns, Setup, Licensing, Code Map, Gacks

> **Action Plan troubleshooting lives in [`action-plans-patterns.md`](action-plans-patterns.md)** — the authoritative source for Patterns 1–10 (successor tasks, DocumentChecklistItem, PSL, reordering, mobile, ReadOnly deploy, guest user, Sales-vs-FSC, inactive-user dependency engine, item-list refresh), plus the Setup Checklist, PSL/licensing reference, Code Reference Map, and Gack lookup. Not duplicated here.

The patterns below (K–R) cover the **rest of FSC** — Referrals, Rollup Summary Rules, Financial Accounts, Timeline, Households, Interaction Summaries, Goals/Life Events, and the data model.

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
sf data query --query "SELECT Name, Subject__c, Status__c FROM ADM_Work__c WHERE Product_Tag__r.Name LIKE '%FSC Referrals%' AND Type__c = 'Bug' AND Status__c NOT IN ('Closed','Duplicate','Never Fix') LIMIT 10" --target-org gus --json
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
