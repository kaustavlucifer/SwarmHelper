---
name: manufacturing-known-patterns
description: Manufacturing Cloud diagnostic map — symptom to subsystem to confirmation to GUS search. Leads, not verdicts.
---

# Manufacturing Cloud — Diagnostic Patterns

> Symptom → likely subsystem → how to confirm → GUS search. Patterns are leads; confirm root cause against the current case (Phase 4/5). Release branches use {CURRENT_GA} — resolve per capabilities/codesearch.md.
>
> **Product tags are area-specific** — there is no bare `Manufacturing Cloud` tag. Use the verified tag closest to the feature area (each grounded against live GUS bug counts):
> - Sales Agreement / Account Forecasting / AAF / Rebates / PBB (umbrella): `Manufacturing Cloud - Sales Agreement, Account Forecasting, AAF, Rebates, PBB`
> - Warranty: `Manufacturing Service - Warranty Management`
> - Program Based Business: `Manufacturing Program Based Business`
> - Rebates calc engine: `Manufacturing Rebates Calculation - New` · Rebates setup: `Manufacturing Rebates Setup`
> - Inventory: `Manufacturing - Inventory visibility and actions`

## Pattern: Cannot edit fields on an expired or renewed Sales Agreement
- **Likely subsystem:** Sales Agreement entity layer — `core/industries-manufacturing/java/` (SalesAgreement / SalesAgreementProduct / SalesAgreementProductSchedule entity functions)
- **How to confirm:** Check the SA `Status` (Expired/Activated) and whether the edit targets a `SalesAgreementProductSchedule` custom field; reproduce on a record in the same status. Edits are gated by SA lifecycle state.
- **GUS search:** `Product_Tag__r.Name LIKE '%Manufacturing Cloud - Sales Agreement, Account Forecasting, AAF, Rebates, PBB%' AND Subject__c LIKE '%Sales Agreement%'` (apply build-staleness rule)

## Pattern: Renewed Sales Agreement has wrong start date / schedule period
- **Likely subsystem:** Sales Agreement renewal + schedule generation — `core/industries-manufacturing/java/`, `core/industries-manufacturing/calcTemplates/`
- **How to confirm:** Compare parent SA `StartDate`/`EndDate` against the renewed SA; inspect the schedule frequency and whether period boundaries align with the calc template. Check for a renewal-job error in `gslog`.
- **GUS search:** `Product_Tag__r.Name LIKE '%Manufacturing Cloud - Sales Agreement, Account Forecasting, AAF, Rebates, PBB%' AND Subject__c LIKE '%Renew%'` (apply build-staleness rule)

## Pattern: Sales Agreement grid — wrong cell selected, adjustment hover misaligned, can't navigate pages
- **Likely subsystem:** Sales Agreement grid LWC — `core/ui-industries-manufacturing-components/modules/` (and `components/`)
- **How to confirm:** Reproduce in the SA editor grid; note whether it triggers after editing a product on page 2+ (pagination state) or on the adjustments hover. UI-layer regression — check the PTC/component build, not the entity layer.
- **GUS search:** `Product_Tag__r.Name LIKE '%Manufacturing Cloud - Sales Agreement, Account Forecasting, AAF, Rebates, PBB%' AND Subject__c LIKE '%Grid%'` (apply build-staleness rule)

## Pattern: Account forecast not generating or actuals not syncing
- **Likely subsystem:** Advanced Account Forecasting + Data Processing Engine — `core/industries-manufacturing/batchJobDefinitions/`, `core/industries-manufacturing/java/`
- **How to confirm:** Check the DPE definition run status and input/output mapping; verify the forecasting type, product-category mapping, and the actuals data source. Look for batch failures in `gslog`/`axlim` (large-volume forecasts hit governor limits).
- **GUS search:** `Product_Tag__r.Name LIKE '%Manufacturing Cloud - Sales Agreement, Account Forecasting, AAF, Rebates, PBB%' AND Subject__c LIKE '%Forecast%'` (apply build-staleness rule)

## Pattern: Program Based Business — mass update or flow input parameter fails
- **Likely subsystem:** PBB processing — `core/industries-manufacturing/java/` (manufacturing program / program forecast entity + flow-invocable layer)
- **How to confirm:** For flow failures, check the invocable input type — collection params (e.g. `String[]`) passed as ANYTYPE can fail to bind. For mass update, reproduce against the PBB mass-update modal and capture the `axerr`.
- **GUS search:** `Product_Tag__r.Name LIKE '%Manufacturing Program Based Business%' AND Subject__c LIKE '%PBB%'` (apply build-staleness rule)

## Pattern: Rebate claim details / payout incorrect or inaccessible
- **Likely subsystem:** Rebate Management calc + claim engine — `core/industries-mfg-rebates-impl/java/`, `core/industries-mfg-rebates/java/`, `core/industries-mfg-rebates/esTemplates/` (Expression Set / decision logic)
- **How to confirm:** Trace the rebate program → member → payout chain; verify the Expression Set / calc template version and effective dates. For "cannot fetch claim details," check sharing / permission set (rebate claim access is permission-gated).
- **GUS search:** `Product_Tag__r.Name LIKE '%Manufacturing Rebates Calculation - New%' AND Subject__c LIKE '%Rebate%'` (apply build-staleness rule)

## Pattern: Warranty / supplier claim flex card errors or denied claim
- **Likely subsystem:** Warranty Management — `core/industries-manufacturing/java/` (Claim / ClaimItem / ClaimCoverage), warranty claim FlexCard
- **How to confirm:** For denied claims, verify `WarrantyTerm` effective dates and coverage-verification rules against the asset's in-service date. For flex card errors, reproduce the card render and capture the `axerr`/console error — distinguish UI regression from entity-rule denial.
- **GUS search:** `Product_Tag__r.Name LIKE '%Manufacturing Service - Warranty Management%' AND Subject__c LIKE '%Claim%'` (apply build-staleness rule)

## Pattern: Inventory visibility / availability data wrong or missing
- **Likely subsystem:** Inventory Management — `core/industries-manufacturing/java/`, DPE definitions in `core/industries-manufacturing/batchJobDefinitions/`
- **How to confirm:** Check inventory object permissions and the DPE/data-source feeding stock levels; verify the distribution/demand mapping. Confirm whether the gap is ingestion (DPE) or display (component).
- **GUS search:** `Product_Tag__r.Name LIKE '%Manufacturing - Inventory visibility and actions%' AND Subject__c LIKE '%Inventory%'` (apply build-staleness rule)

## Pattern: Privilege-escalation / sharing gack in a Manufacturing flow
- **Likely subsystem:** Entity sharing layer — `core/industries-manufacturing/java/` and `core/industries-mfg-rebates-impl/java/` (look for `runAs` / `withoutSharing` paths flagged by Productivity Tools)
- **How to confirm:** Pull the gack stack via columbo; identify the entity function performing the privileged read. These surface as "Push Privilege Escalation" hardening items, not customer-facing errors.
- **GUS search:** `Product_Tag__r.Name LIKE '%Manufacturing Cloud - Sales Agreement, Account Forecasting, AAF, Rebates, PBB%' AND Subject__c LIKE '%Privilege Escalation%'` (apply build-staleness rule)
