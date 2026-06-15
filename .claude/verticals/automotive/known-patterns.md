---
name: automotive-known-patterns
description: Automotive Cloud diagnostic map — symptom to subsystem to confirmation to GUS search. Leads, not verdicts.
---

# Automotive Cloud — Diagnostic Patterns

> Symptom → likely subsystem → how to confirm → GUS search. Patterns are leads; confirm root cause against the current case (Phase 4/5). Release branches use {CURRENT_GA} — resolve per capabilities/codesearch.md.
>
> **Product tags are area-specific** — there is no bare `Automotive Cloud` tag. Use the verified tag closest to the feature area (each grounded against live GUS bug counts):
> - Vehicle records / Lead Mgmt / appraisal: `Automotive Cloud - Vehicle 360`
> - Auto finance / lease / repossession: `Automotive - Captive Finance`
> - Telematics / connected asset / telemetry: `Automotive - Connected Cars`
> - Dealer / criteria search: `Automotive - Criteria Based Search`
> - OEM service / warranty processes: `Automotive - OEM Service Processes`

## Pattern: Vehicle / appraisal record fails to create or sync
- **Likely subsystem:** Vehicle 360 entity layer — `core/industries-automotive/java/src/`, `core/industries-automotive-udd/java/` (VehicleDefinition + AppraisalItem / OpportunityInterest entities)
- **How to confirm:** Check the DPE job status and data-source mapping for bulk loads; for single-record failures capture the `axerr`. Watch for unresolved field references (e.g. an `AppraisalItem__c` lookup) that indicate a UDD/packaging mismatch rather than data.
- **GUS search:** `Product_Tag__r.Name LIKE '%Automotive Cloud - Vehicle 360%' AND Subject__c LIKE '%Vehicle%'` (apply build-staleness rule)

## Pattern: Auto finance application form won't submit (agent or self-intake flow)
- **Likely subsystem:** Captive Finance intake — `core/industries-automotive/java/src/` (application form / DCI junction entities), Agentforce intake flow
- **How to confirm:** Reproduce both the agent path and the self-intake flow; check the DocumentChecklistItem / application-form junction records and the flow's submit action. Capture `axerr` and confirm whether it is flow-config or entity-validation.
- **GUS search:** `Product_Tag__r.Name LIKE '%Automotive - Captive Finance%' AND Subject__c LIKE '%Application form%'` (apply build-staleness rule)

## Pattern: Lease extension / finance process template loads slowly
- **Likely subsystem:** Captive Finance service process — `core/industries-automotive/java/src/`, finance service-process templates (Financial Account step)
- **How to confirm:** Time the offending step (e.g. "Load Financial Accounts" in Lease Extension); pull `axlim` for SOQL/CPU and check the financial-account query volume. Perf bug, not a functional denial — look at row counts and step query design.
- **GUS search:** `Product_Tag__r.Name LIKE '%Automotive - Captive Finance%' AND Subject__c LIKE '%Lease%'` (apply build-staleness rule)

## Pattern: Repossession activity / item create errors or missing fields
- **Likely subsystem:** Captive Finance repossession entities — `core/industries-automotive/java/src/` (Repossession / RepossessionItem entity functions)
- **How to confirm:** Check the create modal for required-field/Name-field gaps; verify the entity is enabled for the org's license. Reproduce the create flow and capture the `axerr`.
- **GUS search:** `Product_Tag__r.Name LIKE '%Automotive - Captive Finance%' AND Subject__c LIKE '%Repossession%'` (apply build-staleness rule)

## Pattern: Connected-asset telemetry / remote action not firing, or telemetry metadata not deployable
- **Likely subsystem:** Connected Cars telemetry — `core/industries-automotive/java/src/` (TelemetryDefinition, TelemetryActionDefinition and related metadata)
- **How to confirm:** For remote actions (e.g. HVAC control), trace the telemetry action definition → step → attribute chain. For deploy/extract gaps, check whether the Telemetry* metadata types are exposed to SF CLI/Metadata API in the target release.
- **GUS search:** `Product_Tag__r.Name LIKE '%Automotive - Connected Cars%' AND Subject__c LIKE '%Telemetry%'` (apply build-staleness rule)

## Pattern: Dealer / criteria-based search returns no results
- **Likely subsystem:** Criteria Based Search — `core/industries-automotive/java/src/`, search configuration + permission set
- **How to confirm:** Verify the Criteria-Based Search permission set is assigned and the search configuration (object, fields, criteria) is active. Confirm indexed fields and that the running user has record access.
- **GUS search:** `Product_Tag__r.Name LIKE '%Automotive - Criteria Based Search%' AND Subject__c LIKE '%Search%'` (apply build-staleness rule)

## Pattern: Warranty / claim coverage reserve mode behaves unexpectedly
- **Likely subsystem:** OEM Service / warranty processes — `core/industries-automotive/java/src/` (Claim / ClaimCoverage entity functions, reserve-mode defaulting)
- **How to confirm:** Check the ClaimCoverage `Internal Reserve Mode` default and the warranty-term coverage rules; verify effective dates against the asset's in-service date. Distinguish a defaulting bug from a legitimate coverage denial.
- **GUS search:** `Product_Tag__r.Name LIKE '%Automotive - OEM Service Processes%' AND Subject__c LIKE '%Coverage%'` (apply build-staleness rule)

## Pattern: Lead management / dealer performance privilege-escalation gack
- **Likely subsystem:** Vehicle 360 lead/dealer layer — `core/industries-automotive/java/src/` (LeadMgmtCommonUtils and related sharing paths)
- **How to confirm:** Pull the gack stack via columbo and identify the privileged read; these surface as Productivity Tools hardening items ("Unnecessary Privilege Escalation"), not customer-facing failures.
- **GUS search:** `Product_Tag__r.Name LIKE '%Automotive Cloud - Vehicle 360%' AND Subject__c LIKE '%Privilege Escalation%'` (apply build-staleness rule)

## Pattern: Automotive setup / Go (Accelerate) install or config failure
- **Likely subsystem:** Setup Home + Actionable Event Orchestration — `core/industries-automotive-setup-home/java/`, `core/industries-automotive/config/`
- **How to confirm:** For "Go/Go Accelerate" install failures (e.g. installing Actionable Event Orchestration for Connected Assets), check the setup flow logs and the orchestration package dependency; confirm the org's license/feature gate.
- **GUS search:** `Product_Tag__r.Name LIKE '%Automotive - Captive Finance%' AND Subject__c LIKE '%Go Accelerate%'` (apply build-staleness rule)
