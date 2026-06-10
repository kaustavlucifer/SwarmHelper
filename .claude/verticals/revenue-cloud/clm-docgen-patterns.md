## A. Triage & Classification

When a support engineer brings a Revenue Cloud CLM/DocGen issue, first classify it:

```
Is the issue about...

├── 1. DOCGEN PERMISSIONS & ACCESS
│   ├── "Insufficient privileges" / "You don't have access"
│   ├── DocGen Designer/User permission sets not working
│   ├── CLM Runtime User permission set issues
│   ├── Cannot access Document Builder
│   ├── OmniStudio Document Generation feature missing
│   └── Document Template Designer access denied
│   → Go to: Pattern 1 (DocGen Permission/Access)
│
├── 2. CONTRACT STATUS & ACTIVATION
│   ├── Cannot change contract status (Active → Expired)
│   ├── Contract stuck in Draft/Negotiating
│   ├── Contract locked/unlocked issues after Send for Signature
│   ├── Assetization failure during contract activation
│   ├── "Context Use Case Mapping" error
│   └── Contract activation/order activation errors
│   → Go to: Pattern 2 (Contract Status/Activation)
│
├── 3. EXTERNAL STORAGE & INTEGRATIONS
│   ├── Microsoft 365 / SharePoint integration issues
│   ├── External Document Storage Configuration errors
│   ├── Word Connector check-in/check-out problems
│   ├── Clause library not accessible in Word Add-in
│   ├── Named Credentials configuration issues
│   └── Document version sync issues with M365
│   → Go to: Pattern 3 (External Storage)
│
├── 4. MERGE FIELDS & DATA POPULATION
│   ├── Merge fields not populating in generated document
│   ├── Data mapping errors during document generation
│   ├── NULL pointer exceptions in template rendering
│   ├── Quote/Contract field values not pulling through
│   ├── Custom field tokens not resolving
│   └── Context path resolution failures (leanerQueryTags)
│   → Go to: Pattern 4 (Merge Field/Data Issues)
│
├── 5. FONTS & FORMATTING
│   ├── Font not rendering correctly in PDF
│   ├── Rich text formatting lost
│   ├── Custom fonts not appearing
│   ├── Numbered list counter not restarting
│   ├── Image/text overlapping in document
│   └── Locale-specific formatting issues (currency, date, etc.)
│   → Go to: Pattern 5 (Font/Formatting)
│
├── 6. CONTEXT SERVICE & INTEGRATION
│   ├── "Couldn't fetch context definition name"
│   ├── Context Use Case Mapping configuration errors
│   ├── Integration Procedure attachment issues
│   ├── OmniScript DocGen function errors
│   ├── "Exception occurred" in document generation
│   └── Context service timeout/unavailable
│   → Go to: Pattern 6 (Context Service)
│
├── 7. PERFORMANCE & TIMEOUTS
│   ├── Document generation takes too long
│   ├── Timeout errors during PDF generation
│   ├── Performance degradation after release (e.g., Spring '26)
│   ├── Image download delays from Content Files
│   ├── Slow Content Document Download
│   └── Batch job failures due to timeouts
│   → Go to: Pattern 7 (Performance/Timeout)
│
├── 8. E-SIGNATURE INTEGRATION
│   ├── DocuSign authentication errors
│   ├── Named Credentials configuration issues
│   ├── Envelope creation failures
│   ├── Envelope status not updating
│   ├── Anchor tag/signature field configuration
│   └── "Bad Request" errors during envelope creation
│   → Go to: Pattern 9 (E-signature)
│
└── 9. SESSION ID / OUTBOUND MESSAGE DEPRECATION
    ├── Session ID removal enforcement
    ├── Outbound Message without Session ID
    ├── Nintex DocGen Session ID issues
    ├── Integration fails after Session ID deprecation
    └── Extension requests for Session ID support
    → Go to: Pattern 10 (Session ID Deprecation)
```

### Quick Diagnostic Questions

Ask these to narrow down quickly:
1. **What is the exact error message?** Full text, including any error codes
2. **When did this start?** After a release/deployment/configuration change?
3. **Which user/profile?** System Admin or specific user with permission sets?
4. **Production or Sandbox?** If sandbox, when was last refresh?
5. **Which feature?** Contract creation, document generation, e-signature, external storage?
6. **Which integration?** DocuSign, Microsoft 365, Nintex, custom integration?
7. **Can System Admin reproduce?** Helps isolate permission vs. configuration issues

---

## B. Known Issue Patterns

### Pattern 1: DocGen Permission/Access Denied

**Frequency:** 41 cases (13.7% of total CLM/DocGen issues)  
**Severity:** Priority 45: 20 cases | Priority 0: 5 cases  
**Average Resolution Rate:** 68% closed (28/41 cases)  
**Data Source:** OrgCS case analysis (300 cases, last 6 months)

**Symptoms:**
- "Insufficient privileges" when accessing Document Builder
- CLM Runtime User permission set assigned but not working
- OmniStudio Document Generation feature missing from setup
- Cannot access Document Template Designer
- "You don't have access to the place sales transaction service"
- DocGen permission sets not visible or assignable

**Common Error Messages:**
```
"We can't log you in because of an authentication error"
"You don't have access to the place sales transaction service"
"Insufficient privileges" (when accessing DocGen features)
"The OmniStudio Document Generation feature is missing"
"Cannot access Document Builder in Revenue Cloud"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Permission sets not assigned (DocGen Designer, DocGen User, CLM Runtime User) | Check assigned permission sets on user record |
| 2 | OmniStudio DocGen not enabled in org | Setup → Document Generation → General Settings |
| 3 | User license doesn't include DocGen access | Check user license type (needs Platform or appropriate license) |
| 4 | Named Credentials misconfiguration (for external integrations) | Setup → Named Credentials → verify authentication |
| 5 | Feature provisioning issue (missing DocGen in org) | Check if DocGen appears in Setup menu |

**Resolution Steps:**

1. **Verify Permission Sets:**
   ```sql
   SELECT PermissionSet.Name, AssigneeId, Assignee.Name 
   FROM PermissionSetAssignment 
   WHERE AssigneeId = '<user_id>' 
   AND (PermissionSet.Name LIKE '%DocGen%' OR PermissionSet.Name LIKE '%CLM%')
   ```
   Required permission sets:
   - `OmniStudio DocGen Designer` (for template creation)
   - `OmniStudio DocGen User` (for document generation)
   - `CLM Runtime User` (for contract operations)

2. **Check DocGen Enablement:**
   - Navigate to: Setup → Document Generation → General Settings
   - Verify "Enable OmniStudio Document Generation" is checked
   - If missing → Contact Salesforce to provision DocGen

3. **Verify User License:**
   - Check user license type supports DocGen (Platform, Unlimited, Enterprise with OmniStudio)
   - Guest/Community licenses do NOT support DocGen

4. **Check Named Credentials (if using external storage):**
   - Setup → Named Credentials → verify authentication status
   - Test connection to external service (DocuSign, M365, etc.)
   - Re-authenticate if expired

5. **Verify Object-Level Permissions:**
   ```
   Required objects: Contract, ContractDocumentVersion, DocumentTemplate, 
   ContentVersion, ContentDocument
   ```

**Verification:** User can access Setup → Document Generation → Create/Edit template → Generate document

**Known Limitations:**
- Guest/Community users cannot use DocGen (requires internal license)
- Some DocGen features require specific license tiers (check with AE/Sales)

**Escalation Criteria:**
- DocGen not provisioned in org → Escalate to Provisioning team
- Permission sets assigned but still no access → Escalate to Platform Engineering with debug logs
- Named Credentials issue persists → Escalate with authentication logs

**Sample Case Numbers:** 473359860, 473525828, 472974889, 473370982, 473301551

---

### Pattern 2: Contract Status/Activation Issues

**Frequency:** 31 cases (10.3% of total)  
**Severity:** Priority 45: 12 cases | Priority 0: 8 cases  
**Average Resolution Rate:** 55% closed (17/31 cases)

**Symptoms:**
- Cannot change contract status from Active → Expired
- Contract activation fails during order assetization
- "Context Use Case Mapping" error when updating contract
- Contract stuck in Draft/Negotiating status
- Contract locked after "Send for Signature" cannot be unlocked
- Assetization log shows errors but contract appears active

**Common Error Messages:**
```
"We couldn't fetch the context definition name and its associated mapping name 
from the context use case mapping object"
"Cannot move status from Active to Expired for contract with record type Pricing Contract"
"Unlock is allowed only in Draft or Negotiating status"
"Assetization process fails with log errors"
"Status progression blocked - document is locked"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Context Use Case Mapping misconfigured | Check Setup → Context Use Case Mapping for target object |
| 2 | Validation rules blocking status transition | Check Validation Rules on Contract object |
| 3 | Document locked during signature process | Check ContractDocumentVersion.IsLocked field |
| 4 | Assetization flow failure (assets not created) | Check Contract → Assets → verify creation |
| 5 | Record Type mismatch in configuration | Verify Contract Record Type matches Context Use Case Mapping |

**Resolution Steps:**

1. **Check Context Use Case Mapping:**
   - Navigate to: Setup → Context Use Case Mapping
   - Verify Target Object = Contract (or specific Record Type)
   - Verify Context Definition Name is populated
   - If missing: Create new Context Use Case Mapping record
     - Target Object: Contract
     - Target Record Type: (specific record type if applicable)
     - Context Definition Name: (from Context Definition object)
     - Context Mapping Name: (from Context Mapping object)

2. **Verify Contract Status Picklist Values:**
   ```sql
   SELECT Id, Status, RecordType.Name, IsLocked__c 
   FROM Contract 
   WHERE Id = '<contract_id>'
   ```
   - Valid status transitions (typically): Draft → Negotiating → Activated → Expired/Terminated
   - Check if custom status values exist

3. **Check for Locking Issues:**
   ```sql
   SELECT Id, Contract__c, IsLocked, Status__c, DocumentName 
   FROM ContractDocumentVersion 
   WHERE Contract__c = '<contract_id>' 
   ORDER BY CreatedDate DESC LIMIT 1
   ```
   - If IsLocked = true and status change blocked → Document must be unlocked first
   - **Workaround:** If stuck, update IsLocked = false via API/Workbench (admin only)

4. **Investigate Assetization Failure:**
   - Check: Contract → Related → Assets (should show created assets)
   - If missing: Check Assetization Log (custom object or debug logs)
   - Common causes: Order not activated, Product missing asset configuration, validation rule failure

5. **Review Validation Rules:**
   - Setup → Object Manager → Contract → Validation Rules
   - Identify rules that fire on status change
   - Temporarily deactivate to isolate if blocking (admin only)

**Verification:** 
- Contract status changes successfully
- Assets created during activation
- Document can be locked/unlocked as expected

**Pro Tips:**
- Context Use Case Mapping is REQUIRED for Revenue Cloud contract operations
- Always check Contract Record Type matches Context Use Case Mapping configuration
- Document lock state is separate from contract status — unlock document first if needed

**Escalation Criteria:**
- Context Use Case Mapping error persists after configuration → Escalate with screenshots of setup
- Assetization fails with no logs → Escalate with order/contract details
- Platform bug suspected → Escalate with before/after behavior (e.g., worked pre-release)

**Sample Case Numbers:** 473490996, 473388354, 473427259, 473523664, 472929093

---

### Pattern 3: DocGen External Storage Issues

**Frequency:** 25 cases (8.3% of total)  
**Severity:** Priority 45: 12 cases | Priority 0: 3 cases  
**Average Resolution Rate:** 72% closed (18/25 cases)

**Symptoms:**
- External Document Storage Configuration error for Microsoft 365
- Cannot check-in/check-out documents in Word Connector
- Clause library not accessible in Word Add-in
- Named Credentials authentication failure
- Document version sync issues between Salesforce and M365
- "Fetch contract document version failed" error

**Common Error Messages:**
```
"We couldn't fetch the contract document version because of a configuration error or API failure"
"Cannot check-in the word document in the Microsoft 365 word connector"
"External Document Storage Configuration issue"
"Cannot get clause library and clause insertion working"
"Configuration error when selecting Document Storage Type = Microsoft 365"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Named Credentials not configured or expired | Setup → Named Credentials → check authentication status |
| 2 | External Storage Configuration not set on Contract Type | Check Contract Type Config → Document Storage Type |
| 3 | Microsoft 365 permissions insufficient | Test M365 API access with admin account |
| 4 | Callback URL mismatch in Connected App | Verify callback URL matches authentication provider |
| 5 | SharePoint site not accessible from Salesforce | Test direct SharePoint API call |

**Resolution Steps:**

1. **Configure Named Credentials (Microsoft 365):**
   - Setup → Named Credentials → Create New
   - Name: `Microsoft365_DocStorage`
   - URL: `https://graph.microsoft.com`
   - Identity Type: Named Principal
   - Authentication Protocol: OAuth 2.0
   - Scope: `Files.ReadWrite.All Sites.ReadWrite.All`
   - Auth Provider: (link to M365 Auth Provider)
   - Test authentication → should show "Authenticated"

2. **Configure External Storage on Contract Type:**
   - Setup → Contract Type Config → Select contract type
   - Document Storage Type: Microsoft 365
   - External Storage Configuration: (select Named Credential)
   - SharePoint Site URL: (org's SharePoint site)
   - Document Library Name: (target library, e.g., "Contracts")
   - Save and verify

3. **Set Up Authentication Provider (Microsoft 365):**
   - Setup → Auth. Providers → New → Microsoft
   - Consumer Key: (from Azure App Registration - Application ID)
   - Consumer Secret: (from Azure App Registration - Client Secret)
   - Authorize Endpoint URL: `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/authorize`
   - Token Endpoint URL: `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token`
   - Default Scopes: `https://graph.microsoft.com/.default`
   - Save

4. **Configure Connected App (for callback):**
   - Setup → App Manager → New Connected App
   - Callback URL: `https://<your-instance>.salesforce.com/services/authcallback/<auth-provider-name>`
   - OAuth Scopes: Full access (API)
   - Save and note Consumer Key/Secret

5. **Test External Storage:**
   - Create a test contract
   - Generate a document → Save to External Storage
   - Verify document appears in SharePoint/M365
   - Open in Word Connector → Edit → Check-in
   - Verify version sync back to Salesforce

**Verification:**
- Document saves to external storage (M365/SharePoint)
- Word Connector can check-out/check-in documents
- Clause library accessible in Word Add-in
- Document versions sync between Salesforce and M365

**Known Limitations:**
- External storage requires specific Salesforce licenses (check with AE)
- M365 integration requires Azure app registration (admin access needed)
- SharePoint site must be accessible via Microsoft Graph API

**Escalation Criteria:**
- Named Credentials authentication fails after correct setup → Escalate with auth provider config
- SharePoint API access denied → Escalate to customer's M365 admin
- Configuration correct but still fails → Escalate with API logs and error details

**Sample Case Numbers:** 473176396, 473400863, 473396901, 473462125, 473508606

---

### Pattern 4: DocGen Merge Field/Data Population Issues

**Frequency:** 23 cases (7.7% of total)  
**Severity:** Priority 0: 7 cases | Priority 45: 7 cases  
**Average Resolution Rate:** 61% closed (14/23 cases)

**Symptoms:**
- Merge fields showing as blank in generated document
- NULL pointer exception during document generation
- Custom field tokens not resolving
- Quote/Contract field values not pulling through
- E-signature configuration prepopulates only some fields (name, email but not others)
- Context path resolution failures (leanerQueryTags response shape change)

**Common Error Messages:**
```
"The Quote node path resolved to null"
"Null pointer exception in template rendering"
"Electronic signature configuration works only 20% - prepopulates recipient name 
and email but nothing after that"
"leanerQueryTags() Header Query Response Shape Change Breaks Custom Pricing Post Hook"
"Cannot create a contract due to mapping issue"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Incorrect merge field syntax in template | Review template tokens: {{ObjectName.FieldName}} |
| 2 | Field API name mismatch (custom fields) | Verify field exists: Setup → Object Manager → Fields |
| 3 | Context path resolution changed after release | Check if issue started after Salesforce release |
| 4 | Data not populated on source record | Query the record to verify field has value |
| 5 | Permission issue (field-level security) | Check user's profile/permission set has field access |

**Resolution Steps:**

1. **Verify Merge Field Syntax:**
   - Standard fields: `{{Contract.ContractNumber}}`
   - Custom fields: `{{Contract.Custom_Field__c}}`
   - Related fields: `{{Contract.Account.Name}}`
   - Quote fields: `{{Quote.TotalAmount}}` or `{{SalesTransaction.TotalAmount}}`
   - **NEW (post-release):** Use `SalesTransactionSource` path for quotes: `{{SalesTransactionSource.TotalAmount}}`

2. **Check Field API Name:**
   ```sql
   SELECT Id, ContractNumber, Custom_Field__c, Account.Name 
   FROM Contract 
   WHERE Id = '<contract_id>'
   ```
   - Verify field exists and has value
   - Check spelling/case matches exactly (API names are case-sensitive)

3. **Review Context Path Resolution (Spring '26+ issue):**
   - **OLD behavior:** `Context.IndustriesContext.leanerQueryTags()` returned shared `recordIds` list
   - **NEW behavior:** Context path uses `SalesTransactionSource` instead of direct Quote path
   - **Fix:** Update custom Apex/template to use `SalesTransactionSource` path
   - Example: Replace `{{Quote.TotalAmount}}` with `{{SalesTransactionSource.TotalAmount}}`

4. **Check Field-Level Security:**
   ```sql
   SELECT ParentId, Parent.Profile.Name, Field, PermissionsRead 
   FROM FieldPermissions 
   WHERE Field = 'Contract.Custom_Field__c' 
   AND PermissionsRead = true
   ```
   - Ensure running user has read access to all fields used in template

5. **Test with Simple Template:**
   - Create minimal template with only 1-2 merge fields
   - Generate document → verify fields populate
   - If works: issue is with specific field in original template
   - If fails: broader configuration/permission issue

**Verification:**
- All merge fields populate correctly in generated document
- No NULL values or blank spaces where data should appear
- E-signature configuration prepopulates all fields

**Pro Tips:**
- Always use Field API names in merge tokens (not Field Labels)
- Test templates with records that have all fields populated
- After Salesforce releases, check release notes for context service changes
- For quote fields, try both `Quote.*` and `SalesTransactionSource.*` paths

**Escalation Criteria:**
- Merge fields correct but still not populating → Escalate with template file and sample record
- Context path resolution issue post-release → Escalate to Product Team with before/after behavior
- NULL pointer exception persists → Escalate with debug logs and stack trace

**Sample Case Numbers:** 473442063, 473388354, 473503460, 473348039, 473396901

---

### Pattern 5: DocGen Font/Formatting Issues

**Frequency:** 21 cases (7.0% of total)  
**Severity:** Priority 45: 14 cases | Priority 27: 3 cases  
**Average Resolution Rate:** 67% closed (14/21 cases)

**Symptoms:**
- Custom fonts not rendering in generated PDF
- Rich text formatting lost in document generation
- Numbered list counter not restarting in repeating blocks
- Image and text overlapping in generated document
- Currency/date formatting incorrect for locale (French, German, etc.)
- Font settings not applied from rich text tokens

**Common Error Messages:**
```
"<w:startOverride w:val=" (appears in generated document - indicates numbering.xml issue)
"Font not rendering correctly in PDF"
"Rich text formatting lost in document generation"
"Custom font not appearing"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Custom font not uploaded to Salesforce | Check Setup → Document Generation → Custom Fonts |
| 2 | Font file exceeds size limit (typically 5MB) | Check font file size |
| 3 | Rich text token not using correct syntax | Review template: use `{{{RichTextField}}}` (triple braces) |
| 4 | Numbered list `startOverride` not generated in repeating block | Check .docx → word/numbering.xml for startOverride entries |
| 5 | Locale-specific formatting not configured | Check user locale vs. org default locale |

**Resolution Steps:**

1. **Upload Custom Fonts:**
   - Navigate to: Setup → Document Generation → Custom Fonts
   - Click "Upload Font"
   - Select .ttf or .otf file (max 5MB typically)
   - Font Name: (name to reference in template)
   - Upload → Save
   - In template, reference: Font Family = "CustomFontName"

2. **Fix Rich Text Rendering:**
   - **Correct syntax:** `{{{Contract.RichTextField__c}}}` (triple curly braces)
   - **Wrong syntax:** `{{Contract.RichTextField__c}}` (double braces = plain text)
   - Triple braces preserve HTML formatting

3. **Fix Numbered List Counter (Repeating Blocks):**
   - Issue: Numbered lists in `{{#items}}...{{/items}}` blocks don't restart
   - Root cause: DocGen doesn't auto-generate `<w:startOverride>` in numbering.xml
   - **Workaround:**
     1. Create template with numbered list inside repeating block
     2. Generate test document → Save as .docx
     3. Extract .docx → Open word/numbering.xml
     4. Manually add: `<w:startOverride w:val="1"/>` to each abstract num definition
     5. Repackage .docx → Upload as template
   - **Known Limitation:** This is a platform limitation (logged with Product Team)

4. **Fix Locale-Specific Formatting:**
   - Check user locale: Setup → Users → View user → Locale
   - Currency: Use `{{Contract.Amount | currency}}` filter
   - Date: Use `{{Contract.StartDate | date('DD/MM/YYYY')}}` format
   - For French locale: Ensure decimal separator = comma, thousands separator = space

5. **Fix Image/Text Overlap:**
   - Check image size in template (large images cause overlap)
   - Set image width explicitly in template: `<img width="400px" ...>`
   - Use page breaks to separate content sections
   - Check Word template margins/padding

**Verification:**
- Custom fonts render correctly in generated PDF
- Rich text fields preserve formatting (bold, italic, lists, etc.)
- Numbered lists restart at 1 in each repeating block section
- Currency/date formats match user locale
- Images don't overlap with text

**Known Limitations:**
- Font file size limit: typically 5MB (varies by org)
- Numbered list restart requires manual numbering.xml edit (no UI support)
- Some complex Word formatting may not preserve in PDF conversion

**Escalation Criteria:**
- Custom font uploaded but not rendering → Escalate with font file and template
- Numbered list workaround doesn't work → Escalate to Product Team (known limitation)
- Locale formatting incorrect despite correct filters → Escalate with template and sample data

**Sample Case Numbers:** 473481791, 473456968, 472984252, 473337912

---

### Pattern 6: DocGen Context Service/Integration Issues

**Frequency:** 19 cases (6.3% of total)  
**Severity:** Priority 45: 8 cases | Priority 27: 3 cases  
**Average Resolution Rate:** 79% closed (15/19 cases)

**Symptoms:**
- "Couldn't fetch the context definition name" error
- Context Use Case Mapping configuration error
- Integration Procedure cannot be attached to Document Template
- OmniScript DocGen function fails with "exception occurred"
- Context service timeout or unavailable
- "User license does not allow Visualforce page access" during contract creation

**Common Error Messages:**
```
"We couldn't fetch the context definition name and its associated mapping name 
from the context use case mapping object"
"OmniScript document generation error: exception occurred"
"The user license does not allow Visualforce page access"
"Cannot attach Integration Procedure in Document Template setup"
"Context service timeout"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Context Use Case Mapping not configured for target object | Check Setup → Context Use Case Mapping |
| 2 | Integration Procedure not activated | Check OmniStudio → Integration Procedures → Status |
| 3 | User license doesn't support OmniStudio | Check user license type |
| 4 | Context Definition or Context Mapping record missing | Query ContextDefinition and ContextMapping objects |
| 5 | Visualforce page access blocked by license | Guest/Community users cannot access VF pages |

**Resolution Steps:**

1. **Configure Context Use Case Mapping:**
   - Navigate to: Setup → Context Use Case Mapping
   - Click "New" (if mapping doesn't exist)
   - Required fields:
     - **Target Object:** Contract (or custom object)
     - **Target Record Type:** (leave blank for all, or specify record type)
     - **Context Definition Name:** (from Context Definition object - typically "ContractContext")
     - **Context Mapping Name:** (from Context Mapping object - e.g., "ContractMapping")
     - **Use Case Name:** (e.g., "Contract Generation")
   - Save
   - **Verify:** Query to confirm:
     ```sql
     SELECT Id, TargetObject, TargetRecordType, ContextDefinitionName, ContextMappingName 
     FROM ContextUseCaseMapping__c 
     WHERE TargetObject = 'Contract'
     ```

2. **Check Context Definition and Context Mapping:**
   ```sql
   -- Verify Context Definition exists
   SELECT Id, Name, ContextId__c 
   FROM ContextDefinition__c 
   WHERE Name = 'ContractContext'
   
   -- Verify Context Mapping exists
   SELECT Id, Name, SourceObjectName__c, TargetObjectName__c 
   FROM ContextMapping__c 
   WHERE Name = 'ContractMapping'
   ```
   - If missing: May need to be created via managed package or custom setup
   - Contact Salesforce if standard context definitions missing

3. **Attach Integration Procedure to Document Template:**
   - Setup → Document Generation → Document Templates → Select template
   - **Before Generation Integration Procedure:** (select IP)
   - **After Generation Integration Procedure:** (optional)
   - **Note:** Integration Procedure must be:
     - **Activated** (Version Status = Active)
     - **Owned by running user** or shared via OmniStudio sharing
   - If IP not visible: Check activation status and permissions

4. **Fix User License Issues (VF page access):**
   - Guest/Community users: Cannot access Visualforce pages
   - **Workaround:** Use Platform Event to trigger contract creation in system context
   - **Alternative:** Use Apex Invocable with `without sharing` annotation

5. **Troubleshoot OmniScript DocGen Function:**
   - Enable debug logs for running user
   - Reproduce error → Download debug log
   - Search for: "DocGen", "exception", "error"
   - Common causes: Missing field, incorrect merge syntax, permission issue

**Verification:**
- Context Use Case Mapping exists and populated
- Document generation completes without "context" errors
- Integration Procedure executes before/after document generation
- OmniScript DocGen function returns success

**Pro Tips:**
- Context Use Case Mapping is REQUIRED for Revenue Cloud contract operations
- Always activate Integration Procedures before attaching to templates
- Test with System Admin first to isolate license/permission issues

**Escalation Criteria:**
- Context Use Case Mapping configured but error persists → Escalate with configuration screenshots
- Context Definition/Mapping missing and cannot be created → Escalate to Provisioning
- OmniScript DocGen exception with no clear error → Escalate with debug logs

**Sample Case Numbers:** 473427259, 473197057, 473240016, 473502960, 473479932

---

### Pattern 7: DocGen Performance/Timeout

**Frequency:** 18 cases (6.0% of total)  
**Severity:** Priority 45: 8 cases | Priority 13: 4 cases  
**Average Resolution Rate:** 50% closed (9/18 cases)

**Symptoms:**
- Document generation takes several minutes (should be seconds)
- Timeout errors during PDF generation
- Performance degradation after Salesforce release (e.g., Spring '26)
- Image download delays from Content Files
- Slow Content Document Download (especially via Nintex DocGen)
- Batch job failures due to DocGen timeouts

**Common Error Messages:**
```
"DocGen Runtime timeout error"
"Nintex DocGen delays post Spring 26 release"
"Completed with failures" (batch job)
"Content Document Download slow"
"DocGen Process stalling when waiting for Salesforce to respond with downloaded image"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Large images in ContentVersion causing slow download | Check image file sizes in document template |
| 2 | Spring '26 release throttling third-party image downloads | Check if issue started after Spring '26 release |
| 3 | Template complexity (many merge fields, loops, images) | Count merge fields and repeating blocks in template |
| 4 | Bulk document generation (e.g., batch of 100+ contracts) | Check if timeout occurs in batch context |
| 5 | External service latency (M365, SharePoint, DocuSign) | Test external service response times |

**Resolution Steps:**

1. **Optimize Template Performance:**
   - **Reduce image sizes:** Compress images before uploading to Content Files
     - Target: < 500KB per image
     - Use tools like TinyPNG, ImageOptim
   - **Limit merge fields:** Avoid unnecessary fields in template
   - **Simplify repeating blocks:** Reduce nesting of `{{#loop}}...{{/loop}}`
   - **Use conditional rendering sparingly:** Too many `{{#if}}` blocks slow rendering

2. **Address Spring '26 Image Download Throttling:**
   - **Issue:** Salesforce may throttle third-party image downloads (Nintex DocGen affected)
   - **Workaround 1:** Reduce image count in templates
   - **Workaround 2:** Use Salesforce-hosted images (Files/Content) instead of external URLs
   - **Workaround 3:** Request extension for Session ID removal (if related)
   - **Escalate:** If business-critical, escalate to Product Team with org ID and case details

3. **Optimize Bulk Document Generation:**
   - **Best Practice:** Generate documents asynchronously (Batch Apex, Queueable)
   - **Avoid:** Synchronous generation of > 10 documents at once
   - **Pattern:** 
     ```apex
     // Instead of: for each contract, generate document immediately
     // Use: Batch Apex with batch size = 10
     Database.executeBatch(new ContractDocGenBatch(), 10);
     ```

4. **Test External Service Performance:**
   - **Microsoft 365:** Test SharePoint API latency
   - **DocuSign:** Test envelope creation API latency
   - **Nintex:** Contact Nintex support to verify their service health

5. **Enable DocGen Server-Side Generation:**
   - Navigate to: Setup → Document Generation → General Settings
   - Check: "Enable Server-Side Document Generation"
   - **Benefit:** Offloads rendering to Salesforce servers (faster for large documents)

**Verification:**
- Document generation completes in < 30 seconds (for typical contract)
- No timeout errors during PDF generation
- Batch jobs complete successfully
- Image download performance acceptable (< 5 seconds per image)

**Known Issues (Spring '26+):**
- Nintex DocGen experiencing image download delays (Salesforce throttling suspected)
- External services may be throttled when downloading images from Content Files
- **Recommendation:** Test in sandbox after each Salesforce release

**Escalation Criteria:**
- Performance degradation after release → Escalate to Product Team with before/after metrics
- Timeout persists despite optimization → Escalate with template file and debug logs
- Third-party service (Nintex) affected → Escalate with service logs and Salesforce logs

**Sample Case Numbers:** 472848155, 473396671, 473516586, 472826552, 473487943

---

### Pattern 8: DocGen Image/Content Download Issues

**Frequency:** 15 cases (5.0% of total)  
**Severity:** Priority 45: 6 cases | Priority 13: 3 cases  
**Average Resolution Rate:** 87% closed (13/15 cases)

**Symptoms:**
- Images not appearing in generated PDF
- Content Document Download slow (> 1 minute)
- Image/text overlapping in document
- "Slow Content Document Download" error
- External applicant cannot download offer letter

**Common Error Messages:**
```
"OmniScript document generation error: exception occurred"
"Slow Content Document Download"
"Image not rendering in PDF"
"Document download fails for external users"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | ContentVersion file size too large | Check ContentVersion.ContentSize field |
| 2 | External user (guest) lacks ContentDocument access | Check guest user permissions |
| 3 | Image URL in template is broken/inaccessible | Test image URL in browser |
| 4 | Spring '26 throttling image downloads | Check if issue started post-release |
| 5 | Image format not supported (BMP, TIFF, etc.) | Check image file type (use PNG, JPG, GIF) |

**Resolution Steps:**

1. **Optimize Image Files:**
   - Compress images to < 500KB per image
   - Use supported formats: PNG, JPG, GIF (avoid BMP, TIFF)
   - Upload compressed images to Salesforce Files
   - Reference in template: `<img src="{{ImageURL}}">`

2. **Fix External User Access:**
   - Grant Guest User access to ContentDocument:
     - Setup → Sharing Settings → ContentDocument → Guest User Access = Read Only
   - OR: Use Platform Event to generate document in system context
   - OR: Store generated PDF in external storage (M365, public URL)

3. **Verify Image URLs:**
   - Test image URL in browser (should load immediately)
   - For Salesforce Files: Use ContentDistribution public URL
   - For external URLs: Ensure CORS headers allow Salesforce access

4. **Address Spring '26 Throttling:**
   - Same as Pattern 7 (Performance/Timeout)
   - Reduce image count in templates
   - Use Salesforce-hosted images

**Verification:**
- Images appear correctly in generated PDF
- Content download completes in < 10 seconds
- External users can download documents

**Sample Case Numbers:** 473438106, 473181826, 473460395, 473487943, 473323637

---

### Pattern 9: E-signature Configuration Issues

**Frequency:** 15 cases (5.0% of total)  
**Severity:** Priority 0: 7 cases | Priority 45: 5 cases  
**Average Resolution Rate:** 47% closed (7/15 cases)

**Symptoms:**
- DocuSign authentication error ("We can't log you in because of an authentication error")
- Named Credentials configuration fails
- Envelope creation error ("Bad Request")
- Envelope status not updating after signing
- Anchor tag/signature field configuration not working
- E-signature prepopulates only some fields (name, email but not custom fields)

**Common Error Messages:**
```
"We can't log you in because of an authentication error. For help, contact 
your Salesforce administrator."
"We couldn't create the envelope because of an error: Bad Request"
"Envelope status not updating"
"Authentication error while connecting DocuSign via Named Credentials"
"JSON structure error when calling DocuSign API"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Named Credentials not configured or expired | Check Setup → Named Credentials → Authentication status |
| 2 | DocuSign Connected App misconfigured | Verify DocuSign Integration Key and Callback URL |
| 3 | Envelope JSON structure incorrect | Review JSON payload in API logs |
| 4 | Anchor tag syntax incorrect in template | Check template for anchor tag format: `/sn1/` |
| 5 | E-signature configuration prepopulation mapping incomplete | Review field mappings in E-signature Configuration |

**Resolution Steps:**

1. **Configure DocuSign Named Credentials:**
   - Setup → Named Credentials → New
   - **Name:** `DocuSign_API`
   - **URL:** `https://na3.docusign.net` (or your DocuSign environment)
   - **Identity Type:** Named Principal
   - **Authentication Protocol:** OAuth 2.0
   - **Scope:** `signature impersonation`
   - **Auth Provider:** DocuSign_AuthProvider (must create first)
   - Test authentication → should show "Authenticated"

2. **Configure DocuSign Authentication Provider:**
   - Setup → Auth. Providers → New → Custom
   - **Name:** `DocuSign_AuthProvider`
   - **Consumer Key:** (from DocuSign - Integration Key)
   - **Consumer Secret:** (from DocuSign)
   - **Authorize Endpoint URL:** `https://account.docusign.com/oauth/auth`
   - **Token Endpoint URL:** `https://account.docusign.com/oauth/token`
   - **Default Scopes:** `signature impersonation`
   - **Include Consumer Secret in API Requests:** Yes
   - Save

3. **Configure DocuSign Connected App (in DocuSign):**
   - Log in to DocuSign → Settings → Integrations → Apps and Keys
   - Create new app or edit existing
   - **Redirect URI:** `https://<your-instance>.salesforce.com/services/authcallback/DocuSign_AuthProvider`
   - **Scopes:** `signature impersonation`
   - Copy Integration Key and Secret Key → paste into Salesforce Auth Provider

4. **Fix Envelope Creation JSON:**
   - Review API call logs for JSON structure
   - Common issues:
     - Missing required fields: `emailSubject`, `recipients`, `documents`
     - Incorrect recipient format
     - Missing anchor tag references
   - **Correct structure:**
     ```json
     {
       "emailSubject": "Please sign this document",
       "status": "sent",
       "documents": [
         {
           "documentId": "1",
           "name": "Contract.pdf",
           "documentBase64": "<base64-encoded-pdf>"
         }
       ],
       "recipients": {
         "signers": [
           {
             "email": "signer@example.com",
             "name": "Signer Name",
             "recipientId": "1",
             "tabs": {
               "signHereTabs": [
                 {
                   "anchorString": "/sn1/",
                   "anchorXOffset": "0",
                   "anchorYOffset": "0"
                 }
               ]
             }
           }
         ]
       }
     }
     ```

5. **Configure Anchor Tags in Template:**
   - In Word template, insert anchor tags where signatures needed:
     - **Signature:** `/sn1/` (for signer 1), `/sn2/` (for signer 2), etc.
     - **Date:** `/ds1/`
     - **Text field:** `/txt1/`
   - **Note:** Anchor strings must match exactly in DocuSign envelope JSON

6. **Fix E-signature Configuration Prepopulation:**
   - Setup → E-signature Configuration → Edit
   - Verify field mappings for ALL fields (not just Name and Email)
   - Example:
     - Recipient Name → `{{Contact.Name}}`
     - Recipient Email → `{{Contact.Email}}`
     - Company → `{{Account.Name}}`
     - Title → `{{Contact.Title}}`
   - **Important:** Use correct merge field syntax for your object context

**Verification:**
- DocuSign authentication succeeds (can log in)
- Envelope creation succeeds (no "Bad Request" error)
- Signature request email sent to recipients
- After signing, envelope status updates in Salesforce
- All e-signature configuration fields prepopulate

**Known Limitations:**
- DocuSign anchor tags must be configured in template OR DocuSign template (not both)
- Some advanced DocuSign features require DocuSign template setup (not anchor tags)
- E-signature configuration field mapping depends on object context (Contract, Quote, etc.)

**Escalation Criteria:**
- Named Credentials authentication fails after correct setup → Escalate with auth provider config
- Envelope creation fails with "Bad Request" despite correct JSON → Escalate with API logs
- Anchor tags not working → Escalate with template file and envelope JSON

**Sample Case Numbers:** 473388354, 473305889, 473420006, 473515633, 473254710

---

### Pattern 10: Session ID/Outbound Message Deprecation

**Frequency:** 10 cases (3.3% of total)  
**Severity:** Priority 45: 6 cases | Priority 26: 2 cases  
**Average Resolution Rate:** 50% closed (5/10 cases)

**Symptoms:**
- Nintex DocGen integration failing after Session ID removal
- Outbound Message without Session ID not working
- Request for extension on Session ID removal
- DocGen integration using Session ID needs migration
- Third-party integration (Nintex) broken post-deprecation

**Common Error Messages:**
```
"Session ID removed from Outbound Message"
"Nintex DocGen delays post Spring 26 release"
"DocGen Outbound Message w/o Session ID failing"
"Extension request for Session ID removal enforcement"
```

**Root Causes:**

| # | Root Cause | How to Identify |
|---|-----------|-----------------|
| 1 | Integration relies on Session ID in Outbound Message | Check Workflow Outbound Message configuration |
| 2 | Salesforce enforced Session ID deprecation | Check if issue started after specific enforcement date |
| 3 | Third-party vendor (Nintex) not updated to remove Session ID dependency | Contact Nintex support |
| 4 | Custom integration using Session ID needs refactor | Review custom Apex/integration code |

**Background:**
- **Salesforce announcement:** Session ID in Outbound Messages deprecated (security risk)
- **Enforcement:** Rolling enforcement starting Feb 2026 (varies by org)
- **Impact:** Third-party services (Nintex DocGen, Conga Composer, etc.) that relied on Session ID for authentication

**Resolution Steps:**

1. **Identify Outbound Messages Using Session ID:**
   - Setup → Workflow Actions → Outbound Messages
   - Check each Outbound Message configuration
   - If "Include Session ID" is checked → needs migration

2. **Migration Options:**

   **Option A: Use Named Credentials (RECOMMENDED)**
   - Replace Session ID-based authentication with OAuth 2.0 Named Credentials
   - Setup → Named Credentials → New
   - Configure OAuth 2.0 authentication for target service
   - Update target service to accept Named Credential token (not Session ID)

   **Option B: Use Platform Event + Flow**
   - Replace Workflow Outbound Message with Platform Event
   - Flow listens to Platform Event → calls external service via HTTP Callout
   - HTTP Callout uses Named Credential (OAuth 2.0)
   - **Benefit:** More control, better error handling, retries

   **Option C: Use Apex Callout**
   - Replace Workflow Outbound Message with Apex Trigger → Callout
   - Callout uses Named Credential (OAuth 2.0)
   - **Benefit:** Full programmatic control

3. **For Nintex DocGen Specifically:**
   - **Nintex announcement:** Nintex is updating their service to remove Session ID dependency
   - **Timeline:** Nintex aiming for Q1 2026 update
   - **Workaround (temporary):** Request extension from Salesforce via support case
     - Provide: Org ID, business justification, vendor timeline
     - Salesforce may grant 3-4 week extension
   - **Long-term:** Upgrade to latest Nintex DocGen package (post-Session ID removal)

4. **Request Extension (Temporary):**
   - Open support case with Salesforce
   - Subject: "Request extension for Session ID removal enforcement"
   - Provide:
     - Org ID
     - Business impact (customer count, critical workflows)
     - Vendor migration timeline (if applicable)
     - Justification for extension
   - **Note:** Extensions granted case-by-case, not guaranteed

**Verification:**
- Outbound Message no longer uses Session ID
- External service integration works without Session ID
- Named Credentials authentication succeeds
- Document generation completes successfully

**Known Timeline:**
- **Feb 2026:** Initial enforcement wave
- **Q1 2026:** Nintex DocGen update expected
- **Q2 2026:** Universal enforcement (all orgs)

**Escalation Criteria:**
- Extension request denied but business-critical → Escalate to Account Team
- Vendor (Nintex) not providing migration path → Escalate with vendor and Salesforce
- Migration complex and timeline tight → Engage Professional Services

**Sample Case Numbers:** 473487943, 473323637, 473471500, 473507303, 472178820

---

### Pattern 11: DocGen Feature Missing/Setup Issues

**Frequency:** 7 cases (2.3% of total)  
**Severity:** Priority 45: 5 cases  
**Average Resolution Rate:** 86% closed (6/7 cases)

**Symptoms:**
- Revenue Cloud Document Builder action missing from Quote
- OmniStudio Document Generation feature missing from Setup
- DocGen permission sets not visible
- Cannot locate Document Template Designer
- "Generate PDF Document" action not available

**Root Causes:**
- OmniStudio DocGen not provisioned in org
- Feature flag not enabled
- User lacks DocGen Designer/User permission sets
- License doesn't include DocGen access

**Resolution Steps:**
1. Check org has OmniStudio DocGen provisioned
2. Verify Setup → Document Generation appears
3. Assign permission sets: OmniStudio DocGen Designer, OmniStudio DocGen User
4. If missing: Contact Salesforce to provision DocGen

**Sample Case Numbers:** 473519099, 473502960, 473504124, 473472122, 473420988

---

### Pattern 12: Currency/Number Formatting Issues

**Frequency:** 7 cases (2.3% of total)  
**Severity:** Priority 31: 2 cases | Priority 45: 2 cases  
**Average Resolution Rate:** 57% closed (4/7 cases)

**Symptoms:**
- Currency fields showing incorrect decimal places
- Currency symbol incorrect ($ instead of € or vice versa)
- Number formatting incorrect for locale (French uses comma as decimal)
- Contract document showing user currency instead of quote currency

**Resolution Steps:**
1. Use currency filter in merge fields: `{{Amount | currency}}`
2. Check user locale: Setup → Users → Locale
3. Specify currency in template: `{{Amount | currency('EUR')}}`
4. For quote currency: Use `{{Quote.CurrencyIsoCode}}` with amount

**Sample Case Numbers:** 473283724, 473492971, 472665221, 473488902, 473013840

---

### Pattern 13: DocGen Template Errors

**Frequency:** 4 cases (1.3% of total)  
**Symptoms:**
- Template fails merge (generic error)
- Runtime timeout error during template rendering
- Unable to access Document Template Designer
- DocGen template fails with no specific error

**Resolution Steps:**
1. Test template with minimal merge fields
2. Check template syntax (valid Mustache syntax)
3. Enable debug logs for DocGen service
4. Check for corrupted .docx file (re-create template if needed)

**Sample Case Numbers:** 473238455, 473413691, 473132688, 473206459

---

## C. Setup & Configuration Guide

### Complete CLM + DocGen Setup Checklist

```
1. LICENSE VERIFICATION
   □ OmniStudio DocGen provisioned (Setup → Document Generation appears)
   □ Revenue Cloud license active
   □ User licenses support DocGen (Platform, Unlimited, Enterprise)

2. PERMISSION SETS (assign to all CLM/DocGen users)
   □ OmniStudio DocGen Designer (for template creators)
   □ OmniStudio DocGen User (for document generators)
   □ CLM Runtime User (for contract operations)
   □ Contract: Create, Read, Edit, Delete
   □ ContractDocumentVersion: Create, Read, Edit, Delete
   □ DocumentTemplate: Read (end users) / CRUD (admins)
   □ ContentVersion: Create, Read
   □ ContentDocument: Read

3. DOCGEN SETTINGS
   □ Setup → Document Generation → General Settings
   □ Enable OmniStudio Document Generation: ✓
   □ Enable Server-Side Document Generation: ✓ (for performance)
   □ Custom Fonts uploaded (if needed)

4. CONTEXT USE CASE MAPPING
   □ Setup → Context Use Case Mapping → New
   □ Target Object: Contract
   □ Context Definition Name: ContractContext (or org-specific)
   □ Context Mapping Name: ContractMapping
   □ Verify mapping exists for all contract record types

5. CONTRACT TYPE CONFIGURATION
   □ Setup → Contract Type Config → Select type
   □ Document Storage Type: Internal or Microsoft 365
   □ If External: External Storage Configuration (Named Credential)

6. EXTERNAL STORAGE (if using Microsoft 365)
   □ Azure App Registration created
   □ Auth Provider configured (OAuth 2.0)
   □ Named Credentials configured and authenticated
   □ SharePoint Site URL and Document Library configured
   □ Test: Generate document → Save to M365 → Verify appears

7. E-SIGNATURE (if using DocuSign)
   □ DocuSign account active
   □ Connected App in DocuSign with correct Redirect URI
   □ Auth Provider configured in Salesforce
   □ Named Credentials configured and authenticated
   □ E-signature Configuration created with field mappings
   □ Test: Send envelope → Verify recipient receives email

8. DOCUMENT TEMPLATES
   □ Create template in Word (.docx)
   □ Add merge fields: {{ObjectName.FieldName}}
   □ Upload: Setup → Document Generation → Document Templates → New
   □ Test: Generate document → Verify fields populate
```

### Common Misconfigurations

| Misconfiguration | Error/Symptom | Fix |
|-----------------|---------------|-----|
| Permission sets not assigned | "Insufficient privileges" | Assign DocGen Designer/User + CLM Runtime User |
| Context Use Case Mapping missing | "Couldn't fetch context definition" | Create mapping: Target Object = Contract |
| Named Credentials not authenticated | External storage/e-signature fails | Re-authenticate Named Credential |
| Merge field syntax incorrect | Field blank in document | Use {{Object.Field__c}} (exact API name) |
| Rich text field double braces | Formatting lost | Use triple braces: {{{Field}}} |
| Integration Procedure not activated | IP not attachable to template | Activate IP in OmniStudio |
| Custom font not uploaded | Font not rendering | Upload font: Setup → Document Generation → Custom Fonts |
| Image too large | Slow generation/timeout | Compress image to < 500KB |

---

## D. Licensing & Entitlements

### Required Licenses

| Feature | Required License | Notes |
|---------|-----------------|-------|
| Revenue Cloud CLM | Revenue Cloud License | Includes contract management |
| OmniStudio DocGen | OmniStudio License OR Platform License | Required for document generation |
| External Storage (M365) | Revenue Cloud + Storage Add-on | Check with AE if included |
| E-signature (DocuSign) | DocuSign Account + RCA DocuSign | Separate DocuSign subscription |
| Word Connector | Revenue Cloud + M365 Integration | Requires M365 license for users |

### License Troubleshooting Flowchart

```
Cannot access DocGen?
│
├── Check: Setup → Document Generation menu visible?
│   │
│   ├── NO → DocGen not provisioned
│   │   └── Contact Salesforce to provision OmniStudio DocGen
│   │
│   └── YES → Check permission sets
│       ├── DocGen Designer assigned? (for template creation)
│       ├── DocGen User assigned? (for document generation)
│       └── CLM Runtime User assigned? (for contract operations)
│           │
│           ├── NO → Assign permission sets
│           │
│           └── YES → Check user license type
│               ├── Guest/Community → CANNOT use DocGen (requires internal license)
│               └── Platform/Unlimited/Enterprise → Should work (escalate if not)
```

---

## E. Key Behavioral Design Rules

These are **by design** — not bugs. Educate the customer:

1. **Session ID Deprecation:** Session ID in Outbound Messages is deprecated (security). Must migrate to Named Credentials.
2. **Numbered List Restart:** Numbered lists in repeating blocks do NOT auto-restart. Requires manual numbering.xml edit (no UI support).
3. **Rich Text Rendering:** Must use triple braces `{{{Field}}}` to preserve formatting (double braces = plain text).
4. **Context Use Case Mapping Required:** Revenue Cloud contract operations require Context Use Case Mapping configuration.
5. **External Storage:** Microsoft 365 integration requires Azure App Registration (admin access needed).
6. **Guest User Limitation:** Guest/Community users cannot use DocGen (requires internal license).
7. **Font Upload Limit:** Custom font files typically limited to 5MB (varies by org).
8. **Template Syntax:** Use exact Field API names (not labels) in merge tokens.
9. **Quote Path Resolution:** After Spring '26, use `SalesTransactionSource` path for quote fields (not direct `Quote` path).
10. **E-signature Anchor Tags:** Anchor tags must be consistent in template AND DocuSign envelope JSON.

---

## F. Escalation Paths

### When to Escalate

| Scenario | Team | Priority |
|----------|------|----------|
| DocGen not provisioned in org | Provisioning Team | P2 |
| Context Use Case Mapping error persists after config | Revenue Cloud Engineering | P2 |
| Performance degradation after release | Product Team (release regression) | P1 |
| Session ID deprecation blocking business | Product Management + Account Team | P1 |
| Named Credentials authentication fails | Platform Engineering | P2 |
| Third-party integration broken (Nintex, Conga) | Third-party vendor + Salesforce liaison | P2 |
| Template rendering bug (platform issue) | OmniStudio Engineering | P3 |

### Gather Before Escalating

```
□ Org ID (18-char)
□ User ID(s) affected
□ Exact error message (full text)
□ Steps to reproduce (numbered)
□ Template file (.docx)
□ Sample record ID (Contract, Quote, etc.)
□ Debug logs (Apex + System) from failing operation
□ Screenshots: Error state, configuration (Context Use Case Mapping, Named Credentials, etc.)
□ Last known working date (if regression)
□ Salesforce release version (if performance/behavior change)
□ Third-party service logs (if integration issue)
```

### High-Priority Issue References

Based on OrgCS case analysis (last 6 months):
- **41 cases:** DocGen Permission/Access issues (Priority 45: 20 cases)
- **31 cases:** Contract Status/Activation issues (Priority 45: 12 cases)
- **25 cases:** External Storage issues (Priority 45: 12 cases)
- **23 cases:** Merge Field/Data Population issues (Priority 45: 7 cases)

**To pull similar cases (if OrgCS connected):**
```
FIND {DocGen permission} IN ALL FIELDS RETURNING Case(Id, CaseNumber, Subject, Status, CreatedDate LIMIT 50)
FIND {Context Use Case Mapping} IN ALL FIELDS RETURNING Case(Id, CaseNumber, Subject, Status, CreatedDate LIMIT 50)
```

---

## G. Tribal Knowledge & Pro Tips

*Source: Revenue Cloud support channels, OrgCS case resolutions*

**Pro Tips (from resolved cases):**

1. **Always test with System Admin first** — Isolates permission vs. configuration issues
2. **Context Use Case Mapping is REQUIRED** — Don't skip this. It's the #1 cause of "couldn't fetch context definition" errors
3. **Triple braces for rich text** — `{{{RichField}}}` preserves formatting, `{{RichField}}` strips it
4. **Compress images** — Target < 500KB per image to avoid timeouts
5. **After releases, check Context path resolution** — Spring '26 changed quote path resolution (use `SalesTransactionSource`)
6. **Named Credentials over Session ID** — Always. Session ID deprecated, Named Credentials more secure
7. **Integration Procedures must be Activated** — Can't attach inactive IP to template
8. **External storage needs Azure admin** — M365 integration requires Azure App Registration (customer's IT admin)
9. **Anchor tags are case-sensitive** — `/sn1/` ≠ `/SN1/`
10. **Template syntax matters** — Use exact Field API names (not labels): `Custom_Field__c` not `Custom Field`

**Common "Gotchas":**
- Merge field blank? Check field API name (case-sensitive), check field has value on record, check FLS
- E-signature not prepopulating fields? Check field mappings in E-signature Configuration (all fields, not just Name/Email)
- Numbered list not restarting? Known limitation — requires manual numbering.xml edit
- Image not rendering? Check file size (< 500KB), check format (PNG/JPG), check URL accessible
- Template not attachable? Check Integration Procedure is Activated

---

## H. Active Known Issues (Live Lookup)

**If GUS is connected, run this query to get current open bugs:**

```bash
sf data query --query "SELECT Name, Subject__c, Priority__c, Status__c, CreatedDate FROM ADM_Work__c WHERE (Product_Tag__r.Name LIKE '%Revenue Cloud%' OR Product_Tag__r.Name LIKE '%DocGen%' OR Product_Tag__r.Name LIKE '%OmniStudio%') AND Type__c = 'Bug' AND Status__c NOT IN ('Closed', 'Duplicate', 'Never Fix', 'Not a Bug') ORDER BY Priority__c, CreatedDate DESC LIMIT 20" --target-org gus --json
```

**If OrgCS is connected, search for recent similar cases:**
```
FIND {<your_error_message_keyword>} IN ALL FIELDS RETURNING Case(Id, CaseNumber, Subject, Status, Resolution__c, CreatedDate WHERE CreatedDate = LAST_N_MONTHS:3 ORDER BY CreatedDate DESC LIMIT 20)
```

**Known Issues (based on case analysis):**
- **Spring '26 Release:** Image download performance degradation (Nintex DocGen affected)
- **Session ID Deprecation:** Rolling enforcement causing integration failures
- **Context Path Resolution:** `Quote.*` path changed to `SalesTransactionSource.*` in some contexts
- **Numbered List Restart:** Not supported in repeating blocks (manual numbering.xml edit required)

---

## I. Code Reference Map

**For deep investigation, use CodeSearch to trace:**

### Key Modules

```
Revenue Cloud CLM:
├── Contract Lifecycle Management
│   ├── ContractDocumentVersion (object)
│   ├── ContractActivation (process)
│   └── ContextUseCaseMapping (configuration)
│
├── Document Generation (OmniStudio DocGen)
│   ├── DocumentTemplate (object)
│   ├── DocGenService (Apex service)
│   └── MergeFieldResolver (template rendering)
│
├── Context Service
│   ├── ContextDefinition (object)
│   ├── ContextMapping (object)
│   └── leanerQueryTags() (API method)
│
└── E-signature Integration
    ├── ESignatureConfiguration (object)
    ├── EnvelopeService (Apex service)
    └── DocuSign API callouts
```

### CodeSearch Queries for Live Investigation

**When investigating a specific case, use these queries:**

| Scenario | Query |
|----------|-------|
| Trace contract activation | `content:"ContractActivation" lang:java` |
| Find Context Use Case Mapping logic | `content:"ContextUseCaseMapping" lang:java` |
| Trace merge field resolution | `content:"MergeFieldResolver" OR content:"leanerQueryTags" lang:java` |
| Find DocGen service | `content:"DocGenService" lang:java` |
| Trace e-signature envelope creation | `content:"EnvelopeService" OR content:"DocuSign" lang:apex` |
| Find external storage integration | `content:"ExternalStorage" OR content:"Microsoft365" lang:java` |
| Trace Session ID usage | `content:"SessionId" content:"OutboundMessage" lang:apex` |
| Find contract document version logic | `content:"ContractDocumentVersion" lang:java` |

---

## J. Troubleshooting Decision Tree

```
User reports CLM/DocGen issue
│
├── 1. CLASSIFY ISSUE (Section A)
│   ├── Permission/Access? → Pattern 1
│   ├── Contract Status? → Pattern 2
│   ├── External Storage? → Pattern 3
│   ├── Merge Fields? → Pattern 4
│   ├── Fonts/Formatting? → Pattern 5
│   ├── Context Service? → Pattern 6
│   ├── Performance/Timeout? → Pattern 7
│   ├── Images/Content? → Pattern 8
│   ├── E-signature? → Pattern 9
│   └── Session ID? → Pattern 10
│
├── 2. ASK QUICK DIAGNOSTIC QUESTIONS (Section A)
│   ├── What's the exact error?
│   ├── When did it start?
│   ├── Which user/profile?
│   ├── Prod or sandbox?
│   └── Can System Admin reproduce?
│
├── 3. FOLLOW PATTERN RESOLUTION STEPS
│   ├── Check configuration
│   ├── Verify permissions
│   ├── Test with minimal example
│   ├── Review logs
│   └── Apply workaround if available
│
├── 4. VERIFY FIX
│   ├── Test with original scenario
│   ├── Test with System Admin
│   ├── Test with affected user
│   └── Document resolution
│
└── 5. ESCALATE IF NEEDED (Section F)
    ├── Gather diagnostic info
    ├── Identify correct team
    ├── Provide context
    └── Reference similar cases
```
