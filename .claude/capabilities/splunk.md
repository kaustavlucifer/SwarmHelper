# Capability: Splunk — Core Application Log Search

## Key Concepts

| Concept | Rule |
|---|---|
| **Index** | Pod/instance name (e.g., `usa664s`, `deu72`). NOT per-org, NOT product name. |
| **organizationId** | 15-character org ID. Truncate 18-char IDs. |
| **requestId** | Links related log lines for one operation. |
| **rootRequestId** | Original request that started the entire chain — use for end-to-end traces. |

## Tools

### Tool: `mcp__plugin_monitoring_vmcp-monitoring__query_splunk`
```
Params:
  query: "search index=<POD_ID> organizationId=<ORG_ID_15> ..."
  splunk_api_base_url: "https://splunk-api.log-analytics.monitoring.aws-esvc1-useast2.aws.sfdc.cl"
```

### API URLs

| Environment | URL |
|---|---|
| Core (Production + Sandbox) | `https://splunk-api.log-analytics.monitoring.aws-esvc1-useast2.aws.sfdc.cl` |
| Non-core | `https://splunk-api-noncore.log-analytics.monitoring.aws-esvc1-useast2.aws.sfdc.cl` |
| DNR | `https://splunk-api-dnr.log-analytics.monitoring.aws-esvc1-useast2.aws.sfdc.cl` |

## Auth Troubleshooting

- **401 Unauthorized**: Run `/salesforce-trust-foundations:mcp-auth`
- Use fallback tool if primary fails

## Operational Limits

- Query scope: most recent **15 days**
- Results capped at **500 log lines**
- Stats queries: event collection capped at **5,000 events**
- Query timeout: **60 seconds**
- Max **4 concurrent** executions

## Log Record Types (logRecordType)

### Apex
| Type | Description |
|---|---|
| `axerr` | Uncaught Apex exceptions (LimitException, QueryException, DmlException) |
| `axapx` | Apex execution — transaction summary, VFRemote calls |
| `axlim` | Governor limit consumption per transaction |
| `axgen` | General Apex log messages |
| `axque` | Queueable job tracking |
| `axftr` | @future method debugging |

### Industries (All Verticals)
| Type | Description | Availability |
|---|---|---|
| `r1log` | **Industries managed package instrumentation** — primary logRecordType for ALL Industry verticals. Use `instKey` field to filter by feature area. | All Industries packages |

Query pattern:
```spl
index=<POD> organizationId=<ORG_15> logRecordType=r1log instKey=<feature> earliest=-7d
| head 50
```

Common `instKey` values: `makeHttpCall`, `InvokeService`, `DREngine`, `CPQService`, etc.

### OmniStudio
| Type | Description | Availability |
|---|---|---|
| `iposs` | OmniScript | Standard Runtime 242+; Package 244+ |
| `ipipr` | Integration Procedure | Package 242+; Standard 244+ |
| `ipipa` | IP actions/blocks | Same as ipipr |
| `ipdar` | DataRaptor | Package 242+; Standard 242+ |
| `ipfbc` | FlexCard | Confirm with engineering |
| `izilg` | OIC flags (LoggingEnabled, etc.) | All runtimes |
| `stsci` | Experience Sites | All runtimes |
| `ipcul` | Component usage instrumentation | All runtimes |

### Platform
| Type | Description |
|---|---|
| `gslog` | Platform Java exceptions / **suppressed (rate-limited) gack** records with stack traces |
| `txerr` | Error logs (ERROR, SEVERE) |
| `txlog` | Transaction logs (INFO, WARN) |
| `cosis` | Individual SOQL statements (carries `sqlId`; internal-only when `loginId`/`sessionId` null) |
| `ullog` | Universal performance / latency logs |

### Request summary family (single-char legacy codes — end-of-request lines)
| Type | Description |
|---|---|
| `U` | SFDC UI request | 
| `V` | Visualforce UI request |
| `A` | SOAP API request | 
| `a` | Apex SOAP (customer Apex SOAP method) |
| `aprst` | REST API request · `apars` Apex REST request · `vfrmt` VF Remoting (JS remoting) |
| `R` / `rrlog` | Report view / async report (incl. dashboard refresh) |
| `Q` / `I` | SearchQuery / SearchIndex completion |

> For Apex execution (`axapx`), the `quiddity` field disambiguates type (R=sync, F=future, C=scheduled, T=test, V=VF/remoting, batch variants A/S).

### Gack cluster (ready-made hunt)
Gacks span several codes — search them together:
```spl
index=<POD> organizationId=<ORG_15> `logRecordType(G, gslog, gglog, maerr)` earliest=-7d | head 20
```
`gslog` = suppressed gacks (solid), `gglog` = gack sibling (still valid in Splunk though dropped from governance), `G` = gack record, `maerr` = metadata error (deploy investigations).

### Async / Message Queue family (trace Queueable / Batch / @future end-to-end)
`mq*` lines (40+ codes) are how async execution is traced across the platform: `mqenq` enqueue → `mqdeq` dequeue → `mqhdl` handler invoked → `mqfrm` handler finished (perf stats) → `mqend` processed; `mqded` dead message, `mqreq` re-enqueue. Pair with `axque`/`axftr`/`axbth` (ApexAsyncBatch, 252+) on the Apex side.

### Metadata / deploy family (package install, change set, MDAPI deploy failures)
`maopr` operation start · `madop` deploy options · `madep` deploy stats (filter `operation="meta_deploy"`) · `mades` per-component-type stats · `maret`/`mazip` retrieve/zip · `pkgop` package install/update · `csopr`/`csevt` change set op/event.

### Extra Apex codes (the `ax*` family is large — these are the support-relevant ones)
`axhlt` heap-limit tracker · `axlul` limits-usage log · `axqfe` queueable finalizer · `axats`/`axtlt` test summary/limit · `axbth` async batch (252+) · `axpdc` post-deploy compiler · `axcpe` compile errors (256+) · `t` ApexTrigger · `qxjob` queued execution.

## Query Patterns

### Time Window Strategy

**User-provided timestamp:**
- If user gives exact time (e.g., "error at 2026-06-26 14:23:40"), use ±5 minutes: `earliest="06/26/2026:14:18:40" latest="06/26/2026:14:28:40"`

**No timestamp (component-based search):**
- For known error signature (exception class, component name), use **7-day window** initially: `earliest=-7d`
- If 7-day returns too many results (>500), narrow to 24h or ask user for time range
- For recurring errors, use `timechart` to see frequency over time

**Default investigation window:** `-7d` (covers most issues; 15-day max due to Splunk retention)

### 1. Find errors for an org (component-based, no timestamp)
```spl
index=<POD> organizationId=<ORG_15> level=error earliest=-7d
| stats count by logRecordType
| sort -count
```

### 2. Search specific error message (7-day window)
```spl
index=<POD> organizationId=<ORG_15> "System.CalloutException" earliest=-7d
| head 50
```

### 3. Two-pass trace (error → full request)
**Pass 1:**
```spl
index=<POD> organizationId=<ORG_15> "ERROR_MESSAGE" earliest=-24h logRecordType=txerr
| fields requestId
| head 5
```
**Pass 2:**
```spl
index=<POD> organizationId=<ORG_15> requestId="<REQ_ID>" earliest=-1h
| reverse
```

### 4. Apex uncaught exceptions
```spl
index=<POD> organizationId=<ORG_15> `logRecordType(axerr)` earliest=-7d
| head 20
```

### 5. OmniStudio DataRaptor logs
```spl
index=<POD> organizationId=<ORG_15> logRecordType=ipdar earliest=-7d
| head 100
```

### 6. Error frequency over time
```spl
index=<POD> organizationId=<ORG_15> "ERROR_SIGNATURE" earliest=-7d
| timechart span=1h count
```

### 7. Trace by rootRequestId (full end-to-end)
```spl
index=<POD> organizationId=<ORG_15> rootRequestId="<ROOT_REQ_ID>" earliest=-1h
| reverse
```

### 8. Platform Java exceptions (gslog)
```spl
index=<POD> organizationId=<ORG_15> `logRecordType(gslog)` ("Exception" OR "SEVERE") earliest=-7d
| head 20
```

### 9. Falcon / HyperForce orgs (if standard index returns nothing)
```spl
index=coreprod dc::aws-prod1-useast1 sp::core1 pod::<POD> organizationId=<ORG_15> earliest=-7d
```

### 10. Support two-pass with timing
```spl
index=<POD> organizationId=<ORG_15> <REQUEST_ID>
| eval runTimeInSeconds=runTime/1000
| eval endTime=_time
| eval startTimeTemp=_time-runTimeInSeconds
| convert ctime(endTime) as EndTime
| convert ctime(startTimeTemp) as StartTime
| table logRecordType, StartTime, EndTime, runTimeInSeconds, message
```

### Falcon / Off-Platform Services (Revenue Cloud, TPM)

For services running on Falcon (k8s), use the `distapps` index:
```spl
index=distapps functional_domain=core1 k8s_namespace=revenue-cloud earliest=-7d
| head 50
```

| Namespace | Vertical |
|---|---|
| `revenue-cloud` | Revenue Cloud (RLM) Falcon services |
| `consumer-goods` | TPM off-platform services |

---

## Log Governance (source-of-truth for logRecordType)

The canonical, machine-readable catalog is the **Log Governance Service schema** (~4,500 codes; every per-team Confluence list is a derived snapshot of it). Look up any code's fields here:
- **`https://log-governance-service.sfproxy.monitoring.aws-esvc1-useast2.aws.sfdc.cl/schema/main?logRecordType=<TYPE>`** (current proxy; version-pinned views exist, e.g. `/schema/246.9.7`)
- Older UI: `https://log-governance.eng.sfdc.net:8080/schema/main?logRecordType=<TYPE>`
- Underlying registry in core: `monitoringlib/submodules/logging/config/app-logging-format.xml`
- OmniAnalytics Dashboard: `https://org62.lightning.force.com/analytics/dashboard/0FK0M000000TPcMWAW`

Curated human references (verified 2026-06-15):
- **Industries Splunk cheat-sheet** (copy-paste query templates): https://confluence.internal.salesforce.com/spaces/IN/pages/1240990289 (IN — Splunk Queries Quick Reference)
- Apex family: https://confluence.internal.salesforce.com/spaces/APEX/pages/449478700 (Apex Log Lines)
- Request/deploy/bulk families: https://confluence.internal.salesforce.com/spaces/CCE/pages/667336957 (Request Log Lines)

> Naming convention: 2-letter family prefix + 3-letter subtype (`ax`=Apex, `mq`=Message Queue, `ma`/`md`=Metadata/deploy, `tx`/`co`=Context Service/cache, `gs`/`gg`/`G`=gacks), plus a few legacy single-char codes (`U`/`V`/`A`/`Q`/`I`/`G`/`t`/`a`). If a code isn't in this doc, look it up in the governance service — don't guess.

---

## Time Ranges

| Need | Syntax |
|---|---|
| Last hour | `earliest=-1h` |
| Last 24h | `earliest=-24h` |
| Last 7 days | `earliest=-7d` |
| Specific time | `earliest="11/07/2025:14:23:40" latest="11/07/2025:14:23:50"` |

## Important Notes

- **Caught exceptions** (try/catch in managed package) do NOT appear in Splunk `axerr`
- Always include both `index` AND `organizationId`
- Always specify `earliest` and `latest`
- Use `| head N` for initial exploration
- Use `| reverse` for chronological order
- Check `izilg` for `LoggingEnabled=true` FIRST on performance issues
- Use `` `logRecordType(ax*)` `` to search all Apex log types at once
