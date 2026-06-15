---
name: net-zero
description: Net Zero Cloud troubleshooting — carbon accounting, emissions, ESG, energy/waste management, sustainability. Routed by /swarm-helper.
---

# Net Zero Cloud Debugger

**Trigger:** Net Zero Cloud, sustainability, carbon footprint, emissions tracking, ESG, carbon accounting, energy management, waste management, vehicle emissions, `NetZero`, `Sustainability`.

---

## Product Areas

| Area | Scope |
|---|---|
| Carbon Accounting | Scope 1/2/3 emissions, carbon footprint calculation, emission factors |
| Energy Management | Energy consumption tracking, renewable energy, energy audits |
| Waste Management | Waste tracking, recycling rates, circular economy metrics |
| Vehicle Emissions | Fleet emissions, travel emissions, commute tracking |
| Water Management | Water consumption, water recycling, water intensity |
| ESG Reporting | Sustainability reports, regulatory compliance (CSRD, TCFD) |
| Forecasting | Emissions forecasting, reduction targets, pathway modeling |
| Reference Data | Emission factors, conversion factors, activity types |

---

## Repository Architecture

### GitHub Repos (CONFIRMED via CodeSearch)

| Repository | Path | Content |
|---|---|---|
| `Sustainability-App` | `sf-industries/Sustainability-App` | Net Zero Cloud managed package app |

### Core Monorepo Paths (CONFIRMED via CodeSearch)

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-sustainability/                     ← Net Zero Cloud base (CarbonFootprint, EmissionSource)
  core/industries-sustainability-api/                 ← API layer
  core/industries-sustainability-impl/                ← Implementation
  core/industries-sustainability-udd/                 ← UDD (objects, config)
  core/industries-sustainability-shared/              ← Shared utilities
  core/ui-industries-sustainability-components/       ← LWC components
  core/ui-industries-sustainability-impl/             ← UI implementation
  core/ui-industries-sustainability-api/              ← UI API layer
```

> **Note:** Hybrid architecture — managed package app + core Java services. Key classes: `CarbonFootprintDAO`, energy-to-carbon-footprint automation.

### Key Core Classes
- `CarbonFootprintDAO` — `core/industries-sustainability/java/src/industries/sustainability/energyusetocarbonfootprintautomation/CarbonFootprintDAO.java`

---


### PTC Layer

Net Zero Cloud is hybrid (managed app + core Java). No dedicated PTC layer — uses shared OmniStudio PTC if OmniStudio components are involved.

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| Net Zero Cloud (CSG) | https://confluence.internal.salesforce.com/spaces/CSGPAK/pages/458640848/Net+Zero+Cloud |
| Net Zero Cloud Onboarding | https://confluence.internal.salesforce.com/spaces/IN/pages/399485909/Onboarding |
| Architecture Doc (Quip) | https://salesforce.quip.com/o91VAyhoAgj0 |
| ERD Diagram | https://lucid.app/lucidchart/282131b9-47f8-49eb-95fc-579a8c79caff/edit |

---

## Key Objects

| Object | Description |
|---|---|
| `StnryAssetEnvrSrc` | Stationary Asset Environmental Source |
| `StnryAssetCrbnFtprnt` | Stationary Asset Carbon Footprint |
| `VehicleAssetEmssnSrc` | Vehicle Asset Emission Source |
| `PrcsdCrbnFtprntData` | Processed Carbon Footprint Data |
| `SustainabilityGoal` | Reduction targets and goals |
| `BldgEnrgyIntensity` | Building Energy Intensity |
| `WasteFootprint` | Waste tracking records |
| `WaterFootprint` | Water consumption records |
| `EmissionFactor` / `EmissionFactorItem` | Emission factor reference data |
| `EnergyUse` / `EnergyUseItem` | Energy consumption records |
| `SustainabilityIndicator` | KPI/metric tracking |

---

## Common Issues

| Symptom | Check |
|---|---|
| Carbon footprint not calculating | Emission factor configuration, activity data completeness |
| Reference data not loading | Reference data load job status, data format |
| Scope 3 emissions missing | Supply chain data configuration, emission factor mapping |
| ESG report incomplete | Data collection period, indicator configuration |
| Energy intensity calculation wrong | Building area data, energy use normalization |
| Forecasting model inaccurate | Historical data quality, pathway model parameters |
| Batch calculation job failing | Data volume, governor limits, DPE configuration |
| Permission errors on sustainability objects | Permission sets, object-level security |
| Vehicle emissions not tracking | Fleet configuration, fuel type mapping |
| Water/waste data not aggregating | Aggregation rules, reporting period alignment |

---

## Sample SOQL Queries

### Carbon footprint records
```soql
SELECT Id, Name, RecordType.DeveloperName, 
       EmissionsScope1, EmissionsScope2, EmissionsScope3,
       ReportingYear, Status
FROM StnryAssetCrbnFtprnt
WHERE OwnerId = '<USER_ID>'
ORDER BY ReportingYear DESC LIMIT 20
```

### Sustainability goals
```soql
SELECT Id, Name, TargetValue, CurrentValue, Status,
       StartDate, TargetDate, Category
FROM SustainabilityGoal
WHERE OwnerId = '<USER_ID>'
ORDER BY TargetDate ASC
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex uncaught exceptions |
| `axlim` | Governor limit consumption |
| `ipipr` | Integration Procedures (if OmniStudio components used) |
| `ipdar` | DataRaptors (if OmniStudio components used) |
| `gslog` | Platform Java exceptions (core implementation) |

## Code Investigation Paths

### Core Implementation
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-262-public content:Sustainability lang:java"
max_matches: 15
```

### Full Module Listing
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-262-public"
ref: "p4/262-patch"
file_path: "core/industries-sustainability-impl/"
```

---

## Debugging Approach

1. **Identify the emission scope** — Scope 1 (direct), Scope 2 (energy), Scope 3 (supply chain)
2. **Check reference data** — Emission factors must be loaded and mapped correctly
3. **Verify calculation jobs** — Carbon footprint calculations are often batch/DPE
4. **Review data collection** — Activity data (fuel, energy, waste) must be complete
5. **Check reporting periods** — Ensure data aligns with configured reporting periods
6. **Verify permissions** — Net Zero requires specific permission sets

---

## Escalation

- GUS product tag: `Net Zero Cloud`
- Slack: `#sustcloud-engineering-only`, `#sc-code-reviews`, `#sc-tf-fix`, `#nzc-scrum-full`
- Operations: `#nzc-ops`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [Carbon Accounting | Energy | Waste | Vehicle | Water | ESG Report | Forecast | Other]
Emission Scope: [1 | 2 | 3 | N/A]
Issue Description:
Calculation job status (if applicable):
Reproduced in Demo org?:
Troubleshooting steps taken?:
```
