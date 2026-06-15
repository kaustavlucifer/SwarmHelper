---
name: docgen
description: Document Generation troubleshooting — DocGen templates, merge fields, tokenDataMap, PDF/Docx rendering. Shared across OmniStudio, Revenue Cloud, Insurance. Routed by /swarm-helper.
---

# Document Generation (DocGen) Debugger

**Trigger:** Document generation, DocGen, document templates, merge fields, template transform, tokenDataMap, `Workspace Not found`, template rendering, PDF generation, Docx generation. Shared feature used across OmniStudio, Revenue Cloud (CLM), Insurance, and other verticals.

> **Cross-cutting vertical:** DocGen is not a standalone product — it's a shared capability. If the issue is specifically about CLM contract documents, also load Revenue Cloud. If it's OmniScript-driven document generation, also load OmniStudio.

---

## Two Flavors of DocGen

| | Vlocity DocGen | OmniStudio DocGen |
|---|---|---|
| Package | `vlocity_cmt` / `vlocity_ins` | OmniStudio package |
| Web Templates | Supported | Not supported |
| License check | Setup → Document Generation Settings (if present = OmniStudio DocGen licensed) |
| Template format | Word (.docx) + Web | Word (.docx) only |

Some orgs have both (customer migrated from Vlocity to Standard DataModel).

---

## Product Areas

| Area | Scope |
|---|---|
| Template management | Template creation, versioning, merge fields, conditional sections |
| Document rendering | PDF/Docx generation, template transform, token resolution |
| CLM DocGen | Contract documents, clause library, e-signature integration |
| OmniStudio DocGen | IP/OS-driven generation, DataRaptor token extraction |
| Batch generation | Bulk document generation, async processing |

---

## Repository Architecture

### GitHub Repos

| Repository | Path | Content |
|---|---|---|
| `via_docgen` | `sf-industries/via_docgen` | DocGen managed package |
| `via_platform` | `sf-industries/via_platform` | OmniStudio engine (DocGen integration classes) |
| `via_contract` | `sf-industries/via_contract` | CLM/Contract DocGen |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public:
  core/industries-interaction-ptc/apex/vlocity_cmt/    ← DocGen PTC classes (CMT namespace)
  core/industries-interaction-ptc/apex/vlocity_ins/    ← DocGen PTC classes (INS namespace)
```

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| DocGen KB (Confluence) | https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650321/ |
| OmniStudio DocGen Setup | https://help.salesforce.com/s/articleView?id=ind.doc_gen_select_templates_for_clm_or_foundation_docgen.htm&type=5 |

---

## Common Errors

| Error | Likely Cause | Resolution |
|---|---|---|
| `Workspace Not found` | DocGen setup incomplete | Complete Document Generation Settings in Setup |
| `List has no rows for assignment to sObject` | Template or DataRaptor misconfiguration | Verify template DR mapping and record existence |
| `Validation Error: Exception Occurred` | Setup issue | Check Document Generation Settings, permissions |
| `Input object is not a Key value Pair` | Incorrect input structure to template | Validate tokenDataMap JSON structure |
| Template renders blank sections | Conditional merge field logic or missing data | Check conditional markers and data availability |
| PDF generation timeout | Large document or complex template | Reduce template complexity or use async generation |
| Font/formatting issues | Template format incompatibility | Use supported fonts; check PDF engine version |

---

## Debugging Technique — Stubbing tokenDataMap

For Docx Document Generation, stub token data from the customer org to a Demo org without replicating all customer data:

1. Capture the `tokenDataMap` from Console Logs while generating the document in the customer org
2. In the Demo org, add a **Set Value** step in the OmniScript and paste the entire captured `DataJSON`
3. Set the `useTemplateDRExtract` flag to **false** under generation options
4. Configure the Template Transform and copy the output path to the input path

This eliminates the need to replicate custom sObjects, DataRaptors, etc. in the Demo org.

---

## Symptom-Driven Fast Path

| Symptom | First Check |
|---|---|
| DocGen button does nothing | Browser console for JS errors; DocGen Setup completeness |
| Template merge renders blank | DataRaptor extract returning data? Token path correct? |
| "Workspace Not found" | Document Generation Settings in Setup |
| Docx generates but PDF fails | PDF engine config; font compatibility |
| CLM DocGen permission error | CLM permission sets; Named Credential config |
| Template not found | Template deployment status; active version check |
| Async generation stuck | Batch job status; governor limits on bulk generation |

---

## Sample SOQL Queries

### DocGen templates
```soql
SELECT Id, Name, IsActive, VersionNumber, Type, LastModifiedDate
FROM OmniDocumentTemplate
WHERE IsActive = true
ORDER BY Name
```

### Document generation jobs (async)
```soql
SELECT Id, Status, CreatedDate, CompletedDate, NumberOfErrors
FROM AsyncApexJob
WHERE ApexClass.Name LIKE '%DocGen%'
ORDER BY CreatedDate DESC LIMIT 10
```

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `ipipr` | Integration Procedures (DocGen orchestration) |
| `ipdar` | DataRaptors (token extraction, data pull) |
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex exceptions (template processing, PDF generation) |
| `axlim` | Governor limits (large document batch generation) |

---

## Code Investigation Paths

### DocGen Package
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "github.com/sf-industries/via_docgen"
ref: "HEAD"
file_path: "classes/<ClassName>.cls"
```

### CLM / Contract DocGen
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "github.com/sf-industries/via_contract"
ref: "HEAD"
file_path: "classes/<ClassName>.cls"
```

### DocGen in via_platform
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:github.com/sf-industries/via_platform DocGen"
```

### PTC Layer (DocGen classes)
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public content:DocumentGeneration path:industries-interaction-ptc"
max_matches: 10
```

---

## Escalation

| Channel | Use |
|---|---|
| `#support-omnistudio-collaboration` (C03GSNY2GVC) | OmniStudio DocGen issues |
| `#support-rev-dev-amer` (C0275QG40LE) | CLM DocGen / Revenue Cloud |
| `#support-swarm-industries` (C02BEHKLWES) | General swarm |

**GUS product tags:** `Industries Interaction platform`, `Revenue Cloud`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
DocGen Flavor: [OmniStudio DocGen | Vlocity DocGen | CLM DocGen | Unknown]
Template Type: [Word/Docx | Web | PDF]
Issue Description:
Template name:
Reproduced in Demo org?:
Troubleshooting steps taken?:
tokenDataMap captured?:
```
