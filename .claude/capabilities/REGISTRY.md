# MCP Plugin Registry — Tool Fallback Chains & Auth Probes

This registry defines the primary→secondary→tertiary tool chains for each MCP plugin. When a primary tool fails (401, timeout, error), the orchestrator automatically tries the next tool in the chain.

**Philosophy:** We probe at the **plugin level**, not individual tool level. If one tool in a plugin is authenticated, all tools in that plugin share the same auth. This keeps probes fast and token-efficient.

---

## Plugin Matrix

| Plugin | Primary Tool (any from plugin) | Fallback Plugin | Probe Call | What It Covers |
|---|---|---|---|---|
| **OrgCS** | `mcp__orgcs__soqlQuery` | — | `getUserInfo` (fast user record) | Case resolution, org/pod lookup, all OrgCS queries |
| **Splunk** | `mcp__plugin_monitoring_vmcp-monitoring__query_splunk` | — | `query_splunk("search index=usa664s earliest=-1m | head 1")` | Log search across all pods and instances |
| **GUS** | `mcp__plugin_gus_gus_server__query_gus_records` | — | `get_object_description("Work")` (schema call, no data) | Bug tracking, chatter, all GUS queries |
| **Columbo** | `mcp__plugin_columbo_columbo__get_auth_status` | — | `get_auth_status` (built-in, no query) | Gack investigation, stacktrace parsing |
| **Slack** | `mcp__plugin_slack_slack__slack_search_channels` | — | `slack_search_channels(query:"test")` (lightweight) | Channel search, message search, thread reading |
| **CodeSearch** | `mcp__plugin_deep-research_codesearch__search` | `mcp__mcp-adaptor__search` | `search("repo:github.com/sf-industries/via_platform")` | Package code, core monorepo, git.soma - all repos |
| **git.soma** | `mcp__plugin_git-soma_vmcp-git-soma__list_branches` | — | `list_branches("industries-rcg/rcgps-retail-tpm")` | git.soma.salesforce.com repos (TPM, etc.) |
| **Salesforce Docs** | `mcp__salesforce-docs__salesforce_docs_search` | — | `salesforce_docs_search(query:"API", limit:1)` | Official Salesforce product documentation |
| **Internal Docs** | `mcp__plugin_search_search__search` | `mcp__mcp-adaptor__doc_search`, `mcp__plugin_deep-research_search__doc_search` | `search(query:"test", sources:["confluence"])` | Confluence, Stack Overflow, Quip, Glossary |
| **SF CLI** | Bash → `sf --version` | — | `sf --version` | Salesforce CLI operations |

---

## Auth Status Interpretation

| Response | Status | Remediation |
|---|---|---|
| Success (data returned) | ✓ **Authenticated** | — |
| 401 Unauthorized | ⚠ **Auth expired** | Run `/salesforce-trust-foundations:mcp-auth` |
| Timeout / network error | ⚠ **Unreachable** | Check network; retry |
| Tool not found | ✗ **Not registered** | Add MCP server to `~/.claude.json`, then run `/mcp` |
| Empty result (but no error) | ✓ **Authenticated** (probe succeeded, no data matched) | — |

---

## Probe Strategy (Parallel Execution)

**Phase 0 spawns ONE agent per plugin** to probe in parallel. Each agent:
1. Checks if the tool prefix exists in `<system-reminder>` deferred tools
2. If registered → attempts the simplest probe call for that plugin
3. Returns status: `authenticated | auth_expired | not_registered | unreachable`

**Total probe time**: ~3-5 seconds (all run concurrently via parallel agents).

**Fallback logic**: If a plugin fails during investigation (Phases 4-5), the orchestrator automatically tries the next plugin in the chain from this registry. Users see inline notation: `(via mcp-adaptor fallback)`.

---

## Probe Call Design

Each probe is designed to be:
- **Fast**: Metadata calls or 1-result queries
- **Cheap**: Minimal tokens consumed
- **Reliable**: Catches 401 auth failures without false positives

### Probe Call Notes

- **OrgCS**: `getUserInfo` returns user metadata instantly (faster than SOQL query)
- **Splunk**: 1-minute window on stable pod (usa664s), head 1 for minimal load
- **GUS**: Schema describe (no data fetch) - fast metadata operation
- **Columbo**: Built-in auth status endpoint (no actual data query)
- **Slack**: Search for "test" (safe, always returns something)
- **CodeSearch**: Minimal repo filter (validates index access) - ONE probe covers package, core, and git.soma repos
- **git.soma**: List branches (metadata only, no file reads)
- **Salesforce Docs**: 1-result search for "API" (guaranteed hit, lightweight)
- **Internal Docs**: Search "test" in Confluence (lightweight)
- **SF CLI**: Version check (no authentication required)

---

## When to Skip Probes

**Never skip Phase 0.** It's mandatory regardless of input (case number, org+pod, error). The ~3-5 second probe time is worth the reliability gain.

---

## Updating This Registry

When adding a new MCP plugin:
1. Add a row to the Plugin Matrix
2. Define the simplest probe call for that plugin
3. Update Phase 0 in `swarm-helper.md` to include the new probe agent
4. Update Phase 4 to reference the new plugin's fallback chain
