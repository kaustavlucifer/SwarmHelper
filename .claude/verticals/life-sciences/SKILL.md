# Life Sciences Cloud Debugger

**Trigger:** Life Sciences Cloud, clinical trials, drug programs, MedTech, patient services, consent management, referral management (LS context), REMS, companion diagnostics.

---

## Product Areas

| Area | Scope |
|---|---|
| Patient Services | Patient enrollment, consent, benefit verification |
| Clinical | Trial management, site selection, patient recruitment |
| Commercial | HCP engagement, samples, formulary |
| MedTech | Device tracking, remote monitoring (shared with HC) |

---

## Repository Architecture

### GitHub Repos

| Repository | Path | Content |
|---|---|---|
| `lifesciences` | `sf-industries-ls/lifesciences` | Life Sciences main repo (note: separate org) |
| `HealthCloudCCDAandEMRDatakit` | `sf-industries/HealthCloudCCDAandEMRDatakit` | CCDA/EMR data kit (shared with HC) |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-lifesciences-impl/                   ← Life Sciences Java (trial mgmt, research study)
  core/ui-industries-common-components/omnistudio/     ← Shared components
```

### Engineering Playbooks

| Resource | URL |
|---|---|
| Life Sciences | https://confluence.internal.salesforce.com/spaces/IN/pages/827098968/Life+Sciences |
| Healthcare and Life Sciences (CCEIND) | https://confluence.internal.salesforce.com/spaces/CCEIND/pages/601662751/Healthcare+and+Life+Sciences |
| Epic Narratives (Quip) | https://salesforce.quip.com/GEpaAnwMbhiH |

> **Note:** No dedicated engineering support playbook found for Life Sciences. This is a documentation gap — consider researching `#support-swarm-industries` Slack for tribal knowledge.

---

## Common Issues

| Symptom | Check |
|---|---|
| Patient enrollment IP failing | PTC layer, callout configuration |
| Consent management errors | Consent object permissions, data sharing |
| Clinical trial data model | Custom objects, relationship configuration |
| Drug program setup | Program configuration, eligibility rules |
| HCP engagement tracking | Activity capture config, compliance rules |
| Sample management errors | Inventory objects, allocation rules |

---

## Escalation

- GUS product tag: `Life Sciences Cloud`
- Slack: `#support-swarm-industries`
