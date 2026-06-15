---
name: nonprofit-known-patterns
description: Nonprofit Cloud / NPSP diagnostic patterns — symptom to subsystem to confirmation to GUS search.
---

# Nonprofit Cloud — Diagnostic Patterns

> Symptom → likely subsystem → how to confirm → GUS search. Patterns are leads; confirm against the current case (Phase 4/5). Release branches use {CURRENT_GA} — resolve per capabilities/codesearch.md.
>
> **Two product layers — route first:**
> - **NPSP (managed package, `npsp__`/`npe01__`/`npo02__`/`npe03__`/`npe4__`/`npe5__`)** — open-source managed pkg; GUS tag `Nonprofit Success Pack (NPSP)`. Trigger-handler (TDTM) framework, Opportunity-as-gift model, rollups, recurring donations.
> - **Nonprofit Cloud (core)** — Java under `core/industries-nonprofit-impl/java/src/industries/nonprofit/impl/` plus `core/industries-fundraising/`. Verified core subsystems (from the feature taxonomy): **Fundraising** (`BusinessProcessAPI`, `GiftEntry`, `GiftCommitmentManagement`, `RollUps`, `GiftDesignation`, `ExternalIDSupport`), **Grantmaking** (`ApplicationManagement`), **ProgramManagement** (`ScheduleSessionRegistration`, `UnscheduledBenefits`, `AttendanceTracking`, `BenefitScheduleSetup`). The bare tag `Nonprofit Cloud` carries no bugs — use the layer-specific tags below.

## Pattern: Recurring donation installment not creating / high CPU on rollups
- **Likely subsystem:** NPSP Recurring Donations (`npe03__Recurring_Donation__c`, Enhanced RD) and the rollup engine; fixed-length + customizable rollups + custom naming is a known CPU hot path.
- **How to confirm:** Check RD schedule config and the `npe03__Recurring_Donations_Settings__c` custom setting; for slow/failed runs check `axlim` (CPU/SOQL) on the rollup batch; isolate whether rollups or naming is the cost.
- **GUS search:** `Product_Tag__r.Name LIKE '%Nonprofit Success Pack (NPSP)%' AND Subject__c LIKE '%Recurring Donation%'` (apply build-staleness rule)

## Pattern: Donation rollups wrong / soft credit not rolling up (NPSP)
- **Likely subsystem:** NPSP Rollup settings — hard vs soft credit, Contact Roles, customizable rollups.
- **How to confirm:** Check NPSP Rollup configuration; verify Opportunity Contact Roles for soft credits; re-run the rollup and compare; check `axlim` if it's also slow.
- **GUS search:** `Product_Tag__r.Name LIKE '%Nonprofit Success Pack (NPSP)%' AND Subject__c LIKE '%rollup%'` (apply build-staleness rule)

## Pattern: Gift entry batch errors (core Nonprofit Cloud)
- **Likely subsystem:** Core `Fundraising.GiftEntry` (`industries.nonprofit.impl` gift-entry), distinct from NPSP GEM batches.
- **How to confirm:** Confirm whether the org uses core Nonprofit Cloud Gift Entry vs NPSP/GEM; for core, check `gslog` for the gift-entry class and any commitment-validation skip telemetry; verify batch template + field mapping.
- **GUS search:** `Product_Tag__r.Name LIKE '%Nonprofit Success Pack (NPSP)%' AND Subject__c LIKE '%Gift Entry%'` (apply build-staleness rule)

## Pattern: Gift commitment / pledge processing batch fails (core Nonprofit Cloud)
- **Likely subsystem:** Core `Fundraising.GiftCommitmentManagement` — next-gen commitment processing batch job.
- **How to confirm:** Check the commitment batch job status; `gslog` / `axlim` for the commitment-processing batch; confirm gift designation (GAU equivalent) is resolvable.
- **GUS search:** `Product_Tag__r.Name LIKE '%Nonprofit Success Pack (NPSP)%' AND Subject__c LIKE '%commitment%'` (apply build-staleness rule)

## Pattern: Donation created via API not appearing / Business Process API error
- **Likely subsystem:** Core `Fundraising.BusinessProcessAPI` (Nonprofit Cloud Connect/Business Process layer).
- **How to confirm:** Check the API request/response and `gslog` for the BusinessProcessAPI class; confirm the donation payload and required references resolve; check telemetry for `DONATION_CREATED_VIA_API`.
- **GUS search:** `Product_Tag__r.Name LIKE '%Nonprofit Success Pack (NPSP)%' AND Subject__c LIKE '%API%'` (apply build-staleness rule)

## Pattern: Address management conflict / address verification wrong format
- **Likely subsystem:** NPSP address model (`npsp__Address__c`) vs standard address, or core Nonprofit Cloud Addresses; verification can emit a street format the locale doesn't expect.
- **How to confirm:** Identify NPSP vs core address path; check the address verification provider settings and locale; compare the stored vs verified street format.
- **GUS search:** `Product_Tag__r.Name LIKE '%Nonprofit Cloud - Addresses%' AND Subject__c LIKE '%Address%'` (apply build-staleness rule)

## Pattern: Allocation / Gift Designation total exceeds 100% or won't save
- **Likely subsystem:** NPSP GAU Allocations (`npsp__Allocation__c` / `npsp__General_Accounting_Unit__c`), or core `Fundraising.GiftDesignation`.
- **How to confirm:** Check GAU allocation rules and the default allocation; verify percent vs amount allocations sum correctly; route by namespace (NPSP vs core).
- **GUS search:** `Product_Tag__r.Name LIKE '%Nonprofit Success Pack (NPSP)%' AND Subject__c LIKE '%Allocation%'` (apply build-staleness rule)

## Pattern: Grant application / grantmaking record not progressing
- **Likely subsystem:** Core `Grantmaking.ApplicationManagement` (`industries.nonprofit.impl`).
- **How to confirm:** Confirm core Grantmaking (not a custom solution); check `gslog` for the application-management class; verify the application's stage/status transitions and required fields.
- **GUS search:** `Product_Tag__r.Name LIKE '%Nonprofit Success Pack (NPSP)%' AND Subject__c LIKE '%grant%'` (apply build-staleness rule)

## Pattern: Program session registration / benefit schedule / attendance issue
- **Likely subsystem:** Core `ProgramManagement` — `ScheduleSessionRegistration`, `BenefitScheduleSetup`, `AttendanceTracking`, `UnscheduledBenefits`.
- **How to confirm:** Identify the specific program-management feature; check `gslog` for the subsystem class; verify program/cohort/participant records and the benefit schedule setup.
- **GUS search:** `Product_Tag__r.Name LIKE '%Nonprofit Success Pack (NPSP)%' AND Subject__c LIKE '%program%'` (apply build-staleness rule)

## Pattern: NPSP trigger conflict after upgrade / custom trigger interference
- **Likely subsystem:** NPSP TDTM trigger-handler framework — disabled/duplicate handler rows or a customer trigger on the same object.
- **How to confirm:** Check the NPSP Trigger Handler config for the affected sObject; look for custom triggers; `axerr` for the failing handler class; compare NPSP version pre/post upgrade.
- **GUS search:** `Product_Tag__r.Name LIKE '%Nonprofit Success Pack (NPSP)%' AND Subject__c LIKE '%trigger%'` (apply build-staleness rule)
