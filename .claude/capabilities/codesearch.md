# Capability: Code Search — Source Code Investigation

## Two Toolsets

> **Validated 2026-06-15.** Both tools are general code search backed by *different indexes* — they are NOT cleanly split by repo. `deep-research` reliably resolves `github.com/sf-industries/via_platform` AND the `gitcore` core monorepo, so it is the **primary** for almost everything. `mcp-adaptor` indexes a managed-package mirror under a different repo name (`github.com/sf-industries/vlocity-prod-pkg-source`) and returns 0 for a `repo:github.com/sf-industries/via_platform` filter — use it only as a secondary/cross-check for managed-package Apex.

| Tool | Index actually covers | Use For |
|---|---|---|
| `mcp__plugin_deep-research_codesearch__search` / `read_file` / `list_directory` / `commit_search` / `get_commit_diff` / `find_references` / `go_to_definition` | `gitcore.soma.salesforce.com` core monorepo **and** `github.com/sf-industries/via_*` | **Primary** — managed package Apex (via_platform: vlocity_cmt, vlocity_ins, omnistudio) AND core monorepo (Standard Runtime Java/Apex/LWC, PTC layer) |
| `mcp__mcp-adaptor__search` / `read_file` / `list_directory` | `github.com/sf-industries/vlocity-prod-pkg-source` (managed-pkg mirror) | Secondary cross-check for managed package Apex; do NOT filter on `via_platform` (returns 0) |

## Auth Troubleshooting

- CodeSearch tools require no explicit auth (SSO-passed)
- `mcp__mcp-adaptor__*` tools require MCP adaptor to be connected
- If either returns empty results, check the code host / repo name — note the two tools use DIFFERENT repo namespaces (see above)
- The `repo:` filter is a **substring** match, not exact — anchor with `repo:^...$` to disambiguate (e.g. `via_ins` also matches `via_ins_ps`, `via_ins_fsc`)

---

## Tool 1: Managed Package Source (`via_platform` via deep-research codesearch)

**Repository:** `github.com/sf-industries/via_platform` — resolves in the **deep-research** index (validated 2026-06-15). Contains all Apex for vlocity_cmt, vlocity_ins, omnistudio (PS), and telco packages.

### Search
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:github.com/sf-industries/via_platform DREngine"
regex: false   # false = literal/exact; true = regex pattern
```

### Read a file
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "github.com/sf-industries/via_platform"
ref: "HEAD"
file_path: "classes/DREngine.cls"
start_line: 1   # optional
end_line: 100   # optional
```

> The `mcp-adaptor` toolset can also search managed-package Apex, but via the `vlocity-prod-pkg-source` mirror (different repo name) — use it only as a secondary cross-check.

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
query: "repo:gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public content:<keyword> lang:java"
max_matches: 10
```

### Read a file
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_cmt/DataRaptorUtilsPtc.apex"
```

### List directory
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_cmt/"
```

### Check commit history (MANDATORY for regression analysis)
```
Tool: mcp__plugin_deep-research_codesearch__commit_search
code_host: "gitcore.soma.salesforce.com"
org: "core-2206"
repo: "core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
path: "core/industries-interaction-ptc/apex/vlocity_cmt/DataRaptorUtilsPtc.apex"
```

### Find references (where a symbol is used)
```
Tool: mcp__plugin_deep-research_codesearch__find_references
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "<file>"
line: <line_number>
```

### Go to definition
```
Tool: mcp__plugin_deep-research_codesearch__go_to_definition
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "<file>"
line: <line_number>
```

### Get commit diff (view what changed in a specific commit)
```
Tool: mcp__plugin_deep-research_codesearch__get_commit_diff
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
commit_sha: "<sha>"
```

> **Note:** There is no `blame` tool available. For code ownership, use `commit_search` on the file path and check recent commit authors.

---

## Release Resolution (do this FIRST — release numbers change every ~4 months)

Salesforce ships **three major releases per year** (Winter / Spring / Summer), and the core branch number **increments by 2** each release (`…258 → 260 → 262 → 264 → 266 → 268…`). Branch names like `core-262-public` and refs like `refs/heads/release-262` therefore go stale roughly every 4 months. **Never hardcode a release number into a query.** Instead, resolve `{CURRENT_GA}` at the start of an investigation and substitute it everywhere a skill file shows `{CURRENT_GA}` / `core-{CURRENT_GA}-public` / `release-{CURRENT_GA}`.

### Step 1 — resolve from the calendar (no probe needed)

Salesforce GA dates are roughly: **Winter ≈ mid-October**, **Spring ≈ mid-February**, **Summer ≈ mid-June**. Map today's date against this table (verified against official sources 2026-06-15):

| Release | Season | GA (approx) | `{CURRENT_GA}` during this window |
|---|---|---|---|
| 258 | Winter '26 | 2025-10-11 | 258 (from ~2025-10-11 to ~2026-02-13) |
| 260 | Spring '26 | 2026-02-13 | 260 (to ~2026-06-15) |
| 262 | Summer '26 | 2026-06-15 | **262 (to ~2026-10-16)** |
| 264 | Winter '27 | ~2026-10-16 | 264 (to ~2027-02) |
| 266 | Spring '27 | ~2027-02 | 266 (to ~2027-06) |
| 268 | Summer '27 | ~2027-06 | 268 (to ~2027-10) |
| 270 | Winter '28 | ~2027-10 | 270 |
| 272 | Spring '28 | ~2028-02 | 272 |
| 274 | Summer '28 | ~2028-06 | 274 |

`{CURRENT_GA}` = the release whose GA date is the most recent one **on or before today**. `{IN_DEV}` = `{CURRENT_GA} + 2`. `{PREV}` = `{CURRENT_GA} − 2`.

> Dates past 274 follow the same cadence (+2 every release, 3/year). If today is beyond this table, extrapolate: each calendar year adds 6 to the release number (e.g. Summer '28 = 274, Summer '29 = 280).

### Step 2 — confirm by probe (when precision matters near a boundary)

Within ~2 weeks of a GA date, or any time the calendar answer is ambiguous, confirm which branch is actually cut:

```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-{candidate}-public"
directory_path: "core"
```
The highest-numbered `core-N-public` that lists successfully is `{IN_DEV}`; the one below it that is GA-tagged is `{CURRENT_GA}`. (Probed 2026-06-15: `core-262-public` and `core-264-public` both resolve — 262 = GA, 264 = in-dev.) For git.soma TPM, the analog is `mcp__plugin_git-soma_vmcp-git-soma__list_branches` on `industries-rcg/rcgps-retail-tpm` looking for `release-{N}`.

### Substitution

| Placeholder in skill files | Resolves to (as of 2026-06-15) |
|---|---|
| `core-{CURRENT_GA}-public` / `p4/{CURRENT_GA}-patch` | `core-262-public` / `p4/262-patch` — **current GA, default** |
| `core-{IN_DEV}-public` | `core-264-public` — upcoming / in development |
| `core-{PREV}-public` | `core-260-public` — previous |

Use `{CURRENT_GA}` unless specifically checking upcoming changes.

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
| Salesforce CPQ (SBQQ) | Core monorepo `core/qtc/` — SBQQ package source not in indexed git hosts (search `SBQQ__` via codesearch) |

---

## Regression Analysis (MANDATORY)

For every class in an error call stack:
1. **Managed package class** → read via `mcp__plugin_deep-research_codesearch__read_file` (repo `github.com/sf-industries/via_platform`)
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

> **Repo hosts validated 2026-06-15.** Health Cloud and FSC source migrated into the **core monorepo** (`core/industries-healthcare-impl/`, `core/ui-fsc-components/`) — their old `git.soma/industries/*` package repos no longer resolve. Salesforce CPQ (SBQQ) package source is not in any indexed git host (search `SBQQ__` symbols in core `core/qtc/`).

| Vertical | Primary Repo Host |
|---|---|
| Health Cloud | Core monorepo `core/industries-healthcare-impl/` (codesearch) |
| FSC | Core monorepo `core/ui-fsc-components/` (codesearch) + `github.com/sf-industries/via_ins_fsc` |
| TPM/Consumer Goods | `git.soma.salesforce.com/industries-rcg/rcgps-retail-tpm` |
| Life Sciences | `github.com/sf-industries-ls/lifesciences` |
| OmniStudio/Insurance/Comms/E&U/Media | `github.com/sf-industries/via_*` |
| Salesforce CPQ (SBQQ) | Core monorepo `core/qtc/` (codesearch; no standalone package repo) |

---

## Wiki Summaries (code context)

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
