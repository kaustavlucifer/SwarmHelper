---
name: tpm
description: Trade Promotion Management / Consumer Goods troubleshooting — trade promotions, KPI batch, push promotions, claims, Funding Grid, cgcloud. Routed by /swarm-helper.
---

# Trade Promotion Management (TPM) Debugger

**Trigger:** TPM, Consumer Goods Cloud, trade promotions, KPI calculations, nightly batch jobs, push promotions, payment/claim lifecycle, Agentforce for TPM, RTR reports, Funding Grid, `cgcloud` namespace.

---

## Repository Architecture

### GitHub / git.soma Repos

| Repository | Path | Content |
|---|---|---|
| `rcgps-retail-tpm` | `git.soma.salesforce.com/industries-rcg/rcgps-retail-tpm` | TPM managed package (primary, active) |
| `rcg-retail-tpm` | `git.soma.salesforce.com/Localization/rcg-retail-tpm` | TPM localization branch (see known-patterns.md) |
| `cgcloud-solutions` | `github.com/salesforce-internal/cgcloud-solutions` | Consumer Goods Cloud solutions |
| `RCG_RE_TPM` | `github.com/sf-industries/RCG_RE_TPM` | RCG Retail Execution TPM |

> Repo names validated 2026-06-15. The primary TPM package is `industries-rcg/rcgps-retail-tpm` (the old `industries-rcg/rcg-retail-tpm`, `rcg-retail-se`, and `RCGSF_SF_Mobility_Sync` paths do not resolve).

---


### Off-Core Processing Layer (CG Cloud Processing Services — source-validated 2026-06-15)

TPM has **two tiers**: the Salesforce-side managed package (Apex, on git.soma) **and** an off-core Node.js processing service that runs the heavy calculation/batch work on Hyperforce (Falcon/k8s). Most KPI-calculation, batch, and payment-engine failures live in the off-core tier, **not** in Apex.

The off-core code is a **monorepo under `packages/`** in `git.soma.salesforce.com/industries-rcg/rcgps-retail-tpm` — NOT a `classes/*.cls` Apex layout. Use the current release branch `refs/heads/release-{CURRENT_GA}` (see `.claude/capabilities/codesearch.md` → Release Resolution; the default branch is `master`). Confirmed packages under `packages/tpm/`:

| Package | Role |
|---|---|
| `rcgps-tpm-service` | Service entry point (Express REST + queue worker), worker dispatch, K8s deployment |
| `rcgps-accplnprm` | Account-planning & promotions calculation worker |
| `rcgps-calcengine` | Calculation engine (KPI value calculation) |
| `rcgps-kpi` | KPI definitions/processing |
| `rcgps-measureapi` | Measure read/write API |
| `rcgps-productcache` | Product cache |
| `rcgps-rtreporting` | RTR (real-time reporting) + measures export |
| `rcgps-accrual` | Accrual engine |
| `rcgps-reorg` / `rcgps-customcalendar` / `rcgps-updateactivation` / `rcgps-transferdblog` / `rcgps-datacloud` | Supporting workers |

(`rcgps-hierarchycustomer` lives under `packages/retail/`.)

### Service runtime model (read from `rcgps-tpm-service` source)

The container's role is chosen by the `SYSTEM_PROCESS_ID` env var (`entrypoint.js`):
- `TPMSERVICE` → clustered Express **REST web service** (`lib/rest/cluster.js`)
- `TPMCALCULATIONWORKER` (or other) → **queue worker** (`lib/worker/worker-start.js` → `worker-entry.js`) that consumes the SQS queue `TPMCALC` and runs `CalculationWorkerJob`.

**Calculation types** the worker dispatches on (SQS message attr `calculationType`, from `lib/common/constants.js`):
`PromotionCalculation`, `PromotionDistribution`, `AccountPlanScenarioPromotionCalculation`, `AccountPlanCategoryCalculation`, `AccountPlanScenarioCategoryCalculation`, `AccountPlanCalculation`, `AccountPlanScenarioCalculation`, `PNLAnalyzeCalculation`, plus `Test`/`TestCalculationPlan`. Required message attributes (else job skipped): `organizationId`, `txId`, `sessionToken`, `logLevel`, `calculationType`, `calculationTxInternalId`, `calculationPlanId`, `apiId`, `apiMajorVersion`, `apiMinorVersion`.

**Licensing gate:** non-health requests require header `x-is-tpm: true` (else HTTP 403 *"Missing license to use TPM features"*); `updateactivations` / application-limit routes accept `x-is-retail: true` (`lib/rest/app.js` `checkIsTpmHeader`).

### Core Monorepo

TPM is **not** in the core monorepo (`core/industries-rcg/` is not a confirmed path). It is the managed package + off-core service above. No dedicated PTC layer — uses shared OmniStudio PTC classes when IPs/DRs are involved.

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| CGCloud Retail Support Playbook | https://confluence.internal.salesforce.com/spaces/IN/pages/831724707/CGCloud+Retail+Support+Playbook |
| Consumer Goods Cloud | https://confluence.internal.salesforce.com/spaces/IN/pages/188429750/Consumer+Goods+Cloud |
| RCG TPM Product | https://confluence.internal.salesforce.com/spaces/IN/pages/423788550/RCG+TPM+Product |

---

## Key Objects

> **Verified 2026-06-15** against a TPM (`cgcloud`) org. Corrected names that don't exist in the package: KPIs are `cgcloud__KPI_Definition__c` / `cgcloud__KPI_Set__c` / `cgcloud__Promotion_Template_KPI_Definition__c` (not `Promotion_KPI__c`); batch tracking is `cgcloud__Batch_Run_Status__c` (not `Batch_Run__c`); there is no `cgcloud__Claim__c` object.

| Object | Description |
|---|---|
| `cgcloud__Promotion__c` | Trade promotions (✅ verified) |
| `cgcloud__KPI_Definition__c` / `cgcloud__KPI_Set__c` / `cgcloud__Promotion_Template_KPI_Definition__c` | KPI definitions and sets (✅ verified) |
| `cgcloud__Fund__c` | Funding (✅ verified) |
| `cgcloud__Payment__c` / `cgcloud__Payment_Tactic__c` | Payment lifecycle (✅ verified) |
| `cgcloud__Batch_Run_Status__c` / `cgcloud__Batch_Run_Status_Detail__c` | Batch job tracking (✅ verified) |

### Retail Execution objects (Consumer Goods Cloud)

> **Verified 2026-06-15** against a Retail Consumer Goods org. The retail-execution side (store visits, audits, orders, assortments) uses both `cgcloud__` package objects and standard Salesforce Consumer Goods objects.

| Object | Description |
|---|---|
| `cgcloud__Visit_Job__c` / `cgcloud__Visit_Template__c` / `cgcloud__Account_Visit_Setting__c` | Store visit jobs, templates, settings (✅ verified) |
| `cgcloud__Asset_Audit__c` | In-store asset/shelf audit (✅ verified) |
| `cgcloud__Order__c` / `cgcloud__Order_Item__c` / `cgcloud__Order_Template__c` | Retail orders + items + templates (✅ verified) |
| `cgcloud__Product_Assortment_Template__c` / `cgcloud__Product_Assortment_Store__c` | Product assortments by store (✅ verified) |
| `RetailStore` / `InStoreLocation` / `RetailLocationGroup` | Standard store / in-store location / location group (✅ verified) |
| `RetailStoreKpi` / `RetailVisitKpi` | Standard store + visit KPIs (✅ verified) |
| `ProductCategory` / `AssessmentTask` | Standard product category + visit assessment tasks (✅ verified) |

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

## Off-Core REST Endpoints & Log Codes (from `rcgps-tpm-service` source)

All versioned routes sit behind a `/v<N>/` prefix. Health/ops routes have no prefix.

| Method | Path | Purpose |
|---|---|---|
| POST | `/v<N>/calculationplans/schedule` | Schedule a calculation plan (body limit ~2.5 MB) |
| GET | `/v<N>/calculationplans/:id/status` | Calculation plan status |
| GET | `/v<N>/features/:featurename` | Feature flag lookup |
| GET | `/v<N>/rtr/reportdata` · `/rtr/reportmeta` · `/rtr/performancemetrics` · `/rtr/scenariocomparison` | Real-time reporting (rcgps-rtreporting) |
| POST/GET | `/v<N>/measures/export/schedule` · `/measures/export/csv` · `/measures/export/:csvGuid/status` · `/measures/export/:csvGuid/commit` · `/measures/exports` | Measures CSV export |
| POST | `/v<N>/datacloud-export/enqueue` | Data Cloud export (rcgps-datacloud) |
| GET/POST | `/v<N>/updateactivations` · application-limit routes | Update activation / limits (accept `x-is-retail`) |
| GET | `/manage/health`, `/manage/health/live`, `/manage/health/ready`, `/metrics` | k8s liveness/readiness + Prometheus metrics |

**Log-code families** (prefix → area, from `lib/common/log-messages.js`) — use these to grep Splunk/k8s logs:

| Prefix | Area | Notable codes |
|---|---|---|
| `SVC####` | REST server / cluster lifecycle | `SVC0014` soft-limit validation failed (service won't start) |
| `HEALTH####` | readiness/liveness, CPU/mem thresholds | — |
| `TPMWRK####` | worker cluster lifecycle | `TPMWRK0004` not enough resources, `TPMWRK0008` cluster worker died |
| `TPMCALCWRK####` | calculation worker | `0001` unknown process id, `0004` error processing queue msg, `0007` tenant-context mismatch, `030/031` duplicate/already-processed job skipped |
| `CALCPLAN####` | schedule/get calc plan | `CALCPLAN0005` no plan for id, `0009`–`0012` step/promotion count limits exceeded |
| `GETFEATURE####` | feature lookup | `0002` invalid featurename, `0004` feature not found |

> Limits live in `lib/common/constants.js` (e.g. `MAX_PROMOTION_PERCALCULATIONPLAN: 2000`, `MAX_ACCOUNTPLANCATEGORY_STEP: 10`). Worker log-code tables for other packages (kpi, accplnprm, rtreporting, etc.) live in their own `packages/tpm/<pkg>` dirs.

---

## Debugging Approach

1. **Check Batch Run records** — `cgcloud__Batch_Run_Status__c` (+ `_Detail__c`) shows status and errors
2. **Nightly calculation log** — Debug logs during scheduled window
3. **KPI formula audit** — Verify calculation formula matches expected logic
4. **Push promotion hierarchy** — Parent → Child distribution rules
5. **Payment lifecycle state machine** — Valid state transitions

---

## Data Model Relationships

```
cgcloud__Promotion__c (Trade Promotion)                              ✅ verified
├── cgcloud__Tactic__c (promotion tactics/activities)                ✅ verified
│   └── cgcloud__Tactic_Fund__c (tactic↔fund junction)               ✅ verified
├── cgcloud__Fund__c (funding allocation)                            ✅ verified
│   ├── cgcloud__Fund_Transaction__c (fund transactions)             ✅ verified
│   └── cgcloud__Fund_Product__c (fund product scope)                ✅ verified
├── cgcloud__Payment__c (payments)                                   ✅ verified
│   └── cgcloud__Payment_Tactic__c (payment↔tactic detail)           ✅ verified
└── cgcloud__Promotion_Template_KPI_Definition__c (KPI defs)         ✅ verified

cgcloud__KPI_Set__c → cgcloud__KPI_Definition__c (KPI catalog)       ✅ verified
cgcloud__Batch_Run_Status__c (Batch Job Tracking)                    ✅ verified
└── cgcloud__Batch_Run_Status_Detail__c (per-step status)            ✅ verified
```
> Verified against a TPM org 2026-06-15. Earlier diagram objects `Promotion_KPI__c`, `Promotion_KPI_Value__c`, `Tactic_KPI__c`, `Fund_Item__c`, `Claim__c`, `Condition__c`, `Batch_Run__c`, `Batch_Run_Step__c` returned 404 (do not exist in the package).

---

## Sample SOQL Queries

> Field API names below are verified where noted; TPM promotions use **date-range + account-set targeting** (`cgcloud__Date_From__c`/`cgcloud__Date_Thru__c`, `cgcloud__Anchor_Account__c`), not a single `Account__c`/`Status__c`. Always `describe` the object in the target org before finalizing field selections.

### Promotions (by date window)
```soql
SELECT Id, Name, cgcloud__Date_From__c, cgcloud__Date_Thru__c,
       cgcloud__Anchor_Account__c, cgcloud__Commit_Date__c
FROM cgcloud__Promotion__c
WHERE cgcloud__Anchor_Account__c = '<ACCOUNT_ID>'
ORDER BY cgcloud__Date_From__c DESC LIMIT 10
```
> KPIs are not child records of a promotion — they live in the catalog (`cgcloud__KPI_Set__c` → `cgcloud__KPI_Definition__c`) and are attached via `cgcloud__Promotion_Template_KPI_Definition__c`.

### Batch run status (verified fields)
```soql
SELECT Id, Name, cgcloud__Batch_State__c, cgcloud__Batch_Type__c, cgcloud__Job_Name__c,
       cgcloud__Start_Date__c, cgcloud__End_Date__c, cgcloud__Process_Count__c, cgcloud__Items_Error__c
FROM cgcloud__Batch_Run_Status__c
WHERE cgcloud__Batch_State__c != 'Completed'
ORDER BY cgcloud__Start_Date__c DESC LIMIT 10
```

### Payments (verified fields)
```soql
SELECT Id, Name, cgcloud__Payment_Status__c, cgcloud__Payment_Amount__c,
       cgcloud__Family_Status__c, cgcloud__Rejected_Amount__c
FROM cgcloud__Payment__c
ORDER BY CreatedDate DESC LIMIT 20
```

### Fund allocations
```soql
SELECT Id, Name, cgcloud__Budget__c, cgcloud__Committed__c, cgcloud__Spent__c,
       cgcloud__Status__c
FROM cgcloud__Fund__c
WHERE cgcloud__Account__c = '<ACCOUNT_ID>'
ORDER BY Name
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `r1log` | Industries package instrumentation (filter by `instKey`) |
| `axerr` | Apex uncaught exceptions (KPI calculation, batch failures) |
| `axlim` | Governor limits (nightly batch, promotion load) |
| `axque` | Queueable jobs (async KPI processing) |
| `axftr` | @future methods (push promotion distribution) |

---

## Code Investigation Paths

### TPM Managed Package (Apex tier)
```
Tool: mcp__plugin_git-soma_vmcp-git-soma__get_file_contents
owner: "industries-rcg"
repo: "rcgps-retail-tpm"
ref: "refs/heads/release-{CURRENT_GA}"   # resolve per codesearch.md; default branch is master
path: "<apex package path>"   ← managed package Apex
```

### TPM Off-Core Processing Service (Node.js tier — for KPI/batch/calc failures)
```
Tool: mcp__plugin_git-soma_vmcp-git-soma__get_file_contents
owner: "industries-rcg"
repo: "rcgps-retail-tpm"
ref: "refs/heads/release-{CURRENT_GA}"   # resolve per codesearch.md; default branch is master
path: "packages/tpm/<package>/src/..."   ← e.g. packages/tpm/rcgps-calcengine/src
```
Pick the package by failure type: calc → `rcgps-calcengine`/`rcgps-accplnprm`, KPI → `rcgps-kpi`/`rcgps-measureapi`, RTR → `rcgps-rtreporting`, accrual → `rcgps-accrual`, service/dispatch → `rcgps-tpm-service`.

### Consumer Goods Solutions
```
Tool: mcp__plugin_deep-research_codesearch__read_file
repository: "github.com/salesforce-internal/cgcloud-solutions"
ref: "HEAD"
file_path: "<path>"
```

---

## Batch Job Architecture

TPM relies heavily on nightly batch processing:

1. **KPI Calculation Batch** — Runs nightly, calculates all KPIs based on actuals
2. **Push Promotion Distribution** — Distributes parent spend to child promotions
3. **Payment Lifecycle Batch** — Transitions payments through lifecycle states
4. **Fund Reconciliation** — Updates fund committed/spent values

**Debugging batch failures:**
1. Check `cgcloud__Batch_Run_Status__c` for error messages
2. Check scheduled job status (Setup → Scheduled Jobs)
3. Look for governor limit hits in debug logs during batch window
4. Verify no conflicting batch jobs running simultaneously

---

## Escalation

- Slack: `#ent-consumer-goods-se-support` (C098184TA5S) — Consumer Goods SE support
- General: `#support-swarm-industries` (C02BEHKLWES)
- GUS product tag: `Consumer Goods Cloud`

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [KPI Calculation | Batch Jobs | Push Promotions | Payments/Claims | Funding | Agentforce | Other]
Issue Description:
Batch Run status (if applicable):
Nightly job timing:
Reproduced in Demo org?:
Troubleshooting steps taken?:
```

---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `known-patterns.md` — Known issue patterns with resolution steps
