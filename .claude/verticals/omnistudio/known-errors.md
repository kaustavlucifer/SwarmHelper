# Known Error Patterns — Quick Resolution Guide

Check this FIRST before deep investigation. If the error matches a pattern here, provide the resolution immediately.

---

## OmniStudio: OmniScript Errors

| Error / Symptom | Root Cause | Fix |
|---|---|---|
| "This OmniScript requires re-compilation" | Deployment or LWC state mismatch | Deactivate → Reactivate in OmniScript Designer |
| `Value too long for field: Source maximum length is: 131072` | LWC bundle exceeds 128KB | Reduce OmniScript complexity; consider Standard Runtime |
| `Can not get instance of omnistudiocore.IPService` | Element named "SaveForLater" — internal name collision | Rename the SaveForLater element |
| Blank screen post-upgrade | State corruption after package upgrade | Clone with new Type/SubType (case-derived; W-11941259 — confirm symptoms match) |
| OmniScript not loading on Experience site | OmniProcess Read access missing | Add OmniProcess Read to community profile/PS |
| LWC activation fails post-deployment | DB lock during Puppeteer compilation | Use `useBulkOmniScriptLwcCompile` VBT parameter (W-19473960) |
| Child OmniScript "Cannot read properties of undefined" | Multi-tab designer cache over-clearing | Avoid multiple designer tabs in same session (W-22469676, fixed 264) |
| NPM compiler 401 Unauthorized | NPM token expired | Insert `new Document(Name='OmniScript URL Document Do Not Delete', ...)` workaround |
| `No MODULE named markup://c:omniscriptBaseMixin` | Namespace mismatch (Core vs package) | Use correct namespace: `vlocity_ins/utility` (pkg) vs `omnistudio/utility` (std) |
| Reusable OmniScript fails with Standard Runtime | Core security check issue with IP/DR in reusable scripts | Disable Standard Runtime (W-12061569, W-12071523) |

---

## OmniStudio: Integration Procedure Errors

| Error / Symptom | Root Cause | Fix |
|---|---|---|
| "You have uncommitted work pending" `System.CalloutException` | DML before HTTP callout in same transaction | Reorder: HTTP Action BEFORE DataRaptor Save/DML steps |
| `retryCount` node causes BigDecimal error | Type cast bug in IP HTTP action (W-18571605) | Fixed in 258; remove retryCount node as workaround |
| "No Procedure Found with Id Or Integration Procedure Is Inactive" | IP is inactive or name/type mismatch | Activate IP; verify name and type match invocation |
| IP Designer truncated in Chrome (not Incognito) | Edge CDN cached truncated response | Hard reload Chrome or clear Edge pod cache |
| Null error creating new versions | Version mismatch (W-21248701) | Rebuild CME/INS packages; check version compatibility |
| IP response not applied (Invoke Mode = Non-Blocking) | Non-blocking mode doesn't apply response | Use `VlocityNoRootNode` as Response JSON Node |
| `LoggingEnabled` causing severe performance degradation | OIC flag `LoggingEnabled` = true | Set to false in Omni Interaction Configuration (Setup) |

---

## OmniStudio: DataRaptor Errors

| Error / Symptom | Root Cause | Fix |
|---|---|---|
| "Specify a valid bundle name for the Data Mapper and try again" | `IsActive` = false on OmniDataTransform | Set IsActive=true: `SELECT Id, Name FROM OmniDataTransform WHERE IsActive=false` |
| DataRaptor "only works briefly after preview" | OWD for `DRMapItems__c` set to Private | Set `DRMapItems__c` OWD to Public Read Only (W-12194504) |
| "No valid SOQL results" on external objects | DataRaptor doesn't support External Objects (`__x`) | Use Apex Remote Action instead |
| "No valid data returned. Check that Mappings have valid field names" | Field API names incorrect or wrong object | Verify API names in mapping against org schema |
| `Too Many Query Rows: 50001` | Upsert key triggers full-table scan | Uncheck upsert key on the mapping (W-12208324) |
| DataRaptor not returning hidden fields | `VlocityOverrideImplementation.apxc` stripping fields | Check for this class override (W-12788249) |
| `System.SObjectException: Field X is not editable` in DR Load | Formula field, system field, or FLS restriction | Check `getObjectSchema` for `updateable: false`; remove non-writable fields |
| Fields with FLS failing silently | `EnforceDMFLSAndDataEncryption` OIC flag enabled | Check user FLS for all mapped fields, or disable flag |
| `visualforce remoting exception... DRMapperControllerFoundation` | OmniStudio + legacy vlocity package coexistence | W-15899933; separate packages or upgrade |

---

## OmniStudio: FlexCard Errors

| Error / Symptom | Root Cause | Fix |
|---|---|---|
| FlexCard not rendering for community/partner users | Missing PSL, PS, or FLS | Add OmniStudioExperienceCloudUser PSL + Experience Cloud User PS + check FLS |
| `NullPointerException` at `FlexCardController.getflexCardMetadata: line 800` | Post-sandbox-refresh migration incomplete | Complete standard objects migration (W-13462944, fixed 246) |
| `NullPointerException... OmniInteractionAccessConfigService.getEntityInfo()` | File-based Apex gater changes after 254 | Check GUS for W-item fix (HTTP 400 INVALID_API_INPUT) |
| `NullPointerException` at `FlexcardLwcTemplateBuilder.buildCustomLwcElementTemplate` | Child card with custom LWC regression in 242.14 | Fixed in 242.15 (W-a07EE00001NwavJYAR) |
| `Expected ':' after property name in JSON` from `carddesignerUtility.js` | Corrupt FlexCard JSON metadata | File swarm request for engineering investigation |
| `OmnistudioLwcUtilityImpl.deployment failed` | FlexCard activation failure with actions referencing OmniScripts on INS/INSFSC packages pushed from 254 | Check package version compatibility; file swarm |
| FlexCard Action Datasource Variable Resolution not working | Variables not resolving in datasource (W-20966573) | Check GUS W-20966573 for fix release |
| FlexCard parameter corruption with brackets and merge fields | Aggressive regex in `setParams.js` (W-19614666) | Fixed in later release; avoid brackets in merge field paths |

---

## OmniStudio: General / Permissions Errors

| Error / Symptom | Root Cause | Fix |
|---|---|---|
| "An unknown error stopped us from finishing your OmniStudio data migration" | OmniStudio Metadata enablement/migration failure | Contact Salesforce Support (internal escalation) |
| `Entity type 'OmniIntegrationProcedure' is not available in this api version` | API version < 57 | Use API version 57+ (recommended: 60) |
| `OmniScript__c` vs `OmniProcess` confusion | Wrong object for the runtime type | Determine runtime first; use correct object |
| `bulkDml: received EntityObjects that do not refer to current UddInfo` | OmniStudio components deployed with core metadata | Deploy OmniStudio in a separate deployment pass (VBT/IDX) |
| `omniscriptIPAction` LWC not working | Namespace conflict (standard vs package) | Use correct component name for runtime type |

---

## Revenue Cloud / CPQ Errors

| Error / Symptom | Root Cause | Fix |
|---|---|---|
| "Too many SOQL queries: 101" | Queries inside loops (N+1 pattern) | Query ONCE before loop, use Map for lookup |
| "UNABLE_TO_LOCK_ROW" | Two transactions updating same record | Retry; check for background jobs on same records |
| "FIELD_CUSTOM_VALIDATION_EXCEPTION" | Validation rule blocking save | Check debug log for `VALIDATION_FAIL` → find rule in Setup |

---

## General Platform Errors

| Error / Symptom | Root Cause | Fix |
|---|---|---|
| "Apex CPU time limit exceeded" | 10s sync / 60s async CPU limit hit | Find hotspot in debug log; reduce triggers/flows stacking; bulkify |
| "Field X is not editable" | Formula field, system field, or FLS | Check `getObjectSchema` for `updateable: false`; check user FLS |
| "Invalid sobject provided. Schema.describeSObject() does not support..." | Object doesn't exist or double namespace prefix | Check if object exists; fix double namespace (`vlocity_cmt__vlocity_cmt__...`) |
| "Future method cannot be called from a future or batch method" | Chaining @future calls | Refactor to Queueable (can be chained) |
| "INVALID_SESSION_ID" | Session expired | Re-login; check session timeout settings |
| "REQUEST_LIMIT_EXCEEDED" | API request limit hit | Check API usage in Setup; use Bulk API for high-volume |

---

## Debug Log Red Flags

| Pattern | Meaning | Action |
|---|---|---|
| Same exception repeating N times | Loop failing on each record | Fix root cause, don't suppress |
| SOQL inside a loop (duplicates) | N+1 query anti-pattern | Query before loop, use Map |
| >90% of any governor limit | Will break on larger data | Optimize before production break |
| DML before callout | Transaction order violation | Callout first, DML after |
| Double namespace prefix | Config/code error | Fix source of double prefix |
| "Field not editable" on system fields | Code copying all fields including read-only | Filter out non-writable fields |
