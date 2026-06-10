# SwarmHelper Changelog

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
