---
name: loyalty-known-patterns
description: Loyalty Management diagnostic patterns — symptom to subsystem to GUS search, grounded in verified core-only paths and the Loyalty Management product tag.
---

# Loyalty Management — Diagnostic Patterns

> Symptom → likely subsystem → how to confirm → GUS search. Patterns are leads; confirm root cause against the current case (Phase 4/5). Release branches use {CURRENT_GA} — resolve per capabilities/codesearch.md.

> **GUS tag note:** Product tag is **`Loyalty Management`** (verified). Loyalty is core-only (no managed package) — implementation lives in `core/industries-loyalty-impl/`. Apply the build-staleness rule from capabilities/gus.md (compare `Scheduled_Build__r.Name` to the org's build).

## Pattern: Points not accruing / accrual transaction error
- **Likely subsystem:** `core/industries-loyalty-impl/java/src/loyalty/impl/actions/` (accrual/processing actions, e.g. `QueryExecutor`); accrual flows under `core/industries-loyalty/`
- **How to confirm:** Check the TransactionJournal for the member (JournalType = Accrual, Status); verify the accrual rule/promotion is active; trace the loyalty engine execution; check `gslog` for platform Java exceptions
- **GUS search:** `Product_Tag__r.Name LIKE '%Loyalty Management%' AND Subject__c LIKE '%accru%'` (also try `%points%`; apply build-staleness rule)

## Pattern: Member currency / points balance wrong or missing (incl. escrow)
- **Likely subsystem:** `core/industries-loyalty-impl/java/src/loyalty/impl/actions/QueryExecutor.java` (`fetchLoyaltyMemberCurrency`); `LoyaltyMemberCurrency` UDD constants/fields
- **How to confirm:** Compare `LoyaltyMemberCurrency` PointsBalance vs. summed `TransactionJournal` (and `LoyaltyLedger`) entries; check for escrow/stale-check discrepancies (known failure mode: escrow points balance missing from a stale-check record); verify no in-flight reversal
- **GUS search:** `Product_Tag__r.Name LIKE '%Loyalty Management%' AND Subject__c LIKE '%currency%'` (also try `%escrow%` / `%balance%`; apply build-staleness rule)

## Pattern: Promotion not applying or wrong promotion dates
- **Likely subsystem:** `core/industries-loyalty-impl/java/src/loyalty/impl/memberpromotionsview/`; promotion rule templates under `core/industries-loyalty-api/`; `runtime_industries_offermgmt` UI namespace
- **How to confirm:** Verify promotion Start/End dates on the `Promotion` / `LoyaltyProgramMbrPromotion` records (known failure mode: dates overridden with defaults 1/1/1970 & 1/1/3000 on new/edit); check eligibility criteria and member segment; confirm `LoyaltyProgramMbrPromotion` linkage
- **GUS search:** `Product_Tag__r.Name LIKE '%Loyalty Management%' AND Subject__c LIKE '%promotion%'` (apply build-staleness rule)

## Pattern: Voucher won't redeem or issue
- **Likely subsystem:** `core/industries-loyalty-impl/java/src/loyalty/impl/vouchermanagement/`; redeem service `.../services/industriesredeemvoucher/`
- **How to confirm:** Check Voucher Status (Issued/Active vs Expired/Redeemed) and EffectiveDate/ExpirationDate; verify VoucherDefinition and the redemption rule; trace the redeem-voucher service
- **GUS search:** `Product_Tag__r.Name LIKE '%Loyalty Management%' AND Subject__c LIKE '%voucher%'` (apply build-staleness rule)

## Pattern: Tier upgrade/downgrade not triggering
- **Likely subsystem:** `core/industries-loyalty-impl/java/src/loyalty/impl/` tier processing; `LoyaltyMemberTier` UDD; member services `.../services/loyaltyprogrammember/`
- **How to confirm:** Check LoyaltyMemberTier EffectiveDate/ExpirationDate history; verify tier-progression thresholds and qualifying-period config; confirm the tier-assessment batch/async job ran (check `axlim` for governor limits on large member volumes)
- **GUS search:** `Product_Tag__r.Name LIKE '%Loyalty Management%' AND Subject__c LIKE '%tier%'` (apply build-staleness rule)

## Pattern: Partner points not transferring / coalition linking issue
- **Likely subsystem:** `core/industries-loyalty-impl/java/src/loyalty/impl/services/partneraccountlinking/`
- **How to confirm:** Verify the partner-account-linking config and LoyaltyProgramPartner agreement; check transfer rules and the cross-program currency mapping; trace the linking service for exceptions
- **GUS search:** `Product_Tag__r.Name LIKE '%Loyalty Management%' AND Subject__c LIKE '%partner%'` (apply build-staleness rule)

## Pattern: Loyalty engine simulation/execution returns unexpected result
- **Likely subsystem:** `core/industries-loyalty-impl/java/src/loyalty/impl/chain/` and `.../actions/` (loyalty engine connect APIs, e.g. LoyaltyEngineSimulationAndExecution)
- **How to confirm:** Run the engine in simulation mode and compare to execution; check rule chain ordering; review `gslog` for engine exceptions; verify the program process/rule is active and published
- **GUS search:** `Product_Tag__r.Name LIKE '%Loyalty Management%' AND Subject__c LIKE '%engine%'` (also try `%simulation%`; apply build-staleness rule)

## Pattern: Data Processing Engine (DPE) / batch tier or expiration job failed
- **Likely subsystem:** `core/industries-loyalty-impl/java/src/loyalty/impl/dpe/`; `.../transactionjournalcount/`
- **How to confirm:** Check the DPE definition run status and the batch job logs; verify member volume against governor limits (`axlim`); confirm the expiration policy / schedule config for point-expiration runs
- **GUS search:** `Product_Tag__r.Name LIKE '%Loyalty Management%' AND Subject__c LIKE '%batch%'` (also try `%expiration%` / `%DPE%`; apply build-staleness rule)

## Pattern: Member enrollment / benefit assignment fails
- **Likely subsystem:** `core/industries-loyalty-impl/java/src/loyalty/impl/services/loyaltyprogrammember/` and `.../services/benefits/`
- **How to confirm:** Check required fields and duplicate-member rules on LoyaltyProgramMember; verify program capacity; for benefits, confirm BenefitType/MemberBenefit assignment and the benefit service execution
- **GUS search:** `Product_Tag__r.Name LIKE '%Loyalty Management%' AND Subject__c LIKE '%enroll%'` (also try `%member%` / `%benefit%`; apply build-staleness rule)
