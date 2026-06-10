# Trade Promotion Management (TPM) Debugger

**Trigger:** TPM, Consumer Goods Cloud, trade promotions, KPI calculations, nightly batch jobs, push promotions, payment/claim lifecycle, Agentforce for TPM, RTR reports, Funding Grid, `cgcloud` namespace.

---

## Repository Architecture

### GitHub / git.soma Repos

| Repository | Path | Content |
|---|---|---|
| `rcg-retail-tpm` | `git.soma.salesforce.com/industries-rcg/rcg-retail-tpm` | TPM managed package (primary) |
| `RCGSF_SF_Mobility_Sync` | `git.soma.salesforce.com/industries-rcg/RCGSF_SF_Mobility_Sync` | Mobility Sync |
| `rcg-retail-se` | `git.soma.salesforce.com/industries-rcg/rcg-retail-se` | Service Excellence |
| `cgcloud-solutions` | `github.com/salesforce-internal/cgcloud-solutions` | Consumer Goods Cloud solutions |
| `RCG_RE_TPM` | `github.com/sf-industries/RCG_RE_TPM` | RCG Retail Execution TPM |

---


### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-rcg/                               ← Consumer Goods / TPM (if present)
```

> **Note:** TPM is primarily managed package (rcg-retail-tpm on git.soma). No dedicated PTC layer — uses shared OmniStudio PTC classes when IPs/DRs are involved.

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| CGCloud Retail Support Playbook | https://confluence.internal.salesforce.com/spaces/IN/pages/831724707/CGCloud+Retail+Support+Playbook |
| Consumer Goods Cloud | https://confluence.internal.salesforce.com/spaces/IN/pages/188429750/Consumer+Goods+Cloud |
| RCG TPM Product | https://confluence.internal.salesforce.com/spaces/IN/pages/423788550/RCG+TPM+Product |

---

## Key Objects

| Object | Description |
|---|---|
| `cgcloud__Promotion__c` | Trade promotions |
| `cgcloud__Promotion_KPI__c` | KPI records |
| `cgcloud__Fund__c` | Funding |
| `cgcloud__Payment__c` / `cgcloud__Claim__c` | Payment/claim lifecycle |
| `cgcloud__Batch_Run__c` | Batch job tracking |

---

## Common Issues

| Symptom | Check |
|---|---|
| KPI calculation failures | Nightly batch status, calculation engine config |
| Nightly batch not running | Scheduled job status, batch run records |
| Push promotion child spend reset | Distribution rules, child promotion config |
| Payment stuck "ToBeClosed" | Claim lifecycle state, payment rules |
| Apex governor limits on promotion load | Number of KPIs per promotion, SOQL depth |
| Agentforce for TPM errors | Agentforce config, action definitions |
| RTR report data discrepancies | Report filters, KPI sync timing |
| Hyperforce KPI sync issues | Cross-environment sync config |
| Licensing/permission errors | TPM permission sets, feature licenses |
| Funding Grid not loading | LWC component access, data visibility |
| CONFIGURATION_ERROR | TPM settings, missing required config |

---

## Debugging Approach

1. **Check Batch Run records** — `cgcloud__Batch_Run__c` shows status and errors
2. **Nightly calculation log** — Debug logs during scheduled window
3. **KPI formula audit** — Verify calculation formula matches expected logic
4. **Push promotion hierarchy** — Parent → Child distribution rules
5. **Payment lifecycle state machine** — Valid state transitions

---

## Data Model Relationships

```
cgcloud__Promotion__c (Trade Promotion)
├── cgcloud__Promotion_KPI__c (master-detail — KPIs)
│   └── cgcloud__Promotion_KPI_Value__c (calculated values)
├── cgcloud__Tactic__c (promotion tactics/activities)
│   └── cgcloud__Tactic_KPI__c (tactic-level KPIs)
├── cgcloud__Fund__c (funding allocation)
│   └── cgcloud__Fund_Item__c (fund line items)
├── cgcloud__Payment__c (payments)
│   └── cgcloud__Claim__c (claims against payments)
└── cgcloud__Condition__c (promotion conditions/rules)

cgcloud__Batch_Run__c (Batch Job Tracking)
├── cgcloud__Batch_Run_Step__c (individual steps)
└── Error logs / status records
```

---

## Sample SOQL Queries

### Promotions with KPIs
```soql
SELECT Id, Name, cgcloud__Promotion_Status__c, cgcloud__Start_Date__c, cgcloud__End_Date__c,
       cgcloud__Account__r.Name,
       (SELECT Id, Name, cgcloud__KPI_Type__c, cgcloud__Planned_Value__c, cgcloud__Actual_Value__c
        FROM cgcloud__Promotion_KPIs__r)
FROM cgcloud__Promotion__c
WHERE cgcloud__Account__c = '<ACCOUNT_ID>'
ORDER BY cgcloud__Start_Date__c DESC LIMIT 10
```

### Batch run status
```soql
SELECT Id, Name, cgcloud__Status__c, cgcloud__Start_Time__c, cgcloud__End_Time__c,
       cgcloud__Error_Message__c, cgcloud__Records_Processed__c
FROM cgcloud__Batch_Run__c
WHERE cgcloud__Status__c != 'Completed'
ORDER BY cgcloud__Start_Time__c DESC LIMIT 10
```

### Payments and claims
```soql
SELECT Id, Name, cgcloud__Status__c, cgcloud__Amount__c,
       cgcloud__Promotion__r.Name,
       (SELECT Id, cgcloud__Status__c, cgcloud__Amount__c FROM cgcloud__Claims__r)
FROM cgcloud__Payment__c
WHERE cgcloud__Promotion__r.cgcloud__Account__c = '<ACCOUNT_ID>'
ORDER BY CreatedDate DESC LIMIT 20
```

### Fund allocations
```soql
SELECT Id, Name, cgcloud__Budget__c, cgcloud__Committed__c, cgcloud__Spent__c,
       cgcloud__Status__c
FROM cgcloud__Fund__c
WHERE cgcloud__Account__c = '<ACCOUNT_ID>'
ORDER BY Name
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex uncaught exceptions (KPI calculation, batch failures) |
| `axlim` | Governor limits (nightly batch, promotion load) |
| `axque` | Queueable jobs (async KPI processing) |
| `axftr` | @future methods (push promotion distribution) |

---

## Code Investigation Paths

### TPM Managed Package
```
Tool: mcp__plugin_git-soma_vmcp-git-soma__get_file_contents
owner: "industries-rcg"
repo: "rcg-retail-tpm"
path: "classes/<ClassName>.cls"
```

### Consumer Goods Solutions
```
Tool: mcp__plugin_git-emu_vmcp-git-emu__get_file_contents
owner: "salesforce-internal"
repo: "cgcloud-solutions"
path: "<path>"
```

---

## Batch Job Architecture

TPM relies heavily on nightly batch processing:

1. **KPI Calculation Batch** — Runs nightly, calculates all KPIs based on actuals
2. **Push Promotion Distribution** — Distributes parent spend to child promotions
3. **Payment Lifecycle Batch** — Transitions payments through lifecycle states
4. **Fund Reconciliation** — Updates fund committed/spent values

**Debugging batch failures:**
1. Check `cgcloud__Batch_Run__c` for error messages
2. Check scheduled job status (Setup → Scheduled Jobs)
3. Look for governor limit hits in debug logs during batch window
4. Verify no conflicting batch jobs running simultaneously

---

## Escalation

- Slack: `#ent-consumer-goods-se-support` (C098184TA5S) — Consumer Goods SE support
- General: `#support-swarm-industries` (C02BEHKLWES)
- GUS product tag: `Consumer Goods Cloud`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [KPI Calculation | Batch Jobs | Push Promotions | Payments/Claims | Funding | Agentforce | Other]
Issue Description:
Batch Run status (if applicable):
Nightly job timing:
Reproduced in Demo org?:
Troubleshooting steps taken?:
```

---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `known-patterns.md` — Known issue patterns with resolution steps
