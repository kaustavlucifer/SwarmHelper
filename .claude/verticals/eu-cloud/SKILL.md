# Energy & Utilities Cloud Debugger

**Trigger:** Energy & Utilities Cloud, E&U, CAM (Customer Acquisition & Management), multisite orders, VEEDigitalGetBasket, DC API cache, package upgrade errors, billing & usage components, E&U Business App performance.

---

## Repository Architecture

### GitHub Repos

| Repository | Path | Content |
|---|---|---|
| `via_energy` | `sf-industries/via_energy` | Energy managed package |
| `vpl_energy` | `sf-industries/vpl_energy` | Energy VPL package |
| `vex_express_energy` | `sf-industries/vex_express_energy` | Energy Express package |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-energy-utilities-udd/                 ← E&U Cloud UDD + configuration
  core/ui-industries-energy-utilities-components/       ← E&U UI (agent console, billing, self-serve)
  core/ui-industries-energy-utilities-sales-components/ ← E&U sales components
  core/industries-sustainability/                       ← Sustainability (energy → carbon footprint)
```

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| E&U Functional Architecture | https://confluence.internal.salesforce.com/spaces/IN/pages/938345591/Energy+Utilities+Functional+Architecture |
| Energy and Utilities Cloud | https://confluence.internal.salesforce.com/spaces/IN/pages/630891843/Energy+and+Utilities+Cloud |
| EUC Overview (Onboarding) | https://confluence.internal.salesforce.com/spaces/IN/pages/1240629424/EUC+Overview |
| E&U Tech Talk | https://confluence.internal.salesforce.com/spaces/IN/pages/1226185780/Energy+Utilities+Tech+Talk |

---

## Key Areas

| Area | Scope |
|---|---|
| CAM | Customer acquisition, quote/order management, multisite |
| DC APIs | Digital Commerce APIs, Regenerate Cache, Batch Jobs |
| Billing & Usage | Billing components, usage tracking, metering |
| EPC | Product catalog (shared with Comms Cloud) |

---

## Common Issues

| Symptom | Check |
|---|---|
| CAM performance degradation | SOQL limits in VEEDigitalGetBasket, cache config |
| DC API cache job failures | Batch job status, regenerate API config |
| Too many SOQL queries in basket | N+1 patterns in custom IP, product hierarchy depth |
| Integration Procedure timeouts | HTTP callout config, external system response time |
| Package upgrade record locks | Concurrent upgrade jobs, lock contention |
| Multisite order performance | Order line count, decomposition complexity |
| CPU time limit exceeded | Trigger stacking, complex pricing calculations |
| Billing component loading | LWC access, data visibility rules |
| Cache regeneration failing | Cache size limits, data volume |

---

## Debugging Approach

1. **Check DC API cache status** — Batch Run records for cache jobs
2. **Basket performance** — Debug log for SOQL count in VEEDigitalGetBasket
3. **Multisite scaling** — Order line count × sites = total processing
4. **Package upgrade** — Check for concurrent jobs, record lock patterns

---

## Escalation

- GUS product tag: `Energy & Utilities Cloud`
- Slack: `#support-swarm-industries`


---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `known-patterns.md` — Known issue patterns with resolution steps
