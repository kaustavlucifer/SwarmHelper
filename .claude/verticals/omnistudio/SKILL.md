---
name: omnistudio
description: OmniStudio troubleshooting — OmniScript, IP, DataRaptor, FlexCard, DocGen, VBT; managed package and standard runtime. Routed by /swarm-helper.
---

# OmniStudio Troubleshooter

**Trigger:** OmniScript, FlexCard, DataRaptor, Integration Procedure, DocGen, VBT, `vlocity_cmt`, `vlocity_ins`, `omnistudio` namespace, `OmniProcess`, `OmniDataTransform`, managed package runtime issues.

---

## Critical First Step: Runtime Type

Before any investigation, determine the runtime:

| Dimension | Managed Package Runtime | Standard Runtime (Core) |
|---|---|---|
| OmniScript object | `vlocity_cmt__OmniScript__c` | `OmniProcess` (API v57+) |
| DataRaptor object | `vlocity_cmt__DRMapItem__c` | `OmniDataTransform` |
| IP object | `vlocity_cmt__OmniScript__c` (Type=IP) | `OmniIntegrationProcedure` |
| FlexCard object | `vlocity_cmt__VlocityCard__c` | `OmniUiCard` |
| Namespace | `vlocity_cmt` or `vlocity_ins` | `omnistudio` or none |
| Deployment | VBT / DataPacks | SFDX metadata (API v57+) |

Ask the engineer to run in **customer org** (NOT OrgCS):
```sql
SELECT Id, DeveloperName, Value FROM OmniInteractionConfig__mdt WHERE DeveloperName = 'OmniStudio' LIMIT 1
```

---

## Component Identification

From error, identify: OmniScript / IP / DataRaptor / FlexCard / DocGen / VBT / General

- **DocGen** → see `docgen-reference.md` in this directory
- **VBT** → see `vbt-reference.md` in this directory
- **Runtime/data model/access/stubbing** → see `reference.md` in this directory

---

## Repository Architecture

### GitHub Repos

| Repository | Path | Content |
|---|---|---|
| `via_platform` | `sf-industries/via_platform` | All managed package Apex (vlocity_cmt, vlocity_ins, omnistudio) |
| `via_platform_foundation` | `sf-industries/via_platform_foundation` | Platform foundation layer |
| `via_core` | `sf-industries/via_core` | DREngine, InvokeService, StateTransition, SecurityChecker |
| `via_oui_lwc` | `sf-industries/via_oui_lwc` | OmniStudio UI LWC components |
| `via_components` | `sf-industries/via_components` | Shared LWC library |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public:
  core/industries-interaction-ptc/apex/vlocity_cmt/    ← PTC layer (CMT namespace)
  core/industries-interaction-ptc/apex/vlocity_ins/    ← PTC layer (INS namespace)
  core/industries-interaction-ptc/apex/omnistudio/     ← PTC layer (Standard Runtime)
  core/industries-interaction-dataraptor-impl/         ← DataRaptor Java engine
  core/omnistudio-omniscript-impl/                     ← OmniScript Java
  core/ui-omnistudio-components/modules/runtime_omnistudio/        ← Runtime LWC
  core/ui-omnistudio-components/modules/runtime_omnistudio_flexcards/  ← FlexCard LWC
  core/ui-omnistudio-components/modules/builder_omnistudio_*/      ← Designer LWC
  core/ui-industries-common-components/omnistudio/     ← Shared Industries components
```

### Key Classes (Managed Package)

| Component | Classes |
|---|---|
| DataRaptor | `DREngine.cls`, `DRQueryRunner.cls`, `DRProcessor.cls`, `MatchingKeyService.cls` |
| Integration Procedure | `IntegrationProcedureService`, `IPEngine` |
| OmniScript | `OmniScriptEngine`, `OmniProcessEngine` |
| FlexCard | `FlexCardController`, `FlexcardLwcTemplateBuilder` |
| FLS | `checkFLS`, `CheckFieldLevelSecurity`, `isSelectableFieldCachedMap`, `EnforceDMFLSAndDataEncryption` |

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| OmniStudio Playbook (main) | https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650321/Omnistudio+Playbook |
| OmniScript Runtime | https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650488/ |
| Integration Procedure | https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650454/ |
| DataRaptor | https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650468/ |
| Performance | https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/668086171/ |
| Diagnostic Tool | https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/1353257910/ |
| DR Validation | https://confluence.internal.salesforce.com/spaces/CMEKB/pages/676318862/ |
| IP Issues | https://confluence.internal.salesforce.com/spaces/CMEKB/pages/675915284/ |

---

## OmniStudio-Specific Splunk logRecordTypes

| Type | Component |
|---|---|
| `r1log` | Industries package instrumentation (shared, filter by `instKey`) |
| `iposs` | OmniScript |
| `ipipr` | Integration Procedure |
| `ipipa` | IP actions/blocks |
| `ipdar` | DataRaptor |
| `ipfbc` | FlexCard |
| `izilg` | OIC flags (check `LoggingEnabled` FIRST on perf issues) |

---

## OmniInteractionConfig (OIC) Flags

| Flag | Effect |
|---|---|
| `ManagedPackageRuntime` | true = package, false = standard |
| `LoggingEnabled` | If true → severe performance degradation |
| `EnforceDMFLSAndDataEncryption` | FLS enforcement on DataMapper |
| `UseLightningOut` | Experience site rendering |

---

## Escalation

| Channel | ID | Use |
|---|---|---|
| `#support-omnistudio-collaboration` | C03GSNY2GVC | Support swarm |
| `#industries-omnistudio-collaboration` | C02LN705BEK | TMP/engineering |
| `#support-swarm-industries` | C02BEHKLWES | General swarm |
| `#omnistudio-migration-support` | C0AJACQQ6H3 | Runtime migration |

**GUS product tag:** `Industries Interaction platform`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Component: [OmniScript | IP | DataRaptor | FlexCard | Other]
Managed Package Runtime?: [Yes / No / Unknown]
Package Name: [vlocity_cmt / vlocity_ins / omnistudio]
Issue Description:
Reproduced in Demo org?:
Troubleshooting steps taken?:
Network/Debug/Splunk logs verified?:
OmniStudio Diagnostic Tool run?:
Playbook used?: [Yes/No]
```

---

## Diagnostic Recommendation

If not already run, suggest the **OmniStudio Diagnostic Tool** to the engineer for additional context.
