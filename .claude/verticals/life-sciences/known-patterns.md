---
name: life-sciences-known-patterns
description: Life Sciences Cloud diagnostic leads — symptom to subsystem to confirmation to GUS search. Verified against core monorepo + sf-industries-ls/lifesciences.
---

# Life Sciences Cloud — Diagnostic Patterns

> Symptom → likely subsystem → how to confirm → GUS search. Patterns are leads; confirm against the current case (Phase 4/5). Release branches use {CURRENT_GA} — resolve per capabilities/codesearch.md.
>
> **GUS tag note:** there is NO `Life Sciences Cloud` bug tag (returns 0). Use `Life Sciences Cloud - Product Management` (real product bugs) as primary; `Health and Life Sciences` as fallback (noisier). Apply the build-staleness rule. Much LS logic lives in the managed package `sf-industries-ls/lifesciences` and shared Health Cloud core, not in `industries-lifesciences-impl` (which is thin: configuration, contentprovider, participantmgmt, utils).

## Pattern: Participant / trial subject enrollment failing
- **Likely subsystem:** `core/industries-lifesciences-impl/java/src/industries/lifesciences/impl/participantmgmt/` (controllers / model / services) operating on ResearchStudy, ResearchStudySubject
- **How to confirm:** Reproduce the enrollment action and pull gslog for participantmgmt services; verify ResearchStudy ↔ ResearchSite ↔ subject relationships; if enrollment runs through an IP, also pull ipipr.
- **GUS search:** `Product_Tag__r.Name LIKE '%Life Sciences Cloud - Product Management%' AND Subject__c LIKE '%enroll%'` (apply build-staleness rule)

## Pattern: Content / presentation not loading or wrong content shown
- **Likely subsystem:** `industries/lifesciences/impl/contentprovider/` (core) + `presentation-player/` in `sf-industries-ls/lifesciences` (managed pkg)
- **How to confirm:** Check content-definition hierarchy and assignment rules; confirm the presentation-player loaded the expected content version; for mobile, check static-resource callout behavior.
- **GUS search:** `Product_Tag__r.Name LIKE '%Life Sciences Cloud - Product Management%' AND Subject__c LIKE '%content%'` (also try `%presentation%`; apply build-staleness rule)

## Pattern: LS mobile app — custom LWC / static resource not loading
- **Likely subsystem:** managed pkg `sf-industries-ls/lifesciences` (force-app LWC + mobile config); known live bug class (mobile app not making callout to load static resources in a custom LWC)
- **How to confirm:** Confirm whether the component is custom vs packaged; check mobile app config and static-resource references; compare web vs mobile behavior.
- **GUS search:** `Product_Tag__r.Name LIKE '%Life Sciences Cloud - Product Management%' AND Subject__c LIKE '%static resource%'` (also try `%mobile%`; apply build-staleness rule)

## Pattern: Consent capture / privacy errors
- **Likely subsystem:** ContactPointConsent + data-sharing model (consent flows often run through OmniStudio IPs; shared with Health Cloud)
- **How to confirm:** Check ContactPointConsent permissions and DataUsePurpose mapping; pull ipipr if consent is captured via an IP; verify effective-date windows on the consent record.
- **GUS search:** `Product_Tag__r.Name LIKE '%Life Sciences Cloud - Product Management%' AND Subject__c LIKE '%consent%'` (fallback tag `Health and Life Sciences`; apply build-staleness rule)

## Pattern: Care program / drug program setup or eligibility
- **Likely subsystem:** CareProgram / CareProgramEnrollee / CareProgramProduct configuration (shared Health Cloud care-program core)
- **How to confirm:** Verify CareProgram config, eligibility rules, and product mapping; check enrollee status transitions; confirm the enrollee's Account/PersonAccount link.
- **GUS search:** `Product_Tag__r.Name LIKE '%Life Sciences Cloud - Product Management%' AND Subject__c LIKE '%care program%'` (also try `%eligib%`; apply build-staleness rule)

## Pattern: HCP engagement / visit tracking issues
- **Likely subsystem:** `Visit` / `VisitedParty` objects + activity-capture config (commercial / HCP engagement). *(`VisitedParty` is the verified object; bare `VisitedPlace` may not exist in all orgs — `describe` to confirm.)*
- **How to confirm:** Check `Visit` object permissions and scheduling config; verify activity-capture and compliance rules; confirm `VisitedParty` data.
- **GUS search:** `Product_Tag__r.Name LIKE '%Life Sciences Cloud - Product Management%' AND Subject__c LIKE '%visit%'` (also try `%HCP%`; apply build-staleness rule)

## Pattern: Agentforce briefing / prompt-template feature failing
- **Likely subsystem:** managed pkg `sf-industries-ls/lifesciences` `prompts/` (GenAI prompt templates) + Briefing object
- **How to confirm:** Check metadata-cache configuration and Briefing object schema; verify the prompt template deployed and resolved; pull gslog for the GenAI prompt path.
- **GUS search:** `Product_Tag__r.Name LIKE '%Life Sciences Cloud - Product Management%' AND Subject__c LIKE '%briefing%'` (also try `%prompt%`; apply build-staleness rule)

## Pattern: Org template / integration-org configuration drift
- **Likely subsystem:** `industries/lifesciences/impl/configuration/` (core config) + LS org-template provisioning; live bug class (LS prod org template using wrong Industries Integration org value)
- **How to confirm:** Compare the org's template/integration-org value against the expected LS value; check whether the issue is provisioning vs runtime.
- **GUS search:** `Product_Tag__r.Name LIKE '%Life Sciences Cloud - Product Management%' AND Subject__c LIKE '%org template%'` (also try `%integration org%`; apply build-staleness rule)

## Pattern: Medication dispense / request data model issues
- **Likely subsystem:** MedicationDispense / MedicationRequest (FHIR-aligned clinical objects, shared with Health Cloud)
- **How to confirm:** Verify object permissions and FHIR mapping; check the dispense ↔ request linkage; if data arrives via integration, inspect the inbound mapping (shared HC FHIR path).
- **GUS search:** `Product_Tag__r.Name LIKE '%Life Sciences Cloud - Product Management%' AND Subject__c LIKE '%medication%'` (fallback tag `Health and Life Sciences`; apply build-staleness rule)

## Pattern: OmniStudio IP / DataRaptor error inside an LS flow
- **Likely subsystem:** OmniStudio runtime (see verticals/omnistudio/) + `core/industries-lifesciences-impl/omnistudio/lifesciences/`; LS heavily uses IPs for enrollment and consent
- **How to confirm:** Pull ipipr/ipdar for the failing IP/DataRaptor; isolate whether the failure is in the OmniStudio layer vs LS core services.
- **GUS search:** `Product_Tag__r.Name LIKE '%Life Sciences Cloud - Product Management%' AND Subject__c LIKE '%<keyword>%'` (apply build-staleness rule)
