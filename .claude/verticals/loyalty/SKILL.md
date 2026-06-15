---
name: loyalty
description: Loyalty Management troubleshooting — programs, points, tiers, rewards, vouchers, partner loyalty. Routed by /swarm-helper.
---

# Loyalty Management Debugger

**Trigger:** Loyalty Management, loyalty programs, rewards, points, tiers, member benefits, promotions (loyalty context), vouchers, partner management (loyalty), `LoyaltyManagement`.

---

## Product Areas

| Area | Scope |
|---|---|
| Program Management | Loyalty program creation, configuration, tiers, benefits |
| Points & Rewards | Point accumulation, redemption, expiration, transfer |
| Member Management | Member enrollment, tier progression, status tracking |
| Promotions | Loyalty promotions, campaigns, targeted offers |
| Vouchers | Voucher creation, distribution, redemption, expiration |
| Partner Management | Partner enrollment, point sharing, coalition programs |
| Analytics | Loyalty analytics, member engagement metrics |
| Transaction Processing | Accrual transactions, redemption transactions, adjustments |

---

## Repository Architecture

### GitHub / git.soma Repos

> **Note:** Loyalty Management has no managed package repo — it is entirely core-only (Java-based).

### Core Monorepo Paths (CONFIRMED via CodeSearch)

```
gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public:
  core/industries-loyalty/                          ← Loyalty Management base
  core/industries-loyalty-api/                      ← API layer (public interfaces)
  core/industries-loyalty-impl/                     ← Implementation (Java services)
  core/industries-loyalty-udd/                      ← UDD (objects, config)
  core/ui-industries-loyalty-api/                   ← UI API layer (LoyaltyProgramTier.java etc.)
```

> **Note:** Loyalty Management is core-only — no dedicated managed package. Has CDP (Customer Data Platform) integration.

---


### PTC Layer

Loyalty is core-only (no PTC layer). No managed package Apex — Java-based implementation in `core/industries-loyalty-impl/`.

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| Loyalty Management (IN space) | https://confluence.internal.salesforce.com/spaces/IN/pages/188429748/Loyalty+Management |
| Loyalty Product Overview | https://confluence.internal.salesforce.com/pages/viewpage.action?pageId=245296521 |
| Loyalty Technical Docs | https://confluence.internal.salesforce.com/pages/viewpage.action?pageId=224271763 |
| Loyalty 101 (Quip) | https://salesforce.quip.com/G708AlxVRsoG |

---

## Key Objects

> **Verified 2026-06-15** against a Loyalty-enabled org via `describe`. Corrected two names that don't exist: `TransactionJournalEntry` (there is only `TransactionJournal`) and `LoyaltyPromotion` (the real promotion-link objects are `LoyaltyProgramMbrPromotion` / `LoyaltyPgmPartnerPromotion`).

| Object | Description |
|---|---|
| `LoyaltyProgram` | Program definition (tiers, rules, benefits) (✅ verified) |
| `LoyaltyProgramMember` | Member enrollment record (✅ verified) |
| `LoyaltyMemberTier` | Member's current/historical tier (✅ verified) |
| `LoyaltyMemberCurrency` | Member's point balances (✅ verified) |
| `TransactionJournal` | Point accrual/redemption transactions (✅ verified) |
| `LoyaltyLedger` / `LoyaltyProgramPartnerLedger` | Point ledger entries (✅ verified) |
| `LoyaltyProgramMbrPromotion` / `LoyaltyPgmPartnerPromotion` | Member / partner promotion links (✅ verified) |
| `Voucher` / `VoucherDefinition` | Voucher records and templates (✅ verified) |
| `LoyaltyPartnerProduct` / `PromotionLoyaltyPtnrProdt` | Partner products for redemption (✅ verified) |
| `LoyaltyProgramPartner` | Coalition partner records (✅ verified) |
| `MemberBenefit` / `BenefitType` | Benefits assigned to members + type definitions (✅ verified) |

---

## Common Issues

| Symptom | Check |
|---|---|
| Points not accruing | Transaction processing rules, accrual trigger configuration |
| Tier upgrade not triggering | Tier progression rules, qualifying period configuration |
| Voucher redemption failing | Voucher status (Active/Expired), redemption rules |
| Member enrollment error | Program capacity, required fields, duplicate check |
| Promotion not applying | Promotion dates, eligibility criteria, member segment |
| Point expiration running unexpectedly | Expiration policy configuration, batch job schedule |
| Partner points not transferring | Partner agreement configuration, transfer rules |
| Transaction journal discrepancy | Transaction type (Accrual/Redemption/Adjustment), reversal records |
| Loyalty API integration failing | Connected App, API versioning, rate limits |
| Analytics dashboard empty | Dataset refresh, permission set, data sync timing |
| Batch job for tier processing failed | Governor limits, member volume, DPE configuration |

---

## Sample SOQL Queries

### Member with balances and tier
```soql
SELECT Id, MembershipNumber, MemberStatus, EnrollmentDate,
       LoyaltyProgram.Name,
       (SELECT Id, LoyaltyMemberCurrencyName, PointsBalance, TotalPointsAccrued 
        FROM LoyaltyMemberCurrencies),
       (SELECT Id, LoyaltyTier.Name, EffectiveDate, ExpirationDate 
        FROM LoyaltyMemberTiers ORDER BY EffectiveDate DESC LIMIT 1)
FROM LoyaltyProgramMember
WHERE MembershipNumber = '<MEMBER_NUMBER>'
LIMIT 1
```

### Recent transactions for a member
```soql
SELECT Id, JournalType, JournalSubType, Status, ActivityDate,
       PointsChange, TransactionAmount, LoyaltyProgram.Name
FROM TransactionJournal
WHERE LoyaltyProgramMemberId = '<MEMBER_ID>'
ORDER BY ActivityDate DESC LIMIT 20
```

### Active vouchers
```soql
SELECT Id, VoucherCode, Status, EffectiveDate, ExpirationDate,
       VoucherDefinition.Name, LoyaltyProgramMember.MembershipNumber
FROM Voucher
WHERE LoyaltyProgramMemberId = '<MEMBER_ID>'
  AND Status = 'Issued'
ORDER BY ExpirationDate ASC
```

---

## Splunk logRecordTypes

| Type | Use |
|---|---|
| `gslog` | **Platform Java exceptions/gacks (primary — core implementation)** |
| `axerr` | Apex uncaught exceptions |
| `axlim` | Governor limit consumption |
| `r1log` | Industries instrumentation — only if the managed-package / OmniStudio layer is in use; core-native features log via `gslog`, not `r1log` |
| `ipipr` | Integration Procedures (only if OmniStudio components used) |
| `ipdar` | DataRaptors (only if OmniStudio components used) |

## Code Investigation Paths

### Core Implementation
```
Tool: mcp__plugin_deep-research_codesearch__search
query: "repo:gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public content:Loyalty lang:java"
max_matches: 15
```

### LWC Components
```
Tool: mcp__plugin_deep-research_codesearch__list_directory
repository: "gitcore.soma.salesforce.com/core-2206/core-{CURRENT_GA}-public"
ref: "p4/{CURRENT_GA}-patch"
file_path: "core/ui-industries-loyalty-components/"
```

---

## Debugging Approach

1. **Identify the feature area** — Points, Tiers, Vouchers, Promotions, Partners
2. **Check transaction journals** — All point movements are tracked here
3. **Verify program configuration** — Accrual rules, tier thresholds, expiration policies
4. **Check batch/async jobs** — Tier processing, point expiration are batch operations
5. **Review member status** — Active vs Inactive members have different processing rules
6. **Check API integrations** — External loyalty systems often integrate via REST API

---

## Scrum Teams

- Loyalty Lannisters
- Loyalty Night Watch
- Loyalty Arryns

---

## Escalation

- GUS product tag: `Loyalty Management`
- Slack: `#support-swarm-industries`
- Mailing lists: IndustriesProductLoyalty@salesforce.com, IndustriesEngineeringLoyalty@salesforce.com

**Swarm template:**
```
Customer Sentiment:
Current Condition:
Feature: [Points/Accrual | Tiers | Vouchers | Promotions | Partners | Enrollment | Other]
Issue Description:
Member Number (if applicable):
Program Name:
Reproduced in Demo org?:
Troubleshooting steps taken?:
```

---

## Detailed Pattern Files

- `known-patterns.md` — Loyalty Management diagnostic patterns (symptom → subsystem → confirm → GUS search).
