# Capability: Confluence / KB / Documentation Search

## Tools

| Tool | Sources |
|---|---|
| `mcp__mcp-adaptor__doc_search` | Confluence, KnowledgeArticle, Slack, Quip, GoogleDrive, Glossary |
| `mcp__plugin_search_search__search` | Confluence, Stack Overflow for Teams, Quip, GlossaryHub |
| `mcp__plugin_deep-research_search__doc_search` | Confluence, Stack Overflow for Teams (fallback if search plugin unavailable) |
| `mcp__plugin_deep-research_search__web_scrape` | Public URLs (GitHub READMEs, Help articles) |

---

## Search Patterns

### Exact error across all sources
```
Tool: mcp__plugin_search_search__search
query: "System.CalloutException uncommitted work pending"
sources: ["confluence", "stackoverflow"]
max_results: 10
```

### Confluence only
```
Tool: mcp__mcp-adaptor__doc_search
query: "Field is not editable OrderItem OmniStudio"
limit: 10
filter_tags: { "Confluence": { "tags": ["*"] } }
```

### Stack Overflow for Teams
```
Tool: mcp__plugin_search_search__search
query: "vlocity_cmt IntegrationProcedureService callHttp"
sources: ["stackoverflow"]
max_results: 10
```

### Quip documents
```
Tool: mcp__plugin_search_search__search
query: "OmniStudio DML before callout workaround"
sources: ["quip"]
max_results: 5
```

### GlossaryHub terms
```
Tool: mcp__plugin_search_search__search
query: "Integration Procedure"
sources: ["glossaryhub"]
max_results: 3
```

### Public URLs (VBT docs, Help articles)
```
Tool: mcp__plugin_deep-research_search__web_scrape
url: "https://raw.githubusercontent.com/vlocityinc/vlocity_build/master/README.md"
```

---

## Search Strategy

1. **Exact error text first** — full exception message
2. **Exception type + class name** — e.g., `CalloutException IntegrationProcedureService`
3. **Error code** — e.g., `FIELD_CUSTOM_VALIDATION_EXCEPTION`
4. **Product + symptom** — e.g., `OmniStudio Integration Procedure fails`
5. **Broaden** — drop specific class names, keep error pattern

---

## Key Playbook URLs

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
