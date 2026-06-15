---
name: education-known-patterns
description: Education Cloud / EDA diagnostic patterns — symptom to subsystem to confirmation to GUS search.
---

# Education Cloud — Diagnostic Patterns

> Symptom → likely subsystem → how to confirm → GUS search. Patterns are leads; confirm against the current case (Phase 4/5). Release branches use {CURRENT_GA} — resolve per capabilities/codesearch.md.
>
> **Two product layers — route first:**
> - **EDA (managed package, `hed__`/`sfedo__`)** — open-source managed pkg; GUS tag `Education Data Architecture` (high bug volume). Trigger-handler (TDTM) framework, Affiliations, Course Connections.
> - **Education Cloud (core)** — Java under `core/industries-education-impl/java/src/industries/education/impl/` (verified subsystems: `admissions`, `applicationprocess`, `academicoperations`, `assessments`, `attendance`, `learnerpath`, `learningcatalog`, `mentoring`, `programmanagement`, `recruiting`, `scheduler`, `studentmanagement`, `studentsuccess`, `hold`, `sharing`) plus `core/industries-education/java/src/industries/education/dpe/` for batch calc. The bare tag `Education Cloud` carries no bugs — use the subsystem-specific tags below or fall through to EDA.

## Pattern: Affiliation not auto-creating / wrong on Contact or Course Offering change
- **Likely subsystem:** EDA managed pkg — Affiliation TDTM trigger handlers (`hed__`); confirm whether it's an Affiliation Mappings config issue vs. a code defect (e.g. list/index errors when multiple offerings update in one transaction).
- **How to confirm:** Check EDA Settings → Affiliation Mappings; reproduce with a bulk update (multiple records in one transaction); inspect `axerr` for `System.ListException` / TDTM handler class in the stack.
- **GUS search:** `Product_Tag__r.Name LIKE '%Education Data Architecture%' AND Subject__c LIKE '%Affiliation%'` (apply build-staleness rule)

## Pattern: EDA trigger errors / unexpected behavior after package upgrade
- **Likely subsystem:** EDA TDTM trigger-handler framework — disabled/duplicate handler rows, or custom trigger conflicting with EDA's.
- **How to confirm:** Compare EDA version pre/post upgrade; check the Trigger Handler custom object for the affected sObject; look for customer triggers on the same object; `axerr` for the failing handler class.
- **GUS search:** `Product_Tag__r.Name LIKE '%Education Data Architecture%' AND Subject__c LIKE '%trigger%'` (apply build-staleness rule)

## Pattern: Admissions application flow / OmniScript step failing
- **Likely subsystem:** Education Cloud core `admissions` / `applicationprocess`, or the OmniStudio UI layer (OmniScript/FlexCard) wired onto EDU.
- **How to confirm:** Determine if the failure is in the OmniScript/IP (check `ipipr`/`ipdar` logRecordTypes) vs. core application processing; reproduce the specific application step.
- **GUS search:** `Product_Tag__r.Name LIKE '%Education - OmniStudio%' AND Subject__c LIKE '%application%'` (apply build-staleness rule)

## Pattern: Program plan / learner path requirements not mapping
- **Likely subsystem:** Education Cloud core `learnerpath` / `programmanagement`, or EDA `hed__Program_Plan__c` / `hed__Plan_Requirement__c` hierarchy (route by namespace on the failing object).
- **How to confirm:** Identify whether objects are `hed__` (EDA) or core; verify plan-requirement hierarchy and course connections; check effective dates.
- **GUS search:** `Product_Tag__r.Name LIKE '%Education Data Architecture%' AND Subject__c LIKE '%Program Plan%'` (apply build-staleness rule)

## Pattern: Student Success alert / advising appointment not firing
- **Likely subsystem:** Education Cloud core `studentsuccess` / `scheduler`, or the Student Success Hub managed pkg.
- **How to confirm:** Check alert criteria and evaluation frequency; for scheduling, verify the `scheduler` config and appointment availability; `gslog` for core Java exceptions.
- **GUS search:** `Product_Tag__r.Name LIKE '%Education Data Architecture%' AND Subject__c LIKE '%alert%'` (apply build-staleness rule)

## Pattern: Attendance / assessment records not generated or rolled up
- **Likely subsystem:** Education Cloud core `attendance` / `assessments`.
- **How to confirm:** Reproduce the record-generation event; check `gslog` for the subsystem class; confirm prerequisite records (offering, enrollment) exist.
- **GUS search:** `Product_Tag__r.Name LIKE '%Education Data Architecture%' AND Subject__c LIKE '%attendance%'` (apply build-staleness rule)

## Pattern: Education batch / DPE calculation job fails or produces no output
- **Likely subsystem:** `core/industries-education/java/src/industries/education/dpe/` — `EducationBatchCalcJobProcessTypeHandler`, file-based template provider (`EducationFilebasedTemplateLocationProvider`).
- **How to confirm:** Check the batch/process-type job status; verify the DPE template location resolves; `gslog` / `axlim` for governor-limit or template-not-found errors in the batch.
- **GUS search:** `Product_Tag__r.Name LIKE '%Education Data Architecture%' AND Subject__c LIKE '%batch%'` (apply build-staleness rule)

## Pattern: Duplicate Contacts / Accounts on import
- **Likely subsystem:** EDA duplicate & matching rules + Account record-type model (Academic vs Administrative/Business).
- **How to confirm:** Check active duplicate rules and matching rules; verify the import path (Data Loader vs flow vs API); confirm the Account record type assignment.
- **GUS search:** `Product_Tag__r.Name LIKE '%Education Data Architecture%' AND Subject__c LIKE '%duplicate%'` (apply build-staleness rule)

## Pattern: Translation / localized label wrong on EDA field
- **Likely subsystem:** EDA managed pkg field metadata / translation (e.g. Preferred Email not excluding standard Email, mis-spelled descriptions).
- **How to confirm:** Confirm the field is `hed__`; check the org's translation workbench and the package's shipped translations; this is typically a managed-pkg defect, not config.
- **GUS search:** `Product_Tag__r.Name LIKE '%Education Data Architecture%' AND Subject__c LIKE '%translation%'` (apply build-staleness rule)
