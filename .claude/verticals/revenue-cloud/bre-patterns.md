## A. Triage & Classification

### Quick Diagnostic Questions

Ask these first to narrow the issue category:

1. **What is the exact error message?**
2. **Which BRE component is affected?** (Decision Table, Expression Set / Pricing Procedure, Context Definition, Rule Library, CML / Constraint Model, Qualification Rules)
3. **Is this a new setup or was it working before?**
4. **Was there a recent sandbox refresh, deployment, or org upgrade?**
5. **What is the Decision Table / Expression Set status?** (Activation In Progress, Inactive, Error)
6. **What permission sets does the affected user have?**
7. **Is this a scratch org, sandbox, or production?**

### Decision Tree

```
Customer reports a BRE issue
│
├── Decision Table not activating / stuck in "Activation In Progress"
│   ├── "Hash Key Group contains more than 50 rows" (or 200 rows in release 260+) → Pattern 1
│   ├── "Source object permissions or fields modified" → Pattern 1
│   ├── Stuck with no error → Pattern 1
│   ├── Special characters in Lookup Table key not allowed → Pattern 1
│   └── Record count exceeds org limit (100K/20M) → Pattern 1
│
├── Decision Table deployment failure
│   ├── "LatestVersionSnapshotId not found" → Pattern 2
│   ├── Expression Set Definition random IDs error → Pattern 2
│   ├── "Ensure lookup table in PriceAdjustmentMatrix step is valid" → Pattern 2
│   ├── "CreatedDate is a required variable in the {1} step" → Pattern 2
│   └── Context Definition deployment failure / migration not supported → Pattern 2
│
├── Context Definition cannot be deactivated
│   ├── Referenced by Expression Set / Rule Library → Pattern 3
│   └── Referenced by Dynamic Rules → Pattern 3
│
├── Permissions / access issues
│   ├── "No Decision Table exists with name: StandardTax" → Pattern 4
│   ├── "Insufficient Privileges" on DecisionTable.Execute → Pattern 4
│   ├── Decision Table access denied after security refresh → Pattern 4
│   ├── BRE Runtime PSL missing / not available in org → Pattern 4
│   ├── "You don't have the permission to access the expression set version" → Pattern 4
│   └── Community users can't refresh prices → Pattern 4
│
├── Configuration Rules / CML errors
│   ├── "Configuration rule refers to different type of sales transaction item" → Pattern 5
│   ├── CML not activating / missing association error → Pattern 5
│   ├── Stale attribute values (CML) → Pattern 5
│   ├── Boolean attribute case mismatch (Instant Pricing) → Pattern 5
│   └── CML type declaration hard limit → Pattern 5
│
├── Qualification / Disqualification Rules not working → Pattern 6
├── Expression Set / Pricing Procedure errors → Pattern 7
│   ├── Expression Set stops working unexpectedly in sandbox (even without changes) → Pattern 7
│   ├── "The rule name X is invalid" when executing ES via Apex in test class → Pattern 7
│   └── "Something went wrong" / Procedure save fails after DT incremental refresh → Pattern 7
├── Decision Table limits exceeded (200-input, record counts) → Pattern 8
│   ├── "consolidated columns exceeds limit of 10/15 table rows" → Pattern 8
│   └── "MaxBulkLookupInputHbpoDt" limit exceeded → Pattern 8
└── Rule Library issues → Pattern 9
```

---

## B. Known Issue Patterns

---

### Pattern 1: Decision Table Not Activating / Stuck in "Activation In Progress"

**Frequency:** High (30+ cases)
**Severity:** P11–P45
**TTR Impact:** 3–7 days

**Symptoms:**
- Decision Table stuck in `Activation In Progress` status with no way to cancel or retry
- Error: `Hash Key Group contains more than 50 rows` (pre-release 260) or `Hash Key Group contains more than 200 rows` (release 260+)
- Error: `We can't show the decision table because the source object permissions or fields linked to it were modified. You need to recreate the decision table with the updated source object configuration.`
- Error: `PricebookEntry contains 231008 rows which exceeds the set Org Limit of 100000 rows per Decision Table`
- Error: `The source object PricebookEntry contains 28157580 rows which exceeds the set Org Limit of 20000000 rows per Decision Table`
- Decision Table activation fails after sandbox refresh or org upgrade
- Cannot create pricing record in Lookup Table with a special character in the key field (e.g., product name with `®` symbol)
- Asset Action Source Entries / Asset Action Source Entries V2 Decision Tables not created by default after enabling Revenue Cloud

**Root Cause:**
- **Hash Key Group limit:** For Advanced (Medium Volume) Decision Tables, no single hash key group can have more than 50 rows per non-equals column. **Increased to 200 in release 260** (`MaxRowsPerHashKey`). Still a hard limit — cannot be further extended beyond the platform default for the release.
- **Source object modified:** A field or object used by the Decision Table was changed (added, removed, or modified) after the DT was created. The DT must be recreated.
- **Record count exceeded:** PricebookEntry or source object has more rows than the org limit (default 100K, max 20M for some DTs). Requires a limit increase via LAP/engineering.
- **Stuck in Activation In Progress:** No error in Splunk, status frozen — requires backend intervention or enable/disable toggle workaround.
- **Special characters in Lookup Table keys:** BRE Lookup Table cannot use product names with special characters (e.g., `®`, `™`) as matching keys. The matching is done at the field level and special characters break the lookup.

**Resolution Steps:**

**For "Source object permissions or fields modified":**
1. Go to Setup → Decision Tables
2. Delete the affected Decision Table (e.g., Price Book Entries, Volume Discount Entries, Tiered Adjustment Entries, Attribute Discount Entries, Bundle Based Adjustment Entries)
3. Remove pricing recipe table mapping references from Workbench if needed
4. Disable the org preference from Salesforce Pricing Settings page
5. Re-enable the org preference — this auto-recreates the Decision Tables and adds mappings to the pricing recipe
6. Reference GUS: a07EE00002ER899YAD

**For "Hash Key Group > 50 rows" (or > 200 in release 260+):**
1. **Release 260+:** The limit was raised from 50 to 200 (`MaxRowsPerHashKey`). Confirm the customer's org release version first.
2. **Still hitting the limit:** Three workarounds available:
   - Use **Low Volume DT** (Standard DT) — no Hash Key Group limit, but requires < 25K rows without groupBy (or up to 1M rows with groupBy)
   - **Add a required EQUALS column** whose values vary — this splits hash groups so each group has fewer rows
   - Apply a **source filter** to reduce rows per hash key group below the limit
3. **Advanced DT with > 25K rows and Hash Key limit still hit:** Must restructure DT shape — make fields that vary most as Required-Equals so rows per hash are reduced. Engage the NGP/pricing team who know the data shape best.
4. Reference GUS doc bug: a07EE00002A7WUFYA3

**For special characters in Lookup Table keys:**
1. Create a new sanitized field on the Product object (e.g., `ProductLookupCode__c`) that strips or replaces special characters
2. Use a Flow or Apex Trigger to auto-populate this field whenever the product is created/updated
3. Update the Lookup Table to use `ProductLookupCode__c` as the matching key instead of the Product Name
4. When calling the pricing procedure, pass the sanitized lookup code — keep the original product name for display
5. This decouples UI display (with special chars) from BRE lookup logic (sanitized key)

**For record count exceeded (100K / 20M):**
1. For 100K limit: Add filters to the Decision Table to reduce the number of source records included
2. For 20M limit: Submit a LAP request to increase the org limit (requires engineering involvement)
3. Reference prior swarm for limit increase process

**For DT refresh triggered per-record instead of per-batch (bulk insert/update):**
1. This occurs when a Flow or Apex trigger calls DT refresh for each record individually during a bulk operation
2. Root cause: the calling code invokes DT refresh inside a record-level loop instead of bulkifying
3. Workaround: restructure the calling Flow or Apex to collect all records first, then invoke refresh once for the batch
4. If the per-record refresh is platform-triggered (not custom code): file GUS investigation — this is unexpected behavior
5. Reference: Case 473536152

**For stuck "Activation In Progress" with no error:**
1. Enable "Rate Management" from Usage Management Settings in Setup (restores access to Rating DTs)
2. Mark all DTs with `ActivationInProgress` status to `InActive` (may need backend/engineering assistance)
3. Refresh the Decision Table
4. If unable to change status via UI: file a GUS investigation — reference GUS a07EE00002Hn9B4YAJ, a07EE000028drFDYAY

**For Asset Action Source Entries V2 not created by default:**
1. After enabling Revenue Cloud, these tables should auto-create — if missing, check if Revenue Cloud was fully activated
2. May need to toggle Revenue Cloud settings off and back on
3. Reference Case 470009650

**Verification:**
- Query: `SELECT Id, DeveloperName, Status, RefreshStatus FROM DecisionTable LIMIT 200`
- Confirm all needed DTs are in `Active` status

**Workarounds (from Slack):**
- Source object modified: disable → delete → re-enable Salesforce Pricing toggle recreates all standard DTs cleanly (Divyanshu Tiwari, #support-rev-rlm-global-swarm-help, 2025-09-01)
- Hash Key Group limit: limit increased from 50 to 200 in release 260. Before that, restructure to Low Volume DT or add more EQUALS columns to split groups. (Satyam Shekhar, #help-bre, 2026-01-25)
- Activation In Progress: enable Rate Management → mark InActive → refresh DT (Anita Kumari, #support-rev-rlm-global-swarm-help, 2026-05-07)
- Special characters in product names: create a sanitized lookup code field on Product; use that for DT matching; preserve original name for UI display (Himanshu Sekhar Sahoo, #help-bre, 2025-09-29)

**Escalation Criteria:**
- Record count exceeds 20M and limit increase via LAP fails → escalate to engineering with org ID
- Stuck in Activation In Progress with no Splunk error and no UI option to reset → escalate with DT ID and org ID

---

### Pattern 2: Expression Set / Pricing Procedure / Context Definition Deployment Failures

**Frequency:** High (25+ cases)
**Severity:** P0–P27
**TTR Impact:** 3–10 days

**Symptoms:**
- `Deployment error: LatestVersionSnapshotId not found`
- `Deploying Expression Set Definition version throws error: We couldn't find a record with the ID: 9QBVF0000000DEz (random IDs). Replace the specified ID with a valid record ID and try again.`
- `Unable to deploy Expression Set Definition VERSION to target org — missing ID for each failure`
- `Expression set version corrupted after moving environments`
- `Failures deploying ContextDefinition`
- `Trouble deploying RCA-specific components from sandbox to production`
- `Deployment issue with Decision Table` (via Change Set, VS Code, or Copado)
- `Ensure that the lookup table in the PriceAdjustmentMatrix step is valid and try again`
- `[긴급] API version mismatch (Spring '26) causing deployment failure`
- `CreatedDate is a required variable in the {1} step` (deploying ExpressionSetDefinition with upsert on TimeSheetValidationError)
- Expression Set upgrade failed — cannot upgrade
- Unable to deploy Context Definition from version 254 to 256
- Customer wants to migrate BRE Context Rules from one org to another — fails or migration not possible
- `Duplicate Context Attribute Mappings` created in destination org after deployment — context attribute mappings duplicated during CD deployment

**Root Cause:**
- Expression Set Definition deployments include internal snapshot IDs that are environment-specific — they must be regenerated in the target org, not copied
- Context Definition CD-to-CD mappings cause deployment issues if target org doesn't have the referenced CDs
- Cloned Context Definitions (instead of extended) don't auto-upgrade and create stale configurations
- API version mismatch between source and target org during release windows
- Pricing Procedure with custom Decision Table: when deploying both together in one package, the DT reference resolution fails
- **BRE Context Rules do not support packaging or migration** via Tooling API / Dynamic Rules Connect API — `ContextRulePilot` permission is not available in production environments, making these APIs unusable for migration. The BRE team has no GA plan for these APIs.

**Resolution Steps:**
1. **For "LatestVersionSnapshotId not found" / random IDs:**
   - Do NOT deploy Expression Set Definition versions directly via change sets if they contain internal IDs
   - Deploy the Expression Set Definition (not the version) first, then recreate/save a new version in the target org
   - Alternatively, use Metadata API deployment with the Expression Set Definition only, then manually activate a version in target
2. **For Context Definition deployment failures:**
   - Verify all referenced Context Definitions exist in the target org before deploying CD-to-CD mappings
   - Use `Extend` instead of `Clone` for Context Definitions — cloned CDs require manual syncing and don't auto-upgrade
3. **For Pricing Procedure + custom Decision Table in same package:**
   - Deploy the Decision Table first as a separate deployment
   - Then deploy the Pricing Procedure referencing the already-deployed DT
4. **For API version mismatch:**
   - Check source and target org API versions — if different (e.g., Spring '26 gap), deploy after both orgs are on same version
   - Or use API version-compatible metadata format
5. **For expression set upgrade failure:**
   - Check if the expression set has references to deprecated fields or objects
   - Engage engineering if upgrade cannot proceed — may require backend fix
6. **For "CreatedDate is a required variable in the {1} step":**
   - This occurs when deploying ExpressionSetDefinition that has a step with an upsert on certain objects (e.g., TimeSheetValidationError)
   - Workaround: remove the upsert step or ensure the required variables (CreatedDate, CreatedById) are explicitly mapped in the definition
   - File escalation to BRE engineering with the deployment metadata XML
7. **For Duplicate Context Attribute Mappings after deployment:**
   - After deploying a Context Definition to a destination org, duplicate mappings may appear
   - Remove duplicate mappings manually via the Context Definition UI or via API
   - To prevent: ensure the target org doesn't already have mappings before deploying — clear them first
   - Reference: Case 470054060 (duplicate context attribute mappings in destination org)

8. **For BRE Context Rules migration between orgs:**
   - **BRE context rules (Rulesets, RuleValues, RuleActions, RuleFilterCriteria) do NOT support packaging or automated migration.** The Tooling API and Dynamic Rules Connect API require `ContextRulePilot` permission which is not available in production.
   - The only path is **manual recreation** in the target org
   - Reference: Tong Zhang investigation, TD-0246994 (#help-bre, 2026-05-07)

**Verification:**
- Deploy succeeds without errors
- Expression Set / Pricing Procedure is active in target org
- Context Definition is active and mappings are correct

**Workarounds (from Slack):**
- Expression Set Definition version IDs: deploy the definition shell only, then recreate the version in the target org manually (Avinash Saklani, #support-rev-rlm-global-swarm-help)
- Copado deployments: same ID-resolution issue — deploy via SFDX CLI as a workaround, or pre-deploy DTs separately (Victor Shapiro, #rlm-office-hours)
- Extend vs Clone: always use `Extend` for Context Definitions — cloned CDs require manual sync and break on upgrades (Engineering Agent, #rlm-office-hours)
- Context Rules migration: cannot be packaged or migrated via API in production — must manually recreate in target org (Anmol Bansal, #help-bre, 2026-04-15)

**Related Documentation:**
- https://help.salesforce.com/s/articleView?id=ind.business_rules_engine.htm&type=5

**Escalation Criteria:**
- Expression Set version deployment fails with new random IDs on every attempt and no pattern — escalate with source and target org IDs and metadata XML

---

### Pattern 3: Context Definition Cannot Be Deactivated

**Frequency:** Medium (10+ cases)
**Severity:** P27–P45
**TTR Impact:** 3–7 days

**Symptoms:**
- `Unable to deactivate Context Definition`
- `Cannot disable context definition due to dynamic rules reference`
- `Context Definition unable to be deactivated for editing`
- `Unable to deactivate Context Definition due to Expression Set reference`
- `Can't disable context definition due to dynamic rules reference`
- "Not able to deactivate the Context Definition"

**Root Cause:**
- Context Definitions cannot be deactivated while active Expression Sets, Pricing Procedures, Rule Libraries, or Dynamic Rules reference them
- Circular references between CDs and Rule Library versions prevent deactivation
- Rulesets referencing the CD must be deactivated/deleted first

**Resolution Steps:**
1. Identify all components referencing the Context Definition:
   ```sql
   -- Find active Rule Library versions referencing the CD
   SELECT ApiName, Status, VersionNumber FROM RulesetVersion 
   WHERE Ruleset.ApiName = '<RULESET_API_NAME>'
   ```
   *(Requires `DebugContextRulePilot` permission to run RulesetVersion queries)*

2. **Steps to deactivate (BRE approach — coordinate with DRO first):**
   1. Identify which rulesets are not needed: `SELECT ApiName FROM Ruleset`
   2. Find active ruleset versions of those rulesets (needs `DebugContextRulePilot` perm)
   3. Update ruleset versions to Inactive state via Connect API (needs `ContextRulePilot` perm)
   4. Delete ruleset via Connect API (needs `ContextRulePilot` perm)
   5. Sample LAP requests for `ContextRulePilot` and `DebugContextRulePilot` perms available in #rlm-office-hours
   6. Connect API details in DRO Handbook

3. Deactivate all referenced Expression Set versions that point to this CD
4. Deactivate all referenced Dynamic Rules
5. Then deactivate the Context Definition

**Required Permissions for Steps:**
- `DebugContextRulePilot` — for querying RulesetVersion
- `ContextRulePilot` — for updating/deleting rulesets via Connect API
- Both require LAP requests

**Verification:**
- Context Definition status changes to `Inactive`
- Can now edit and reactivate

**Workarounds (from Slack):**
- Cannot deactivate via UI: update ruleset versions to Inactive via Connect API first, then delete ruleset — requires `ContextRulePilot` perm via LAP (Shruti Biswal, #rlm-office-hours, 2026-03-06)
- DRO Handbook link: https://docs.google.com/document/d/1LiqwZG1tnD_jjwLZfowNxk6_a8eW4wvvPm-4Q8ZPsy4/edit

**Escalation Criteria:**
- Cannot get `ContextRulePilot` LAP approved → escalate to engineering for backend deactivation
- Circular reference between CDs that cannot be broken → escalate

---

### Pattern 4: BRE Permissions & Access Issues

**Frequency:** High (15+ cases)
**Severity:** P10–P45
**TTR Impact:** 1–3 days

**Symptoms:**
- `No Decision Table exists with name: StandardTax`
- `Insufficient Privileges: This feature is not currently enabled for this user` (on `ConnectApi.DecisionTable.Execute`)
- Decision Table access denied after security refresh
- Decision Tables recreate after sandbox refresh and cannot grant permission
- Community / partner users cannot refresh prices — pricing procedure fails
- `Missing RCA Tables` after sandbox refresh
- Discrepancy in Context Service Runtime Licenses in Production
- BRE Runtime PSL not available in org — only BRE Designer PSL present
- `You don't have the permission to access the expression set version. Your Salesforce admin can help with that.`
- Decision Table Refresh button not visible for non-System Administrator users

**Root Cause:**
- **"No Decision Table exists with name: StandardTax":** Sales Rep user needs Tax Admin or Tax Configurator permission set to access the standard tax Decision Table — there is no minimal permission set that avoids this requirement
- **`Insufficient Privileges` on `ConnectApi.DecisionTable.Execute` run as Automated Process:** Automated Process user cannot execute Decision Tables without special permissions
- **After security refresh:** Decision Tables may be recreated with new IDs; old permissions no longer apply — must reassign
- **Community users:** Need `Rule Engine Community Runtime` permission set to access Pricing Procedures in Experience Cloud
- **BRE Runtime PSL not present:** Some SKUs (e.g., PSF-A, Communications Cloud Advanced Sales & Service Unlimited) include only BRE Designer PSL, not BRE Runtime PSL. Customers with these SKUs must create a custom permission set cloned from BREDesigner as a workaround. **"Employee Experience for Public Sector"** SKU includes BRERuntime PSL — distinct from PSF-A.
- **"You don't have the permission to access the expression set version":** User is missing BRE Designer PSL assignment (even System Admins need the PSL assigned).
- **Refresh button not visible:** DT Refresh button is only available for System Administrator profile or users with BRE Designer permission. Users with only Runtime permission cannot trigger a refresh.

**Resolution Steps:**
1. **"No Decision Table exists with name: StandardTax":**
   - Assign **Tax Admin** OR **Tax Configurator** permission set to the user
   - Note: these include `View Setup and Configuration` — currently no way to grant StandardTax DT access without this. Log VOC if customer objects.
   - Verify the StandardTax Context Definition and DT are both Active
   - Reference: https://help.salesforce.com/s/articleView?id=ind.revenue_cloud_permission_sets_table.htm&type=5

2. **`Insufficient Privileges` on DecisionTable.Execute (Automated Process):**
   - Assign `Business Rules Engine Runtime` PSL + `Run Decision Table` system permission to the running user
   - If running as Automated Process: check if a Named Credential or specific user context needs to be set
   - May require LAP for enabling `Run Decision Table` permission

3. **After sandbox refresh (Decision Tables recreated):**
   - Re-assign `Business Rules Engine Designer` and `Business Rules Engine Runtime` PSLs to affected users
   - Verify all standard DTs are Active (query above)
   - If `Missing RCA Tables`: toggle Salesforce Pricing setting off and back on to recreate standard tables

4. **Community / Experience Cloud users:**
   - Assign `Rule Engine Community Runtime` permission set
   - Note: operation may take ~30 seconds after permission is added (Vani Madhavi Latha, #support-rev-rlm-global-swarm-help)

5. **Context Service Runtime License discrepancy:**
   - Check license counts in Setup → Company Information
   - If production shows fewer CSR licenses than expected: file GUS investigation or LAP request

6. **BRE Runtime PSL missing from org:**
   - Confirm the customer's SKU — BRE Runtime PSL is NOT included in all SKUs
   - **PSF-A / Public Sector Foundation SKU:** Does NOT include BRERuntime PSL. Workaround: clone the `BRE Designer` permission set and assign to users who need runtime access. Reference: Salesforce Quip PSS FAQ https://salesforce.quip.com/MwmRAK2TJyZj#temp:C:aNe810fe5acb6434dc0aac543733
   - **Employee Experience for Public Sector SKU:** Includes BRERuntime PSL — distinct from PSF-A
   - **Communications Cloud Advanced (Sales & Service Unlimited):** Includes only BRE Platform Access and Designer — not Runtime. File a provisioning request or have customer buy BRE Add-on (100K calls SKU)
   - **BRE is not sold standalone** — BRE PSLs come with base industry cloud or Service Cloud license. Additional entitlements: buy `Business Rules Engine - calls 100k` SKU
   - **BRERuntime PSL required for:** invoking any BRE call (ES, DM, DT). Assigning BRERuntime is sufficient for execution.

7. **"You don't have the permission to access the expression set version":**
   - Assign `BRE Designer` PSL to the user — required even for System Admins to open Expression Set Builder
   - After assigning, try refreshing the session
   - If Expression Set was created via Connect API with `ruleId=undefined` in URL: manually append the correct ExpressionSetVersion ID to the ruleBuilder URL path, then activate/reactivate the version (AKASH KUMAR, #industries-bre-officehours, 2026-04-06)

8. **Decision Table Refresh button not visible for custom profile:**
   - Confirm user has `Business Rules Engine Designer` PSL assigned (Runtime alone is insufficient for refresh)
   - Refresh is restricted to Designer + Admin users by platform design

**Required BRE Permission Sets:**

| Capability | BRE Designer PSL | BRE Runtime PSL | Run Decision Table | Tax Admin/Configurator | Rule Engine Community Runtime |
|---|---|---|---|---|---|
| Create/edit Decision Tables | ✅ | | | | |
| Execute Decision Tables (standard users) | | ✅ | ✅ | | |
| Access StandardTax Decision Table | | | | ✅ (one of them) | |
| Automated Process / ConnectAPI | | ✅ | ✅ | | |
| Experience Cloud pricing | | ✅ | | | ✅ |
| Refresh Decision Table | ✅ (Designer) | | | | |
| Open Expression Set Builder | ✅ (Designer) | | | | |

**BRE PSL Availability by SKU (partial list — verify in org):**

| SKU | BRE Designer | BRE Runtime | Notes |
|---|---|---|---|
| Revenue Cloud (RCA/RLM) | ✅ | ✅ | Included |
| Employee Experience for Public Sector | ✅ | ✅ | Runtime intended for employee-constituent use |
| PSF-A (Public Sector Foundation A) | ✅ | ❌ | Clone BREDesigner PS as workaround |
| Communications Cloud Advanced (Sales+Service Unlimited) | ✅ | ❌ | Only Platform Access + Designer |
| Service Cloud (base) | ✅ (limited) | ✅ (limited) | 10K free entitlements |
| Sales Cloud only | ❌ | ❌ | BRE requires industry cloud or add-on |

**Workarounds (from Slack):**
- StandardTax issue: Tax Admin or Tax Configurator is required regardless — no minimal permission exists currently. Log VOC for customers who cannot grant View Setup to sales users. (Gulla Bharath Kumar, #rlm-office-hours, 2026-05-04)
- After security refresh, if Decision Tables have new IDs: disable/enable Salesforce Pricing toggle to recreate standard tables and mappings (multiple cases)
- Community users pricing: add `Rule Engine Community Runtime` PSL — works but takes ~30 seconds (Vani Madhavi Latha, #support-rev-rlm-global-swarm-help, 2026-03-11)
- BRE Runtime PSL not included in SKU: clone BREDesigner PS and assign to users (Scott Johnson, #help-bre, 2026-05-21; PSS FAQ reference)
- Expression Set builder "Builder ID couldn't be determined" (ruleId=undefined): manually add ExpressionSetVersion ID to URL path and activate (AKASH KUMAR, #industries-bre-officehours, 2026-04-06)

**Escalation Criteria:**
- All required PSLs assigned correctly and DTs are Active — still "No Decision Table exists" → escalate with Splunk trace log
- Cannot grant Tax Admin/Configurator PSL due to org security policy — escalate for engineering to provide minimal permission path
- BRE Runtime PSL count dropped to 0 after a provisioning run — check provisioning event and file LAP if legitimate entitlement was removed

---

### Pattern 5: Configuration Rules / CML / Constraint Model Errors

**Frequency:** Medium (15+ cases)
**Severity:** P9–P45
**TTR Impact:** 3–10 days

**Symptoms:**
- `Something went wrong while running configuration rules: The configuration failed because the configuration rule refers to a different type of sales transaction item.`
- `Model '<Name>' is invalid: Missing association for product(s) [<ProductName>] of relation <relation> and parent type <type>.`
- `Cannot invoke "com.salesforce.configurator.variable.InstanceVariable.isBound()" because "instanceVariable" is null`
- CML not activating — silent failure or immediate deactivation on activation attempt
- CML type declaration hard limit hit (cannot add more product types)
- Stale attribute values after back navigation in configurator (CML not re-evaluating dependent attributes)
- Boolean attribute case mismatch: Instant Pricing passes `"FALSE"` (uppercase) but ABA expects `"false"` (lowercase)
- CML multi-OR performance degradation: latency/timeout in quote line editor
- `LOTUS - BRE name is giving 80 character error on standard field`
- `Context Definition Attribute Limit` — cannot add more attributes to SalesTransactionItem node (cases 37, 40 — signature customers blocked by this)

**Root Cause:**
- **"Different type of sales transaction item":** Configuration rule scope (Bundle/Product) doesn't match the transaction type — often caused by enabling then disabling Advanced Configurator, which leaves residual transaction type configuration
- **"Missing association" error on CML activation:** A product was added to the CML but its association was not properly created; or a product was removed/renamed and the association reference is stale. Creating a new CML with similar setup works — suggests cache/state issue on existing CML
- **Context Definition Attribute Limit:** The SalesTransactionItem (STI) node in the Context Definition has a hard limit on the number of attributes. When hit, users cannot add more attributes. Workaround: submit a LAP request to increase the attribute limit per node. Contact Victor Shapiro / Engineering for limit increase.
- **CML type declaration hard limit:** Cannot add more product type declarations beyond the platform limit. Workaround: optimize multi-OR logic using boolean formula fields instead of chaining `||` conditions
- **Boolean case mismatch (Instant Pricing):** Known defect — Instant Pricing passes boolean values as uppercase `"FALSE"` while ABA condition expects lowercase `"false"`. Works correctly after Save & Exit. Engineering investigation in progress.
- **Stale attribute values:** Known issue — CML constraint engine does not re-evaluate dependent attribute values when upstream attribute is changed more than once in same configurator session. Known Issue reference: a02Ka00000lYbeAIAS

**Resolution Steps:**
1. **"Different type of sales transaction item":**
   - Check if Advanced Configurator was previously enabled and then disabled
   - Verify Revenue Settings → Transaction Types are correctly configured for Standard Configuration
   - Test with the OOTB Standard SalesTransactionContextDefinition (not a custom/extended one)
   - In Rule Library configuration: standard SalesTransactionContextDefinition should be referenced; for versioning, the extended SalesTransactionContextDefinition should be referenced
   - If issue appeared after disabling Advanced Configurator: reconfigure Rule Library to reference correct CD

2. **"Missing association" on CML activation:**
   - Verify the association exists for all products referenced in the CML
   - Try importing the relationship (reimport product associations)
   - If existing CML is corrupted: create a new CML with the same setup — new CML typically works correctly
   - Check if this is a cache issue — try in a different browser/session

3. **CML type declaration limit:**
   - Reduce multi-OR chained logic: replace `ProdQuote[Product].count(LineStatus == "New") > 0 || ProdQuote[Product].count(LineStatus == "Renewed") > 0` with a boolean formula field (e.g., `Is_Active_Status__c`) mapped to Context Service, then use `ProdQuote[Product].count(IsActiveStatus == true) > 0` — reduces CML size and improves performance

4. **Boolean case mismatch (ABA + Instant Pricing):**
   - Current workaround: customer completes Save & Exit to create the Quote Line Item, then the ABA applies correctly
   - This is an active known defect — file GUS investigation if not already filed
   - Reference: Case 473510739, prior case 473428919

5. **Stale attribute values after back navigation:**
   - Known issue (a02Ka00000lYbeAIAS) — no workaround currently
   - Advise customer to avoid back navigation in configurator session
   - File GUS investigation if not already tracked

**Workarounds (from Slack):**
- CML type limit / multi-OR performance: use boolean formula field on Quote Line Item mapped to Context Service — single variable check instead of chained `||` conditions reduces both CML size and runtime latency significantly (Kaushik Jagatap, #support-rev-rlm-global-swarm-help, 2026-04-08)
- "Different type" error after disabling Advanced Configurator: reconfigure Rule Library to use Standard SalesTransactionContextDefinition (Aabid Khan, #support-rev-rlm-global-swarm-help, 2025-11-06)
- Missing association — new CML works: create new CML as workaround while investigating root cause on existing CML (Claudio Crocilla, #support-rev-rlm-global-swarm-help, 2026-03-24)

**Escalation Criteria:**
- Boolean case mismatch (Instant Pricing + ABA) on production — escalate as active defect with org ID and reproduction steps
- Stale attribute values in CML — escalate with reference to known issue a02Ka00000lYbeAIAS
- CML hard limit blocking go-live — escalate for engineering review of limit increase

---

### Pattern 6: Qualification / Disqualification Rules Not Working

**Frequency:** Medium (10+ cases)
**Severity:** P9–P90
**TTR Impact:** 3–7 days

**Symptoms:**
- `Product Qualification Procedure not showing any products`
- `Qualification and Disqualification rules are not working`
- `Cannot read the array length because 'inputArray' is null`
- Qualification Procedure input/output parameters not visible in UI
- `Duplicate key Name (attempted merging values ColumnType.AutoNumber and ColumnType.Text)`
- Products not showing in Product Discovery

**Root Cause:**
- Context Definition or Decision Tables not Active — must be active for qualification rules to work
- `inputArray` is null: input variable not properly initialized before being passed to `EvaluateQualification` procedure
- **Duplicate key "Name" conflict:** Custom field with API name `Name__c` (label "Name") on Product Disqualification object conflicts with standard AutoNumber Name field — causes duplicate key error and parameters not rendering in UI
- Product Discovery Context Definition mismatch: context definition in Product Discovery Settings does not match the one used in the Pricing Procedure
- `You cannot update mapping in Product Discovery Context Definition` — occurs when trying to map a newly created attribute to a node in an active Product Discovery CD

**Resolution Steps:**
1. Verify Context Definition is **Active** — qualification rules will not run on inactive CDs
2. Verify all referenced Decision Tables are **Active**
3. **For `inputArray` is null:**
   - Ensure input variable is correctly defined and populated before passing to `EvaluateQualification`
   - Use debugging tools to trace execution and find where array becomes null
4. **For Duplicate key "Name" conflict:**
   - Rename the custom field API name from `Name__c` to something else (e.g., `NameUpdated__c`)
   - After renaming, the Qualification Procedure works as expected
   - Reference: this is reproducible by creating a custom field with label "Name" and API name `Name__c` on the Product Disqualification object
5. **For Product Discovery not showing products:**
   - Verify Product Discovery Context Definition in Revenue Settings matches the CD used in the Pricing Procedure
   - Refresh all Decision Tables used in the Product Discovery Pricing Procedure
   - If standard DTs missing: disable → enable Salesforce Pricing toggle to recreate them

6. **For "You cannot update mapping" on Product Discovery CD:**
   - Deactivate the Context Definition first, make the mapping change, then reactivate
   - If deactivation is blocked by references: follow Pattern 3 (Context Definition deactivation steps)
   - Reference: Case 470059824

**Verification:**
- Products appear when browsing catalog / running qualification procedure
- No errors in pricing procedure execution log

**Workarounds (from Slack):**
- Duplicate key Name conflict: rename custom field API name — verified fix in demo org (Harika Neela, #rlm-office-hours, 2026-01-14)
- Product Discovery missing tables: disable/enable Pricing Settings toggle recreates standard DTs (Pradeepa P, #support-rev-rlm-global-swarm-help, 2025-08-08)

**Escalation Criteria:**
- Context Definition and DTs are Active, input variables are correct, still getting `inputArray` null → escalate with execution trace

---

### Pattern 7: Expression Set / Pricing Procedure Runtime Errors

**Frequency:** Medium (15+ cases)
**Severity:** P0–P26
**TTR Impact:** 3–10 days

**Symptoms:**
- `Activate the Price Book Entries V2 lookup table used in the expression set and try again` (when creating from Expression Set Templates)
- `SF-Pricing-00004: We couldn't find the pricing procedure. Activate a pricing procedure for your org and set it as default, and try again.`
- `RulesEngineException: Ensure that this procedure has at least one active version`
- `Failure point: ExpressionSetCacheAdapter.loadVersionSnapshotObjectFromDb`
- `An error occurred while executing <ElementName> in <ProcedureName>. Error: Decision Table Query failed`
- `Cannot invoke "Object.toString()" because the return value of "TagData.getTagValue()" is null`
- `DUPLICATE_VALUE_FOUND error in Pricing Procedure` (even with no duplicate entries in DT)
- `GoCardless - Error On Pricing Procedure Changing CD`
- Pricing procedure works for System Admin but not for Partner/Sales user
- `Error adding Products from Customer Pricing Catalog due to exceeded RulesEngine calls`
- `Field PricebookEntry.ActivePriceAdjustmentQuantity is inaccessible in this context`
- Decision Matrix (CSV-based) used in Pricing Procedure fails intermittently
- **Expression Set stops working unexpectedly in sandbox** even though no changes were made — deactivate/reactivate resolves it (Dan Appel, #help-bre, 2023-06-01)
- **"The rule name X is invalid. Specify a valid name and try again."** when executing Expression Sets invoked via Lightning Flow or Apex in a test class
- **"Something went wrong. Try again..."** (FAULT_CLIENT: `SearchDecisionTable` / `ResourceApiException`) when saving a Pricing Procedure — caused by incremental DT refresh ClassCastException (Active bug W-22617103)

**Root Cause:**
- **"Activate the Price Book Entries V2 lookup table":** When creating a new Pricing Procedure from Expression Set Templates, this error appears if the standard Price Book Entries V2 Decision Table is not Active. Resolution: go to Setup → Decision Tables → activate PricebookEntriesV2 DT, then retry.
- **"No active version":** Expression Set / Pricing Procedure has no active version, or the active version is not applicable for the given context (date mismatch, incomplete context mapping)
- **NullPointerException (`TagData.getTagValue()` is null):** Input Rule Variable mapped to a formula field (e.g., `Product2.ProductCode`) is null because the context mapping resolves via relationship chain — must map directly in Context Definition
- **DUPLICATE_VALUE_FOUND:** `Enable Output Resolution` checkbox on List Element not functioning correctly — known issue
- **Decision Matrix (CSV-based) intermittent failure:** `AMLTierDiscountsV1 LookUpId is null` — Decision Matrices are not supported in Salesforce Pricing. Must migrate to Decision Table. GUS W-20474114 Closed - No Fix - Working as Documented.
- **`Field ... is inaccessible`:** Data Translation enabled in org causing field accessibility issues in DT metadata generation
- **Exceeded RulesEngine calls:** Large quote (200+ lines) hitting `MaxBulkLookupInputHbpoDt` limit (200 inputs per DT lookup request)
- **ES stops working unexpectedly in sandbox:** ES API stops responding even when no configuration changes were made. Deactivating and reactivating the Expression Set version restores it immediately. This can recur — root cause is platform-level session/cache state in sandbox environments.
- **"The rule name X is invalid" in Apex test:** When Expression Sets are invoked via Lightning Flow or OmniScript within an Apex test class, BRE cannot resolve the rule name due to test isolation. Known issue fixed in release 254 (strategic fix). **Pre-254 workaround:** Add `@isTest(SeeAllData=true)` annotation to the test class so BRE can query rule records. (Prashant Gupta, #help-bre, 2024-10-24)
- **"Something went wrong" when saving Pricing Procedure (W-22617103):** Active bug — occurs when a DT in the pricing procedure was incrementally refreshed. The `lastIncrementalSyncDate` field causes a `ClassCastException` in `DecisionTableQueryManager`. **Active bug W-22617103** filed 2026-05-22. Workaround: use full refresh instead of incremental refresh for affected DTs.

**Resolution Steps:**
1. **"No active version" / `ExpressionSetCacheAdapter` failure:**
   - Verify Pricing Procedure has at least one active version
   - Check that the active version's effective date range includes the current date
   - Verify context mapping is complete — all required input fields are mapped
   - If derived product: check if stricter context requirements are triggered

2. **NullPointerException `TagData.getTagValue()` is null:**
   - Check Input Rule Variable mapping in the List Container — if mapped via formula field referencing another object (e.g., `Product2.ProductCode`), this may return null
   - Workaround: put a constant value to confirm it's the variable mapping causing the issue
   - Fix: Map `Product Code` directly in the Context Definition to `Product2Id.ProductCode` under SalesTransactionItem

3. **DUPLICATE_VALUE_FOUND in Pricing Procedure:**
   - Try enabling the `Output Resolution` checkbox on the List Element
   - If still occurring: known issue — file GUS investigation with Enable Output Resolution workaround details

4. **Decision Matrix (CSV-based) intermittent failure:**
   - Confirm the customer is using a Decision Matrix (CalculationMatrix), not a Decision Table
   - Decision Matrices are NOT supported in Salesforce Pricing — this is documented behavior
   - Recommend migrating from Decision Matrix to Decision Table
   - GUS reference: W-20474114 (Closed - No Fix - Working as Documented)

5. **Field inaccessible / Data Translation:**
   - Temporarily disable `Enable Data Translation` flag under Setup → Company Information as a diagnostic step
   - If issue resolves, the flag is the cause — re-enable and work with engineering for a proper fix

6. **Exceeded RulesEngine calls (200+ quote lines):**
   - Check Decision Table type — must be `MediumVolume` with `UsageType = DefaultPricing`
   - For bulk operations > 200: investigate if `MaxBulkLookupInputHbpoDt` org preference can be increased via `Large Quotes and Orders` setting — owned by BRE Libra team (contact PM Sumaiya Begam Azam)

7. **Expression Set stops working unexpectedly in sandbox:**
   - Deactivate the Expression Set version → then reactivate it
   - This restores functionality immediately
   - If recurs: this is a known sandbox intermittency; log incident pattern for tracking

8. **"The rule name X is invalid" in Apex test class:**
   - Check if this is a test class invoking ES via Flow or OmniScript
   - **Release 254+:** Strategic fix deployed — SeeAllData annotation no longer needed
   - **Pre-254:** Add `@isTest(SeeAllData=true)` to the test class as a workaround
   - Reference: Prashant Gupta, #help-bre, 2024-10-24

9. **"Something went wrong" when saving Pricing Procedure (ClassCastException on incremental refresh):**
   - Check if any DT referenced in the procedure was incrementally refreshed recently
   - **Workaround:** Switch affected DT to full refresh instead of incremental refresh
   - **Bug reference:** W-22617103 (active, filed 2026-05-22) — `DecisionTableQueryManager.getDecisionTableBasicDetailsNonLocalized` `lastIncrementalSyncDate` ClassCastException

**Workarounds (from Slack):**
- Data Translation field inaccessibility: temporarily disabling "Enable Data Translation" in Company Information is a known workaround (Engineering Agent, #rlm-office-hours)
- Decision Matrix → Decision Table migration required for Salesforce Pricing: no workaround for the intermittent failures — migration is the only path (Dishant J, #support-rev-rlm-global-swarm-help, 2026-03-25)
- NullPointerException on TagData: putting a constant value as Input Rule Variable confirms variable mapping issue — fix by mapping directly in Context Definition (Vikram Makhija, #support-rev-rlm-global-swarm-help, 2026-04-27)
- Volume discount not applying on Reprice All: ensure Product Selling Model is populated on all PricebookEntry records for that product (Gulla Bharath Kumar, #support-rev-rlm-global-swarm-help, 2025-10-24)
- Expression Set stopping in sandbox: deactivate + reactivate the ES version — immediate fix (Dan Appel, #help-bre, 2023-06-01)
- "rule name X is invalid" in Apex test: use `@isTest(SeeAllData=true)` annotation pre-254; strategic fix in 254 (Prashant Gupta, #help-bre, 2024-10-24)

**Escalation Criteria:**
- DUPLICATE_VALUE_FOUND with Output Resolution enabled and no actual duplicates in DT → escalate as potential platform bug
- `MaxBulkLookupInputHbpoDt` cannot be increased for customer with large quote/order volume → escalate to BRE Libra PM

---

### Pattern 8: Decision Table Limits Exceeded

**Frequency:** Medium (15+ cases)
**Severity:** P0–P45
**TTR Impact:** 3–14 days

**Symptoms:**
- `Decision Table Query failed: The number of inputs in your lookup request exceeds the limit of 200 for your Salesforce instance.`
- `Decision Table Query failed: Price List decision table has limit exceeded`
- `Decision table limit exceeded. Guardrail: MaxBulkLookupInputHbpoDt. Configured Limit: 200.`
- `The number of inputs in your lookup request exceeds the limit of 200 for your Salesforce instance. Reduce the number of inputs and try again.` (on renewal > 200 assets)
- `PricebookEntry contains 231008 rows which exceeds the set Org Limit of 100000 rows per Decision Table`
- Incremental Refresh Error (100K row limit for incremental refresh — not extensible)
- `Expression Set object: Custom Field Limit Exceeded`
- Org limit for Decision Table refreshes per hour exceeded
- `We couldn't create the decision table because the number of consolidated columns that can be formed after joining columns with the Equals operator by using the AND operator exceeds the limit of 10 table rows.` (Standard DT)

**Root Cause:**
- Default BRE limit: 200 inputs per Decision Table bulk lookup request
  - **Advanced (Medium Volume) DT:** org value `MaxBulkLookupInputHbpoDt` — applies to DefaultPricing, DefaultRating, PricingDiscovery, RatingDiscovery, RevenueStandardTax usage types
  - **Standard (Low Volume) DT:** org value `MaxBulkLookupInputPerDT`
- Default record limit per DT: 100K rows (extensible to 2M by default, up to 40M in release 254+)
- Incremental refresh: hard limit of 100K rows — not extensible
- Expression Set custom field limit: max fields on Expression Set object exceeded
- **Standard DT column limit (`MaxColumnsWithHashingStdDT`):** Default is 10 consolidated columns — columns joined by AND with Required+Equals count as 1. Max can be increased to 15 via LAP. This limit is NOT documented in Help but is enforced by the platform.

**Resolution Steps:**

**For "consolidated columns exceeds limit of 10/15 table rows" (Standard DT):**
1. The column limit is `MaxColumnsWithHashingStdDT` — default 10, max 15 via LAP
2. **Optimize DT shape first:** Group all Required+Equals columns together in sequence — consecutive Required+Equals columns AND-joined count as 1 consolidated column instead of N separate columns
3. Example: if you have 14 columns all Required+Equals in sequence → they count as 1 consolidated column, not 14
4. If optimization not possible: raise a LAP request to increase `MaxColumnsWithHashingStdDT` to max of 15. Once approved, applies permanently to that org.
5. If 15 columns is still not enough: consider migrating to Advanced (Medium Volume) DT — no `MaxColumnsWithHashingStdDT` constraint

**For 200-input limit (bulk lookup):**
1. Verify Decision Table type is `MediumVolume` and `UsageType = DefaultPricing` (most critical for pricing DTs)
2. If already MediumVolume and still failing: identify which org value applies:
   - **Advanced DT:** raise LAP for `MaxBulkLookupInputHbpoDt` — owned by BRE Libra team
   - **Standard DT:** raise LAP for `MaxBulkLookupInputPerDT`
   - Include use case and target limit in LAP request
3. For amendment/renewal > 200 assets: batch the operation into groups of < 200 assets at a time as a workaround

**For record count limits:**
1. Add filters to the Decision Table to reduce source records (e.g., filter by IsActive = true)
2. For limit increase beyond default: submit LAP request
3. For PricebookEntry > 20M: requires engineering involvement

**For incremental refresh 100K limit:**
1. This is a hard limit — cannot be extended
2. Use full refresh instead of incremental refresh
3. Reduce the number of records in the source object with appropriate filters

**For Decision Table refresh rate limit:**
1. Reduce frequency of refreshes — batch refreshes into off-peak hours
2. Submit LAP request to increase refresh limit per hour (case 467700151 reference)

**For Expression Set custom field limit:**
1. Purge deleted custom fields: Setup → Object Manager → Expression Set → Fields → Purge Deleted Fields
2. Reference Salesforce Help: "Purge Deleted Custom Fields"

**Escalation Criteria:**
- `MaxBulkLookupInputHbpoDt` increase needed for production critical path → escalate to BRE Libra PM
- Record count > 20M with business need — LAP not working → escalate to engineering

---

### Pattern 9: Rule Library Issues

**Frequency:** Low-Medium (5+ cases)
**Severity:** P9–P45
**TTR Impact:** 3–7 days

**Symptoms:**
- `Kindly activate your latest rule library version to create rules`
- `RCA Rule Library Issues after Dev Refresh`
- Rule Library reconfiguration needed after refresh
- Unable to set up Configuration Rules with BRE option
- BRE not visible in org after CPQ license

**Root Cause:**
- Rule Library version not active — must activate latest version before creating rules
- After sandbox refresh: Rule Library references may become stale — requires reconfiguration
- BRE not enabled: needs `Set Up Configuration Rules with Business Rules Engine` toggle enabled in Revenue Settings
- CPQ orgs: BRE components may not be visible until RCA/RLM license is properly provisioned

**Resolution Steps:**
1. **"Kindly activate your latest rule library version":**
   - Go to Rule Library → find the latest version → activate it
   - Only one version can be active at a time

2. **BRE not visible / "Set Up Configuration Rules with BRE" not available:**
   - Verify RCA license is active on the org
   - Enable: Revenue Settings → `Set Up Configuration Rules with Business Rules Engine`
   - If option is greyed out: check with licensing team if RCA entitlement is provisioned

3. **After sandbox/dev refresh:**
   - Rule Library may need reconfiguration — verify Rule Library version is active
   - Context Definition references may need to be re-established
   - In one case, reconfiguring the Rule Library resolved all issues (Amit Sonawane, #rlm-office-hours)

4. **BRE not visible with CPQ license:**
   - BRE requires RCA/RLM license, not CPQ — verify the correct license is provisioned
   - Reference Case 46204563

**Escalation Criteria:**
- RCA license is provisioned, Revenue Settings option exists but toggling doesn't enable BRE → escalate with org ID

---

## C. Setup & Configuration Checklist

### New BRE / Decision Table Setup
- [ ] RCA/RLM license active
- [ ] Enable Revenue Settings → `Set Up Configuration Rules with Business Rules Engine`
- [ ] Assign `Business Rules Engine Designer` PSL to admins/configurators
- [ ] Assign `Business Rules Engine Runtime` PSL + `Run Decision Table` to all users who execute pricing/rules
- [ ] Assign `Tax Admin` or `Tax Configurator` PSL to users who need access to StandardTax DT
- [ ] Assign `Rule Engine Community Runtime` to Experience Cloud users
- [ ] Enable Salesforce Pricing Settings to auto-create standard Decision Tables
- [ ] Verify all standard DTs are in `Active` status after setup
- [ ] Use `Extend` (not `Clone`) for Context Definitions
- [ ] Set Decision Table type to `MediumVolume` + `UsageType = DefaultPricing` for pricing DTs
- [ ] For DTs with large datasets: add filters to keep record counts within limits

### Common Misconfigurations

| Misconfiguration | Symptom | Fix |
|---|---|---|
| Context Definition or DT not Active | Rules don't execute, no products shown | Activate both CD and DT |
| Cloned CD instead of Extended | CD doesn't auto-upgrade, stale config | Recreate using Extend |
| Decision Matrix used in Pricing Procedure | Intermittent LookUpId null errors | Migrate to Decision Table |
| Hash Key Group > 50/200 rows (Advanced DT) | DT activation fails | Use Low Volume DT or restructure Required+Equals columns |
| Tax Admin/Configurator PSL missing | "No Decision Table exists: StandardTax" | Assign Tax Admin or Tax Configurator PSL |
| BRE Runtime PSL missing on Automated Process | Insufficient Privileges on DT execute | Assign BRE Runtime + Run Decision Table perms |
| Multi-OR chained conditions in CML | Latency/timeout in configurator | Refactor to boolean formula field + single check |
| Direct DT insert in test class | DML insert not allowed | Use ConnectApi.DecisionTable.Execute instead |
| Expression Set Version deployed directly | Random ID errors in target org | Deploy ES Definition only, recreate version in target |
| Special characters in product name used as DT key | Cannot create pricing record / lookup fails | Create sanitized lookup code field; use that as DT matching key |
| BRE Runtime PSL not in customer SKU | "BRE Runtime PSL not available" | Clone BREDesigner PS as workaround; or buy BRE add-on |
| Standard DT has > 10 non-optimized columns | "consolidated columns exceeds limit of 10" | Group Required+Equals columns in sequence; or raise LAP for max 15 |
| Incremental DT refresh in use (bug W-22617103) | "Something went wrong" when saving Pricing Procedure | Switch to full refresh; bug fixed in 264+ |
| BRE used on Sales Cloud (no industry cloud) | BRE not visible / objects not available | BRE requires industry cloud license — not available on Sales Cloud alone |

---

## D. Licensing & Entitlements

### BRE Component Requirements

| Component | License Required |
|---|---|
| Decision Tables (Lookup Tables) | RCA or BRE add-on |
| Expression Sets / Pricing Procedures | RCA |
| Context Definitions | RCA |
| Configuration Rules (BRE) | RCA |
| CML / Advanced Configurator | RCA + Advanced Configurator Pilot permission |
| Qualification / Disqualification Rules | RCA |
| Context Service Runtime | Included with RCA; check license count in Setup |

### Context Service Runtime Licenses
- Runtime licenses are consumed when Expression Sets / Pricing Procedures execute
- Check available count: Setup → Company Information → Context Service Runtime
- If production shows fewer licenses than expected: file GUS investigation or LAP

---

## E. Documentation Quick-Reference

| Topic | Link |
|---|---|
| BRE Overview | https://help.salesforce.com/s/articleView?id=ind.business_rules_engine.htm&type=5 |
| BRE Default Limits | https://help.salesforce.com/s/articleView?id=ind.business_rules_engine_default_limits.htm&type=5 |
| Set Up Configuration Rules | https://help.salesforce.com/s/articleView?id=ind.set_up_configuration_rules.htm&type=5 |
| Revenue Cloud Permission Sets | https://help.salesforce.com/s/articleView?id=ind.revenue_cloud_permission_sets_table.htm&type=5 |
| Product Catalog Dynamic Attributes | https://help.salesforce.com/s/articleView?id=ind.product_catalog_dynamic_attributes.htm&type=5 |
| Promotions in Revenue Cloud | https://help.salesforce.com/s/articleView?id=ind.comms_set_up_promotions_in_revenue_cloud_for_communications.htm&type=5 |

### Key Behavioral Rules / Design Decisions
- Context Definitions and Decision Tables **must both be Active** for rules/pricing to execute
- Cloned Context Definitions do **not** auto-upgrade — always use `Extend`
- Decision Matrices (CSV-based) are **not supported** in Salesforce Pricing — use Decision Tables only
- `Billing Admin` direct assignment requirement (not via PSG) applies only to Billing — BRE PSLs work via PSG
- Advanced (Medium Volume) DT: max **50** rows per hash key group in pre-260 releases, **200** rows in release 260+
- Standard DT: max **10** consolidated columns by default (max 15 via LAP) — `MaxColumnsWithHashingStdDT`
- Default bulk lookup limit: 200 inputs per DT request (`MaxBulkLookupInputHbpoDt` for Advanced, `MaxBulkLookupInputPerDT` for Standard)
- Expression Set Definition versions contain environment-specific snapshot IDs — cannot be directly deployed between orgs
- `ContextRulePilot` and `DebugContextRulePilot` permissions require LAP requests
- BRE context rules (Rulesets, RuleValues) **do not support packaging or org migration** — must be manually recreated
- BRE is **stateless** — no state maintained between calls; pass all required inputs each time
- BRE is **not available on Sales Cloud** without an industry cloud or BRE add-on license
- `BRERuntime` permission set assignment is sufficient for all BRE execution (ES, DM, DT)

---

## F. Escalation Paths

### When to Escalate
- DT stuck in Activation In Progress with no Splunk error and no UI option to reset
- Expression Set deployment fails with new random IDs on every attempt
- CML boolean case mismatch (Instant Pricing + ABA) on production
- `MaxBulkLookupInputHbpoDt` needs increase for critical path with > 200 lines
- Context Definition deactivation blocked and LAP for `ContextRulePilot` is pending
- Record count > 20M with business need that cannot be filtered

### Information to Gather Before Escalating
1. Org ID + sandbox/production
2. Decision Table ID (from `SELECT Id, DeveloperName FROM DecisionTable`)
3. Exact error message + Error ID
4. Splunk trace log output (if accessible)
5. Steps to reproduce in a clean org
6. Permission sets assigned to running user
7. Whether issue is new or regression (what changed — refresh, deployment, release)

### Key Escalation Contacts (from Slack)
- **#rlm-office-hours** — RLM BRE/Pricing questions (Sakshi Sharma, Frank Shapiro, Vrushabh Mahendrakar, Shruti Biswal)
- **#help-bre** — BRE engineering support channel (BRE experts: Akhand Pratap Singh, Satyam Shekhar, Tulsi Pargain, Amit Kumar, Rajesh Borusu, Prashant Gupta, Dakshayini Bangera, Syed Azher)
- **#industries-bre-officehours** — BRE office hours / new feature integration (Rajmani Kumar, Prashant Gupta, Rohit Mittal)
- **#support-rev-rlm-global-swarm-help** — Active customer swarm requests
- **BRE Libra PM:** Sumaiya Begam Azam — for `MaxBulkLookupInputHbpoDt` limit increases and bulk lookup issues
- **For DT limits LAP requests:** file via GUS/LAP system; use correct org value name (`MaxBulkLookupInputHbpoDt` for Advanced DT, `MaxBulkLookupInputPerDT` for Standard DT, `MaxColumnsWithHashingStdDT` for column limit)
- GUS investigations: file via #support-rev-rlm-global-swarm-help with org ID and error details

---

## G. Tribal Knowledge & Pro Tips

1. **Always check if CD and DT are Active first:** The majority of "No Decision Table exists", qualification rules not working, and pricing procedure failures are caused by an inactive Context Definition or Decision Table. Check status before anything else. (Engineering Agent, #rlm-office-hours)

2. **Extend, never Clone, Context Definitions:** Cloned CDs require manual syncing and don't auto-upgrade with releases. Using Extend ensures the CD inherits platform updates automatically. (Engineering Agent, #rlm-office-hours)

3. **Decision Matrix = Unsupported in Salesforce Pricing:** If a customer has intermittent pricing failures with `LookUpId is null`, check if they're using a Decision Matrix (CalculationMatrix) — not a Decision Table. Migrate is the only fix. GUS W-20474114 is Closed - No Fix. (Dishant J, #support-rev-rlm-global-swarm-help, 2026-03-25)

4. **StandardTax DT access requires Tax Admin or Tax Configurator:** There is currently no minimal permission that grants access to the StandardTax Decision Table without View Setup and Configuration. This is a known gap — log VOC for customers who cannot grant this to sales users. (Gulla Bharath Kumar, #rlm-office-hours, 2026-05-04)

5. **Source Object Modified — toggle fix:** If all standard DTs show "source object permissions or fields modified", disable then re-enable the Salesforce Pricing toggle. It recreates all standard DTs and pricing recipe mappings cleanly. (Divyanshu Tiwari, #support-rev-rlm-global-swarm-help, 2025-09-01)

6. **CML multi-OR performance optimization:** Replace chained `||` conditions (e.g., `count(X == "A") > 0 || count(X == "B") > 0`) with a boolean formula field mapped to Context Service. Single variable check dramatically reduces CML size and runtime latency — brought one customer's configurator response from timeout to < 3 seconds. (Kaushik Jagatap, #support-rev-rlm-global-swarm-help, 2026-04-08)

7. **Deploy Expression Set Definition, not Version:** When deploying Expression Sets between orgs, deploy only the Definition shell. Versions contain internal snapshot IDs that are environment-specific and will fail with random "record not found" errors if deployed directly. Recreate the version in the target org after deploying the definition. (Avinash Saklani, multiple cases)

8. **Volume Discount not applying on Reprice All:** If volume discount works on first pricing but not on Reprice All, check that the Product Selling Model (PSM) field is populated on all PricebookEntry records for that product. A missing PSM causes the volume discount decision table lookup to return null, skipping the discount. (Gulla Bharath Kumar, #support-rev-rlm-global-swarm-help, 2025-10-24)

9. **Community User Pricing (30-second delay):** After assigning `Rule Engine Community Runtime` PSL to an Experience Cloud user, there is a ~30 second processing delay before pricing begins working. Don't assume it's still failing — wait and retest. (Vani Madhavi Latha, #support-rev-rlm-global-swarm-help, 2026-03-11)

10. **"Missing association" CML error — create new CML:** If a CML shows "missing association" errors on activation for associations that demonstrably exist, creating a new CML with the same configuration typically works correctly. The existing CML may have a cached/state issue. (Claudio Crocilla, #support-rev-rlm-global-swarm-help, 2026-03-24)

11. **Deactivating Context Definitions via API:** When UI deactivation is blocked by ruleset references, use Connect API with `ContextRulePilot` permission to update ruleset versions to Inactive, then delete the ruleset. DRO Handbook has the Connect API details. Always coordinate with DRO team first. (Shruti Biswal, #rlm-office-hours, 2026-03-06)

12. **Duplicate key "Name" conflict on Product Disqualification:** If Qualification Procedure shows no input/output parameters, check for a custom field with label "Name" and API name `Name__c` on the Product Disqualification object. This conflicts with the standard AutoNumber Name field. Rename the custom field API name to resolve. (Harika Neela, #rlm-office-hours, 2026-01-14)

13. **BRE not visible in CPQ org:** BRE requires RCA/RLM license, not CPQ. If customer has CPQ but no RCA and BRE is not visible — expected behavior. (Victor Shapiro, Case 46204563)

14. **BRE is not available on Sales Cloud alone:** BRE requires an industry cloud or Service Cloud base license. If a customer with only Sales Cloud or DataBricks/PowerBI integration asks about BRE, they need to purchase an industry cloud or BRE add-on. BRE PSLs are not sold as standalone — they come with base cloud SKUs or via `Business Rules Engine - calls 100k` add-on. (Sumaiya Begam Azam, #help-bre, 2026-03-12)

15. **Standard DT column limit is undocumented:** `MaxColumnsWithHashingStdDT` defaults to 10 and can be raised to max 15 via LAP — but this is NOT documented in the official BRE limits Help article. When customers hit the "consolidated columns exceeds limit" error, they often can't find the limit in docs. Point them to this: group all Required+Equals columns together in sequence to reduce consolidated column count before requesting a LAP increase. (Satyam Shekhar, #help-bre, 2025-08-06)

16. **Two different org values for bulk lookup limit:** `MaxBulkLookupInputHbpoDt` is for Advanced (Medium Volume) DTs used in DefaultPricing/Rating — `MaxBulkLookupInputPerDT` is for Standard (Low Volume) DTs. Filing the wrong one in a LAP request wastes time — always confirm DT type before specifying which org value to raise. (Satyam Shekhar, #help-bre, 2025-12-05)

17. **MaxRowsPerHashKey increased from 50 to 200 in release 260:** If a customer on pre-260 is hitting the 50-row hash group limit, upgrading to 260+ resolves it. However, 200 is the new hard limit — still cannot be increased beyond that. Check the customer's org release version before advising on workarounds. (Satyam Shekhar, #help-bre, 2026-01-25)

18. **Expression Set Builder "ruleId=undefined" after Connect API creation:** When an ES version is created via Connect API, the ruleBuilder URL may populate with `ruleId=undefined`, causing "The Builder ID couldn't be determined" error. Fix: manually copy the ExpressionSetVersion ID and append it to the URL path (`ruleBuilder.app?ruleId=<ID>`), then activate/reactivate. (AKASH KUMAR, #industries-bre-officehours, 2026-04-06)

19. **BRERuntime PSL assignment is sufficient for all BRE execution:** To invoke any BRE call (Expression Set, Decision Matrix, or Decision Table), assigning the `BRERuntime` permission set to the user is sufficient. No other custom permission sets or special configuration needed for execution. (Rajesh Borusu, #help-bre, 2026-05-07)

20. **BRE is stateless:** BRE does not maintain state between calls. If customer needs to apply rules to external data (e.g., DataBricks, Power BI), they need to fetch data externally and pass it into BRE as inputs. Data Cloud data stream → DMO → Flow → BRE is a known integration pattern for external data. (Sumaiya Begam Azam + Rajmani Kumar, #help-bre, 2026-03-06)

---

## H. Active Known Issues

*Connect to GUS for real-time P0/P1/P2 bugs. Query:*
```sql
SELECT Id, Name, Subject__c, Status__c, Priority__c 
FROM ADM_Work__c 
WHERE Product_Tag__r.Name LIKE '%Revenue Cloud (Core)-Business Rules Engine%' 
AND Type__c = 'Bug' 
AND Status__c IN ('New', 'Triaged', 'In Progress') 
AND Priority__c IN ('P0', 'P1', 'P2') 
ORDER BY Priority__c, CreatedDate DESC LIMIT 20
```
*(Run via GUS CLI: `sf data query --query "..." --target-org gus`)*

**Known active as of 2026-05-27:**
- **Boolean case mismatch (Instant Pricing + ABA):** Instant Pricing passes boolean attribute values as uppercase `"FALSE"` — ABA condition expects lowercase `"false"`. Works after Save & Exit. Engineering investigation in progress. (Case 473510739, prior case 473428919)
- **Stale attribute values after back navigation in CML:** CML constraint engine does not re-evaluate dependent attributes after upstream attribute is changed more than once. Known Issue: a02Ka00000lYbeAIAS.
- **"Configuration rule refers to different type of sales transaction item":** Occurs after enabling then disabling Advanced Configurator — residual configuration causes rule type mismatch. Requires Rule Library reconfiguration.
- **W-22617103 — "Something went wrong" when saving Pricing Procedure after incremental DT refresh (Active, 2026-05-22):** `DecisionTableQueryManager.getDecisionTableBasicDetailsNonLocalized` throws ClassCastException on `lastIncrementalSyncDate`. Workaround: use full refresh for affected DT. Fix in progress.
- **Procedure save with DT incremental refresh — "Procedure save is failing with generic error" (FAULT_CLIENT SearchDecisionTable):** Same root cause as W-22617103 above — occurs when a Pricing Procedure step references a DT that was incrementally refreshed. Also related to `"You don't have the permission to access the expression set version"` error in same context. (Yash Garg / Prashant Gupta, #industries-bre-officehours, 2026-05-22)

---

## I. Code Reference Map

*CodeSearch is not currently connected. Connect CodeSearch in AI Suite settings to get code-level root cause traces for errors like NullPointerException in pricing execution, DT lookup failures, and ruleset engine errors.*

**Key classes/components to search for when CodeSearch is available:**
- `PlaceSalesTransactionPricingStep` — pricing execution pipeline (appears in Splunk logs)
- `ExpressionSetCacheAdapter` — expression set version snapshot loading
- `DecisionTableRefreshExecution` — DT refresh/activation execution
- `BillingDecisionTableService` — billing-specific DT service
- `ConnectApi.DecisionTable` — Apex DT execution API

## Completeness Report
