# SwarmHelper Changelog

These skills are developed and maintained by Kaustav Chowdhury.

## v2.1.0 — 2026-06-15

**Consolidation + live validation pass.** All repo paths and Confluence playbooks validated against live MCP sources; structure standardized; tool surface pruned.

### Fixed (repo paths — validated via live codesearch/git probes 2026-06-15)
- **CPQ:** removed non-existent `Steelbrick/CPQ` + `CPQ-REST` repos; SBQQ core-side source pointed at `core/qtc/` in the core monorepo (no standalone package repo exists)
- **Health Cloud / FSC:** removed non-existent `git.soma/industries/healthcare` + `industries/wealth1` + `industries/core` + `industries/build`; pointed at validated core monorepo paths (`core/industries-healthcare-impl/`, `core/ui-fsc-components/`) — these clouds migrated into core
- **TPM:** corrected primary repo `industries-rcg/rcg-retail-tpm` → `industries-rcg/rcgps-retail-tpm`; removed non-existent `rcg-retail-se` + `RCGSF_SF_Mobility_Sync`
- **FSC playbook:** corrected Confluence pageId `187545535` → `187545546`
- Updated `CLAUDE.md` Codebase Access table + Phase 5 to match

### Changed
- **codesearch.md rewritten:** documented the *actual* tool→index mapping — `deep-research_codesearch` is primary (covers `via_platform` + core monorepo); `mcp-adaptor` indexes the `vlocity-prod-pkg-source` mirror, not `via_platform`. Added `repo:` substring-match caveat.
- **All 20 vertical `SKILL.md` now have YAML frontmatter** (`name` + `description`); normalized lead section to `## Product Areas` (except intentional domain-specific first steps)
- **FSC dedupe:** removed ~900 lines of Action Plan content duplicated between `known-patterns.md` and `action-plans-patterns.md`; `known-patterns.md` now points to the authoritative file and keeps only FSC-wide patterns (Referrals, RSR, Financial Accounts, Timeline, Households, Goals)
- **CPQ case-volume stats** date-stamped ("snapshot ~May 2026") so staleness is visible
- Removed personal local paths (`/Users/...`) from comms-cloud pattern files; replaced with repo references

### Removed
- 2 duplicate MCP tool permissions (`dxmcp-gus`, `plugin_codesearch_codesearch`) + 1 unreferenced (`deep-research_search__doc_parse`); settings allow-list 56 → 53
- Codesearch "Tool 3 fallback" section (consolidated to single-path)

---

## v2.0.2 — 2026-06-10

### Changed
- **Phase 0 rewritten:** Clean per-tool status list (✓/⚠️) derived from system context — zero probe calls, instant startup
- Missing tools now show actionable `/mcp` hint instead of generic "pending" label
- Fixed false-pending bug: deferred tools are available, not pending

---

## v2.0.1 — 2026-06-10

### Changed
- **Phase 0 optimized:** Reads MCP connection status from system context instead of making 9 probe calls per invocation (zero-cost startup)
- Added "Building dist" section to CLAUDE.md documenting zip contents (.claude/ only)

---

## v2.0.0 — 2026-06-10

**Verticals:** 12 → 20 | **Capabilities:** 9 → 11 | **MCP probes:** 0 → 9

### Added
- **8 new verticals:** Revenue Lifecycle Mgmt, Manufacturing, Automotive, Public Sector, Loyalty, Education, Nonprofit, Net Zero
- **MCP Health Check (Phase 0):** 9 tool probes at start of every invocation with status table
- **ProductTopic-based routing:** Primary routing signal when case taxonomy available from OrgCS
- **Monitoring capability:** PagerDuty incidents + pod health check
- **Metadata analysis capability:** Apex/Flow/OmniScript export analysis, SF CLI integration
- **Case trend analysis:** Recurring issue detection via OrgCS historical query
- **`r1log` instrumentation:** Industries-specific Splunk logRecordType added to all verticals
- **Falcon/off-platform logging:** `distapps` index for Revenue Cloud and TPM services
- **Verified Slack channels:** 50+ channels with IDs confirmed via Slack MCP search
- **Complete PTC inventory:** 102 PTC classes (54 vlocity_cmt + 48 vlocity_ins) documented
- **Confirmed repo paths:** CodeSearch-validated core monorepo paths for all verticals
- Swarm templates for all 20 verticals
- Sample SOQL queries for all 20 verticals
- Code investigation paths (MCP tool invocations) for all 20 verticals

### Changed
- All existing Silver verticals (FSC, Comms, Revenue Cloud, CPQ, TPM, E&U, Media, Life Sciences, HC) enriched to Gold standard
- Health Cloud `known-patterns.md` rewritten with 12 real troubleshooting patterns (was empty template)
- Comms Cloud `order-management-patterns.md` restructured (removed 776 lines of dev methodology)
- Revenue Cloud `dro-patterns.md` cleaned (removed 190 lines of generic boilerplate)
- Phase 4 investigation table expanded (added Monitoring, Metadata, Case Trends)
- `settings.local.json` updated with 5 new tool permissions

### Removed
- 1,799 lines of boilerplate across 16 files (Update Cadence, Prerequisites & Tool Access, Test Cases, Skill Completeness Reports, Success Metrics)
- "Not yet covered" note from routing table (all verticals now covered)

---

## v1.0.0 — 2026-06-01

**Verticals:** 12 | **Capabilities:** 9

### Initial Release
- 12 verticals: OmniStudio, Health Cloud, Insurance, DocGen, FSC, Comms Cloud, Revenue Cloud, CPQ, TPM, Life Sciences, E&U, Media
- 9 capabilities: OrgCS, Splunk, CodeSearch, GUS, Columbo, Slack, Confluence, Debug Log, HAR Analysis
- Validated OrgCS queries (case 473586604)
- Guard rails for restricted support tiers
- PII protection rules (mandatory)
- Phase-based investigation workflow (Gather → Route → Check Patterns → Investigate → Regression → Post)
