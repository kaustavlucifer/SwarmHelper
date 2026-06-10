# FSC Debugger

**Trigger:** Financial Services Cloud, FSC, financial accounts, account hierarchy, referrals, relationship maps, action plans, `vlocity_ins_fsc` namespace, rollup summary rules, document checklist items, goals, life events, interaction summaries.

---

## Repository Architecture

### GitHub / git.soma Repos

| Repository | Path | Content |
|---|---|---|
| `wealth1` | `git.soma.salesforce.com/industries/wealth1` | FSC managed package (primary) |
| `core` | `git.soma.salesforce.com/industries/core` | Shared Industries core |
| `build` | `git.soma.salesforce.com/industries/build` | Build infrastructure |
| `via_ins_fsc` | `github.com/sf-industries/via_ins_fsc` | Insurance Industries Extension for FSC |
| `via_platform` | `github.com/sf-industries/via_platform` | OmniStudio Apex (vlocity_ins_fsc namespace) |
| `fsc-next-gen-apps` | `github.com/salesforce-internal/fsc-next-gen-apps` | FSC Next-Gen Apps |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/ui-fsc-components/omnistudio/               ← FSC OmniStudio components
  core/ui-fsc-components/java/                     ← FSC Java
  core/ui-fsc-components/modules/                  ← FSC LWC (analytics, KYC, wealth)
  core/ui-fsc-api/                                 ← FSC API layer
  core/industries-interaction-ptc/apex/vlocity_ins_fsc/   ← PTC layer
  core/industries-interaction-ptc/apex/vlocity_fsc_gs0/   ← PTC layer (GS0)
```

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| FSC Main Page | https://confluence.internal.salesforce.com/spaces/IN/pages/187545535/Financial+Services+Cloud+-+FSC |

---

## FSC-Specific Objects

| Object | Description |
|---|---|
| `FinancialAccount` | Core financial account |
| `FinancialAccountRole` | Account roles (owner, beneficiary) |
| `FinancialHolding` | Holdings within accounts |
| `ActionPlan` / `ActionPlanItem` | FSC action plans |
| `Referral` | Referral tracking |
| `AccountContactRelationship` | Relationship maps |
| `RollupSummaryRule__c` | Rollup summary rules |
| `DocumentChecklistItem` | Document checklists |

---

## Common Issues

| Symptom | Check |
|---|---|
| Relationship map not loading | FlexCard datasource, FinancialAccount sharing rules |
| Action plan items not created | ActionPlan triggers, IP logic, field FLS |
| FSC OmniScript errors | `vlocity_ins_fsc` namespace, PTC layer changes |
| Financial account not visible | OWD, sharing rules, person account setup |
| Referral not visible | Referral sharing rules, routing config |
| RSR not calculating | Rollup summary rule config, stale values |
| Document checklist permissions | `DocumentChecklistItem` CRUD access, Site visibility |
| Action plan template access | Licensing (Sales Action Plans vs FSC), permission sets |
| Successor tasks not creating | Task dependency config, action plan item ordering |
| Timeline not showing | Component configuration, record-level access |
| Interaction summary not saving | Related record linking, field permissions |

---

## Escalation

- Slack: `#support-industries-fsc` or `#support-omnistudio-collaboration`
- GUS product tag: `Industries Financial Services Cloud`


---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `action-plans-patterns.md` — Action plan templates, tasks, dependencies, permissions
- `known-patterns.md` — Known issue patterns with resolution steps
