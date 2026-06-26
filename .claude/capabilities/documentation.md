# Capability: Documentation Search — Official & Internal

## Search Strategy

1. **Official Salesforce Docs FIRST** (salesforce-docs MCP) — use for:
   - API field names, object schemas
   - Official error codes and meanings
   - Feature documentation (when it works, how to configure)
   - Release notes for feature changes
   - Wire adapters, decorators, lifecycle hooks
   
2. **Internal Tribal Knowledge** (Confluence, Stack Overflow, Quip) — use for:
   - Troubleshooting patterns and workarounds
   - Team playbooks and diagnostic flows
   - Known issues not yet documented officially
   - Engineer-written guides and war stories
   - Specific product vertical playbooks

3. **Parallel Search** — when the question could be in both:
   - Run official docs + internal docs simultaneously
   - Merge results, prioritizing official docs for accuracy
   - Use internal docs for context/workarounds

---

## Tools (Priority Order with Fallbacks)

| Priority | Tool | Sources | Use For |
|---|---|---|---|
| **1** | `mcp__salesforce-docs__salesforce_docs_search` | Official Salesforce product documentation (Help, Developer Guides, Release Notes) | API names, error codes, feature docs, release notes, wire adapters |
| **2** | `mcp__plugin_search_search__search` | Confluence, Stack Overflow for Teams, Quip, GlossaryHub | Troubleshooting, playbooks, workarounds, team knowledge |
| **3** | `mcp__mcp-adaptor__doc_search` | Confluence, KnowledgeArticle, Slack, Quip, GoogleDrive, Glossary | Fallback for internal docs (if search plugin fails) |
| **4** | `mcp__plugin_deep-research_search__doc_search` | Confluence, Stack Overflow for Teams | Secondary fallback if both above unavailable |
| **5** | `mcp__plugin_deep-research_search__web_scrape` | Public URLs (GitHub, public Help) | VBT README, public Help articles |

---

## Search Patterns

### Pattern 1: Official Docs (API/Field References, Error Codes)

**When to use:** User asks about field names, object schemas, official error meanings, wire adapters, decorators.

```
Tool: mcp__salesforce-docs__salesforce_docs_search
query: "getRecord wire adapter"
search_mode: "hybrid"   # Use hybrid for literal tokens (API names, error codes)
limit: 5
```

**Examples:**
- Field reference: `query: "Account.Industry field" search_mode: "hybrid"`
- Error code: `query: "INVALID_FIELD_FOR_INSERT_UPDATE" search_mode: "hybrid"`
- API name: `query: "getRecord wire adapter" search_mode: "hybrid"`
- Feature docs: `query: "How do I enable Einstein Copilot?" search_mode: "auto"`

**Search mode guide:**
- `auto` (default) — semantic search with lexical fallback; best for natural questions
- `hybrid` — combines semantic + lexical; use for literal tokens (camelCase, snake_case, quoted phrases)
- `semantic` — pure semantic; rarely needed

**Collection filtering:**
```
Tool: mcp__salesforce-docs__salesforce_docs_search
query: "wire adapters"
collection: "developer/lwc"   # Format: {source}/{area} with SLASH separator
limit: 5
```

Common collections: `admin/ai`, `developer/lwc`, `developer/apex`, `admin/release-notes`, `mulesoft/kafka-connector`

### Pattern 2: Internal Troubleshooting (Confluence, Stack Overflow)

**When to use:** User has an error and needs workarounds, playbooks, troubleshooting steps.

```
Tool: mcp__plugin_search_search__search
query: "System.CalloutException uncommitted work pending"
sources: ["confluence", "stackoverflow"]
max_results: 10
```

### Pattern 3: Parallel Search (Both Official + Internal)

**When to use:** Question could benefit from both official docs AND tribal knowledge.

Run both tools in parallel, then merge results:

```
Tool 1: mcp__salesforce-docs__salesforce_docs_search
query: "DataRaptor Extract not returning records"
limit: 5

Tool 2: mcp__plugin_search_search__search
query: "DataRaptor Extract not returning records"
sources: ["confluence", "stackoverflow"]
max_results: 10
```

**Result interpretation:** Official docs for "what it should do", internal docs for "how to fix".

### Pattern 4: Exact Error Across All Internal Sources

```
Tool: mcp__plugin_search_search__search
query: "vlocity_cmt.IntegrationProcedureService.invoke"
sources: ["confluence", "stackoverflow", "quip"]
max_results: 15
```

### Pattern 5: Confluence Only (Team Playbooks)

```
Tool: mcp__mcp-adaptor__doc_search
query: "Field is not editable OrderItem OmniStudio"
limit: 10
filter_tags: { "Confluence": { "tags": ["*"] } }
```

### Pattern 6: Stack Overflow for Teams (Code Patterns)

```
Tool: mcp__plugin_search_search__search
query: "vlocity_cmt IntegrationProcedureService callHttp"
sources: ["stackoverflow"]
max_results: 10
```

### Pattern 7: Quip Documents (Team Runbooks)

```
Tool: mcp__plugin_search_search__search
query: "OmniStudio DML before callout workaround"
sources: ["quip"]
max_results: 5
```

### Pattern 8: GlossaryHub Terms (Definitions)

```
Tool: mcp__plugin_search_search__search
query: "Integration Procedure"
sources: ["glossaryhub"]
max_results: 3
```

### Pattern 9: Public URLs (VBT Docs, GitHub READMEs)

```
Tool: mcp__plugin_deep-research_search__web_scrape
url: "https://raw.githubusercontent.com/vlocityinc/vlocity_build/master/README.md"
```

### Pattern 10: Fetch Full Doc (After Search)

When a search returns a `documentPath`, fetch the full content:

```
Tool: mcp__salesforce-docs__salesforce_docs_fetch
id: "admin/ai/260-0-0/agent_actions_custom.html"   # From search result
```

Or from a full URL:

```
Tool: mcp__salesforce-docs__salesforce_docs_fetch
id: "https://help.salesforce.com/s/articleView?id=ai.agent_actions_custom.htm"
```

**Never construct the id yourself** — always use the documentPath from a search result or a URL provided by the user.

---

## Progressive Search Strategy (When First Search Fails)

1. **Exact error text first** — full exception message
2. **Exception type + class name** — e.g., `CalloutException IntegrationProcedureService`
3. **Error code** — e.g., `FIELD_CUSTOM_VALIDATION_EXCEPTION`
4. **Product + symptom** — e.g., `OmniStudio Integration Procedure fails`
5. **Broaden** — drop specific class names, keep error pattern

---

## Fallback Handling

If `salesforce-docs` returns 401 or errors:
- Note it: `(Official docs unavailable)`
- Continue with internal sources
- During Phase 0, this will be flagged with remediation

If `search_search__search` fails:
- Fall back to `mcp-adaptor__doc_search`
- If that fails → `deep-research_search__doc_search`
- Note inline: `(via mcp-adaptor fallback)`

---

## Key Playbook URLs (Internal Confluence)

| Topic | URL |
|---|---|
| OmniStudio Playbook (main) | `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650321/Omnistudio+Playbook` |
| OmniScript Runtime | `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650488/` |
| Integration Procedure | `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650454/` |
| DataRaptor | `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/644650468/` |
| Performance | `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/668086171/` |
| Diagnostic Tool | `https://confluence.internal.salesforce.com/spaces/OMNISTUDIO/pages/1353257910/` |
| DR Validation | `https://confluence.internal.salesforce.com/spaces/CMEKB/pages/676318862/` |
| IP Issues | `https://confluence.internal.salesforce.com/spaces/CMEKB/pages/675915284/` |

---

## When to Use Each Tool Summary

| Scenario | Tool | Notes |
|---|---|---|
| User asks "what does field X do?" | salesforce-docs | Official field reference |
| User asks "how do I configure feature Y?" | salesforce-docs | Official feature docs |
| User has error code `ERROR_XYZ` | salesforce-docs (hybrid mode) | Official error meaning |
| User has stack trace with custom error | Confluence/SO (search plugin) | Troubleshooting patterns |
| User asks "wire adapter syntax" | salesforce-docs (hybrid mode) | Official API reference |
| User asks "how do teams fix issue X?" | Confluence/SO (search plugin) | Tribal knowledge |
| Both official + workaround needed | Both in parallel | Official for "what", internal for "how to fix" |
