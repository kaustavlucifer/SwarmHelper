# Media Cloud — Ad Sales Debugger

**Trigger:** Media Cloud, Ad Sales Management, media business app, ad inventory, campaign management, `MediaAdSales` classes.

---

## Key Areas

| Area | Scope |
|---|---|
| Ad Sales | Ad inventory management, campaign booking, targeting |
| Pricing | Rate cards, pricing calculations, discounts |
| Integration | External ad server integration, trafficking |

---

## Repository Architecture

### GitHub Repos

| Repository | Path | Content |
|---|---|---|
| `via_media` | `sf-industries/via_media` | Media Cloud managed package |
| `vex_express_media` | `sf-industries/vex_express_media` | Media Cloud Express |
| `media_app_automation` | `sf-industries/media_app_automation` | Media Cloud automation |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-media-revenue/              ← Media revenue management
  core/industries-media-revenue-impl/         ← Media revenue implementation
  core/industries-interaction-ptc/apex/vlocity_cmt/
    MediaAdSalesPtc.apex
    MediaAdSalesRadioPtc.apex
    MediaAdSalesTargetingPtc.apex
  core/industries-interaction-ptc/apex/sfi_media_7/  ← Media PTC (sfi_media namespace)
  core/ui-thunderbird-components/components/setup_thb/mediaManagementSetup/  ← Media setup UI
```

### Managed Package
Search `sf-industries/via_platform` for `MediaAdSales.cls`

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| Communications & Media Cloud (CSG) | https://confluence.internal.salesforce.com/spaces/CSGPAK/pages/522683989/Communications+Media+cloud |

> **Note:** No dedicated engineering support playbook found for Media Cloud. This is a documentation gap.

---

## Common Issues

| Symptom | Check |
|---|---|
| Ad sales pricing errors | `MediaAdSalesPtc.apex`, rate card config |
| Targeting rules failing | `MediaAdSalesTargetingPtc.apex`, audience segment config |
| Campaign booking errors | Inventory availability, scheduling conflicts |
| Radio-specific issues | `MediaAdSalesRadioPtc.apex`, time slot management |
| Performance degradation | SOQL limits in pricing calculations, cache config |

---

## Escalation

- GUS product tag: `Media Cloud`
- Slack: `#support-swarm-industries`


---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `known-patterns.md` — Known issue patterns with resolution steps
