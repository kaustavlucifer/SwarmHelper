# Capability: GUS — Bug Tracking & Work Items

## Tools

| Tool | Use |
|---|---|
| `mcp__plugin_gus_gus_server__query_gus_records` | Primary — query GUS work items via SOQL |
| `mcp__plugin_gus_gus_server__get_object_description` | Schema inspection |

## Auth Troubleshooting

- **Not connected**: Run `/gus` and follow prompts
- **Query returns nothing**: Verify product tag spelling (case-sensitive in LIKE)

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
WHERE Product_Tag__r.Name = 'Industries Interaction platform'
  AND Status__c NOT IN ('Fixed', 'Closed', 'Never Fix')
  AND Subject__c LIKE '%DataRaptor%FLS%'
ORDER BY Priority__c ASC
LIMIT 20
```

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

| Product | Tag |
|---|---|
| OmniStudio / OmniScript / DataRaptor / FlexCard / IP | `Industries Interaction platform` |
| Health Cloud | `Health Cloud` |
| Financial Services Cloud | `Industries Financial Services Cloud` |
| Communications Cloud | `Communications Cloud` |
| Revenue Cloud / CPQ | `Revenue Cloud` |
| Trade Promotion Management | `Consumer Goods Cloud` |
| Life Sciences | `Life Sciences Cloud` |
| Energy & Utilities | `Energy & Utilities Cloud` |
| Media Cloud | `Media Cloud` |

---

## What to Extract

- **W-number** → provide to engineer
- **Status**: Fixed (which release?) / In Progress / Open / Won't Fix
- **`Details__c`** → workaround instructions
- **`Fix_Version__c`** → check if customer's release includes the fix
- **`Known_Issue_URL__c`** → public Known Issue article

---

## Tips

- Search by exception type + class name first (more specific)
- `Subject__c LIKE '%term%'` is case-insensitive in GUS
- Always include `LIMIT`
- If no exact match, broaden to just the exception type
