## A. Triage & Classification

When a support engineer brings a Health Cloud issue, first identify the product area:

```
Is the issue about...

├── CARE MANAGEMENT
│   ├── Care Plan not creating / missing data
│   ├── Care Plan Goal/Problem not linking
│   ├── Care Gap identification not triggering
│   ├── Care Plan IP (Integration Procedure) failing
│   └── Care Team assignment errors
│   → Go to: Pattern 1-2

├── ASSESSMENTS
│   ├── Assessment not saving responses
│   ├── Assessment questions not rendering
│   ├── Assessment scoring calculation wrong
│   ├── Assessment flow/OmniScript errors
│   └── Assessment FLS/permission errors
│   → Go to: Pattern 3

├── UTILIZATION MANAGEMENT
│   ├── Authorization request failing
│   ├── Prior authorization (CarePreauth) errors
│   ├── Benefit verification callout failing
│   ├── UM decision not updating
│   └── Review cycle not progressing
│   → Go to: Pattern 4-5

├── PROVIDER NETWORK
│   ├── Provider search not returning results
│   ├── Provider credentialing errors
│   ├── Network configuration issues
│   ├── Provider-facility mapping wrong
│   └── Taxonomy/specialty not loading
│   → Go to: Pattern 6-7

├── FHIR / HL7 INTEGRATION
│   ├── FHIR callout failing (timeout, 401, 500)
│   ├── FHIR resource mapping errors
│   ├── HL7 message parsing failure
│   ├── External service configuration
│   └── Named Credential issues
│   → Go to: Pattern 8

├── INTELLIGENT APPOINTMENT MANAGEMENT (IAM)
│   ├── Appointment scheduling errors
│   ├── IAM setup verification failures
│   ├── Group scheduling configuration
│   ├── Appointment Guidance Flow issues
│   └── Provider availability not showing
│   → Go to: Pattern 9

├── ENROLLMENT & MEMBERSHIP
│   ├── Member enrollment failing
│   ├── Census data errors
│   ├── MemberPlan not creating
│   └── InsuranceCoverage linking issues
│   → Go to: Pattern 10

└── ACCOUNT PROVISIONING / DORA / GENERAL
    ├── HC Setup Troubleshooter issues
    ├── Permission / license errors
    ├── Person Account configuration
    └── HC-specific OmniScript/IP errors
    → Go to: Pattern 11-12
```

---

## B. Known Issue Patterns

---

### Pattern 1: Care Plan Integration Procedure Failing

**Frequency:** High
**Severity:** P11–P21

**Symptoms:**
- Care Plan creation IP returns error or blank screen
- `InsuranceClaimServicePtc.apex` referenced in stack trace
- IP step fails on DataRaptor extract or save
- "Uncommitted work pending" error in Care Plan flow

**Root Cause:**
HC Care Plans use Integration Procedures that invoke Insurance-layer PTC code (shared payer infrastructure). Common causes:
- OIC flag mismatch between HC and Insurance configurations
- DML before callout in IP step ordering
- PTC layer change broke HC-specific flow

**Resolution:**
1. Check OIC flags: `SELECT Id, DeveloperName, Value FROM OmniInteractionConfig__mdt WHERE DeveloperName = 'OmniStudio'`
2. Review IP step order — ensure HTTP callouts precede DataRaptor Post actions
3. Check PTC layer for recent changes: `core/industries-interaction-ptc/apex/vlocity_ins/InsuranceClaimServicePtc.apex`
4. Verify FLS on Care Plan objects for running user

**Splunk Query:**
```spl
index=<POD> organizationId=<ORG_15> logRecordType=ipipr "CarePlan" earliest=-7d
| head 50
```

**Code Path:** `InsuranceClaimServicePtc.apex` → `IntegrationProcedureService` → Care Plan IP

---

### Pattern 2: Care Gap Not Triggering / Missing Care Gaps

**Frequency:** Medium
**Severity:** P21–P31

**Symptoms:**
- Care gaps not appearing on patient record
- Care gap calculation not running
- Gaps show as "Met" when they should be "Open"
- Historical care gap data missing

**Root Cause:**
Care gaps are calculated based on clinical data and rules. Common causes:
- Care gap definition criteria not matching patient data
- Batch job for gap calculation not running or failing
- Clinical data not properly linked to patient record
- Care gap evaluation rules misconfigured

**Resolution:**
1. Verify care gap definitions and criteria
2. Check batch job status for gap calculation
3. Verify clinical data is linked to correct patient (HealthCondition records)
4. Review care gap evaluation rules and effective dates
5. Check permissions on `CareGap` object

**Splunk Query:**
```spl
index=<POD> organizationId=<ORG_15> "CareGap" (level=error OR logRecordType=axerr) earliest=-7d
| head 20
```

---

### Pattern 3: Assessment Not Saving / FLS Errors

**Frequency:** High
**Severity:** P21–P31

**Symptoms:**
- Assessment responses not persisting after submit
- `NoAccessException` or FLS error on AssessmentResponse
- Assessment OmniScript renders but fails on save
- `AssessmentResponsesPtc.apex` in error trace

**Root Cause:**
Health Cloud assessments use OmniStudio (OmniScripts/IPs) with PTC layer. Common causes:
- FLS not granted on `AssessmentQuestion`, `AssessmentResponse` objects
- `EnforceDMFLSAndDataEncryption` OIC flag is TRUE but permissions not updated
- PTC layer change in `AssessmentResponsesPtc.apex`
- Assessment DataRaptor save step misconfigured

**Resolution:**
1. Check FLS: Verify running user has Read/Create/Edit on `AssessmentQuestion`, `AssessmentResponse`, `AssessmentQuestionResponse`
2. Check OIC flag: `EnforceDMFLSAndDataEncryption` — if TRUE, all field-level permissions must be explicit
3. Check PTC changes:
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_ins/AssessmentResponsesPtc.apex"
```
4. Verify DataRaptor save step maps to correct fields

**Splunk Query:**
```spl
index=<POD> organizationId=<ORG_15> "Assessment" ("NoAccess" OR "FLS" OR "NoAccessException") earliest=-7d
| head 20
```

**Code Path:** `AssessmentResponsesPtc.apex` → OmniScript → DataRaptor Save

---

### Pattern 4: Benefit Verification Callout Failing

**Frequency:** Medium
**Severity:** P11–P21

**Symptoms:**
- Benefit verification returns error or timeout
- `InsuranceRatingPtc.apex` referenced in error
- Named Credential authentication failure (401)
- "You have uncommitted work pending" error
- External payer system not responding

**Root Cause:**
Benefit verification uses Insurance-layer rating PTC to call external payer systems. Common causes:
- Named Credential expired or misconfigured
- External endpoint down or timeout
- DML before callout in verification flow
- Callout timeout too short for external system response time

**Resolution:**
1. Check Named Credential: Setup → Named Credentials → verify auth status
2. Test endpoint directly (if accessible)
3. Check callout timeout configuration
4. Review IP step order — callouts must precede DML operations
5. Check for concurrent callout limits

**Splunk Query:**
```spl
index=<POD> organizationId=<ORG_15> "CalloutException" ("InsuranceRating" OR "BenefitVerification") earliest=-7d
| head 20
```

**Code Path:** `InsuranceRatingPtc.apex` → Named Credential → External Payer System

---

### Pattern 5: Utilization Management Authorization Errors

**Frequency:** Medium
**Severity:** P21–P31

**Symptoms:**
- Authorization request stuck in status
- Prior authorization (CarePreauth) not creating
- UM decision update failing
- Review cycle not advancing to next step

**Root Cause:**
UM uses state machine transitions (shared with Insurance via `StateTransitionService`). Common causes:
- Invalid state transition attempted
- Required fields missing for target state
- Trigger/validation rule blocking state change
- Concurrent modification conflict

**Resolution:**
1. Check current authorization state and valid transitions
2. Verify all required fields for the target state are populated
3. Review validation rules on `AuthorizationRequest` / `CarePreauth`
4. Check for record lock (concurrent modification)
5. Review state model configuration: `VlocityStateModel__c`

**Splunk Query:**
```spl
index=<POD> organizationId=<ORG_15> ("AuthorizationRequest" OR "CarePreauth") (level=error OR logRecordType=axerr) earliest=-7d
| head 20
```

---

### Pattern 6: Provider Search Not Returning Results

**Frequency:** Medium
**Severity:** P31–P45

**Symptoms:**
- Provider search returns empty results
- Search criteria not matching known providers
- Search performance very slow
- Filter options not appearing

**Root Cause:**
Provider search uses Criteria-Based Search framework. Common causes:
- Criteria-Based Search permission set not assigned
- Search configuration (custom metadata) incomplete
- Provider records missing required indexed fields
- Search filter criteria too restrictive

**Resolution:**
1. Verify `Use Criteria-Based Search and Filter` permission set is assigned
2. Check custom metadata for search configuration
3. Verify provider records have required fields populated (Name, Type, Specialty, Location)
4. Test with broadened search criteria
5. Check ProviderSearch__c and ProviderTypeSync__c configuration

---

### Pattern 7: Provider Network / Credentialing Configuration Issues

**Frequency:** Low-Medium
**Severity:** P31–P45

**Symptoms:**
- Provider not showing in network
- Credentialing status not updating
- Provider-facility mapping incorrect
- Taxonomy codes not loading

**Root Cause:**
Provider network configuration involves multiple related objects and data sources. Common causes:
- Provider relationship objects not properly linked
- Credentialing workflow not configured
- Taxonomy data not loaded (HealthcareProviderTaxonomy)
- Network membership criteria not met

**Resolution:**
1. Verify provider record linkage (HealthcareProvider → HealthcareFacility → HealthcareService)
2. Check credentialing workflow and status fields
3. Verify taxonomy data is loaded and mapped
4. Review network membership criteria and effective dates

---

### Pattern 8: FHIR Integration Callout Failures

**Frequency:** Medium
**Severity:** P11–P21

**Symptoms:**
- FHIR API calls returning 401/403/500
- FHIR resource parsing errors
- HL7 message transformation failure
- Named Credential authentication expired
- Timeout on external EHR system

**Root Cause:**
FHIR integrations use Named Credentials and External Services to communicate with EHR systems. Common causes:
- Named Credential OAuth token expired
- FHIR endpoint URL changed
- FHIR R4 schema mismatch (resource structure)
- Network/firewall blocking callout
- EHR system rate limiting

**Resolution:**
1. Check Named Credential status and re-authenticate if needed
2. Verify FHIR endpoint URL is current and accessible
3. Check FHIR resource version compatibility (R4 vs DSTU2)
4. Review External Service configuration
5. Check callout timeout settings
6. Verify IP allowlisting for Salesforce egress IPs

**Splunk Query:**
```spl
index=<POD> organizationId=<ORG_15> "FHIR" ("CalloutException" OR "HttpResponse" OR "timeout") earliest=-7d
| head 20
```

**Code Path:** `core/industries-healthcare-impl/` → External Service → FHIR Endpoint

---

### Pattern 9: Intelligent Appointment Management (IAM) Setup Issues

**Frequency:** Medium
**Severity:** P31–P45

**Symptoms:**
- IAM setup verification shows failures
- Appointment scheduling errors
- Provider availability not displaying
- Group scheduling not working
- Appointment Guidance Flow errors

**Root Cause:**
IAM requires multiple configuration records to be set up correctly. The Health Cloud Troubleshooter diagnostic tool can verify this. Common causes:
- Required records not created (visit types, engagement channels)
- Provider availability not configured
- Permission set not assigned
- Scheduling objects (shifts, service appointments) misconfigured

**Resolution:**
1. Run **Health Cloud Troubleshooter** (Setup → Health Cloud Troubleshooter)
2. Verify all required records per troubleshooter output
3. Check permission sets for scheduling objects
4. Verify provider availability configuration
5. For Group Scheduling: review considerations (specific limitations apply)
6. For Appointment Guidance Flow: verify visit type and channel setup

---

### Pattern 10: Member Enrollment / Census Errors

**Frequency:** Medium
**Severity:** P21–P31

**Symptoms:**
- Member enrollment failing silently or with error
- Census data not creating member records
- `InsEnrollmentServicePtc.apex` in error trace
- `InsCensusServicePtc.apex` data format errors
- MemberPlan records not linking to coverage

**Root Cause:**
Enrollment uses Insurance-layer services (shared payer/carrier infrastructure). Common causes:
- Census data format mismatch (expected vs provided)
- Missing required fields for enrollment
- FLS/CRUD on MemberPlan, InsuranceCoverage objects
- PTC layer change broke enrollment flow

**Resolution:**
1. Check PTC layer:
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_ins/InsEnrollmentServicePtc.apex"
```
2. Verify census data format matches expected schema
3. Check FLS on enrollment objects for running user
4. Review enrollment IP configuration

**Splunk Query:**
```spl
index=<POD> organizationId=<ORG_15> ("InsEnrollmentService" OR "InsCensusService") (level=error OR logRecordType=axerr) earliest=-7d
| head 20
```

**Code Path:** `InsEnrollmentServicePtc.apex` / `InsCensusServicePtc.apex` → via_platform → MemberPlan

---

### Pattern 11: DataRaptor Missing Fields (FLS Enforcement)

**Frequency:** High
**Severity:** P21–P45

**Symptoms:**
- DataRaptor extract returns partial data (fields missing)
- DataRaptor save fails with "field not editable" or silent field drop
- Works for admin but not for standard HC users
- `EnforceDMFLSAndDataEncryption` OIC flag was recently changed

**Root Cause:**
When `EnforceDMFLSAndDataEncryption` OIC flag is TRUE, DataRaptor enforces field-level security on every field accessed. Missing FLS = field silently dropped from results.

**Resolution:**
1. Check OIC flag value: `SELECT Value FROM OmniInteractionConfig__mdt WHERE DeveloperName = 'OmniStudio'` (look for `EnforceDMFLSAndDataEncryption`)
2. If TRUE: grant FLS on ALL fields the DataRaptor accesses to the running user's profile/permission set
3. If recently changed from FALSE to TRUE: identify all DR fields and grant FLS
4. Check `CheckFieldLevelSecurity__c` custom setting (legacy fallback)
5. Use `isSelectableFieldCachedMap` method trace to identify which fields are being filtered

**Splunk Query:**
```spl
index=<POD> organizationId=<ORG_15> logRecordType=ipdar "EnforceDMFLS" earliest=-7d
| head 50
```

**Code Path:** `DREngine.cls` → `CheckFieldLevelSecurity` → FLS enforcement → silent field filtering

---

### Pattern 12: HC Permission / License Errors

**Frequency:** Medium
**Severity:** P31–P45

**Symptoms:**
- "Insufficient Privileges" on HC objects
- HC features not visible in Setup
- HC permission set license not available
- HC components not rendering

**Root Cause:**
Health Cloud requires specific permission set licenses and permission sets. Common causes:
- HC PSL not provisioned to org
- HC permission sets not assigned to user
- Record type access missing
- HC features require specific editions (Enterprise/Unlimited)

**Resolution:**
1. Verify HC PSL is available: Setup → Company Information → Permission Set Licenses
2. Assign required HC permission sets: `HealthCloudFoundation`, `HealthCloudPermissionSetLicense`
3. Check record type access on HC objects
4. Verify edition supports HC features
5. For Experience Cloud: check guest user permissions separately
