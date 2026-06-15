---
name: public-sector-known-patterns
description: Public Sector Solutions diagnostic patterns — symptom to subsystem to GUS search, grounded in verified core paths and the Vlocity Public Sector product tag.
---

# Public Sector Solutions — Diagnostic Patterns

> Symptom → likely subsystem → how to confirm → GUS search. Patterns are leads; confirm root cause against the current case (Phase 4/5). Release branches use {CURRENT_GA} — resolve per capabilities/codesearch.md.

> **GUS tag note:** The managed-package (`vlocity_ps`) product tag is **`Vlocity Public Sector`** (the string `Public Sector Solutions` returns 0 bugs — do not use it). Newer core LPI/Benefits work may be tagged differently; if a tag search returns nothing, widen with `Product_Tag__r.Name LIKE '%Public Sector%'` and inspect results. Apply the build-staleness rule from capabilities/gus.md (a fix in `Scheduled_Build__r.Name` ≤ the org's build means it should already be present).

## Pattern: License/permit application errors or wrong field rendering
- **Likely subsystem:** `core/ui-industries-public-sector-components/java/src/ui/publicsector/components/controller/LicensePermitController.java` (LPI), backed by OmniStudio assets under `.../omnistudio/`
- **How to confirm:** Reproduce the OmniScript/IP step; check OmniStudio runtime logs (`ipipr`/`ipdar`); verify the LicensePermit component config and RegulatoryAuthorizationType setup
- **GUS search:** `Product_Tag__r.Name LIKE '%Vlocity Public Sector%' AND Subject__c LIKE '%license%'` (also try `%permit%`; apply build-staleness rule)

## Pattern: Permit dependency / prerequisite permit not enforced
- **Likely subsystem:** `.../components/controller/PermitDependencyController.java`
- **How to confirm:** Check the permit-dependency configuration on the RegulatoryAuthorizationType; confirm the controller resolves prerequisite records; check for entity-access (`NoAccessException`) on the dependent objects
- **GUS search:** `Product_Tag__r.Name LIKE '%Vlocity Public Sector%' AND Subject__c LIKE '%dependency%'` (also try `%prerequisite%`; apply build-staleness rule)

## Pattern: Inspection history empty / inspection not creating from license
- **Likely subsystem:** `.../components/controller/InspectionHistoryController.java` and `InspectionTaskListController.java`
- **How to confirm:** Controller returns a "missing entity access" result when the user lacks access (`NoAccessException` on `AccountContactRelation`/inspection objects) — verify FLS/sharing first; then check InspectionType config and the RegulatoryAuthorizationId linkage
- **GUS search:** `Product_Tag__r.Name LIKE '%Vlocity Public Sector%' AND Subject__c LIKE '%inspection%'` (apply build-staleness rule)

## Pattern: Government case management / case workflow failure
- **Likely subsystem:** `.../components/controller/CaseManagementController.java` (annotated `@EscalationPermSet`)
- **How to confirm:** Check permission-set escalation state; verify case record type and the gov case workflow; check `axerr` for Apex exceptions and `gslog` for platform Java exceptions
- **GUS search:** `Product_Tag__r.Name LIKE '%Vlocity Public Sector%' AND Subject__c LIKE '%case%'` (apply build-staleness rule)

## Pattern: Benefit disbursement not processing / benefit settings missing
- **Likely subsystem:** `.../components/controller/BenefitDisbursementController.java` and `CarePlanBenefitSettingsController.java` (invocable-action backed)
- **How to confirm:** Trace the invocable action the controller calls; verify CarePlan benefit settings config; for recertification check the `BMRecertEvent__e` platform-event subscription
- **GUS search:** `Product_Tag__r.Name LIKE '%Vlocity Public Sector%' AND Subject__c LIKE '%benefit%'` (also try `%disbursement%`; apply build-staleness rule)

## Pattern: Provider search / referral authorization not working
- **Likely subsystem:** `.../components/controller/ProviderManagementController.java`, `RequestAuthorizationReferralController.java`, `AuthorizeReferralController.java`, `CriteriaBasedSearchAndFilterWrapperController.java`
- **How to confirm:** These controllers are gated by access guards (e.g. `PublicSector.userHasProviderManagement`) — verify the user holds the required permission set first; then check the Criteria-Based Search config and filter setup
- **GUS search:** `Product_Tag__r.Name LIKE '%Vlocity Public Sector%' AND Subject__c LIKE '%provider%'` (also try `%referral%`; apply build-staleness rule)

## Pattern: Assessment / OmniAssessment runtime error
- **Likely subsystem:** `.../components/controller/OmniAssessmentRuntimeController.java` (`@EscalationPermSet`); Expression Set / Decision Table location providers (`PublicSectorFileBasedExpressionSetLocationProviderImpl`, `PublicSectorFileBasedDecisionTableLocationProviderImpl`)
- **How to confirm:** Verify the assessment OmniScript/Expression Set is published; check decision-table location resolution; check `ipipr` runtime logs for the assessment IP
- **GUS search:** `Product_Tag__r.Name LIKE '%Vlocity Public Sector%' AND Subject__c LIKE '%assessment%'` (apply build-staleness rule)

## Pattern: Make-a-payment action fails in portal/console
- **Likely subsystem:** `.../components/controller/MakePaymentActionController.java`; legacy managed-pkg payment via `PSPymnt` (via_ps)
- **How to confirm:** Trace the payment action; confirm payment gateway/named-credential config; check both core controller and the `vlocity_ps` `PSPymnt` path (per CLAUDE.md: always check core AND managed package)
- **GUS search:** `Product_Tag__r.Name LIKE '%Vlocity Public Sector%' AND Subject__c LIKE '%payment%'` (apply build-staleness rule)

## Pattern: Constituent/community user "access denied" or housing card not respecting access
- **Likely subsystem:** Managed-pkg housing/community components (`HousingCards`, `CommunityUserTenantDetailsCards`) in via_ps; access-guard annotations on core controllers
- **How to confirm:** This is a known historical failure mode (components using a System session Id that didn't respect user access) — verify community user sharing/FLS and that the component honours running-user context; check `NoAccessException` patterns in controller results
- **GUS search:** `Product_Tag__r.Name LIKE '%Vlocity Public Sector%' AND Subject__c LIKE '%access%'` (also try `%community%` / `%housing%`; apply build-staleness rule)
