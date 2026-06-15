---
name: media
description: Media Cloud / Ad Sales troubleshooting — ad inventory, campaign management, MediaAdSales, media business app. Routed by /swarm-helper.
---

# Media Cloud — Ad Sales Debugger

**Trigger:** Media Cloud, Ad Sales Management, media business app, ad inventory, campaign management, `MediaAdSales` classes.

---

## Product Areas

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
gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public:
  core/industries-media-revenue/              ← Media revenue management
  core/industries-media-revenue-impl/         ← Media revenue implementation
  core/industries-interaction-ptc/apex/vlocity_cmt/
    MediaAdSalesPtc.apex
    MediaAdSalesRadioPtc.apex
    MediaAdSalesTargetingPtc.apex
  core/industries-interaction-ptc/apex/sfi_media_7/  ← Media PTC (MediaAdSalesPtc, MediaPlanHandlerPtc — validated 2026-06-15)
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

> **Verified 2026-06-15** against a Media Cloud (Ad Sales) org via `describe`. The real Ad Sales schema is the **`Ad*` standard-object family** — the earlier `MediaAdSales*` / `MediaSubscriber__c` names were **fabricated** and have been removed. All objects below returned HTTP 200.

| Object (verified ✅) | Description |
|---|---|
| `AdQuote` / `AdQuoteLine` | Ad quote + lines (core of Ad Sales quoting) |
| `AdQuoteLineUnitsSplit` / `AdQuoteLinePrintIssue` | Quote-line unit splits / print-issue scheduling |
| `AdQuoteLineAdTarget` / `AdQuoteLineCreativeSizeType` / `AdQuoteLineHiatus` | Quote-line targeting, creative sizes, hiatus periods |
| `AdOrderItem` | Ad order line items |
| `AdOrderItemUnitsSplit` / `AdOrderItemPrintIssue` / `AdOrderItemCreativeSizeType` | Order-item splits / print issues / creative sizes |
| `AdOrderLineAdTarget` / `AdOrderLineHiatus` | Order-line targeting / hiatus |
| `AdOpportunity` | Ad sales opportunity |
| `AdServer` / `AdServerAccount` / `AdServerUser` / `AdBuyServerAccount` | Ad-server integration + trafficking accounts |
| `AdDigitalAvailability` / `AdLinearAvailability` / `AdAvailabilityJob` / `AdAvailabilityDimensions` / `AdAvailabilityViewConfig` | Inventory availability (digital + linear) |
| `AdTargetCategory` / `AdTargetCategorySegment` / `AdProductTargetCategory` | Targeting categories / segments |
| `AdCreativeSizeType` / `AdDemographicCode` / `AdPageLayoutType` | Creative sizes, demographic codes, layouts |
| `AdSpaceSpecification` / `AdSpaceCreativeSizeType` / `AdSpaceGroupMember` / `AdSpecMediaPrintIssue` | Ad-space specs and print issues |
| `Product2` | Products (shared) |
| `Order` / `OrderItem` | Orders |
| `vlocity_cmt__PriceList__c` | Pricing (managed pkg, when present) |

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

### Ad quotes + lines for an account
```soql
SELECT Id, Name, Status, TotalAmount,
       (SELECT Id, Name, NetAmount, AdProduct.Name FROM AdQuoteLines)
FROM AdQuote
WHERE Opportunity.Account.Id = '<ACCOUNT_ID>'
ORDER BY CreatedDate DESC LIMIT 10
```
> Field API names (`Status`, `TotalAmount`, `NetAmount`, child-relationship `AdQuoteLines`) should be confirmed via `describe` on the target org — object names are verified; field names per-object are not yet swept.

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex exceptions (ad pricing, targeting) |
| `ipipr` | Integration Procedures |
| `ipdar` | DataRaptors |
| `axlim` | Governor limits (bulk pricing calculations) |

---

## Code Investigation Paths

### Media Managed Package
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "github.com/sf-industries/via_media"
ref: "HEAD"
file_path: "classes/<ClassName>.cls"
```

### Media PTC Layer
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/industries-interaction-ptc/apex/vlocity_cmt/MediaAdSalesPtc.apex"
```

### Media Revenue Core
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/industries-media-revenue-impl/"
```

---

## Escalation

- GUS product tag: `Media Cloud`
- Slack: `#industries-media` (C01JESF340P) — Media & Entertainment community
- General: `#support-swarm-industries` (C02BEHKLWES)

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
