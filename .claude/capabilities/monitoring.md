# Capability: Monitoring — Incident & Pod Health Check

## Purpose

Check for active incidents and pod health BEFORE deep investigation. If the issue is a known outage, skip troubleshooting and report the incident.

## Tools

| Tool | Use |
|---|---|
| `mcp__plugin_monitoring_vmcp-monitoring__get_pagerduty_incidents` | Active P1/P2 incidents affecting the product area |
| `mcp__plugin_monitoring_vmcp-monitoring__get_past_incidents_details` | Historical incidents (was there a recent outage?) |
| `mcp__plugin_monitoring_vmcp-monitoring__check_pod_health` | Pod/instance health status |

---

## When to Use

1. **Before deep investigation** — Quick check: "Is this a known outage?"
2. **Multiple customers reporting same issue** — Blast radius check
3. **Performance degradation** — Pod health might show resource pressure
4. **After identifying the pod** — Correlate with active incidents

---

## Query Patterns

### Check active incidents for a product area
```
Tool: mcp__plugin_monitoring_vmcp-monitoring__get_pagerduty_incidents
service_name: "<product_area>"  # e.g., "Industries", "OmniStudio", "Revenue Cloud"
status: "triggered,acknowledged"
```

### Check pod health
```
Tool: mcp__plugin_monitoring_vmcp-monitoring__check_pod_health
pod: "<POD_ID>"  # e.g., "na44", "usa664s"
```

### Check recent incidents (last 7 days)
```
Tool: mcp__plugin_monitoring_vmcp-monitoring__get_past_incidents_details
service_name: "<product_area>"
since: "7d"
```

---

## Interpreting Results

| Status | Meaning | Action |
|---|---|---|
| Active incident on same service | Known outage | Report incident ID, skip troubleshooting |
| Pod health degraded | Resource pressure | Note in findings, may explain performance issues |
| Recent resolved incident | Possible aftermath | Check if customer issue started during incident window |
| No incidents, pod healthy | Not an outage | Proceed with normal investigation |

---

## Integration with Investigation

- Run monitoring check in Phase 4 (Parallel Investigation) alongside Splunk/GUS/CodeSearch
- If active incident found → mention in Output Format under "Classification: Known Outage"
- If pod unhealthy → correlate with Splunk error timing
