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

## Escalation

- Slack: `#support-consumer-goods` or `#support-swarm-industries`
- GUS product tag: `Consumer Goods Cloud`


---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `known-patterns.md` — Known issue patterns with resolution steps
