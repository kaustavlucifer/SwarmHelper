---
name: net-zero-known-patterns
description: Net Zero Cloud diagnostic leads — symptom to subsystem to confirmation to GUS search. Verified against core monorepo + Sustainability-App.
---

# Net Zero Cloud — Diagnostic Patterns

> Symptom → likely subsystem → how to confirm → GUS search. Patterns are leads; confirm against the current case (Phase 4/5). Release branches use {CURRENT_GA} — resolve per capabilities/codesearch.md.
>
> Canonical GUS tag: `Net Zero Cloud`. Engineering-shared fallbacks seen live: `Net Zero Engineering Shared`, `Net Zero Cloud - OmniStudio`. Apply the build-staleness rule (a bug Closed in a build older than the org's GA is not your repro).

## Pattern: Carbon footprint not calculating / stale after data load
- **Likely subsystem:** `core/industries-sustainability/java/src/industries/sustainability/calculations/` and `energyusetocarbonfootprintautomation/` (energy-use → carbon-footprint automation, `CarbonFootprintDAO`)
- **How to confirm:** Check the calculation batch ran (gslog/axlim for the batch job); verify activity data (EnergyUse/EnergyUseItem) is complete for the reporting period; confirm an emission factor mapped to the activity.
- **GUS search:** `Product_Tag__r.Name LIKE '%Net Zero Cloud%' AND Subject__c LIKE '%carbon footprint%'` (apply build-staleness rule)

## Pattern: Emission factor not found / wrong factor applied
- **Likely subsystem:** `fetchEmissionsFactor/` + `referencedata/` / `referencedatav2/` / `referencedatav3/` (versioned reference-data layers)
- **How to confirm:** Confirm which reference-data version the org is on (v2 vs v3 behave differently); check EmissionFactor/EmissionFactorItem load completed; verify activity-type → factor mapping and the factor's effective date range covers the reporting period.
- **GUS search:** `Product_Tag__r.Name LIKE '%Net Zero Cloud%' AND Subject__c LIKE '%emission factor%'` (apply build-staleness rule)

## Pattern: Reference data load job failing or partial
- **Likely subsystem:** `referencedatav3/` + `batchjob/` + `messagequeue/` (async load pipeline)
- **How to confirm:** Check batch/queue job status and gslog for the load; inspect data format vs expected schema; look for governor-limit (axlim) truncation on large factor sets.
- **GUS search:** `Product_Tag__r.Name LIKE '%Net Zero Cloud%' AND Subject__c LIKE '%reference data%'` (apply build-staleness rule)

## Pattern: Scope 3 / supplier emissions missing or not aggregating
- **Likely subsystem:** `supplier/` + `suppliermanagement/` + `topdown/` / `bottomup/` (supplier-side Scope 3 estimation)
- **How to confirm:** Determine whether the org uses top-down (spend-based) or bottom-up (activity-based) estimation; verify supplier records and their emission data exist; check the chosen pathway is configured.
- **GUS search:** `Product_Tag__r.Name LIKE '%Net Zero Cloud%' AND Subject__c LIKE '%scope 3%'` (also try `%supplier%`; apply build-staleness rule)

## Pattern: Rollup / aggregated totals wrong across org or hierarchy
- **Likely subsystem:** `rollup/` + `scorecardallocation/` (aggregation and allocation across org structure)
- **How to confirm:** Check whether the rollup batch re-ran after the source records changed; verify allocation rules and reporting-period alignment; reconcile a single leaf record's value vs the aggregate.
- **GUS search:** `Product_Tag__r.Name LIKE '%Net Zero Cloud%' AND Subject__c LIKE '%rollup%'` (also try `%aggregat%`; apply build-staleness rule)

## Pattern: Calculation export to downstream / file output failing
- **Likely subsystem:** `netzerocalculationexport/` + `filebased/` + `serialization/`
- **How to confirm:** Check the export job in gslog; verify the target/file config; inspect serialization errors on large result sets.
- **GUS search:** `Product_Tag__r.Name LIKE '%Net Zero Cloud%' AND Subject__c LIKE '%export%'` (apply build-staleness rule)

## Pattern: Building / energy intensity metric wrong
- **Likely subsystem:** `calculations/` + `flexibleformula/` (custom-formula-driven intensity) operating on BldgEnrgyIntensity / EnergyUse
- **How to confirm:** Verify building-area normalization input; check whether a flexible/custom formula overrides the standard calc; confirm units.
- **GUS search:** `Product_Tag__r.Name LIKE '%Net Zero Cloud%' AND Subject__c LIKE '%intensity%'` (apply build-staleness rule)

## Pattern: Record locked / cannot edit after calculation
- **Likely subsystem:** `recordlockunlock/`
- **How to confirm:** Check whether the record was locked by a completed calculation run; confirm the unlock path / permission to unlock; look for a stuck lock from a failed batch.
- **GUS search:** `Product_Tag__r.Name LIKE '%Net Zero Cloud%' AND Subject__c LIKE '%lock%'` (apply build-staleness rule)

## Pattern: Program enrollment / EA credit distribution issues
- **Likely subsystem:** `programenrollmentservices/` + `eaccreditdistribution/`
- **How to confirm:** Verify program enrollment records and eligibility; check the credit-distribution run and its inputs.
- **GUS search:** `Product_Tag__r.Name LIKE '%Net Zero Cloud%' AND Subject__c LIKE '%credit%'` (also try `%enrollment%`; apply build-staleness rule)

## Pattern: OmniStudio component error inside a Net Zero flow
- **Likely subsystem:** OmniStudio runtime (IPs/DataRaptors) — see verticals/omnistudio/. Net Zero is hybrid (managed app `github.com/sf-industries/Sustainability-App` + core Java); UI/flows may use OmniStudio.
- **How to confirm:** Pull ipipr/ipdar for the failing IP/DataRaptor; confirm whether the failure is in the OmniStudio layer vs the sustainability core calc.
- **GUS search:** `Product_Tag__r.Name LIKE '%Net Zero Cloud - OmniStudio%' AND Subject__c LIKE '%<keyword>%'` (apply build-staleness rule)
