# Capability: Columbo — Gack / Platform Exception Investigation

## What is a Gack?

A "Gack" is Salesforce's internal term for an unhandled server-side exception. Each generates:
- **Gack ID** — unique identifier (e.g., `1912526670-12345`)
- **Stacktrace ID** — hash of the stack trace, shared across all occurrences of same error

## Tools

| Tool | Use |
|---|---|
| `mcp__plugin_columbo_columbo__search_gacks` | Search by subject, stacktrace ID, or gack ID |
| `mcp__plugin_columbo_columbo__fetch_gack_details` | Full details: subject, stack trace, source class |
| `mcp__plugin_columbo_columbo__fetch_gacks_by_stacktrace` | All occurrences of same error |
| `mcp__plugin_columbo_columbo__fetch_bulk_gacks_by_stacktrace_ids` | Parallel multi-gack fetch |
| `mcp__plugin_columbo_columbo__get_latest_accessible_gack` | Most recent within Splunk 15-day window |
| `mcp__plugin_columbo_columbo__fetch_kodama_analysis` | Owning team, recent code changes, regression |
| `mcp__plugin_columbo_columbo__parse_delphi_stack_trace` | Parse raw stack trace into structured data |
| `mcp__plugin_columbo_columbo__get_auth_status` | Check SSO session |
| `mcp__plugin_columbo_columbo__refresh_auth` | Re-authenticate |

## Auth Troubleshooting

Check first:
```
Tool: mcp__plugin_columbo_columbo__get_auth_status
```
If expired:
```
Tool: mcp__plugin_columbo_columbo__refresh_auth
```

---

## Search Methods

### By error subject text
```
Tool: mcp__plugin_columbo_columbo__search_gacks
query: "CalloutException uncommitted work pending"
search_type: "SUBJECT"
```

### By gack ID
```
query: "1912526670-12345"
search_type: "GACK"
```

### By stacktrace ID
```
query: "1817477665"
search_type: "STACKTRACE"
```

### Get full details
```
Tool: mcp__plugin_columbo_columbo__fetch_gack_details
gack_id: "1912526670-12345"
```

### Get latest accessible gack (within Splunk 15-day window)
```
Tool: mcp__plugin_columbo_columbo__get_latest_accessible_gack
stacktrace_id: "1817477665"
```

### Get Kodama analysis (owning team + regression)
```
Tool: mcp__plugin_columbo_columbo__fetch_kodama_analysis
gack_id: "1912526670-12345"
```
Returns: owning team, recent code changes, regression indicators.

### Parse raw stack trace (no gack ID)
```
Tool: mcp__plugin_columbo_columbo__parse_delphi_stack_trace
stack_trace: "java.lang.NullPointerException\n\tat com.salesforce..."
```

---

## Investigation Flow

1. **Search by subject** with exception message from debug log
2. If found → **get gack details** for stack trace and source class
3. **Get Kodama analysis** → owning team + regression check
4. **Fetch by stacktrace ID** → count occurrences / blast radius
5. Cross-reference with **GUS** — Kodama often links to existing bugs

---

## Delphi Web UI (if MCP unavailable)

| Environment | URL |
|---|---|
| Production | `https://delphi-app-production.sfproxy.monitoring.aws-esvc1-useast2.aws.sfdc.cl/gacks/gackFullText?environment=PRODUCTION` |
| Internal | `https://delphi-app-internal.sfproxy.monitoring.aws-esvc1-useast2.aws.sfdc.cl/gacks/gackFullText` |

---

## Notes

- Not all exceptions become gacks — only unhandled server-side
- Splunk log data has 15-day retention; gack metadata stored indefinitely
- Local environment gacks are NOT uploaded to Delphi
