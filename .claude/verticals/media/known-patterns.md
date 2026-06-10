## 🔥 TOP 10 ISSUE PATTERNS (Ranked by Frequency/Severity)

### PATTERN #1: Add Placement Button Not Working
**Frequency**: HIGH | **Severity**: Level 3 High | **Cases**: Multiple (43925592, 44075772)

**Symptoms**:
- Click "Add Placement" button in Media Plan
- New blank row NOT added to Data Table
- No error message displayed
- Media Plan creation journey blocked

**Root Causes**:
1. Missing permissions on `MediaAdSalesAppHandler` Apex class
2. Integration Procedure `sfiAds-DigitalHandler` permission issue for portal users
3. Field-level security on Media Plan related objects
4. LWC component cache issue

**Diagnosis Steps**:
```
1. Check debug logs for errors when clicking "Add Placement"
2. Verify user has access to:
   - MediaAdSalesAppHandler Apex class
   - sfiAds-DigitalHandler Integration Procedure
   - Media Plan object CRUD + FLS on all required fields
3. Check browser console for LWC errors
4. Verify Media Plan is in correct status (not submitted/locked)
```

**Resolution**:
```
STEP 1: Verify Apex Class Access
- Navigate to Setup → Apex Classes → MediaAdSalesAppHandler
- Click "Security" and add customer's profile/permission set

STEP 2: Grant Integration Procedure Access
- Setup → OmniStudio → Integration Procedures → sfiAds-DigitalHandler
- Click "Activate" if inactive
- Add profile to "Permitted Profiles" (especially for portal users)

STEP 3: Check Field-Level Security
- Verify FLS on Media_Plan__c object for:
  - Status__c (Read/Write)
  - PlacementData__c (Read/Write)
  - All placement-related fields

STEP 4: Clear LWC Cache (if above don't resolve)
- Have user hard refresh browser (Ctrl+Shift+R / Cmd+Shift+R)
- Or: Setup → Custom Code → Lightning Components → Clear Cache
```

**Code References**:
- `MediaAdSalesAppHandler.cls` - Main handler for Add Placement action (lines 116-122)
- `sfiAdsDigitalHandler` - Integration Procedure for digital placements
- `runtime_media_asm_mediaplan` - LWC component namespace

**Related Cases**: 43925592, 44075772, 44653314

---

### PATTERN #2: Next Button Greyed Out After Placement Configuration
**Frequency**: HIGH | **Severity**: Level 2-4 | **Cases**: 43959003, 44033313

**Symptoms**:
- User configures placement details in Media Plan
- Clicks to proceed to next step
- "Next" button is greyed out/disabled
- Cannot proceed with Media Plan creation journey

**Root Causes**:
1. Required field validation not met (but not showing error)
2. OmniScript conditional logic blocking next step
3. Browser session state issue
4. Missing data in placement configuration

**Resolution** (abbreviated - see full documentation for complete steps):
- Check browser console for validation errors
- Review OmniScript conditional logic
- Verify all required fields populated
- Clear browser cache if needed

**Related Cases**: 43959003, 44033313, multiple reports

---

### PATTERN #3: CalculationProcedure Query Error ⚠️ P1 CRITICAL
**Frequency**: MEDIUM | **Severity**: CRITICAL | **Bug**: Known Issue | **Case**: 44075772

**Symptoms**:
```
System.QueryException: sObject type 'CalculationProcedure' is not supported
```
- Error occurs during pricing calculation
- Blocks Media Plan creation/updates
- Error in logs: `AdSalesPricingPlanHelper.getCalculationProcedurePrice: line 111`

**Root Cause**:
- Missing namespace prefix in SOQL query
- Code queries `CalculationProcedure` but should query `CalculationProcedure__c`
- Standard vs. Custom object naming confusion

**Resolution**:
```
⚠️ THIS IS A CODE BUG - Cannot be fixed via configuration

IMMEDIATE WORKAROUND:
1. Contact Industries Media Cloud product team
2. Reference this known issue in #ad_sales_management_tech_group
3. Request hotfix or patch release

CODE FIX REQUIRED (for dev team):
File: AdSalesPricingPlanHelper.cls
Method: getCalculationProcedurePrice (line ~110)

BEFORE:
String query = 'SELECT Id, Name FROM CalculationProcedure WHERE Name = :procName';

AFTER:
String query = 'SELECT Id, Name FROM CalculationProcedure__c WHERE Name = :procName';
```

**Escalation**: REQUIRED - Escalate to dev team immediately via #ad_sales_management_tech_group

**Related Cases**: 44075772

---

### PATTERN #4: Undefined Ad Space Size Display
**Frequency**: MEDIUM | **Severity**: Level 2 Urgent | **Bug**: W-12964525 | **Cases**: 44381590, 44437205

**Symptoms**:
- Ad Space size displays as "undefinedxundefined" for seconds-based units
- Works correctly for pixel-based units (e.g., "300x250")
- Display issue in UI, but underlying data may be correct

**Root Cause**:
- LWC component logic handles pixel dimensions but not time-based units
- Missing null check for Width__c and Height__c fields for non-pixel ad types

**Code References**:
- LWC: `runtime_media_asm_mediaplan/adSpaceSelector`
- Object: `Ad_Space__c` (Width__c, Height__c, Duration__c, AdSpaceType__c)

**Workaround**: Ignore the display (cosmetic issue), backend processing uses correct Duration field

**Related Cases**: 44381590, 44437205

---

### PATTERN #5: Rate Type Value Missing on Clone
**Frequency**: MEDIUM | **Severity**: Level 3 High | **Bug**: W-13853569 | **Cases**: 44613109, 45042469

**Symptoms**:
- Clone existing Media Plan
- Rate Type field comes as blank in cloned placements
- Other fields clone correctly

**Root Cause**:
- Integration Procedure `sfiAds-DigitalCloneLineItems` missing Rate Type in field mapping
- Remote Class `MediaAdSalesAddPlacementRespHandler` not copying Rate Type

**Code References**:
- Integration Procedure: `sfiAds-DigitalCloneLineItems`
- Apex: `MediaAdSalesAddPlacementRespHandler.cls` (method: clonePlacement)

**Status**: Bug W-13853569, target fix in Summer '26 release

**Related Cases**: 44613109, 45042469

---

### PATTERN #6: LWC Component Error on Ad Placement Creation
**Frequency**: MEDIUM | **Severity**: Level 2 Urgent | **Cases**: Multiple

**Symptoms**:
```
Error: Cannot read properties of null (reading 'split')
Component: runtime_media_asm_mediaplan/typeSelector
Location: typeSelector.js:21:752
```
- Error when creating new Ad Placement
- Console shows JavaScript null pointer exception

**Root Causes**:
1. Missing data in parent Media Plan record (MediaType__c, Category__c, AdFormat__c)
2. Null value in picklist field being processed by LWC
3. Browser cache serving stale component version
4. Missing field-level security causing field to return null

**Code References**:
- LWC: `runtime_media_asm_mediaplan/typeSelector.js` line 21
- Related components: `adSpaceSelector`, `placementForm`

---

### PATTERN #7: Pricing Recalculation Not Triggered
**Frequency**: MEDIUM | **Severity**: Level 2 Urgent | **Bug**: W-13811293 | **Cases**: 44933298

**Symptoms**:
- Change targeting criteria on Ad Placement
- Price does not automatically recalculate
- Stale price remains even after save

**Root Cause**:
- Trigger on Ad_Placement__c not firing on targeting field updates
- Pricing recalculation logic checks wrong fields for changes

**Workaround**:
```apex
// Manual pricing recalculation via Anonymous Apex
List<Ad_Order_Item__c> items = [
  SELECT Id FROM Ad_Order_Item__c WHERE Id = :placementId
];
AdSalesPricingService.recalculatePricing(items);
```

**Code References**:
- `AdOrderItemTriggerHandler.cls` (afterUpdate method)
- `AdSalesPricingService.recalculatePricing()`

**Status**: Bug W-13811293, investigating trigger field detection logic

**Related Cases**: 44933298

---

### PATTERN #8: Check Inventory IP Errors
**Frequency**: MEDIUM | **Severity**: Level 2 Urgent | **Bug**: W-13497563 | **Cases**: 44560855

**Symptoms**:
- Call VPL-CheckInventory Integration Procedure
- Always returns: `Available=true, AvailableUnits=null`
- Should return actual available units from GAM (Google Ad Manager)

**Root Cause**:
- GAM API authentication failing (expired/invalid credentials)
- Response parsing error (null units not handled)

**Resolution**:
```
STEP 1: Re-authenticate GAM Integration
- Setup → Named Credentials → Google_Ad_Manager
- Click "Edit" → "Authenticate"
- Follow OAuth flow to re-authorize

STEP 2: Update Integration Procedure Response Mapping
Integration Procedure: VPL-CheckInventory

AFTER (correct):
Available = IF(%response.availableUnits% > 0, true, false)
AvailableUnits = %response.availableUnits%
ErrorMessage = %response.errors[0].message%
```

**Code References**:
- Integration Procedure: `VPL-CheckInventory`
- Named Credential: `Google_Ad_Manager`
- Apex: `SfiAdsAvailabilityService.cls`

**Related Cases**: 44560855

---

### PATTERN #9: Ad Order Item Units Split Not Populated
**Frequency**: MEDIUM | **Severity**: Level 2 Urgent | **Bug**: W-13798762 | **Cases**: 44814059

**Symptoms**:
- Create and Submit Ad Order
- Ad Order Item Units Split records are empty/not created
- Expected: Units split by flight date automatically populated

**Root Cause**:
- Maintenance job `ASMTSOCoreDataSetup` not run or failed
- Trigger that creates Units Split records disabled

**Resolution**:
```apex
// Run ASMTSOCoreDataSetup Maintenance Job
ASMTSOCoreDataSetup setupJob = new ASMTSOCoreDataSetup();
Database.executeBatch(setupJob, 200);

// Monitor job
SELECT Id, Status, JobItemsProcessed, TotalJobItems
FROM AsyncApexJob
WHERE ApexClass.Name = 'ASMTSOCoreDataSetup'
ORDER BY CreatedDate DESC LIMIT 1
```

**Code References**:
- Batch Job: `ASMTSOCoreDataSetup.cls`
- Object: `Ad_Order_Item_Units_Split__c`

**Related Cases**: 44814059

---

### PATTERN #10: Daily Calendar Off By One Day
**Frequency**: LOW-MEDIUM | **Severity**: Level 2 Urgent | **Bug**: W-13484755 | **Cases**: 44671865

**Symptoms**:
- Ad Space Specification: Set Start/End Week Day (Monday-Friday)
- Should gray out weekend days but grays out Sunday-Monday instead (off by one)

**Root Cause**:
- Day-of-week calculation uses 0-based index (Sunday=0)
- UI component uses 1-based index (Monday=1)
- Mismatch between backend and frontend day indexing

**Code References**:
- LWC: `runtime_media_asm_mediaplan/dailyCalendar.js`
- Object: `Ad_Space_Specification__c` (StartWeekDay__c, EndWeekDay__c)

**Workaround**: Document the off-by-one behavior for users

**Status**: Bug W-13484755, fix targeted for next minor release

**Related Cases**: 44671865

---

## 🛠️ SETUP & CONFIGURATION GUIDE

### Prerequisites
1. **Salesforce Industries Media Cloud License**
   - Required for all Media Cloud users
   - Check: Setup → Company Information → Licenses

2. **Permission Sets**
   - `IndustriesMediaCloudUser` - Base permission set
   - `MediaAdSalesManager` - Full CRUD access
   - `MediaAdSalesRepresentative` - Limited access for sales reps

3. **OmniStudio Licenses**
   - Required for OmniScripts and Integration Procedures

### Initial Setup (12-Step Process)

```
STEP 1: Install Industries Media Cloud Package
- AppExchange: "Salesforce Industries Media Cloud"
- Run Media Cloud Configuration Wizard

STEP 2: Assign Permission Sets
- Users → Add Permission Set Assignments
- Assign: IndustriesMediaCloudUser + MediaAdSalesManager

STEP 3: Configure Ad Sales Settings
- Industries Media Cloud → Ad Sales Configuration
- Enable: Media Plans, Placement Management, Pricing Calculation

STEP 4: Create Ad Space Catalog
- Media Business App → Ad Spaces
- Create spaces for: Digital, Print, Linear TV, Radio

STEP 5: Configure Pricing Rules
- Calculation Procedures → Create "Ad Sales Pricing Procedure"

STEP 6: Set Up Time Slot Objects (TSO)
Execute Anonymous Apex:
ASMTSOCoreDataSetup setupJob = new ASMTSOCoreDataSetup();
Database.executeBatch(setupJob, 200);

STEP 7-8: Activate OmniScripts and Integration Procedures
- MediaPlanCreation, AddPlacement, CheckInventory, SubmitOrder

STEP 9: Set Up GAM Integration (Optional)
- Named Credentials → Google_Ad_Manager (OAuth 2.0)

STEP 10: Create Page Layouts
- Add all required fields to Media_Plan__c layout

STEP 11: Configure Validation Rules
- Start Date Before End Date
- Budget Must Be Positive

STEP 12: Test End-to-End Flow
- Create Media Plan → Add Placement → Submit Order
```

### Common Configuration Issues Table

| Issue | Cause | Fix |
|-------|-------|-----|
| "Add Placement" button not visible | Missing Apex class permission | Grant access to MediaAdSalesAppHandler |
| OmniScript not loading | OmniStudio license issue | Verify license assigned |
| Pricing not calculating | Calculation Procedure not mapped | Map to Ad_Order_Item__c |
| Inventory check returns null | GAM credentials expired | Re-authenticate Named Credential |
| Units Split not populating | ASMTSOCoreDataSetup not run | Execute batch job |

---


---

## MANDATORY DATA HANDLING RULES

> **These rules apply to every action taken by this skill. Read before proceeding.**

### Required Licenses
1. **Salesforce Industries Media Cloud** (SKU: INDMC-MEDIA)
2. **OmniStudio** (for OmniScripts and Integration Procedures)
3. **Optional**: Industries Media Cloud - Ad Sales Trust

### Permission Sets Matrix

| Permission Set | Purpose | CRUD on Objects |
|----------------|---------|-----------------|
| IndustriesMediaCloudUser | Base access | Read all |
| MediaAdSalesManager | Full management | Full CRUD |
| MediaAdSalesRepresentative | Sales operations | Create/Edit Plans |
| OmniStudioUser | Execute IPs | N/A |

### Critical Apex Classes to Grant Access
- MediaAdSalesAppHandler
- MediaAdSalesUtil
- MediaAdSalesDigitalService
- AdSalesPricingService
- SfiAdsAvailabilityService

### Integration Procedure Access (Important for Portal Users)
- sfiAds-DigitalHandler
- sfiAds-DigitalCloneLineItems
- VPL-CheckInventory

**Note**: Portal users need explicit "Permitted Profiles" configuration in Integration Procedures.

---

## 🚨 ESCALATION PATHS

### When to Escalate

**Escalate to Dev Team (L3) when**:
- PATTERN #3 (CalculationProcedure query error) - P1 escalation
- Code bug requiring patch/hotfix
- Governor limits exceeded in product code
- GAM integration consistently failing
- Data corruption identified

**Escalation Channels**:
1. **Slack**: #ad_sales_management_tech_group (tag @media-adsales-dev-oncall)
2. **GUS**: Work Type = Bug, Scrum Team = Media Ad Sales
3. **Weekly Office Hours**: Wednesdays 10 AM PST

### Escalation Template

```
**Case Summary**: [One-line description]
**Customer Org ID**: [15/18 char]
**Issue Pattern**: [Pattern # from this skill]
**Symptoms**: [What user experiences]
**Reproduction Steps**: [1, 2, 3...]
**Debug Logs**: [Attached/Link]
**Root Cause**: [Hypothesis]
**Customer Impact**: [Blocking production?]
**Urgency**: [P1/P2/P3]
```

---

## 💬 TRIBAL KNOWLEDGE FROM SLACK

### Key Q&A

**Q: Why does "Add Placement" fail for portal users but work for internal users?**
A: Integration Procedures require explicit permission. Add portal profile to "Permitted Profiles" in IP definition. Apex class access alone is not sufficient.

**Q: When should I run ASMTSOCoreDataSetup batch job?**
A: Run after initial install, after enabling Ad Sales features, or if Units Split not auto-populating. Recommended: Schedule weekly (Sundays 2 AM).

**Q: How do I troubleshoot "SOQL 101 query limit" errors?**
A: Common in Ad Sales triggers. Check AdOrderItemTriggerHandler for queries in loops. Solution: Bulkify code to use single query for all items.

**Q: What's the difference between Media Plan and Ad Order?**
A:
- **Media Plan**: Pre-sales artifact for planning/quoting
- **Ad Order**: Post-sales artifact for sold/committed inventory
- Workflow: Media Plan → Approval → Convert to Ad Order → Fulfillment

**Q: Best practice for ad space hierarchy?**
A:
```
Ad_Space__c (Website Home Page - Top Banner)
  ├─ Ad_Space_Specification__c (300x250 pixels)
  ├─ Ad_Space_Specification__c (728x90 pixels)
  └─ Ad_Space_Specification__c (300x600 pixels)
```

---

## 🗺️ CODE REFERENCE MAP

### Key Apex Classes (from 430+ codebase)

**Core Handlers**:
- `MediaAdSalesAppHandler.cls` - Main entry point, routes to media-type handlers
- `MediaAdSalesUtil.cls` - Common utility methods
- `MediaAdSalesException.cls` - Custom exception handling

**Media Type Handlers**:
- `MediaAdSalesLinearTVHandler.cls` - Linear TV placement logic
- `MediaAdSalesDigitalService.cls` - Digital/banner/video ads
- `MediaAdSalesPrintService.cls` - Print (magazine, newspaper)
- `MediaAdSalesRadioService.cls` - Radio spot ads

**Pricing & Calculation**:
- `AdSalesPricingService.cls` - Main pricing engine
- `AdSalesPricingPlanHelper.cls` - ⚠️ Contains bug in PATTERN #3
- `AdSalesPricingCalculationJob.cls` - Queueable job for async pricing

**Inventory Management**:
- `SfiAdsAvailabilityService.cls` - Check inventory availability
- `SfiAdsMediaApiQueryUtil.cls` - Query utilities

**Order Processing**:
- `AdOrderItemTriggerHandler.cls` - Trigger logic
- `DefaultAdOrderItemServiceImplementation.cls` - Service implementation
- `AdOrderItemUnitsSplitHelper.cls` - Create Units Split records

**Batch Jobs**:
- `ASMTSOCoreDataSetup.cls` - Setup Time Slot Objects (critical for PATTERN #9)

### Integration Procedures
- `sfiAds-DigitalHandler` - Handle digital placements
- `sfiAds-DigitalCloneLineItems` - ⚠️ Bug in PATTERN #5
- `VPL-CheckInventory` - ⚠️ Bug in PATTERN #8

### LWC Components (Namespace: runtime_media_asm_mediaplan)
- `typeSelector` - ⚠️ Error in PATTERN #6
- `adSpaceSelector` - ⚠️ Display bug in PATTERN #4
- `dailyCalendar` - ⚠️ Off-by-one in PATTERN #10
- `placementForm` - Placement detail entry
- `pricingCalculator` - Pricing calculation and display

### Custom Objects
- `Media_Plan__c` - Pre-sales planning
- `Ad_Placement__c` - Placement within plan
- `Ad_Order__c` - Sold/committed order
- `Ad_Order_Item__c` - Line item in order
- `Ad_Order_Item_Units_Split__c` - Daily unit breakdown
- `Ad_Space__c` - Inventory definition
- `Ad_Space_Specification__c` - Size/format variant
- `CalculationProcedure__c` - Pricing rules (⚠️ PATTERN #3)
- `TimeSlotObject__c` - Daily calendar slots

---

## 🐛 ACTIVE KNOWN ISSUES (May 2026)

| Bug ID | Title | Severity | Status | Workaround? |
|--------|-------|----------|--------|-------------|
| W-12964525 | Undefined Ad Space Size Display | P2 | In Progress | Yes (PATTERN #4) |
| W-13853569 | Rate Type Missing on Clone | P3 | Prioritized | Yes (PATTERN #5) |
| W-13811293 | Pricing Recalculation Not Triggered | P2 | Investigating | Yes (PATTERN #7) |
| W-13497563 | Check Inventory IP Errors | P2 | Pending GAM Team | Yes (PATTERN #8) |
| W-13798762 | Units Split Not Populated | P2 | Doc Updated | Yes (PATTERN #9) |
| W-13484755 | Daily Calendar Off By One Day | P2 | Fix in Review | Yes (PATTERN #10) |
| [Pending] | CalculationProcedure Query Error | P1 | Not Yet Filed | No - Code fix required (PATTERN #3) |

### Release Notes Watch

**Summer '26 Release** (expected June 2026):
- Fix for Rate Type on Clone (W-13853569)
- Fix for Daily Calendar day indexing (W-13484755)
- Enhanced error messages for "Add Placement" failures

---

## 📊 DEBUGGING TECHNIQUES

### Debug Log Analysis

**Key Categories to Monitor**:
- `SOQL_EXECUTE_BEGIN/END` - Check query count (limit 101)
- `EXCEPTION_THROWN` - Obvious errors
- `CODE_UNIT_STARTED` - Trace execution path

**Common Error Patterns**:
```
System.QueryException: sObject type 'CalculationProcedure' is not supported
→ See PATTERN #3

System.LimitException: Too many SOQL queries: 101
→ Trigger bulkification issue

System.NullPointerException
→ Check null guards in LWC or Apex
```

### Anonymous Apex Scripts

**Check User Permissions**:
```apex
List<PermissionSetAssignment> perms = [
  SELECT PermissionSet.Name
  FROM PermissionSetAssignment
  WHERE AssigneeId = :UserInfo.getUserId()
];
System.debug('Permission Sets: ' + perms);
```

**Test Pricing Calculation**:
```apex
List<Ad_Order_Item__c> items = [
  SELECT Id FROM Ad_Order_Item__c WHERE Id = :itemId
];
AdSalesPricingService.recalculatePricing(items);
```

---

## 🔄 COMMON WORKFLOWS

### Workflow 1: Create Media Plan → Submit Order

```
1. Create Media_Plan__c (Name, MediaType, Dates, Budget)
2. Add Placements → See PATTERN #1 if button doesn't work
3. Calculate Pricing → See PATTERN #3 if CalculationProcedure error
4. Check Inventory (optional) → See PATTERN #8 if failing
5. Submit for Approval
6. Convert to Ad_Order__c → See PATTERN #9 if Units Split not created
7. Verify: Ad_Order_Item__c and Units Split records exist
```

### Workflow 2: Clone Media Plan

```
1. Navigate to Media_Plan__c
2. Click "Clone"
3. System calls sfiAds-DigitalCloneLineItems → See PATTERN #5 if Rate Type blank
4. Review and modify cloned plan
5. Submit per standard workflow
```

---

## 📚 ADDITIONAL RESOURCES

### Documentation
- [Industries Media Cloud Documentation](https://help.salesforce.com/s/articleView?id=sf.industry_media_cloud.htm)
- [Media Ad Sales Management Admin Guide](https://help.salesforce.com/s/articleView?id=sf.media_adsales_admin.htm)

### Slack Channels (Internal)
- #ad_sales_management_tech_group - Primary technical channel
- #industries-media-cloud - General questions
- #omnistudio-help - OmniStudio questions

---

## 🎯 QUICK REFERENCE CARD

### Top 5 Most Common Issues

1. **"Add Placement" not working** → Check Apex/IP permissions (PATTERN #1)
2. **CalculationProcedure query error** → P1 code bug, escalate (PATTERN #3)
3. **Pricing not recalculating** → Manual recalculate (PATTERN #7)
4. **Units Split empty** → Run ASMTSOCoreDataSetup (PATTERN #9)
5. **"Next" button greyed** → Check required fields (PATTERN #2)

### Essential Commands

**Run Time Slot Setup**:
```apex
ASMTSOCoreDataSetup job = new ASMTSOCoreDataSetup();
Database.executeBatch(job, 200);
```

**Recalculate Pricing**:
```apex
AdSalesPricingService.recalculatePricing([SELECT Id FROM Ad_Order_Item__c WHERE Id = :itemId]);
```

---

## ✅ CASE RESOLUTION CHECKLIST

Before closing a case:

- [ ] Root cause identified and documented
- [ ] Resolution applied and tested
- [ ] Customer confirmed fix works
- [ ] Debug logs reviewed
- [ ] If code bug: GUS bug filed
- [ ] If new pattern: Shared in Slack
- [ ] Case comments: Symptom → Diagnosis → Root Cause → Resolution

---

**END OF SKILL**

*Maintained by: Industries Media Cloud Support Team*  
*Last updated: May 29, 2026*  
*Version: 1.0.0*  
*Data sources: GUS investigations (W-tickets), OrgCS cases (2-3 years), via_cmex-260.9 codebase (430+ classes)*