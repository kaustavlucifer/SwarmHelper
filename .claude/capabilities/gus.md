# Capability: GUS — Bug Tracking & Work Items

## Tools

| Tool | Use |
|---|---|
| `mcp__plugin_gus_gus_server__query_gus_records` | Primary — query GUS work items via SOQL |
| `mcp__plugin_gus_gus_server__get_object_description` | Schema inspection |

## Auth Troubleshooting

- **Not connected**: Run `/gus` and follow prompts
- **Query returns nothing**: Match product tags with `LIKE '%...%'` (never `=`); if still empty, broaden to the cloud keyword and inspect `Product_Tag__r.Name` in results

---

## Common Searches

### By error message
```soql
SELECT Id, Name, Subject__c, Status__c, Priority__c, Fix_Version__c,
       Assignee__r.Name, Scrum_Team__r.Name, Details__c
FROM ADM_Work__c
WHERE Subject__c LIKE '%CalloutException%IntegrationProcedure%'
  AND RecordType.Name = 'Bug'
ORDER BY LastModifiedDate DESC
LIMIT 20
```

### By W-number
```soql
SELECT Id, Name, Subject__c, Status__c, Priority__c, Fix_Version__c,
       Assignee__r.Name, Details__c, Known_Issue_URL__c
FROM ADM_Work__c
WHERE Name = 'W-12345678'
LIMIT 1
```

### By product tag (open bugs)
```soql
SELECT Id, Name, Subject__c, Status__c, Priority__c, Fix_Version__c
FROM ADM_Work__c
WHERE Product_Tag__r.Name LIKE '%Industries Interaction platform%'
  AND Status__c NOT IN ('Fixed', 'Closed', 'Never Fix')
  AND Subject__c LIKE '%DataRaptor%FLS%'
ORDER BY Priority__c ASC
LIMIT 20
```

> **Always match product tags with `LIKE '%...%'`, not `=`.** Tags get renamed and proliferate into area-specific variants (e.g. there is no bare `Manufacturing Cloud` tag — only `Manufacturing Cloud - Sales Agreement, ...`, `Manufacturing Service - Warranty Management`, etc.; likewise `Vlocity Public Sector` not `Public Sector Solutions`). A substring `LIKE` survives renames and catches sibling tags; an `=` silently returns 0 the day a tag is reworded. If a tag `LIKE` still returns nothing, broaden to the cloud keyword (e.g. `LIKE '%Public Sector%'`) and inspect `Product_Tag__r.Name` in the results.

### GUS feed (engineering chatter on a work item)
```soql
SELECT Id, ParentId, Type, Body, CreatedDate, CreatedBy.Name
FROM ADM_Work__Feed
WHERE ParentId = '<GUS_WORK_ITEM_ID>'
  AND Type IN ('TextPost', 'ContentPost', 'LinkPost')
ORDER BY CreatedDate ASC
```

---

## Product Tags

Use these as the **substring** inside `LIKE '%...%'` (not exact match). Where a bare cloud name returns 0 bugs, the verified working tag is noted.

| Product | Tag substring (for LIKE) |
|---|---|
| OmniStudio / OmniScript / DataRaptor / FlexCard / IP | `Industries Interaction platform` |
| Health Cloud | `Health Cloud` |
| Financial Services Cloud | `Industries Financial Services Cloud` |
| Communications Cloud | `Communications Cloud` |
| Revenue Cloud / CPQ | `Revenue Cloud` |
| Trade Promotion Management | `Consumer Goods Cloud` |
| Life Sciences | `Life Sciences Cloud - Product Management` (bare `Life Sciences Cloud` = 0 bugs); fallback `Health and Life Sciences` |
| Energy & Utilities | `Energy & Utilities Cloud` |
| Media Cloud | `Media Cloud` |
| Manufacturing | area-specific (no bare tag): `Manufacturing Cloud - Sales Agreement`, `Manufacturing Service - Warranty Management`, `Manufacturing Program Based Business`, `Manufacturing Rebates` |
| Automotive | area-specific (no bare tag): `Automotive - Captive Finance`, `Automotive Cloud - Vehicle 360`, `Automotive - Connected Cars` |
| Public Sector | `Vlocity Public Sector` (bare `Public Sector Solutions` = 0 bugs) |
| Loyalty | `Loyalty Management` |
| Education | `Education Data Architecture` (EDA); `Education - OmniStudio` |
| Nonprofit | `Nonprofit Success Pack (NPSP)`; `Nonprofit Cloud - Addresses` |
| Net Zero | `Net Zero Cloud`; `Net Zero Cloud - OmniStudio` |

> Verified live 2026-06-15. These are starting substrings — if a `LIKE` returns nothing, broaden to the cloud keyword and inspect `Product_Tag__r.Name` in results.

---

## What to Extract

- **W-number** → provide to engineer
- **Status**: Fixed (which release?) / In Progress / Open / Won't Fix
- **`Details__c`** → workaround instructions
- **`Fix_Version__c`** → check if customer's release includes the fix
- **`Known_Issue_URL__c`** → public Known Issue article

---

## Build / Staleness Fields (ALWAYS pull when surfacing related bugs)

Release numbers move every ~4 months, so a bug that was real two releases ago is usually already fixed in the customer's org. To avoid pointing engineers at dead bugs, **always select the build fields and reason about them** (resolve `{CURRENT_GA}` per `.claude/capabilities/codesearch.md` → Release Resolution):

| Field | Meaning |
|---|---|
| `Found_in_Build__r.Name` | Release the bug was first found in (e.g. `"262"`) |
| `Scheduled_Build__r.Name` | Release the fix is scheduled / shipped in |
| `Resolution__c` | How it closed (`App Bug Fix`, `Won't Fix`, `Duplicate`, `Test Change`, etc.) |
| `Closed_On__c` | Close timestamp |

> The `*_Build__c` fields are **lookups** — you must traverse `__r.Name` to get the readable release number; selecting the raw `__c` returns an opaque record Id.

### Staleness rule when reporting related bugs

1. **Open / In Progress bugs** → always surface, regardless of build.
2. **Closed/Fixed bugs scheduled in `{CURRENT_GA}` or `{IN_DEV}`** → surface as current and relevant.
3. **Closed/Fixed bugs scheduled in `{PREV}` (GA−2) or older** → still list them (a regression can resurface), but **append an explicit note**, e.g.:
   > ⚠️ *W-####### was resolved in build {N} (≥2 releases before current GA {CURRENT_GA}). Per GUS it is likely already fixed in this org — verify the org's current build before treating it as the cause.*
4. Always show `Found_in_Build` → `Scheduled_Build` so the engineer can judge whether the customer's org predates the fix.

Recommended SELECT for related-bug queries:
```soql
SELECT Id, Name, Subject__c, Status__c, Priority__c,
       Found_in_Build__r.Name, Scheduled_Build__r.Name,
       Resolution__c, Closed_On__c, Known_Issue_URL__c
FROM ADM_Work__c
WHERE Product_Tag__r.Name LIKE '%<TAG>%'
  AND RecordType.Name = 'Bug'
  AND Subject__c LIKE '%<SYMPTOM>%'
ORDER BY Status__c, Scheduled_Build__r.Name DESC NULLS LAST
LIMIT 20
```

---

## Additional Queries

### By Scrum Team
```soql
SELECT Id, Name, Subject__c, Status__c, Priority__c, Fix_Version__c
FROM ADM_Work__c
WHERE Scrum_Team__r.Name = 'CME - OmniStudio Starlord'
  AND Status__c NOT IN ('Fixed', 'Closed', 'Never Fix')
ORDER BY Priority__c ASC
LIMIT 20
```

### Known Issue articles
```soql
SELECT Id, Name, Subject__c, Status__c, Known_Issue_URL__c
FROM ADM_Work__c
WHERE Known_Issue_URL__c != null
  AND Product_Tag__r.Name LIKE '%Industries Interaction platform%'
ORDER BY LastModifiedDate DESC
LIMIT 10
```

### GUS Chatter (engineering discussion on a work item)
```
Tool: mcp__plugin_gus_gus_server__query_gus_chatter
parent_id: "<GUS_WORK_ITEM_ID>"
```

---

## Tips

- Search by exception type + class name first (more specific)
- `Subject__c LIKE '%term%'` is case-insensitive in GUS
- Always include `LIMIT`
- If no exact match, broaden to just the exception type
- Use `query_gus_chatter` for engineering context on open bugs
