# Capability: OrgCS — Support Case Management Org

## What OrgCS Is

OrgCS is the **"OrgCS and H&T" support case management org** (instance USA26). It is Salesforce's internal org for managing support cases. It is **NOT** connected to the customer's org.

## What You Can Query

| Object | Key Fields | Purpose |
|---|---|---|
| `Case` | `CaseNumber`, `Subject`, `Status`, `Severity_Level__c`, `InstanceId__c`, `Account.CustomerOrgId15__c`, `Account.OrgInstance__c` | Case metadata, org/pod resolution |
| `CaseComment` | `CommentBody`, `IsPublished`, `CreatedBy.Name`, `CreatedDate` | Engineer-customer dialogue |
| `CaseFeed` | `Type`, `Body`, `LinkUrl`, `Title`, `CreatedBy.Name` | Chatter, emails, status changes |
| `Case_Relationship__c` | `GUS_Work__r.Name`, `GUS_Work__r.Subject__c`, `GUS_Work__r.Status__c`, `GUS_Work__r.Details__c` | Linked GUS work items |
| `Swarm` | `CollaborationUrl`, `HelpNeeded`, `SwarmMembers` subquery | Active swarms with Slack threads |
| `ADM_Work__Feed` | `Body`, `CreatedBy.Name`, `CreatedDate` | GUS work item chatter |
| `HT_Tenant__c` | `Org62_Tenant_Name__c`, `External_Id__c` | Tenant information |

## What You CANNOT Query

OmniStudio objects (`OmniProcess`, `OmniDataTransform`, `OmniInteractionConfig__mdt`, `DRMapItems__c`, `vlocity_cmt__OmniScript__c`) do **not** exist in OrgCS. For customer org data, ask the engineer to run queries in their org.

## Available Tools

| Tool | Use |
|---|---|
| `mcp__orgcs__soqlQuery` | SOQL queries |
| `mcp__orgcs__find` | SOSL text search |
| `mcp__orgcs__getObjectSchema` | Check object/field existence |
| `mcp__orgcs__getRelatedRecords` | Traverse relationships |
| `mcp__orgcs__getUserInfo` | Auth check / current user |
| `mcp__orgcs__listRecentSobjectRecords` | Recently viewed records |

## Auth Troubleshooting

- **401 / not authenticated**: Add OrgCS config to `~/.claude.json`, then click Authenticate in Claude settings
- **Field not found / sObject not found**: Run `mcp__orgcs__getObjectSchema` first to verify field exists

## Validated Query Patterns

All patterns below have been validated against live data (case 473586604, 2026-06-01).

### Full Case Resolution (run Steps 1-5 in parallel once you have Case ID)

#### Step 1 — Core Case + Org/Pod Resolution

```soql
SELECT Id, CaseNumber, Subject, Status, Priority, Severity_Level__c,
       Origin, CreatedDate, Age_days__c, IsEscalated,
       Management_Escalation_Count__c, Escalation_Reason__c, Reopen_count__c,
       Case_Cause__c, Sub_cause__c, Bug__c, Resolution_Summary__c,
       Active_Swarm__c, Support_Region__c, SupportLevel__c, Sub_Status__c,
       RecordType.Name, Owner.Name,
       CaseRoutingTaxonomy__r.Product__c,
       CaseReportingTaxonomy__r.Name,
       CaseReportingTaxonomy__r.Feature__c,
       CaseReportingTaxonomy__r.ProductTopic__r.Name,
       Account.CustomerOrgId15__c, Account.OrgInstance__c,
       InstanceId__c
FROM Case
WHERE CaseNumber = '<CASE_NUMBER>'
LIMIT 1
```

Key fields:
- `Account.CustomerOrgId15__c` → 15-char org ID for Splunk
- `InstanceId__c` → Splunk index / pod name
- `CaseReportingTaxonomy__r.ProductTopic__r.Name` → product routing

#### Step 2 — Case Comments

```soql
SELECT Id, ParentId, CommentBody, IsPublished, CreatedDate, CreatedBy.Name
FROM CaseComment
WHERE ParentId = '<CASE_ID>'
ORDER BY CreatedDate ASC
```

#### Step 3 — Case Feed (chatter, emails, status changes)

```soql
SELECT Id, ParentId, Type, Body, LinkUrl, Title, CreatedDate, CreatedById, CreatedBy.Name
FROM CaseFeed
WHERE ParentId = '<CASE_ID>'
  AND Type IN ('TextPost','ContentPost','LinkPost','EmailMessageEvent','ChangeStatusPost')
ORDER BY CreatedDate ASC
```

#### Step 4 — Linked GUS Work Items

```soql
SELECT Id, Case__c, Work_Record_Type__c, Work_Status__c, Work_Priority__c,
       Product_Tag_Name__c, Scrum_Team_Name__c, Work_Subject__c,
       GUS_Work__r.Name, GUS_Work__r.Subject__c, GUS_Work__r.Status__c,
       GUS_Work__r.Priority__c, GUS_Work__r.Severity_Level__c,
       GUS_Work__r.Assigned_To_Name__c, GUS_Work__r.Scrum_Team_Name__c,
       GUS_Work__r.Product_Tag_Name__c, GUS_Work__r.Bug_Number__c,
       GUS_Work__r.GUS_Link__c, GUS_Work__r.Details__c,
       GUS_Work__r.Due_Date__c, GUS_Work__r.CreatedDate,
       GUS_Work__r.LastModifiedDate
FROM Case_Relationship__c
WHERE Case__c = '<CASE_ID>'
```

#### Step 5 — Active Swarm

```soql
SELECT Id, Name, Status, RelatedRecordId, HelpNeeded,
       StartedDateTime, EndedDateTime, CollaborationTool, CollaborationUrl,
       SwarmAge__c, Type__c, CreatedDate, LastModifiedDate,
       (SELECT Id, OwnerId, Owner.Name, Status, AssignedDateTime, CompletedDateTime
        FROM SwarmMembers ORDER BY AssignedDateTime ASC)
FROM Swarm
WHERE RelatedRecordId = '<CASE_ID>'
ORDER BY CreatedDate DESC
LIMIT 5
```

#### Step 6 — GUS Work Item Feed (only if Step 4 returns GUS IDs)

> **Note:** `ADM_Work__Feed` is NOT queryable from OrgCS. Use the GUS plugin instead:
```
Tool: mcp__plugin_gus_gus_server__query_gus_records
Query:
  SELECT Id, ParentId, Type, Body, CreatedDate, CreatedBy.Name
  FROM ADM_Work__Feed
  WHERE ParentId IN ('<GUS_WORK_ID_1>', '<GUS_WORK_ID_2>')
    AND Type IN ('TextPost','ContentPost','LinkPost')
  ORDER BY CreatedDate ASC
```

### Quick Lookups

**By org ID (no case number):**
```soql
SELECT Id, CaseNumber, Subject, Status, CreatedDate, InstanceId__c, Account.CustomerOrgId15__c
FROM Case
WHERE Account.CustomerOrgId15__c = '<15_CHAR_ORG_ID>'
ORDER BY CreatedDate DESC LIMIT 5
```

**By subject keyword:**
```soql
SELECT Id, CaseNumber, Subject, Status, CreatedDate
FROM Case
WHERE Subject LIKE '%OmniScript%'
ORDER BY CreatedDate DESC LIMIT 10
```

## Post-Fetch Actions

After retrieving case data:
1. **Extract Slack URLs** from `CaseFeed.Body`, `CaseFeed.LinkUrl`, `Swarm.CollaborationUrl`, and `Swarm.HelpNeeded` — pattern: `https://*.slack.com/archives/[CHANNEL_ID](/p[TIMESTAMP])?`
2. **Extract W-numbers** from `CaseComment.CommentBody` and `CaseFeed.Body` — pattern: `W-\d+`
3. **Map to Splunk**: `InstanceId__c` = Splunk index, `Account.CustomerOrgId15__c` = `organizationId`

## Rules

- Always use `LIMIT` clauses
- All operations are READ-ONLY
- Use `CommentBody` (not `Description`) on CaseComment
- Field API names are case-insensitive in SOQL but case-sensitive in schema lookups
- Custom fields end with `__c`; managed package fields have namespace prefix
