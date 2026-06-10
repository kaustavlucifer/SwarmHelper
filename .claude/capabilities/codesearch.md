# Capability: Code Search — Source Code Investigation

## Two Toolsets

| Tool | Access | Use For |
|---|---|---|
| `mcp__mcp-adaptor__search` / `read_file` / `list_directory` | `github.com/sf-industries/via_platform` | **Managed package Apex** (vlocity_cmt, vlocity_ins, omnistudio) |
| `mcp__plugin_deep-research_codesearch__search` / `read_file` / `list_directory` / `commit_search` / `get_commit_diff` / `find_references` / `go_to_definition` | `gitcore.soma.salesforce.com` | **Core monorepo** (Standard Runtime Java/Apex/LWC, PTC layer) |

## Auth Troubleshooting

- CodeSearch tools require no explicit auth (SSO-passed)
- `mcp__mcp-adaptor__*` tools require MCP adaptor to be connected
- If either returns empty results, check that the code host / repo is correct

---

## Tool 1: Managed Package Source (`mcp-adaptor`)

**Repository:** `github.com/sf-industries/via_platform`

Contains all Apex for vlocity_cmt, vlocity_ins, omnistudio (PS), and telco packages.

### Search
```
Tool: mcp__mcp-adaptor__search
query: "repo:github.com/sf-industries/via_platform content:<keyword>"
regex: false   # exact match
regex: true    # regex pattern
```

### Read a file
```
Tool: mcp__mcp-adaptor__read_file
repository: "github.com/sf-industries/via_platform"
ref: "HEAD"
file_path: "classes/DREngine.cls"
start_line: 1   # optional
end_line: 100   # optional
```

### List directory
```
Tool: mcp__mcp-adaptor__list_directory
repository: "github.com/sf-industries/via_platform"
directory_path: "classes"
```

### Key Classes by Component

| Component | Classes |
|---|---|
| DataRaptor | `DREngine.cls`, `DRProcessor.cls`, `DRQueryRunner.cls`, `DREngineUtilitiesFoundation.cls`, `DRDataModelUtility.cls`, `DRGlobal.cls`, `DRQueryService.cls` |
| FLS / Permissions | `MatchingKeyService.cls`, `PlatformSecurityChecker.cls`, `DRUtilityService.cls` |
| Platform Cache | `ScaleCacheService.cls`, `OrgCacheManager.cls` |
| VBT / DataPacks | `DRDataPackService.cls`, `DRDataPackServiceInternal.cls` |
| Schema | `ObjectDescriberV2.cls`, `PlatformObjectMappings.cls`, `NamespaceUtilities.cls` |

### Package Manifests
- `package_ins.xml` — Insurance (INS)
- `package_os.xml` — OmniStudio base
- `package_ps.xml` — Public Sector
- `package_telco.xml` — Telco / Communications

---

## Tool 2: Core Monorepo (deep-research codesearch)

### Search
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-262-public content:<keyword> lang:java"
max_matches: 10
```

### Read a file
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_cmt/DataRaptorUtilsPtc.apex"
```

### List directory
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_cmt/"
```

### Check commit history (MANDATORY for regression analysis)
```
Tool: mcp__plugin_deep-research_codesearch__commit_search
code_host: "gitcore.soma.salesforce.com"
org: "core-2206"
repo: "core-262-public"
ref: "p4/262-patch"
path: "core/industries-interaction-ptc/apex/vlocity_cmt/DataRaptorUtilsPtc.apex"
```

### Find references (where a symbol is used)
```
Tool: mcp__plugin_deep-research_codesearch__find_references
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "<file>"
line: <line_number>
```

### Go to definition
```
Tool: mcp__plugin_deep-research_codesearch__go_to_definition
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "<file>"
line: <line_number>
```

### Get commit diff (view what changed in a specific commit)
```
Tool: mcp__plugin_deep-research_codesearch__get_commit_diff
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
commit_sha: "<sha>"
```

> **Note:** There is no `blame` tool available. For code ownership, use `commit_search` on the file path and check recent commit authors.

---

## Release Versions

| Repo | Branch | Release |
|---|---|---|
| `core-262-public` / `p4/262-patch` | **Current GA (default)** | Spring '26 |
| `core-264-public` / `p4/264-patch` | Upcoming (in development) | Summer '26 |
| `core-260-public` / `p4/260-patch` | Previous | Winter '26 |

Always use `core-262-public` unless specifically checking upcoming changes.

---

## Key Paths in Core Monorepo

### PTC Layer (Protected Apex — package ↔ Java bridge)
Path: `core/industries-interaction-ptc/apex/<namespace>/`
Namespaces: `vlocity_cmt`, `vlocity_ins`, `omnistudio`, `vlocity_ins_fsc`, `vlocity_fsc_gs0`, `vlocity_erg`, `vlocity_ps`, `vlocity_bmk`

Key PTC files:
```
DataRaptorUtilsPtc.apex          OmniScriptServicePtc.apex
IntegrationProcedureUtilsPtc.apex FlexcardUtilsPtc.apex
OmniStudioLoggingServicePtc.apex CPQServicePtc.apex
CalculationServicePtc.apex       InsuranceClaimServicePtc.apex
AssessmentResponsesPtc.apex      OrchestrationPlanCompositionServicePtc.apex
```

### Engine & Runtime (Java)
| Path | Contents |
|---|---|
| `core/industries-interaction-dataraptor-impl/` | DataRaptor Java engine |
| `core/omnistudio-omniscript-impl/` | OmniScript Java |
| `core/ui-omnistudio-impl/` | OmniStudio Apex + Java glue |

### LWC Components
| Path | Contents |
|---|---|
| `core/ui-omnistudio-components/modules/runtime_omnistudio/` | Runtime OmniScript LWC |
| `core/ui-omnistudio-components/modules/runtime_omnistudio_flexcards/` | FlexCard runtime |
| `core/ui-omnistudio-components/modules/builder_omnistudio_*/` | Designer LWC |

### Vertical-Specific Paths (CONFIRMED via CodeSearch)
| Path | Vertical |
|---|---|
| `core/ui-fsc-components/omnistudio/` | FSC |
| `core/industries-healthcare-impl/` | Health Cloud |
| `core/industries-communications-msm-impl/omnistudio/` | Communications/Telco |
| `core/ui-industries-cpq-components/` | CPQ / Revenue Cloud |
| `core/ui-industries-public-sector-components/omnistudio/` | Public Sector |
| `core/industries-manufacturing/` | Manufacturing Cloud |
| `core/industries-automotive/` | Automotive Cloud |
| `core/industries-loyalty/` | Loyalty Management |
| `core/industries-education/` | Education Cloud (9 modules) |
| `core/industries-nonprofit-*` | Nonprofit Cloud |
| `core/industries-sustainability/` | Net Zero Cloud |
| `core/ui-industries-digitallending-components/omnistudio/` | Digital Lending |
| `core/ui-industries-common-components/omnistudio/` | Shared Industries |

### Managed Package Repos (Non-via_platform)
| Vertical | Repo |
|---|---|
| Public Sector | `github.com/sf-industries/via_ps` (Apex package, vlocity_ps namespace) |
| Net Zero Cloud | `github.com/sf-industries/Sustainability-App` |
| Insurance | `github.com/sf-industries/via_ins` |
| Insurance-FSC Bridge | `github.com/sf-industries/via_ins_fsc` |
| Comms/Telco | `github.com/sf-industries/via_telco` |
| Energy | `github.com/sf-industries/via_energy` |
| Media | `github.com/sf-industries/via_media` |
| Revenue Mgmt | `github.com/sf-industries/via_rm` |
| CLM/Contracts | `github.com/sf-industries/via_contract` |
| DocGen | `github.com/sf-industries/via_docgen` |
| Industries CPQ | `github.com/sf-industries/via_cpq` |
| Salesforce CPQ | `git.soma.salesforce.com/Steelbrick/CPQ` |

---

## Regression Analysis (MANDATORY)

For every class in an error call stack:
1. **Managed package class** → read via `mcp__mcp-adaptor__read_file`
2. **PTC class** → read via `mcp__plugin_deep-research_codesearch__read_file`, then check commits with `commit_search`
3. **Core Java class** → search via `mcp__plugin_deep-research_codesearch__search`

**Interpreting commit history results:**
- Changes in last 30 days → potential regression
- Changes in last 90 days → possible regression (correlate with report date)
- No changes in 6+ months → not code regression; focus on config/data
- Note commit author as escalation contact

---

## GitHub EMU / Internal Repos

For repos beyond via_platform:
```
mcp__plugin_git-emu_vmcp-git-emu__search_repositories   — find repos on GitHub EMU
mcp__plugin_git-emu_vmcp-git-emu__get_file_contents     — read file from GitHub EMU repo
mcp__plugin_git-soma_vmcp-git-soma__search_repositories — find repos on git.soma
mcp__plugin_git-soma_vmcp-git-soma__get_file_contents   — read file from git.soma repo
```

**Authoritative repo mapping:** https://confluence.internal.salesforce.com/spaces/IN/pages/1079189041/Industries+Managed+Packages+and+Repository+Mapping

### Key Repo Hosts by Vertical

| Vertical | Primary Repo Host |
|---|---|
| Health Cloud | `git.soma.salesforce.com/industries/healthcare` |
| FSC | `git.soma.salesforce.com/industries/wealth1` |
| TPM/Consumer Goods | `git.soma.salesforce.com/industries-rcg/rcg-retail-tpm` |
| Life Sciences | `github.com/sf-industries-ls/lifesciences` |
| OmniStudio/Insurance/Comms/E&U/Media | `github.com/sf-industries/via_*` |
| Salesforce CPQ (SBQQ) | `git.soma.salesforce.com/Steelbrick/CPQ` |

---

## Tool 3: Alternative CodeSearch (Fallback)

If `mcp__plugin_deep-research_codesearch__*` tools are unavailable, use:

```
Tool: mcp__plugin_codesearch_codesearch__search
query: "content:<keyword> lang:java"
max_matches: 10
```

This tool searches the same core monorepo but through a different plugin. Already in permissions.

### Wiki Summaries (code context)
```
Tool: mcp__plugin_deep-research_codesearch__search_wiki_summaries
query: "OmniStudio DataRaptor"
max_results: 5
```

---

## Not Accessible / Returns Empty

| Source | Status |
|---|---|
| `git.soma.salesforce.com` OmniStudio | 0 results |
| `github.com/sf-industries-hc` | Not indexed |
| `github.com/salesforce-cts` | Not indexed |
| `github.com/forcedotcom` managed package | 0 results |
| Perforce / Tableau repos | No OmniStudio content |
