# OmniStudio Reference

Architecture, runtime settings, data models, access patterns, playbook URLs, escalation channels, and debugging techniques for OmniStudio troubleshooting.

---

## Component Architecture

### OmniScript
Declarative wizard/form builder rendered as LWC (Standard Runtime) or Aura (Package Runtime).
- Compilation generates LWC bundle at activation via Puppeteer/VF
- Experience Sites: `stsci` log type; OmniProcess Read access required
- Key failures: compilation errors, blank screens post-upgrade, 4MB LWC payload limit, LWS interference

### Integration Procedure (IP)
Server-side data processing pipeline — no UI. Invoked by OmniScript, Apex, REST, or Flow.
- Apex class: `omnistudio.IntegrationProcedureService` (standard) / `vlocity_cmt.IntegrationProcedureService` (package)
- Async mode: `Invoke Mode = Non-Blocking` — use `VlocityNoRootNode` as Response JSON Node
- Key failures: DML before callout, HTTP retryCount bug, BigDecimal cast, inactive procedure

### DataRaptor / Data Mapper
SOQL-based ETL — Extract, Load, Transform, Turbo Extract.
- Objects: `OmniDataTransform` (standard) / `vlocity_cmt__DRMapItem__c` (package)
- Key failures: IsActive=false, OWD on DRMapItems__c, external objects unsupported, FLS enforcement

### FlexCard
Declarative card UI — Lightning pages, Experience sites, OmniScript.
- Objects: `OmniUiCard` (standard) / `vlocity_cmt__VlocityCard__c` (package)
- Key failures: parameter corruption, datasource variable resolution, null getflexCardMetadata post-refresh

### OmniInteractionConfig (OIC)
Master configuration flags — check ALL in the **customer org** (NOT OrgCS):
```sql
SELECT Id, DeveloperName, Label, Value FROM OmniInteractionConfig__mdt ORDER BY DeveloperName
```

| Flag | Effect |
|---|---|
| `ManagedPackageRuntime` | true = package runtime, false = standard |
| `LoggingEnabled` | If true → severe performance degradation (set to false) |
| `EnforceDMFLSAndDataEncryption` | FLS enforcement on DataMapper fields |
| `UseLightningOut` | Affects Experience site rendering |
| `UseStandardRuntime` | Transitional flag during migration |

---

## Runtime Settings

### Managed Package Runtime
- OmniScripts and FlexCards **deploy a custom LWC upon activation**
- Generated LWC can be embedded on record pages like any other custom LWC

### Standard Runtime
- OmniScripts and FlexCards do **NOT** automatically deploy custom LWC on activation
- Can be used without generated LWC via the Standard OmniScript / Standard FlexCard component
- To generate LWC manually: open the component → upper-right arrow → deploy Standard Runtime-compatible LWC
- To auto-deploy on activation: enable **"Deploy Custom Lightning Web Components in Standard Runtime"**

---

## Data Models

### Custom Data Model (Legacy)
| Object | Stores |
|---|---|
| `Vlocity OmniScript` | OmniScripts & IPs |
| `Vlocity DataRaptor Bundle` | DataRaptors |
| `Vlocity Cards` | FlexCards & Vlocity Cards |

### Standard Data Model (Newer)
| Object | Stores |
|---|---|
| `OmniProcess` | OmniScripts & IPs |
| `OmniUiCard` | FlexCards |
| `OmniDataTransform` | DataRaptors |

---

## Package Structure

| Package | Contains |
|---|---|
| CMT Package | OmniStudio Designer + MP Classes + CMT-specific MP Class |
| INS Package | OmniStudio Designer + MP Classes + INS-specific MP Class |
| OmniStudio Package | OmniStudio Designer + MP Classes |

---

## Identifying Org Data Model

1. If org has OmniStudio license → **OmniStudio Settings** and **Omni Interaction Configuration** are visible in Setup
2. If not visible → no OmniStudio license
3. Check `OmniInteractionConfig__mdt` entries:

| Scenario | OIC Entries |
|---|---|
| OmniStudio package only | `TheFirstInstalledOmniPackage` = `omnistudio` |
| CMT/INS on Custom Data Model, no OmniStudio license | OmniStudio Settings not visible |
| CMT/INS on Custom Data Model, OmniStudio license present | OmniStudio Settings visible, no OIC entries |
| CMT/INS on Standard Data Model | `TheFirstInstalledOmniPackage` + `InstalledIndustryPackage` (both = CMT/INS namespace) |

---

## Access Issues

### Diagnostic Query
```sql
SELECT RecordId, HasReadAccess, HadEditAccess
FROM UserRecordAccess
WHERE UserId = '<user_id>' AND RecordId = '<record_id>'
LIMIT 200
```

### Internal Users — Check All of:
- Managed Package Classes access
- Managed Package VF Pages access
- sObject access for OmniStudio objects
- OmniStudio Permission Set License
- System Permission for OmniStudio (Standard Runtime only)

### Community Site Users — Check All Internal +:
- Sharing rules for OmniStudio sObjects
- OmniStudio Community license

### Community Guest Users — Check All Community +:
- Sharing rules for OmniStudio sObjects (OWDs NOT enforced on Guest Users — explicit sharing rules required)
- Create a Permission Set, assign everything, and test

---

## Debugging — Stubbing OmniScript Data

For issues in a later step of a multi-step OmniScript (avoids re-running all steps every time):

1. Preview the OS and go through steps until you reach the problem step
2. Copy the `OmniDataJSON` from that point
3. Create a new version of the OmniScript
4. Disable all steps and elements **before** the problem step
5. Add a **Set Value** element first and paste the captured JSON
6. Preview — the OS will start with pre-populated data at the problem step

> Note: may not work if the flow must create records or run validations before the problem step.

---

## Common LWC Activation Errors

| Error | Cause / Action |
|---|---|
| `forceGenerated-******** Error` | Missing access or improper activation/deployment |
| `No MODULE markup found` | Missing access or improper activation/deployment |
| `List Index Out of Bounds` | Missing access or improper activation/deployment |

**For all three above:** Check Network Logs — the callout response usually contains the exact error. Capture Managed Package Debug logs and check for access exceptions.

---

## Splunk logRecordTypes for OmniStudio

| logRecordType | Component | Availability |
|---|---|---|
| `iposs` | OmniScript | Standard Runtime 242 core+; Package Runtime 244+ |
| `ipipr` | Integration Procedure | Package Runtime 242+; Standard Runtime 244+ |
| `ipipa` | IP actions/blocks | Same as ipipr |
| `ipdar` | DataRaptor | Package Runtime 242+; Standard Runtime 242 core+ |
| `ipfbc` | FlexCard | Confirm with engineering if no results |
| `izilg` | OIC settings/feature flags | All runtimes |
| `stsci` | Experience Sites | All runtimes |
| `ipcul` | Component usage instrumentation | All runtimes |

---

## Confluence Playbook URLs

- Main: `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650321/Omnistudio+Playbook`
- OmniScript Designer: `https://confluence.internal.salesforce.com/display/OMNISTUDIO/Plays%3A%2BOmniscript%2BDesigner`
- OmniScript Runtime: `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650488/Plays+Omniscript+Runtime`
- Integration Procedure: `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650454/Plays+Integration+Procedure`
- DataRaptor: `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650468/Plays+DataRaptor`
- Configuration: `https://confluence.internal.salesforce.com/display/OMNISTUDIO/Plays%3A+Configuration`
- Performance: `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/668086171/Plays+Performance`
- CMEKB IP Issues: `https://confluence.internal.salesforce.com/spaces/CMEKB/pages/675915284/Integration+Procedure+Issues+and+Debugging`
- CMEKB DR Validation: `https://confluence.internal.salesforce.com/spaces/CMEKB/pages/676318862/Dataraptor+Validation`
- Diagnostic Tool: `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/1353257910/Omnistudio+Diagnostic+Tool`

---

## Escalation Channels

| Issue Type | Channel | ID |
|---|---|---|
| General OmniStudio support | `#support-omnistudio-collaboration` | C03GSNY2GVC |
| TMP/engineering escalation | `#industries-omnistudio-collaboration` | C02LN705BEK |
| Swarm escalation | `#support-swarm-industries` | C02BEHKLWES |
| General OmniStudio engineering | `#omnistudio` | C01H0U55GD8 |
| Migration to Core Runtime | `#omnistudio-migration-support` | C0AJACQQ6H3 |
| Spring '26 changes | `#omnistudio-support` | C0A693YH82H |
| OmniScript-specific | `#industries-omniscript` | C026ZKUL5EG |
| Revenue Cloud | `#support-rev-dev-amer` | — |
| Government/GIA | `#cce-gia` | — |
| Splunk/Monitoring | `#moncloud-support` | — |

**Named SMEs:** Tilak Gujjer (permissions, OIC flags), Krishna Kruthiventi (migration to core), Dileep Bommineni (APAC swarm, migration readiness)

---

## Swarm Escalation Template

When filing in `#support-swarm-industries`:
```
Customer Sentiment:
Current Condition:
Component causing the issue: [OmniScript | Integration Procedure | DataRaptor | FlexCard | Other]
Managed Package Runtime enabled?: [Yes / No / Unknown]
Package Name: [vlocity_cmt / vlocity_ins / omnistudio / other]
Issue Description:
Reproduced in Demo org?:
If not, Why?:
Troubleshooting steps taken?:
Network logs, Debug logs and Splunk logs Verified?:
OmniStudio Diagnostic Tool run?:
Have you used the Omnistudio Troubleshooting Playbook?: [Yes/No — link: https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650321/Omnistudio+Playbook]
```

---

## Performance Tools

1. **Ferret** — Apex Log Analyzer (VS Code extension or `http://ferret-prod.herokuapp.com`)
2. **IDX Profiler** — Step-level timing for OS/IP/DR
3. **Chrome DevTools Network** — Copy request as cURL to replay individual API calls
4. **`LoggingEnabled` flag** — First thing to check for perf issues; if true, set to false

---

## Demo Org Provisioning

- Heroku QSignups: `https://q-central.herokuapp.com/q-signups`
- Template IDs: `https://confluence.internal.salesforce.com/pages/viewpage.action?spaceKey=OMNISTUDIO&title=Production`

---

## CodeSearch for OmniStudio

OmniStudio UI components live in the core monorepo:
```
code_host: gitcore.soma.salesforce.com
org: core-2206
repo: core-262-public
ref: p4/262-patch
path: core/ui-omnistudio-components/
```

Managed package Apex (`vlocity_cmt`, `vlocity_ins`) is readable via `mcp__mcp-adaptor__read_file` and `mcp__mcp-adaptor__search` against `github.com/sf-industries/via_platform` — always check this for any class in the call stack. The `mcp__plugin_deep-research_codesearch__*` tools return zero results for managed package classes (those tools only cover the core monorepo at gitcore).
