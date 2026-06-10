# Capability: Slack — Internal Knowledge & Escalation

## Tools

| Tool | Use |
|---|---|
| `mcp__plugin_slack_slack__slack_search_public_and_private` | Search all accessible channels |
| `mcp__plugin_slack_slack__slack_search_public` | Public channels only |
| `mcp__plugin_slack_slack__slack_read_thread` | Read full thread from a URL |
| `mcp__plugin_slack_slack__slack_read_channel` | Read recent messages from a channel |
| `mcp__plugin_slack_slack__slack_search_channels` | Find channels by name |
| `mcp__plugin_slack_slack__slack_search_users` | Find people by name/title/email |
| `mcp__plugin_slack_slack__slack_read_user_profile` | Get person's profile/role |

## Auth Troubleshooting

- **Error / not connected**: Connect via AI Suite Settings → MCP Servers → Slack

---

## Search Patterns

### Error message search
```
query: "\"uncommitted work pending\" IntegrationProcedure"
limit: 10
```

### With date filter
```
query: "CalloutException vlocity_cmt after:2026-01-01"
limit: 10
```

### In specific channel
```
query: "\"Field is not editable\" in:support-omnistudio-collaboration"
limit: 10
```

### By org ID (find if case was discussed)
```
query: "00Dxx0000001234 in:support-swarm-industries"
limit: 5
```

### Read thread from CaseFeed/Swarm URL
```
Tool: mcp__plugin_slack_slack__slack_read_thread
thread_url: "https://salesforce.slack.com/archives/C03GSNY2GVC/p1234567890"
```

---

## Key Channels

| Channel | Coverage | ID |
|---|---|---|
| `#support-omnistudio-collaboration` | OmniStudio support swarm | C03GSNY2GVC |
| `#industries-omnistudio-collaboration` | OmniStudio TMP/engineering | C02LN705BEK |
| `#support-swarm-industries` | General industry swarm | C02BEHKLWES |
| `#omnistudio` | OmniStudio engineering | C01H0U55GD8 |
| `#omnistudio-support` | Spring '26 OmniStudio support | C0A693YH82H |
| `#omnistudio-migration-support` | Migration to Core Runtime | C0AJACQQ6H3 |
| `#industries-omniscript` | OmniScript-specific | C026ZKUL5EG |
| `#support-rev-dev-amer` | Revenue Cloud / CPQ (Americas) | — |
| `#support-health-cloud` | Health Cloud | — |
| `#support-industries-fsc` | FSC | — |
| `#support-swarm-service-agentforce-datacloud-ind` | Industry + Data Cloud | — |
| `#moncloud-support` | Splunk / monitoring | — |

---

## Search Modifiers

| Modifier | Example |
|---|---|
| `in:channel-name` | `in:support-omnistudio-collaboration` |
| `from:@username` | `from:@jsmith` |
| `after:YYYY-MM-DD` | `after:2026-01-01` |
| `before:YYYY-MM-DD` | `before:2026-06-01` |
| `"exact phrase"` | `"uncommitted work pending"` |
| `-word` | Exclude a term |
| `has:link` | Messages with links |
| `has:file` | Messages with attachments |

---

## What to Look For

- Same error reported by others → check if solution was shared
- Escalation threads → engineering team engagement
- `#case-XXXXXXX` channels → full investigation context
- Workarounds shared before formal KB articles
- SME identification → use `slack_read_user_profile` for contact info
