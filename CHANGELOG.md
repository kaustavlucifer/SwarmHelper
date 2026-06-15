# SwarmHelper Changelog

These skills are developed and maintained by Kaustav Chowdhury.

## v2.3.0 — 2026-06-15

**Durability + enrichment pass.** Release numbers made dynamic, GUS bug-staleness handling added, TPM off-core enriched from source, 8 thin verticals given diagnostic-map pattern files, Splunk reference expanded from authoritative sources. All facts live-probed (auth-gated MCP).

### Added
- **Dynamic release resolution** (codesearch.md → *Release Resolution*): all `core-{N}-public` / `p4/{N}-patch` / `release-{N}` refs replaced with `{CURRENT_GA}` / `{IN_DEV}` / `{PREV}` placeholders across ~60 sites in 24 files. Resolver maps today's date to a Slack-verified tri-annual release table (Winter/Spring/Summer, +2 each) through ~2028, with an optional gitcore probe near GA boundaries. Orchestrator resolves once per session. **Fixes a latent error:** the old table mislabeled 262 as Spring '26 — it is Summer '26 (GA 2026-06-15).
- **GUS bug-staleness rule** (gus.md + orchestrator Phase 4/6): related-bug queries now pull `Found_in_Build__r.Name` / `Scheduled_Build__r.Name`; bugs fixed in ≤ GA−2 are flagged "likely already resolved in this org — verify build" instead of presented as the cause.
- **`known-patterns.md` for 8 thin verticals** (Manufacturing, Automotive, Public Sector, Loyalty, Education, Nonprofit, Net Zero, Life Sciences) — symptom → verified subsystem/class → how-to-confirm → GUS search. Diagnostic maps, **not** static bug inventories (no release-pegged W-number lists to maintain). Every core subsystem path list-dir-verified on the current GA branch.
- **TPM off-core enrichment** from `rcgps-tpm-service` source: real REST endpoints, the `SYSTEM_PROCESS_ID` worker/web split, `calculationType` dispatch values, the `x-is-tpm` licensing gate, and log-code families (`SVC*`, `TPMWRK*`, `TPMCALCWRK*`, `CALCPLAN*`, `GETFEATURE*`).
- **Splunk reference expanded** (splunk.md) from authoritative sources: Log Governance Service source-of-truth URL (~4,500 codes), gack cluster `logRecordType(G,gslog,gglog,maerr)`, `mq*` async family (trace Queueable/Batch/@future), `ma*` metadata-deploy family, request-summary single-char codes, extra `ax*` codes, and the IN Industries Splunk cheat-sheet link.

### Fixed
- **GUS product tags** — 6 SKILL escalation tags returned **0 bugs** (silent-fail): `Manufacturing Cloud`, `Automotive Cloud`, `Public Sector Solutions`, `Education Cloud`, `Nonprofit Cloud`, `Life Sciences Cloud` → corrected to the real (often area-specific) tags. **All product-tag queries repo-wide switched from `=` to `LIKE '%...%'`** so they survive tag renames.
- **Splunk per-vertical tables** — 7 core-only verticals led with `r1log` (managed-package-only instrumentation that doesn't fire on core-native features) → reordered to lead with `gslog`, `r1log` qualified as conditional. Public Sector kept `r1log` primary (real `vlocity_ps` package).
- **Slack** — re-probed the 6 `⚠️ verify` channels: only `#ad_sales_management_tech_group` (C0349HJ7M0A) is real (promoted); the other 5 don't exist (removed). Added `#industries-netzerocloud-all` (C027WNNCTJL).

## v2.2.0 — 2026-06-15

**Full verification sweep + TPM off-core enrichment.** Every factual reference across all 20 verticals was live-probed against MCP sources; only confirmed facts ship. ~335 references checked.

### Verified (live, 2026-06-15)
- **core/ monorepo paths:** 96/99 confirmed via codesearch `list_directory` on `core-262-public`.
- **GUS W-numbers:** all 110 cited W-numbers exist (zero fabrications); subjects cross-checked against claims.
- **Confluence:** 45/46 pages resolve to correct space + pageId.
- **Slack:** every claimed `C0…` channel ID matched its name.

### Fixed (verification failures — corrected or removed)
- **BRE paths:** `core/industries-bre-impl` / `-bre-api` don't exist → corrected to `industries-bre-near-core-impl`, `-near-core-api`, `-engine-runtime` (RLM).
- **Manufacturing rebate:** `core/industries-rebate-impl` (dead) → `core/industries-mfg-rebates-impl`.
- **Media:** removed wrong-domain `setup_thb/media` path (that dir is contact-center voice setup; Media Ad Sales is `industries-media-revenue` + PTC `sfi_media_7`).
- **Action Plans:** stale `core-260-public` refs → `core-262-public` (module exists in both).
- **GUS citations:** removed mis-cited W-22450490 (comms) and W-12428921 (omnistudio); reworded W-19402617 to its real subject; retagged W-11941259 as case-derived.
- **Confluence:** AITB1 slug pointed at wrong pageId → corrected to `CSGPAK/489918917`; unverified net-zero "Onboarding" → real `IN/842107431 Net Zero Cloud Troubleshooting Help`.
- **Slack:** replaced literal `#C0…`-as-name placeholders in CPQ files; fixed name-drift (`#sc-code-reviews`→`#nzc-code-reviews`, `#sc-tf-fix`→`#nzc-tf-fix`, `#support-industries-fsc`→`#support-industry-fsc-hc`, `#support-health-cloud`→`#industries-healthcare`); validated IDs added to `slack.md` table.
- Unconfirmable channels (e.g. `#ask-sfdo-tech-expert`, `#automotive-cloud-experts`, `#sustcloud-engineering-only`) tagged `⚠️ verify` rather than asserted.

### Added
- **Provenance principle** (swarm-helper Phase 3): verified facts cited directly; case-derived patterns treated as leads requiring Phase 4/5 confirmation; object/field names tagged "verify in target org".
- **TPM off-core architecture** (validated): documented the `packages/tpm/` Node.js processing monorepo (rcgps-calcengine, rcgps-accplnprm, rcgps-kpi, rcgps-tpm-service, etc.) on `industries-rcg/rcgps-retail-tpm` release-262 — TPM's real calc/batch tier, distinct from the Apex managed package. Corrected the code-investigation path (was assuming `classes/*.cls`).

### Removed
- `Archive/` (original per-vertical submissions), `RCG TPM SKILL.md` (703 KB raw dev guide), `SampleTrainingData/` — source material for building `.claude/`, not shipped artifacts. The shipped deliverable is `.claude/` + `CHANGELOG.md`.

---

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
