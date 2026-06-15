# SwarmHelper Changelog

These skills are developed and maintained by Kaustav Chowdhury.

## v2.5.0 — 2026-06-15

**Live-org object verification sweep.** With demo-org credentials supplied per vertical, every `Key Objects` table was probed via live org connection (SOAP `describe` for SOAP-enabled orgs; `sf` CLI web-auth + `EntityDefinition`/`FieldDefinition`/Tooling API for SOAP-disabled orgs). Fabricated sObject/field names — caught only as "⚠️ pending org sweep" before — are now replaced with real, verified API names. **Ten verticals corrected against live orgs; two more (E&U, Manufacturing) corrected/tagged from cross-vertical findings.**

### Fixed (object API names — verified against live orgs)
- **Media** — the entire `MediaAdSales*` / `MediaSubscriber__c` object family was **fabricated**. Real Ad Sales schema is the **`Ad*` standard-object family**: `AdQuote`, `AdQuoteLine`, `AdQuoteLineUnitsSplit`, `AdQuoteLinePrintIssue`, `AdOrderItem`, `AdOpportunity`, `AdServer`/`AdServerAccount`, `AdDigitalAvailability`/`AdLinearAvailability`/`AdAvailabilityJob`, `AdTargetCategory`, `AdCreativeSizeType`, `AdSpaceSpecification` (all 200). Key Objects table + `MediaSubscriber__c` SOQL rewritten. *(Apex class names `MediaAdSales*Ptc`/`MediaAdSalesAppHandler` are legitimate and untouched.)*
- **TPM** — `Promotion_KPI__c`, `Promotion_KPI_Value__c`, `Tactic_KPI__c`, `Fund_Item__c`, `Condition__c`, `Claim__c`, `Batch_Run__c`, `Batch_Run_Step__c` all returned 404 (fabricated). Corrected: KPIs = `cgcloud__KPI_Definition__c` / `cgcloud__KPI_Set__c` / `cgcloud__Promotion_Template_KPI_Definition__c`; batch = `cgcloud__Batch_Run_Status__c` (+ `_Detail__c`); funds = `cgcloud__Fund_Transaction__c` / `cgcloud__Tactic_Fund__c`. Data-model diagram + 3 SOQL queries rewritten with verified fields (`Batch_State__c`, `Batch_Type__c`, `Payment_Status__c`, `Payment_Amount__c`, `Date_From__c`/`Date_Thru__c`, `Anchor_Account__c`).
- **Health Cloud** — `AssessmentResponse` is not a queryable parent; corrected to `Assessment` (subject field `AccountId`, status `AssessmentStatus`) with child `AssessmentQuestionResponse`. Bare `CarePlanProblem`/`CarePlanGoal` don't exist as standard objects → managed-pkg `HealthCloudGA__CarePlanProblem__c`/`__CarePlanGoal__c`. `CareGap`→ org analog `CareBarrier`. SOQL + known-patterns FLS prose fixed. `Claim`/`InsuranceCoverage`/`AuthorizationRequest` kept and tagged *(license-gated)*.
- **FSC** — `AccountContactRelationship` → standard `AccountContactRelation` (no "-ship"). Documented the two editions: `FinServ__` managed package (`FinServ__FinancialAccount__c` etc.) vs. FSC-on-core standard objects.
- **Comms** — `vlocity_cmt__Product2__c` is fabricated → EPC uses standard **`Product2`** plus `vlocity_cmt__*` helper objects (`ProductChildItem`, `ProductRelationship`). Fixed Key Objects + 4 code/SOQL examples in `performance-patterns.md`.
- **Loyalty** — `TransactionJournalEntry` doesn't exist (only `TransactionJournal`); `LoyaltyPromotion` → `LoyaltyProgramMbrPromotion` / `LoyaltyPgmPartnerPromotion`. Added verified `LoyaltyLedger` / `LoyaltyProgramPartnerLedger`. 11/13 originals confirmed.
- **Life Sciences** — `CareProgram*` family fully confirmed. `VisitedPlace` → org analog `VisitedParty`. `ResearchStudy`/`ResearchStudySubject`/`ResearchSite`/`Briefing` kept and tagged *(license-gated)* (real objects, Clinical feature license required).
- **Net Zero** — 6 fabricated objects (`PrcsdCrbnFtprntData`, `SustainabilityGoal`, `WaterFootprint`, `EmissionFactor`/`EmissionFactorItem`, `EnergyUse`/`EnergyUseItem`, `SustainabilityIndicator`) replaced with the real terse-API schema: `StnryAssetCrbnFtprnt`, `StnryAssetEnrgyUse`, `StnryAssetWaterFtprnt`, `Scope3CrbnFtprnt`, `EmssnReductionTarget`/`AnnualEmssnRdctnTarget`, `ProductEmissionsFactor`, `CrbnCreditProject`/`CrbnCreditAlloc`, `SustainabilityScorecard`. Both SOQL queries rewritten with `describe`-verified fields (`Co2EmissionsInKg`, `FootprintStage`, `BaseYearEmissions`).
- **CPQ (SBQQ)** — `SBQQ__PricingRule__c` → real object **`SBQQ__PriceRule__c`**; Subscription term dates `SBQQ__TermStartDate__c`/`TermEndDate__c` → `SBQQ__StartDate__c`/`SBQQ__EndDate__c`. Settings caveat added: legacy `SBQQ__QuoteSettings__c`/`LineEditorSettings__c`/`Plugins__c` not present in the verified org — consolidated to `SBQQ__GeneralSettings__c` (varies by package version). All other SBQQ objects + ~25 key fields verified present; 3 SOQL queries corrected.
- **Nonprofit (NPSP)** — Program Management objects `Program__c`/`ProgramCohort__c`/`Participant__c`/`Outcome__c` don't exist as bare names → **`pmdm__` namespace** (`pmdm__Program__c`, `pmdm__ProgramCohort__c`, `pmdm__ProgramEngagement__c`, `pmdm__ServiceParticipant__c`, `pmdm__Service__c`/`ServiceDelivery__c`/`ServiceSchedule__c`/`ServiceSession__c`). Added the **`outfunds__` Grantmaking** family (`Funding_Program__c`, `Funding_Request__c`, `Disbursement__c`, `Requirement__c`, `Review__c`, `outfundsnpspext__GAU_Expenditure__c`). Fixed SOQL `npsp__Next_Payment_Date__c` → `npe03__Next_Payment_Date__c` and recurring-donation setting `npsp__RecurringDonations2__c` → `npe03__Recurring_Donations_Settings__c`. NPSP fundraising objects (`npsp__`/`npe03__`/`npe4__`/`npe5__`) all confirmed.

### Added
- **TPM Retail Execution objects** (verified against a Retail Consumer Goods org): new subsection documenting `cgcloud__Visit_Job__c`/`Visit_Template__c`, `cgcloud__Asset_Audit__c`, `cgcloud__Order__c`/`Order_Item__c`, `cgcloud__Product_Assortment_*`, and standard `RetailStore`/`InStoreLocation`/`RetailStoreKpi`/`RetailVisitKpi`/`ProductCategory`. The org also re-confirmed all 8 TPM promotion/fund/KPI/batch objects (cross-org consistency).

### Changed
- **E&U Cloud** — carried the Comms verification finding (E&U shares the EPC catalog): `vlocity_cmt__Product2__c` → standard `Product2` (the managed-pkg object does not exist). Remaining E&U-specific objects (`ServicePoint__c`, `Premise__c`, `EnergyUsage__c`) tagged ⚠️ unverified (no E&U org in the sweep).
- **Manufacturing** — `ManufacturingProgram__c` / `ManufacturingProgramForecast__c` tagged ⚠️ unverified (no Manufacturing org; recent Manufacturing Cloud may use standard program-based-business objects — confirm via `describe`).
- **RLM** object tag updated — `PriceAdjustmentSchedule` / `PriceAdjustmentTier` confirmed standard (200). `SalesTransaction`/`SalesTransactionItem`/`UsageGrant`/`ConstraintEngineNodeStatus__c` still unverified: the two revenue-flavored orgs available (Comms CPQ + a Revenue demo) returned 404 — neither is RLM-SalesTransaction–enabled. Remains ⚠️ pending a true RLM org.
- "License-gated" convention introduced: real Salesforce objects absent from a given demo org (license/feature-gated) are kept and explicitly tagged, distinct from fabricated names which are removed.

### Verification method
- **SOAP-enabled orgs:** SOAP `login()` → REST `describe`/global `/sobjects/`. Covered Health Cloud, Life Sciences, FSC, TPM, Media (fullcopy), Comms CPQ, a Revenue demo org.
- **SOAP-disabled orgs:** `sf org login web` (browser auth) → `sf data query` against `EntityDefinition`/`FieldDefinition`/Tooling API. Covered Net Zero, Retail Consumer Goods, Salesforce CPQ, NPSP.
- **Still pending:** RLM transaction/usage objects (no RLM-enabled org available — both revenue-flavored orgs probed returned 404).

### Audit
- Two rounds of parallel read-only audits (3 agents each) across the whole filebase confirmed: version consistent in all 3 locations; 20 verticals / 11 capabilities structurally complete; no broken sibling-file references; release placeholders intact (no hardcoded `core-26X`); MCP tool routing correct (deep-research codesearch / git-soma; git-emu not a read path); no dead repos; all GUS `Product_Tag` queries use `LIKE`; no stale cruft/personal paths. Findings caught + fixed: HC `AssessmentResponse` FLS prose, Comms `vlocity_cmt__Product2__c` batch examples, Net Zero known-patterns prose, plus the E&U `Product2` and Manufacturing program-object tags above.

## v2.3.0 — 2026-06-15

**Durability + enrichment pass.** Release numbers made dynamic, GUS bug-staleness handling added, TPM off-core enriched from source, 8 thin verticals given diagnostic-map pattern files, Splunk reference expanded from authoritative sources. All facts live-probed (auth-gated MCP).

### Added
- **Dynamic release resolution** (codesearch.md → *Release Resolution*): all `core-{N}-public` / `p4/{N}-patch` / `release-{N}` refs replaced with `{CURRENT_GA}` / `{IN_DEV}` / `{PREV}` placeholders across ~60 sites in 24 files. Resolver maps today's date to a Slack-verified tri-annual release table (Winter/Spring/Summer, +2 each) through ~2028, with an optional gitcore probe near GA boundaries. Orchestrator resolves once per session. **Fixes a latent error:** the old table mislabeled 262 as Spring '26 — it is Summer '26 (GA 2026-06-15).
- **GUS bug-staleness rule** (gus.md + orchestrator Phase 4/6): related-bug queries now pull `Found_in_Build__r.Name` / `Scheduled_Build__r.Name`; bugs fixed in ≤ GA−2 are flagged "likely already resolved in this org — verify build" instead of presented as the cause.
- **`known-patterns.md` for 8 thin verticals** (Manufacturing, Automotive, Public Sector, Loyalty, Education, Nonprofit, Net Zero, Life Sciences) — symptom → verified subsystem/class → how-to-confirm → GUS search. Diagnostic maps, **not** static bug inventories (no release-pegged W-number lists to maintain). Every core subsystem path list-dir-verified on the current GA branch.
- **TPM off-core enrichment** from `rcgps-tpm-service` source: real REST endpoints, the `SYSTEM_PROCESS_ID` worker/web split, `calculationType` dispatch values, the `x-is-tpm` licensing gate, and log-code families (`SVC*`, `TPMWRK*`, `TPMCALCWRK*`, `CALCPLAN*`, `GETFEATURE*`).
- **Splunk reference expanded** (splunk.md) from authoritative sources: Log Governance Service source-of-truth URL (~4,500 codes), gack cluster `logRecordType(G,gslog,gglog,maerr)`, `mq*` async family (trace Queueable/Batch/@future), `ma*` metadata-deploy family, request-summary single-char codes, extra `ax*` codes, and the IN Industries Splunk cheat-sheet link.

### Fixed
- **GUS product tags** — 6 SKILL escalation tags returned **0 bugs** (silent-fail): `Manufacturing Cloud`, `Automotive Cloud`, `Public Sector Solutions`, `Education Cloud`, `Nonprofit Cloud`, `Life Sciences Cloud` → corrected to the real (often area-specific) tags. **All product-tag queries repo-wide switched from `=` to `LIKE '%...%'`** so they survive tag renames.
- **Splunk per-vertical tables** — 7 core-only verticals led with `r1log` (managed-package-only instrumentation that doesn't fire on core-native features) → reordered to lead with `gslog`, `r1log` qualified as conditional. Public Sector kept `r1log` primary (real `vlocity_ps` package).
- **Slack** — re-probed the 6 `⚠️ verify` channels: only `#ad_sales_management_tech_group` (C0349HJ7M0A) is real (promoted); the other 5 don't exist (removed). Added `#industries-netzerocloud-all` (C027WNNCTJL).

## v2.2.0 — 2026-06-15

**Full verification sweep + TPM off-core enrichment.** Every factual reference across all 20 verticals was live-probed against MCP sources; only confirmed facts ship. ~335 references checked.

### Verified (live, 2026-06-15)
- **core/ monorepo paths:** 96/99 confirmed via codesearch `list_directory` on `core-262-public`.
- **GUS W-numbers:** all 110 cited W-numbers exist (zero fabrications); subjects cross-checked against claims.
- **Confluence:** 45/46 pages resolve to correct space + pageId.
- **Slack:** every claimed `C0…` channel ID matched its name.

### Fixed (verification failures — corrected or removed)
- **BRE paths:** `core/industries-bre-impl` / `-bre-api` don't exist → corrected to `industries-bre-near-core-impl`, `-near-core-api`, `-engine-runtime` (RLM).
- **Manufacturing rebate:** `core/industries-rebate-impl` (dead) → `core/industries-mfg-rebates-impl`.
- **Media:** removed wrong-domain `setup_thb/media` path (that dir is contact-center voice setup; Media Ad Sales is `industries-media-revenue` + PTC `sfi_media_7`).
- **Action Plans:** stale `core-260-public` refs → `core-262-public` (module exists in both).
- **GUS citations:** removed mis-cited W-22450490 (comms) and W-12428921 (omnistudio); reworded W-19402617 to its real subject; retagged W-11941259 as case-derived.
- **Confluence:** AITB1 slug pointed at wrong pageId → corrected to `CSGPAK/489918917`; unverified net-zero "Onboarding" → real `IN/842107431 Net Zero Cloud Troubleshooting Help`.
- **Slack:** replaced literal `#C0…`-as-name placeholders in CPQ files; fixed name-drift (`#sc-code-reviews`→`#nzc-code-reviews`, `#sc-tf-fix`→`#nzc-tf-fix`, `#support-industries-fsc`→`#support-industry-fsc-hc`, `#support-health-cloud`→`#industries-healthcare`); validated IDs added to `slack.md` table.
- Unconfirmable channels (e.g. `#ask-sfdo-tech-expert`, `#automotive-cloud-experts`, `#sustcloud-engineering-only`) tagged `⚠️ verify` rather than asserted.

### Added
- **Provenance principle** (swarm-helper Phase 3): verified facts cited directly; case-derived patterns treated as leads requiring Phase 4/5 confirmation; object/field names tagged "verify in target org".
- **TPM off-core architecture** (validated): documented the `packages/tpm/` Node.js processing monorepo (rcgps-calcengine, rcgps-accplnprm, rcgps-kpi, rcgps-tpm-service, etc.) on `industries-rcg/rcgps-retail-tpm` release-262 — TPM's real calc/batch tier, distinct from the Apex managed package. Corrected the code-investigation path (was assuming `classes/*.cls`).

### Removed
- `Archive/` (original per-vertical submissions), `RCG TPM SKILL.md` (703 KB raw dev guide), `SampleTrainingData/` — source material for building `.claude/`, not shipped artifacts. The shipped deliverable is `.claude/` + `CHANGELOG.md`.

### Code-name verification pass (RLM + Media, codesearch-probed)
- **RLM SKILL — fabricated class names replaced with probe-verified ones.** The PTC list, Key Classes table, and Code Investigation Paths cited classes that returned **0 results** in core-262 (`AssetLifecycleServicePtc`, `TransactionServicePtc`, `ConfiguratorServicePtc`, `AssetLifecycleManager`, `AmendmentService`, `RenewalService`, `CancelService`, `SalesTransactionService`, `TransactionRollbackService`, `ConstraintEngine`, `CMLRuleEvaluator`, `ProductConfigurationService`, `PriceCalculationService`, `PriceAdjustmentEngine`, `CPIUpliftCalculator`, `QuoteLineItemService`, `OrderCaptureService`, `CSVImportService`, `ConstraintModelLanguageParser`, `ConstraintNodeEvaluator`). Replaced with the real surface (probed): `SwapService`/`PearCancelService` (`revenue-amend-renew-api`), `CalmCancelService`/`PhoenixCalmCancelService` (`asset-management-impl`), `AssetPriceRetainService`/`AssetPriceMaintenanceService`/`AssetToOrderSchedulerService`/`CpqAssetToBasketService`/`CMEAssetizationService`/`CpqTransactionContextService` (`industries-cpq-impl`), `BSAmendmentService`/`BSGRenewalService` (`billing-services`), `PlaceSalesTransactionService`/`PreprocessSalesTransactionService`/`ReadSalesTransactionService`/`RevenueCloneSalesTransactionService` (sales-transaction modules), `PricingService`/`CartPricingService`/`CpqPricingService`/`NGPPricingService` (pricing modules), `ConstraintEngineNodeStatus`/`ConstraintEngineConstants`/`ConstraintEngineOverallOutput` + `CmlParseResult` (constraint-studio). Added the real core modules to the path list.
- **RLM `via_rm` relabeled** as legacy Vlocity Revenue Management (Story/scoring engine), NOT modern RLM — investigation routed to core paths.
- **RLM + Media objects tagged ⚠️ pending org sweep** — `SalesTransaction`/`UsageGrant`/`PriceAdjustmentTier` etc. and `MediaAdSales*` could not be confirmed via codesearch (standard objects); Media PTC actually references `AdQuote`/`AdQuoteLine`/`AdOrderItem`. Media PTC class names (`MediaAdSalesPtc`/`MediaAdSalesRadioPtc`/`MediaAdSalesTargetingPtc`) re-probed and confirmed correct.

### Pre-distribution deep audit (post-enrichment polish)
- **Tool-routing corrected (probe-verified):** removed all `git-emu` read routing (SSO-blocked) — sf-industries/salesforce-internal repos now route to deep-research codesearch (confirmed indexed: via_platform, via_ins, via_telco, via_media, via_docgen, via_cpq, via_xom); `sf-industries-ls/lifesciences` flagged ⚠️ not-reachable (codesearch returns 0 for that org). Fixed `mcp-adaptor`-on-`via_platform` examples (returns 0) → deep-research in codesearch.md, omnistudio/reference.md, health-cloud, docgen.
- **OmniProcess IP discriminator fixed (verified on live OmniDemo + CMTLatest orgs):** Standard Runtime uses `OmniProcessType`/`IsIntegrationProcedure`; managed package uses `vlocity_cmt__IsProcedure__c`/`vlocity_cmt__OmniProcessType__c`. `Type`/`SubType` are business labels, NOT the discriminator (metadata-analysis.md had `Type = 'Integration Procedure'` — corrected; bogus `OmniIntegrationProcedure` metadata type removed).
- **OrderDecompService path fixed (codesearch-verified):** the PTC class is replicated per-namespace dir (`vlocity_cmt/`, `common/`, `omnistudio/`, …); there is NO `vlocity_ins_fsc` copy. Comms now points at the `vlocity_cmt` copy (old "in vlocity_ins_fsc, not vlocity_cmt" claim was backwards).
- **CLAUDE.md docs synced to disk:** File Structure tree expanded 13→20 verticals + all `known-patterns.md`; capability diagrams now list all 11 (added monitoring + metadata-analysis); Codebase Access table re-routed off git-emu.
- **CPQ** investigation paths: removed dead `Steelbrick/CPQ` / `CPQ-REST` repos → core `core/qtc/` SBQQ search. **Phase 3** OmniStudio reference files (reference/docgen/vbt) + **Output Format** 20-vertical list updated. 3 orphaned `known-patterns.md` (loyalty, net-zero, public-sector) linked from their SKILLs. Stale `mcp-adaptor auth --provider gus` → `/gus`.

---

## v2.1.0 — 2026-06-15

**Consolidation + live validation pass.** All repo paths and Confluence playbooks validated against live MCP sources; structure standardized; tool surface pruned.

### Fixed (repo paths — validated via live codesearch/git probes 2026-06-15)
- **CPQ:** removed non-existent `Steelbrick/CPQ` + `CPQ-REST` repos; SBQQ core-side source pointed at `core/qtc/` in the core monorepo (no standalone package repo exists)
- **Health Cloud / FSC:** removed non-existent `git.soma/industries/healthcare` + `industries/wealth1` + `industries/core` + `industries/build`; pointed at validated core monorepo paths (`core/industries-healthcare-impl/`, `core/ui-fsc-components/`) — these clouds migrated into core
- **TPM:** corrected primary repo `industries-rcg/rcg-retail-tpm` → `industries-rcg/rcgps-retail-tpm`; removed non-existent `rcg-retail-se` + `RCGSF_SF_Mobility_Sync`
- **FSC playbook:** corrected Confluence pageId `187545535` → `187545546`
- Updated `CLAUDE.md` Codebase Access table + Phase 5 to match

### Changed
- **codesearch.md rewritten:** documented the *actual* tool→index mapping — `deep-research_codesearch` is primary (covers `via_platform` + core monorepo); `mcp-adaptor` indexes the `vlocity-prod-pkg-source` mirror, not `via_platform`. Added `repo:` substring-match caveat.
- **All 20 vertical `SKILL.md` now have YAML frontmatter** (`name` + `description`); normalized lead section to `## Product Areas` (except intentional domain-specific first steps)
- **FSC dedupe:** removed ~900 lines of Action Plan content duplicated between `known-patterns.md` and `action-plans-patterns.md`; `known-patterns.md` now points to the authoritative file and keeps only FSC-wide patterns (Referrals, RSR, Financial Accounts, Timeline, Households, Goals)
- **CPQ case-volume stats** date-stamped ("snapshot ~May 2026") so staleness is visible
- Removed personal local paths (`/Users/...`) from comms-cloud pattern files; replaced with repo references

### Removed
- 2 duplicate MCP tool permissions (`dxmcp-gus`, `plugin_codesearch_codesearch`) + 1 unreferenced (`deep-research_search__doc_parse`); settings allow-list 56 → 53
- Codesearch "Tool 3 fallback" section (consolidated to single-path)

---

## v2.0.2 — 2026-06-10

### Changed
- **Phase 0 rewritten:** Clean per-tool status list (✓/⚠️) derived from system context — zero probe calls, instant startup
- Missing tools now show actionable `/mcp` hint instead of generic "pending" label
- Fixed false-pending bug: deferred tools are available, not pending

---

## v2.0.1 — 2026-06-10

### Changed
- **Phase 0 optimized:** Reads MCP connection status from system context instead of making 9 probe calls per invocation (zero-cost startup)
- Added "Building dist" section to CLAUDE.md documenting zip contents (.claude/ only)

---

## v2.0.0 — 2026-06-10

**Verticals:** 12 → 20 | **Capabilities:** 9 → 11 | **MCP probes:** 0 → 9

### Added
- **8 new verticals:** Revenue Lifecycle Mgmt, Manufacturing, Automotive, Public Sector, Loyalty, Education, Nonprofit, Net Zero
- **MCP Health Check (Phase 0):** 9 tool probes at start of every invocation with status table
- **ProductTopic-based routing:** Primary routing signal when case taxonomy available from OrgCS
- **Monitoring capability:** PagerDuty incidents + pod health check
- **Metadata analysis capability:** Apex/Flow/OmniScript export analysis, SF CLI integration
- **Case trend analysis:** Recurring issue detection via OrgCS historical query
- **`r1log` instrumentation:** Industries-specific Splunk logRecordType added to all verticals
- **Falcon/off-platform logging:** `distapps` index for Revenue Cloud and TPM services
- **Verified Slack channels:** 50+ channels with IDs confirmed via Slack MCP search
- **Complete PTC inventory:** 102 PTC classes (54 vlocity_cmt + 48 vlocity_ins) documented
- **Confirmed repo paths:** CodeSearch-validated core monorepo paths for all verticals
- Swarm templates for all 20 verticals
- Sample SOQL queries for all 20 verticals
- Code investigation paths (MCP tool invocations) for all 20 verticals

### Changed
- All existing Silver verticals (FSC, Comms, Revenue Cloud, CPQ, TPM, E&U, Media, Life Sciences, HC) enriched to Gold standard
- Health Cloud `known-patterns.md` rewritten with 12 real troubleshooting patterns (was empty template)
- Comms Cloud `order-management-patterns.md` restructured (removed 776 lines of dev methodology)
- Revenue Cloud `dro-patterns.md` cleaned (removed 190 lines of generic boilerplate)
- Phase 4 investigation table expanded (added Monitoring, Metadata, Case Trends)
- `settings.local.json` updated with 5 new tool permissions

### Removed
- 1,799 lines of boilerplate across 16 files (Update Cadence, Prerequisites & Tool Access, Test Cases, Skill Completeness Reports, Success Metrics)
- "Not yet covered" note from routing table (all verticals now covered)

---

## v1.0.0 — 2026-06-01

**Verticals:** 12 | **Capabilities:** 9

### Initial Release
- 12 verticals: OmniStudio, Health Cloud, Insurance, DocGen, FSC, Comms Cloud, Revenue Cloud, CPQ, TPM, Life Sciences, E&U, Media
- 9 capabilities: OrgCS, Splunk, CodeSearch, GUS, Columbo, Slack, Confluence, Debug Log, HAR Analysis
- Validated OrgCS queries (case 473586604)
- Guard rails for restricted support tiers
- PII protection rules (mandatory)
- Phase-based investigation workflow (Gather → Route → Check Patterns → Investigate → Regression → Post)
