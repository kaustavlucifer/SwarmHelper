# Industry Cloud Support Troubleshooter

Single-command troubleshooting framework for Industry Cloud support cases. Engineers interact with `/swarm-helper` — it handles routing, investigation, and resolution.

---

## Architecture

```
/swarm-helper (single command — intelligent orchestrator)
    │
    ├── capabilities/        ← Reusable MCP integration patterns
    │   ├── REGISTRY.md      ← Tool fallback chains & auth probe definitions
    │   ├── orgcs.md         ← Case resolution, org/pod lookup (validated)
    │   ├── splunk.md        ← Log search patterns
    │   ├── codesearch.md    ← Source code investigation
    │   ├── gus.md           ← Bug tracking queries
    │   ├── slack.md         ← Internal knowledge search
    │   ├── documentation.md ← Official Salesforce docs + internal (Confluence/KB/SO)
    │   ├── columbo.md       ← Gack/exception investigation
    │   ├── monitoring.md    ← Incident & pod health check
    │   ├── metadata-analysis.md   ← Apex/Flow/OmniScript export analysis
    │   ├── debug-log-analysis.md  ← Debug log parsing rules
    │   └── har-analysis.md  ← HAR file parsing (+ PII redaction)
    │
    └── verticals/           ← Domain-specific knowledge (auto-routed, 20 verticals)
        ├── omnistudio/      ← OmniScript, IP, DataRaptor, FlexCard, DocGen, VBT
        ├── health-cloud/    ← Care plans, assessments, payer-side (UM, benefit verify), FHIR
        ├── insurance/       ← Policy admin, quoting, rating, enrollment, billing, commissions
        ├── docgen/          ← Document generation (shared: OmniStudio, Revenue Cloud, CLM)
        ├── fsc/             ← Financial accounts, action plans, referrals
        ├── comms-cloud/     ← EPC, CPQ, order management, MSM
        ├── revenue-cloud/   ← Billing, BRE, configurator, CLM, approvals
        ├── revenue-lifecycle-mgmt/ ← Asset lifecycle, quote-to-order, transactions, usage selling
        ├── cpq/             ← Salesforce CPQ (SBQQ), QCP, QLE
        ├── tpm/             ← Trade Promotion Management, Consumer Goods
        ├── life-sciences/   ← Clinical, patient services, drug programs
        ├── eu-cloud/        ← Energy & Utilities, CAM, DC APIs
        ├── media/           ← Ad Sales, media business app
        ├── manufacturing/   ← Sales agreements, forecasting, warranty, fleet
        ├── automotive/      ← Vehicle lifecycle, dealer, auto finance, VIN
        ├── public-sector/   ← Permits, licensing, inspections, complaints (PSS)
        ├── loyalty/         ← Points, tiers, rewards, vouchers, member programs
        ├── education/       ← EDA, admissions, student success, academic operations
        ├── nonprofit/       ← NPSP, fundraising, donations, grants, gift entry
        └── net-zero/        ← Carbon accounting, emissions, ESG, sustainability
```

---

## How to Use

Just type `/swarm-helper` and describe the problem:
- Paste an error message or stack trace
- Provide a case number (org + pod auto-resolved)
- Give an org ID + pod name
- Drop a debug log file in `data/`

The command runs a fast plugin health check first (~3-5s), then determines which vertical applies, which MCP sources to query, and produces a structured investigation report.

---

## How Routing Works

`swarm-helper` reads the error/case and matches signals to verticals:
- `vlocity_cmt`, `OmniScript`, `DataRaptor` → OmniStudio
- `CarePlan`, `InsuranceClaim` (payer-side), FHIR, UM → Health Cloud
- `InsurancePolicy`, `InsPolicyService`, `InsQuoteService`, rating, quoting → Insurance
- `FinancialAccount`, action plans → FSC
- EPC, order decomposition, MSM → Comms Cloud
- BRE, billing, Decision Tables → Revenue Cloud
- Asset lifecycle, RLM, sales transactions, transaction rollback, CPI uplift, usage selling, contract cotermination → Revenue Lifecycle Management
- `SBQQ__`, QCP, QLE → CPQ
- `cgcloud`, KPI batch → TPM
- And so on.

Multiple verticals can be invoked for cross-cutting issues.

**Guard Rails:** Cases with restricted support tiers (EU Premier, US Premier, Gov, US Sig, Non-Hyperforce) are blocked from AI analysis per data governance policy.

---

## File Structure

```
SwarmHelper/
├── CLAUDE.md                              ← You are here
├── data/                                  ← Drop case files here (logs, HARs, CSVs)
└── .claude/
    ├── settings.local.json               ← Tool permissions
    ├── commands/
    │   └── swarm-helper.md              ← THE command (orchestrator + router)
    ├── capabilities/                    ← Shared MCP integration patterns
    │   ├── REGISTRY.md                  ← Tool fallback chains & auth probe definitions
    │   ├── orgcs.md                     ← OrgCS queries (validated against live data)
    │   ├── splunk.md                    ← Splunk log search
    │   ├── codesearch.md               ← Code search (managed pkg + core monorepo)
    │   ├── gus.md                       ← GUS bug search
    │   ├── slack.md                     ← Slack channel search
    │   ├── documentation.md            ← Official Salesforce docs + internal (Confluence/KB/SO)
    │   ├── columbo.md                  ← Gack investigation
    │   ├── monitoring.md               ← Incident (PagerDuty) & pod health check
    │   ├── metadata-analysis.md        ← Apex/Flow/OmniScript export analysis
    │   ├── debug-log-analysis.md       ← Debug log parsing (+ PII redaction)
    │   └── har-analysis.md             ← HAR file parsing (+ PII redaction)
    └── verticals/                       ← Domain knowledge (auto-routed by swarm-helper)
        ├── omnistudio/
        │   ├── SKILL.md                 ← OmniStudio troubleshooting
        │   ├── known-errors.md          ← 50+ known error patterns with fixes
        │   ├── reference.md             ← Architecture, OIC flags, runtime, stubbing
        │   ├── docgen-reference.md      ← DocGen flavors and debugging
        │   └── vbt-reference.md         ← VBT full reference
        ├── health-cloud/
        │   ├── SKILL.md                 ← Health Cloud (payer-side)
        │   └── known-patterns.md        ← Known issue patterns
        ├── insurance/
        │   └── SKILL.md                 ← Insurance (carrier-side: policy, quoting, rating)
        ├── docgen/
        │   └── SKILL.md                 ← Document Generation (shared across clouds)
        ├── fsc/
        │   ├── SKILL.md                 ← Financial Services Cloud
        │   ├── known-patterns.md        ← Known issue patterns
        │   └── action-plans-patterns.md ← Action plan templates, tasks, dependencies
        ├── comms-cloud/
        │   ├── SKILL.md                 ← Communications Cloud
        │   ├── dc-cache-patterns.md     ← DC Regenerate Cache APIs, batch jobs
        │   ├── order-management-patterns.md  ← Order decomposition, XOM
        │   ├── performance-patterns.md  ← Timeouts, cache, SOQL limits
        │   └── promotion-discount-patterns.md ← CPQ promotions, discounts
        ├── revenue-cloud/
        │   ├── SKILL.md                 ← Revenue Cloud
        │   ├── approvals-patterns.md    ← Advanced Approvals, orchestration
        │   ├── billing-patterns.md      ← Billing schedules, invoicing, DPE
        │   ├── bre-patterns.md          ← Decision Tables, Expression Sets
        │   ├── clm-docgen-patterns.md   ← CLM contracts, DocGen templates
        │   ├── configurator-patterns.md ← Product Configurator, CML rules
        │   ├── dro-patterns.md          ← Revenue recognition, ASC 606/IFRS 15
        │   └── product-to-order-patterns.md ← Product catalog, pricing config
        ├── revenue-lifecycle-mgmt/
        │   └── SKILL.md                 ← Revenue Lifecycle Management (asset lifecycle, quote-to-order, usage selling)
        ├── cpq/
        │   ├── SKILL.md                 ← Salesforce CPQ (SBQQ)
        │   ├── known-patterns.md        ← Known issue patterns
        │   └── developer-patterns.md    ← QCP, API integration, governor limits
        ├── tpm/
        │   ├── SKILL.md                 ← Trade Promotion Management
        │   └── known-patterns.md        ← Known issue patterns
        ├── life-sciences/
        │   ├── SKILL.md                 ← Life Sciences Cloud
        │   └── known-patterns.md        ← Diagnostic patterns
        ├── eu-cloud/
        │   ├── SKILL.md                 ← Energy & Utilities Cloud
        │   └── known-patterns.md        ← Known issue patterns
        ├── media/
        │   ├── SKILL.md                 ← Media Cloud / Ad Sales
        │   └── known-patterns.md        ← Known issue patterns
        ├── manufacturing/
        │   ├── SKILL.md                 ← Manufacturing Cloud
        │   └── known-patterns.md        ← Diagnostic patterns
        ├── automotive/
        │   ├── SKILL.md                 ← Automotive Cloud
        │   └── known-patterns.md        ← Diagnostic patterns
        ├── public-sector/
        │   ├── SKILL.md                 ← Public Sector Solutions
        │   └── known-patterns.md        ← Diagnostic patterns
        ├── loyalty/
        │   ├── SKILL.md                 ← Loyalty Management
        │   └── known-patterns.md        ← Diagnostic patterns
        ├── education/
        │   ├── SKILL.md                 ← Education Cloud / EDA
        │   └── known-patterns.md        ← Diagnostic patterns
        ├── nonprofit/
        │   ├── SKILL.md                 ← Nonprofit Cloud / NPSP
        │   └── known-patterns.md        ← Diagnostic patterns
        └── net-zero/
            ├── SKILL.md                 ← Net Zero Cloud
            └── known-patterns.md        ← Diagnostic patterns
```

---

## Design Principles

1. **Single entry point** — `/swarm-helper` is the only command. No need to know which skill applies.
2. **Capabilities are reusable** — MCP patterns shared across all verticals. Change once, used everywhere.
3. **Verticals are domain knowledge** — Objects, common issues, code paths, escalation channels, playbooks, repos.
4. **Validated patterns** — OrgCS queries validated against live case data (473586604, 2026-06-01).
5. **Smart routing** — Signals in error text/case taxonomy auto-select the right vertical(s).
6. **Graceful degradation** — If a tool is unavailable, investigation continues with others.
7. **Guard rails** — Restricted tiers (EU Premier, US Premier, Gov, US Sig, Non-Hyperforce) are blocked before any data is fetched.
8. **PII protection** — Customer PII is NEVER output. Debug logs, HAR files, case comments are redacted before analysis. Only code paths, error types, and technical identifiers are reported.

---

## New User Setup

Run `/swarm-helper` — it will use whatever MCP tools are available. If a tool fails:

| Tool | Fix |
|---|---|
| OrgCS 401 | Add to `~/.claude.json`, click Authenticate |
| Splunk 401 | Run `/salesforce-trust-foundations:mcp-auth` |
| Columbo expired | It auto-runs `get_auth_status` → `refresh_auth` |
| GUS not connected | Run `/gus` and follow prompts |

---

## Codebase Access

> **Tool selection (validated 2026-06-15):** `github.com/sf-industries/*` and `gitcore.soma.salesforce.com/*` are read via **`mcp__plugin_deep-research_codesearch__*`** (it indexes both). `git.soma.salesforce.com/*` (e.g. TPM `rcgps-retail-tpm`) uses **`mcp__plugin_git-soma_vmcp-git-soma__get_file_contents`**. The `git-emu` tool is SSO-blocked (403 on everything) and is NOT a reliable path — do not route here. See `.claude/capabilities/codesearch.md` for full detail. Release branches use `{CURRENT_GA}` — resolve per codesearch.md.

| Source | Tool | Content |
|---|---|---|
| `github.com/sf-industries/via_platform` | `mcp__plugin_deep-research_codesearch__search` / `read_file` | All managed package Apex (OmniStudio, INS, CMT) |
| `github.com/sf-industries/via_ins` | `mcp__plugin_deep-research_codesearch__*` | Insurance service Apex |
| `github.com/sf-industries/via_ins_fsc` | `mcp__plugin_deep-research_codesearch__*` | Insurance ↔ FSC bridge |
| `github.com/sf-industries/via_media` | `mcp__plugin_deep-research_codesearch__*` | Media Cloud package |
| `github.com/sf-industries/via_rm` | `mcp__plugin_deep-research_codesearch__*` | Revenue Management |
| `github.com/sf-industries/via_contract` | `mcp__plugin_deep-research_codesearch__*` | CLM / Contract management |
| `github.com/sf-industries/via_docgen` | `mcp__plugin_deep-research_codesearch__*` | DocGen package |
| `core/qtc/` in `gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public` | `mcp__plugin_deep-research_codesearch__search` | Salesforce CPQ (SBQQ) core-side — no standalone package repo (validated 2026-06-15) |
| `github.com/sf-industries/via_core` | `mcp__plugin_deep-research_codesearch__*` | Platform foundation (InvokeService, DREngine) |
| `github.com/sf-industries/via_telco` | `mcp__plugin_deep-research_codesearch__*` | Comms Cloud / Telco package |
| `github.com/sf-industries/via_cpq` | `mcp__plugin_deep-research_codesearch__*` | Industries CPQ package |
| `github.com/sf-industries/via_energy` | `mcp__plugin_deep-research_codesearch__*` | Energy & Utilities package |
| `git.soma.salesforce.com/industries-rcg/rcgps-retail-tpm` | `mcp__plugin_git-soma_vmcp-git-soma__get_file_contents` | TPM managed pkg + off-core Node service (`packages/tpm/`) |
| `gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public` | `mcp__plugin_deep-research_codesearch__*` | Standard Runtime Java + PTC + LWC |
| `github.com/salesforce-internal/fsc-next-gen-apps` | `mcp__plugin_deep-research_codesearch__search` | FSC patterns |
| `github.com/salesforce-internal/cgcloud-solutions` | `mcp__plugin_deep-research_codesearch__*` | Consumer Goods / TPM |
| `github.com/sf-industries-ls/lifesciences` | `mcp__plugin_deep-research_codesearch__*` | Life Sciences managed pkg (presentation-player, prompts) |

---

## Building dist

The `dist/` folder contains a distributable zip with a versioned wrapper folder containing `.claude/`, `CHANGELOG.md`, **and an empty `data/` folder** (the drop-zone for case files). It excludes `settings.local.json` (user-specific permissions).

```bash
VERSION="v2.5.0"
rm -rf /tmp/swarm-helper-${VERSION} dist/swarm-helper-${VERSION}.zip
mkdir -p /tmp/swarm-helper-${VERSION}
cp -r .claude /tmp/swarm-helper-${VERSION}/
cp CHANGELOG.md /tmp/swarm-helper-${VERSION}/
rm -f /tmp/swarm-helper-${VERSION}/.claude/settings.local.json
# empty data/ drop-zone (.gitkeep keeps the dir in the zip)
mkdir -p /tmp/swarm-helper-${VERSION}/data
touch /tmp/swarm-helper-${VERSION}/data/.gitkeep
(cd /tmp && zip -r - swarm-helper-${VERSION}/) > dist/swarm-helper-${VERSION}.zip
rm -rf /tmp/swarm-helper-${VERSION}
```

When unzipped, recipients get: `swarm-helper-v2.5.0/.claude/` + `CHANGELOG.md` + `data/` — they copy `.claude/` and `data/` into their project root.

`dist/` is gitignored — rebuild locally before sharing.

---

## Rules

- All operations are READ-ONLY
- Every claim cites its source
- Truncate org IDs to 15 characters for Splunk
- Splunk index = pod/instance name (NOT org ID)
- ALWAYS check PTC layer for regressions
- ALWAYS check both managed package AND core for same class
