# Salesforce CPQ (SBQQ) Debugger

**Trigger:** Salesforce CPQ, SBQQ, QCP (Quote Calculator Plugin), quote line editor (QLE), amendments, contracts, pricing rules, CPQ API, subscription pricing, renewals, order contracting, MDQ, document generation (CPQ flavor).

---

## Key Differences: Salesforce CPQ vs Revenue Cloud CPQ

| | Salesforce CPQ (SBQQ) | Revenue Cloud (Industries CPQ) |
|---|---|---|
| Namespace | `SBQQ__` | `vlocity_cmt__` or standard objects |
| Package | Managed package (Steelbrick) | Industries package or core |
| Objects | `SBQQ__Quote__c`, `SBQQ__QuoteLine__c` | `Quote`, `QuoteLineItem` or standard |
| Pricing | QCP (JavaScript), Pricing Rules | Calculation Procedures, Decision Tables |

---

## Repository Architecture

### GitHub Repos

| Repository | Path | Content |
|---|---|---|
| `via_cpq` | `sf-industries/via_cpq` | Industries CPQ managed package |
| `CPQ` | `git.soma.salesforce.com/Steelbrick/CPQ` | Salesforce CPQ (SBQQ) managed package — Apex, VF, JS, CSS |
| `CPQ-REST` | `git.soma.salesforce.com/Steelbrick/CPQ-REST` | CPQ JS services (calculator, amend/renew) on Heroku |
| `org62-cpq-aws-*` | `SF-BT/org62-cpq-aws-*` | CPQ pricing, approval rules, config loader, evaluation engine |

### Core Monorepo Paths

```
gitcore.soma.salesforce.com/core-2206/core-262-public:
  core/industries-cpq/                    ← Industries CPQ core (rule engine)
  core/industries-cpq-impl/              ← Industries CPQ implementation
  core/industries-cpq-pricing-common-api/ ← CPQ pricing common API
  core/cpq/                              ← Core CPQ functionality
  core/cpq-config-rules/                 ← CPQ configuration rules
```

---

## Engineering Playbooks

| Resource | URL |
|---|---|
| CPQ Onboarding (Asset Lifecycle) — includes full investigation workflow | https://confluence.internal.salesforce.com/spaces/QUOTETOC/pages/1187813893/Salesforce+CPQ+Onboarding+Asset+Lifecycle |
| How to Troubleshoot CPQ Issues (Industries CPQ) | https://confluence.internal.salesforce.com/spaces/CSGPAK/pages/432075488/How+to+Troubleshoot+CPQ+Issues |
| Industries CPQ: Project CPQNext | https://confluence.internal.salesforce.com/spaces/VLCOM/pages/449598012/Industries+CPQ+Project+CPQNext |
| CPQ (IN Space) | https://confluence.internal.salesforce.com/spaces/IN/pages/591534674/CPQ |
| Rev Cloud Development Briki (internal) | https://sites.google.com/salesforce.com/rev-cloud-development/ |

---

## Common Issues

| Symptom | Check |
|---|---|
| QLE not loading | QCP errors, JavaScript console, Quote status |
| Apex heap size error | Large quote (1000+ lines), QCP complexity |
| Too many SOQL queries | Triggers stacking on quote save, flow loops |
| Amendment failing | Contract status, subscription fields, amendment start date |
| Contract not creating | Order → Contract flow, required fields, validation rules |
| Pricing rule not firing | Rule order, evaluation event, conditions |
| Renewal quote issue | Renewal model, subscription term, opportunity stage |
| Order contracting error | Contract field mapping, order status transition |
| UNABLE_TO_LOCK_ROW | Concurrent quote saves, background jobs on same records |
| Calculation service auth | Service URL, certificate, endpoint config |
| MDQ (Multi-Dimensional Quoting) | Segment config, dimension mapping |
| CPQ document generation | Template merge fields, PDF generation |

---

## Developer-Tier Issues

| Symptom | Check |
|---|---|
| QCP custom code failing | JavaScript syntax, async/await patterns, API limits |
| CPQ API integration | REST API endpoints, bulk operations, field mapping |
| Managed trigger failures | Trigger order, recursion guards, namespace conflicts |
| Custom Flow + CPQ | Flow trigger order, CPQ calculator not re-firing |
| Queueable job failures | Chaining limits, async context restrictions |
| Middleware query failures | Connected App config, API versioning |

---

## Debugging Approach

1. **Browser console** (F12) — QLE/QCP errors appear here
2. **Debug logs** — Apex execution for quote save/calculate
3. **SBQQ Settings** — Custom Setting `SBQQ__QuoteSetting__c` controls behavior
4. **Calculation order** — Price Rules → QCP → Calculator → Plugins
5. **Package version** — Check installed version vs known bug fixes

---

## Escalation

- Slack: `#support-rev-dev-amer`
- GUS product tag: `Revenue Cloud` (CPQ bugs filed here)


---

## Detailed Pattern Files

The following files contain comprehensive troubleshooting patterns with specific error messages, root causes, resolution steps, Splunk queries, and GUS references:

- `developer-patterns.md` — Developer-tier issues: QCP, API integration, Apex governor limits
- `known-patterns.md` — Known issue patterns with resolution steps
