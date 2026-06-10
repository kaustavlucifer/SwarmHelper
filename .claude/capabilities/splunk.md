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
| `gslog` | Platform Java exceptions/gacks with full stack traces |
| `txerr` | Error logs (ERROR, SEVERE) |
| `txlog` | Transaction logs (INFO, WARN) |
| `A` | Transaction summary (main request log) |
| `cosis` | Individual SOQL statements |

## Query Patterns

### 1. Find errors for an org
```spl
index=<POD> organizationId=<ORG_15> level=error earliest=-24h
| stats count by logRecordType
| sort -count
```

### 2. Search specific error message
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

## Log Governance

Schema validation for any logRecordType:
- URL: `https://log-governance.eng.sfdc.net:8080/schema/main?logRecordType=<TYPE>`
- OmniAnalytics Dashboard: `https://org62.lightning.force.com/analytics/dashboard/0FK0M000000TPcMWAW`

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
