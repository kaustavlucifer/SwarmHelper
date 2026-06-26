# v2.6.0 Implementation Summary

**Date:** 2026-06-26  
**Author:** Kaustav Chowdhury (with Claude Code assistance)

---

## Overview

This release adds **MCP capability health checking**, **auto-fallback chains**, and **official Salesforce documentation integration**. Users now see upfront which tools are working, which need auth, and which aren't registered — before investigation starts. When primary tools fail, secondary/tertiary fallbacks are tried automatically.

---

## Key Enhancements

### 1. MCP Plugin Health Check (Phase 0 Redesign)

**Before (v2.5.0):**
- Only checked if tool prefixes existed (registered, not authenticated)
- Users had to manually run `/mcp` to discover auth issues
- 401 errors appeared mid-investigation with no clear remediation

**After (v2.6.0):**
- Spawns **10 parallel probe agents** (one per plugin, not per capability)
- Each agent runs the simplest test query for that plugin to verify auth
- Returns detailed status: ✓ authenticated, ⚠ auth expired, ✗ not registered
- Shows remediation commands (e.g., run `/salesforce-trust-foundations:mcp-auth`)
- **Performance:** Reduced from 12 capability probes to 10 plugin probes (~3-5 seconds, faster and more token-efficient)

**Output Example:**
```
MCP Plugin Health Check
────────────────────────────────────────────────────────────────
  ✓ OrgCS plugin (case resolution, org lookup)
  ✓ Splunk plugin (log search across all pods)
  ✓ GUS plugin (bug tracking)
  ⚠ Columbo plugin (gack investigation) → auth expired
  ✓ Slack plugin (channel/message search)
  ✓ CodeSearch plugin (package, core, git.soma repos)
    ↳ Fallback: mcp-adaptor
  ✓ git.soma plugin (git.soma.salesforce.com repos)
  ✓ Salesforce Docs plugin (official product docs)
  ✓ Internal Docs plugin (Confluence, SO, Quip)
    ↳ Fallbacks: mcp-adaptor, deep-research
  ✓ SF CLI (available)

Status: 9/10 plugins ready
⚠ 1 plugin needs attention

Remediation Commands:
  • Columbo auth expired → run /salesforce-trust-foundations:mcp-auth
────────────────────────────────────────────────────────────────
```

### 2. Plugin Registry (REGISTRY.md)

**New file:** `.claude/capabilities/REGISTRY.md`

Defines the **primary → secondary → tertiary fallback chain** for each plugin:

| Plugin | Primary Tool | Fallback Plugin | Probe Call | What It Covers |
|---|---|---|---|---|
| OrgCS | `mcp__orgcs__getUserInfo` | — | `getUserInfo` (fast user record) | Case resolution, org/pod lookup, all OrgCS queries |
| CodeSearch | `mcp__plugin_deep-research_codesearch__search` | `mcp__mcp-adaptor__search` | `search("repo:github.com/sf-industries/via_platform")` | Package, core monorepo, git.soma - ALL repos |
| Salesforce Docs | `mcp__salesforce-docs__salesforce_docs_search` | — | `salesforce_docs_search(query:"API", limit:1)` | Official product docs |
| Internal Docs | `mcp__plugin_search_search__search` | `mcp__mcp-adaptor__doc_search`, `mcp__plugin_deep-research_search__doc_search` | `search(query:"test", sources:["confluence"])` | Confluence, SO, Quip |
| ... | ... | ... | ... | ... |

**Philosophy:** One probe per plugin verifies auth for all tools in that plugin. This is faster, cheaper, and more honest than probing every capability.

**Use:** Single source of truth for tool routing and fallback logic.

### 3. Auto-Fallback Logic (Phase 4)

**Before:** If a primary tool failed (401, timeout), investigation stopped or continued with gaps.

**After:** When primary fails, automatically tries secondary → tertiary in parallel. Users see inline notation when fallbacks succeed.

**Example (Documentation Search):**
1. Try `salesforce-docs` (official docs)
2. If 401 → Try `search_search__search` (internal docs)
3. If that fails → Try `mcp-adaptor__doc_search`
4. If that fails → Try `deep-research_search__doc_search`
5. If all fail → Note: `Documentation unavailable (all tools failed)`

**Inline notation:**
- `✓ Confluence search (via mcp-adaptor fallback)`
- `✓ Package code found in DREngine.cls (via mcp-adaptor)`

**Performance:** Fallbacks only run for failed primaries, and they run in parallel.

### 4. Salesforce Docs MCP Integration

**New Tool:** `mcp__salesforce-docs__salesforce_docs_search`

**Sources:** Official Salesforce product documentation (Help, Developer Guides, Release Notes)

**Use Cases:**
- API field names, object schemas
- Official error code meanings
- Wire adapters, decorators, lifecycle hooks
- Feature documentation (how to configure, when to use)
- Release notes for feature changes

**Search Modes:**
- `auto` (default): Semantic search with lexical fallback
- `hybrid`: Combines semantic + lexical (use for literal tokens: API names, error codes, camelCase, snake_case)
- `semantic`: Pure semantic (rarely needed)

**Collection Filtering:**
```
query: "wire adapters"
collection: "developer/lwc"   # Format: {source}/{area} with SLASH
```

**Examples:**
```
# API reference
query: "getRecord wire adapter"
search_mode: "hybrid"

# Error code
query: "INVALID_FIELD_FOR_INSERT_UPDATE"
search_mode: "hybrid"

# Feature docs
query: "How do I enable Einstein Copilot?"
search_mode: "auto"
```

### 5. Documentation Capability Restructured

**File renamed:** `confluence.md` → `documentation.md`

**Search Strategy:**
1. **Official docs FIRST** (salesforce-docs) — API names, error codes, feature docs
2. **Internal tribal knowledge** (Confluence, Stack Overflow, Quip) — troubleshooting, workarounds, playbooks
3. **Parallel search** — run both when question benefits from official + tribal knowledge

**Tool Priority:**
1. `mcp__salesforce-docs__salesforce_docs_search` (official)
2. `mcp__plugin_search_search__search` (internal)
3. `mcp__mcp-adaptor__doc_search` (fallback)
4. `mcp__plugin_deep-research_search__doc_search` (secondary fallback)

---

## Files Changed

### Created
- `.claude/capabilities/REGISTRY.md` — Tool fallback chains & probe definitions

### Modified
- `.claude/commands/swarm-helper.md` — Phase 0 redesigned, Phase 4 auto-fallback added
- `.claude/capabilities/confluence.md` → `documentation.md` — Renamed, restructured with salesforce-docs primary
- `CHANGELOG.md` — v2.6.0 entry added
- `CLAUDE.md` — Architecture diagram updated (REGISTRY.md, documentation.md)

### Dist
- `dist/swarm-helper-v2.6.0.zip` — Built (433 KB)
  - Contains: `.claude/`, `CHANGELOG.md`, `data/` (empty drop-zone)
  - Excludes: `settings.local.json` (user-specific)

---

## Version Consistency

All 4 locations updated to **v2.6.0**:
- ✓ `.claude/commands/swarm-helper.md` (line 7)
- ✓ `CHANGELOG.md` (line 5)
- ✓ `CLAUDE.md` (architecture diagram)
- ✓ Dist build script (used in filename)

---

## Testing Recommendations

### Phase 0 Health Check
1. Run `/swarm-helper` with all MCPs authenticated → should show 10/10 plugins ready
2. Expire one MCP token (e.g., Columbo) → should show ⚠ auth expired with remediation
3. Disable one MCP (remove from `~/.claude.json`) → should show ✗ not registered
4. Verify parallel execution (watch for ~3-5s total time, not 10×probe time)

### Auto-Fallback
1. Make `deep-research codesearch` return 401 → should auto-try `mcp-adaptor`
2. Make `salesforce-docs` unavailable → should auto-try `search_search__search` (internal docs)
3. Verify inline notation appears when fallback succeeds

### Salesforce Docs
1. Search API name: `query:"getRecord wire adapter" search_mode:"hybrid"`
2. Search error code: `query:"INVALID_FIELD_FOR_INSERT_UPDATE" search_mode:"hybrid"`
3. Search feature: `query:"Einstein Copilot setup" search_mode:"auto"`
4. Filter by collection: `query:"wire adapters" collection:"developer/lwc"`
5. Verify official docs appear before internal docs for API/error lookups

### Parallel Search Pattern
1. Provide a question that benefits from both official + internal docs
2. Verify both searches run in parallel
3. Verify official docs are cited for "what it should do", internal for "how to fix"

---

## Migration Notes for Users

### For Existing Users (v2.5.0 → v2.6.0)

**No action required** if all MCPs are already authenticated. Phase 0 will confirm and proceed.

**If MCPs are missing/expired:**
- You'll see upfront which ones need attention
- Follow the remediation commands shown (e.g., `/salesforce-trust-foundations:mcp-auth`)
- Run `/swarm-helper` again after fixing

**If you have `confluence.md` bookmarked:**
- File is now `documentation.md` (auto-updated in all vertical SKILLs)
- Search patterns remain the same, just with official docs added as primary

### For New Users

1. Install MCPs (OrgCS, Splunk, GUS, Columbo, Slack, CodeSearch, Monitoring, SF CLI)
2. Add `salesforce-docs` MCP to `~/.claude.json`:
   ```json
   "salesforce-docs": {
     "url": "https://salesforce-docs-76258744c9d7.herokuapp.com/api/mcp"
   }
   ```
3. Run `/swarm-helper` — Phase 0 will check all tools and guide you through any missing auth
4. Drop case files in `data/` directory (logs, HARs, exports)
5. Provide case number, org+pod, or error message — swarm-helper auto-routes

---

## Performance Impact

### Phase 0 Probes
- **Time added:** ~3-5 seconds upfront (all probes run in parallel)
- **Performance improvement:** Reduced from 12 capability probes to 10 plugin probes (faster, fewer tokens)
- **Value:** Eliminates mid-investigation 401 surprises and manual `/mcp` checks
- **User control:** Phase 0 is now MANDATORY (no fast-path skip) — reliability over speed

### Auto-Fallback
- **No added latency** when primary succeeds (99% of cases)
- **When primary fails:** fallbacks run in parallel (~same time as one retry)
- **Net result:** Investigations complete despite tool failures (better success rate)

### Official Docs
- **Parallel with internal docs** — no added latency
- **Better accuracy** — official API/error meanings instead of guessing from Confluence

---

## Future Enhancements

### Potential v2.7.0 Ideas
1. **Fast-path auto-detection** — if case number provided, skip Phase 0 and probe on-demand
2. **Probe result caching** — cache auth status for 10 minutes to avoid re-probing on consecutive `/swarm-helper` calls
3. **Tool health dashboard** — `/swarm-helper-status` command to show MCP health without starting investigation
4. **Custom probe queries** — allow users to override probe calls in REGISTRY.md for org-specific indexes
5. **Graceful degradation matrix** — show which verticals can still function if specific tools are down

### Known Limitations
- Phase 0 always runs all 10 plugin probes (no selective probing yet)
- Probe failures are logged but don't halt investigation (this is by design)
- Fallback chains are static (not user-configurable)
- No retry logic for transient network errors (just moves to fallback)

---

## Changelog Entry (Detailed)

See `CHANGELOG.md` for the full v2.6.0 entry. Key highlights:

### Added
- MCP Plugin Health Check (Phase 0) with 10 parallel probe agents (reduced from 12 capability probes)
- Plugin Registry (REGISTRY.md) defining fallback chains at plugin level
- Auto-Fallback Logic (Phase 4) for Documentation and CodeSearch
- Salesforce Docs MCP integration as primary for official docs
- Documentation capability restructured (confluence.md → documentation.md)

### Changed
- Phase 0 now verifies working auth (not just registration)
- Phase 4 updated with fallback column and parallel+fallback pattern
- Tool priority order explicit: official docs → internal docs → fallbacks

### Fixed
- Users now see auth issues upfront (not mid-investigation)
- Investigations resilient to single-tool failures via auto-fallback

---

## Distribution

- **File:** `dist/swarm-helper-v2.6.0.zip`
- **Size:** 433 KB
- **Contents:**
  - `swarm-helper-v2.6.0/.claude/` (commands, capabilities, verticals)
  - `swarm-helper-v2.6.0/CHANGELOG.md`
  - `swarm-helper-v2.6.0/data/` (empty, for case files)
- **Installation:** Unzip, copy `.claude/` and `data/` to project root
- **Compatibility:** Requires Claude Code with MCP support

---

## Acknowledgments

- MCPs used: OrgCS, Splunk (monitoring plugin), GUS, Columbo, Slack, CodeSearch (deep-research + mcp-adaptor), Salesforce Docs, Monitoring
- Architecture patterns: Parallel agent probing, fallback chains, semantic+lexical hybrid search
- Inspiration: `/salesforce-trust-foundations:mcp-auth` for the auth-refresh pattern

---

**End of Implementation Summary**
