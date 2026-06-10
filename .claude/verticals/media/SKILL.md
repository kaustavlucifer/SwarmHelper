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

## Key Objects

| Object | Description |
|---|---|
| `MediaAdSalesProduct` | Ad products (inventory types) |
| `MediaAdSalesSlot` | Available ad slots/inventory |
| `MediaAdSalesCampaign` | Ad campaigns |
| `MediaAdSalesBooking` | Booked ad placements |
| `MediaAdSalesTargeting` | Targeting criteria |
| `MediaAdSalesReservation` | Reserved inventory |
| `MediaSubscriber` | Subscriber records (media subscriptions) |
| `Product2` | Products (shared) |
| `Order` / `OrderItem` | Orders |
| `vlocity_cmt__PriceList__c` | Pricing |

---

## Sample SOQL Queries

### Ad campaigns for an account
```soql
SELECT Id, Name, Status, StartDate, EndDate, Budget,
       Account.Name
FROM Campaign
WHERE Account.Id = '<ACCOUNT_ID>'
  AND RecordType.DeveloperName LIKE '%AdSales%'
ORDER BY StartDate DESC LIMIT 10
```

### Subscriber details
```soql
SELECT Id, Name, Status__c, SubscriptionType__c, Account.Name,
       StartDate__c, EndDate__c
FROM MediaSubscriber__c
WHERE Account.Id = '<ACCOUNT_ID>'
ORDER BY StartDate__c DESC
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `axerr` | Apex exceptions (ad pricing, targeting) |
| `ipipr` | Integration Procedures |
| `ipdar` | DataRaptors |
| `axlim` | Governor limits (bulk pricing calculations) |

---

## Code Investigation Paths

### Media Managed Package
```
Tool: mcp__plugin_git-emu_vmcp-git-emu__get_file_contents
owner: "sf-industries"
repo: "via_media"
path: "classes/<ClassName>.cls"
```

### Media PTC Layer
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_cmt/MediaAdSalesPtc.apex"
```

### Media Revenue Core
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/industries-media-revenue-impl/"
```

---

## Escalation

- GUS product tag: `Media Cloud`
- Slack: `#support-swarm-industries`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [Ad Sales | Pricing | Targeting | Campaign Booking | Radio | Subscriber Lifecycle | Other]
Issue Description:
Reproduced in Demo org?:
Troubleshooting steps taken?:
```

---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `known-patterns.md` — Known issue patterns with resolution steps
