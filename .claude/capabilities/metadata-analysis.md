# Capability: Metadata & Component Analysis

## Purpose

Analyze customer Salesforce metadata (Apex, Flows, OmniScripts, LWC, custom objects) when engineers provide exported files or retrieve them via SF CLI. This enables troubleshooting component-level issues without needing direct org access.

---

## Input Methods

### 1. Files dropped in `data/` directory

Engineers can drop exported metadata files:
- `.cls` / `.cls-meta.xml` — Apex classes/triggers
- `.flow-meta.xml` — Flow definitions
- `.json` — OmniScript/IP/DataRaptor/FlexCard exports (DataPack format)
- `.object-meta.xml` — Custom object definitions
- `.permissionset-meta.xml` — Permission sets
- `.profile-meta.xml` — Profiles
- `.xml` — Any SFDX metadata format

### 2. SF CLI retrieval (if org connected)

```bash
# Retrieve specific component
sf project retrieve start --metadata ApexClass:MyClassName --target-org <alias>

# Retrieve OmniStudio components
sf project retrieve start --metadata OmniProcess:MyOmniScript --target-org <alias>

# Retrieve Flow
sf project retrieve start --metadata Flow:MyFlowName --target-org <alias>

# Retrieve custom object with fields
sf project retrieve start --metadata CustomObject:MyObject__c --target-org <alias>
```

### 3. SOQL export of OmniStudio components

For OmniStudio components that don't deploy via standard metadata:
```soql
-- OmniScript definition (Standard Runtime)
SELECT Id, Name, Type, SubType, Language, IsActive, VersionNumber, PropertySetConfig
FROM OmniProcess
WHERE Name = '<OS_NAME>' AND IsActive = true

-- DataRaptor/DataTransform
SELECT Id, Name, Type, IsActive, InputType, OutputType
FROM OmniDataTransform
WHERE Name = '<DR_NAME>'

-- Integration Procedure (Standard Runtime)
SELECT Id, Name, Type, SubType, IsActive, PropertySetConfig
FROM OmniProcess
WHERE Type = 'Integration Procedure' AND Name = '<IP_NAME>'
```

---

## Analysis Patterns

### Apex Class Analysis
When `.cls` file is provided:
1. Check for governor limit risks (SOQL in loops, DML in loops)
2. Identify callout patterns (DML before callout violations)
3. Check FLS/CRUD enforcement (SYSTEM_MODE vs USER_MODE)
4. Look for known anti-patterns per vertical
5. Trace the call chain (which service/trigger invokes this)

### Flow XML Analysis
When `.flow-meta.xml` is provided:
1. Parse flow type (Screen Flow, Record-Triggered, Autolaunched)
2. Identify decision elements and their criteria
3. Check for bulkification issues (loops with DML/SOQL inside)
4. Verify fault paths exist for all connectors
5. Check for hardcoded IDs or environment-specific values

### OmniScript/IP JSON Analysis
When DataPack JSON export is provided:
1. Parse the step structure (elements, actions, conditions)
2. Identify DataRaptor references and their types (Extract/Transform/Load)
3. Check for HTTP Action callouts (Named Credential, endpoint)
4. Look for Set Values with hardcoded data
5. Identify conditional visibility rules and their logic
6. Check Remote Action step configurations

### Custom Object Analysis
When `.object-meta.xml` is provided:
1. Review field definitions (types, required, defaults)
2. Check validation rules for conflicts with automation
3. Identify sharing model (OWD, sharing rules)
4. Review record types and their field assignments
5. Check for lookup filters that might block operations

---

## Tools

| Tool | Use |
|---|---|
| `Bash(sf project *)` | SF CLI metadata retrieve/deploy operations (already permitted) |
| `Read` | Parse local metadata files dropped in `data/` |
| Local file analysis | Claude can directly parse XML/JSON metadata structures |
| `afv-metadata` plugin | Agentforce metadata structure analysis |
| `sfdx-deploy-doctor` plugin | Deployment failure diagnosis |

---

## When to Use

- Engineer says "I have the customer's Apex class / Flow / OmniScript export"
- Engineer drops files in `data/` directory
- Debug log shows a custom class/flow name that needs source review
- Deployment failure (use `sfdx-deploy-doctor` skill)
- Need to understand component configuration without org access

---

## Integration with Verticals

| Vertical | Common Metadata to Analyze |
|---|---|
| OmniStudio | OmniScript JSON, IP JSON, DataRaptor JSON, FlexCard JSON |
| Insurance | Service class Apex, Trigger factory config, State model JSON |
| Health Cloud | Care Plan IPs, Assessment OmniScripts, FHIR mapping |
| Revenue Cloud | BRE Expression Sets, CML rules, Approval Flows |
| CPQ | QCP JavaScript, Pricing Rules config, Quote template |
| FSC | Action Plan templates, Rollup rules, Relationship config |
| Comms Cloud | EPC product catalog, Orchestration plan config |

---

## Key Salesforce Metadata Types Reference

| Category | Metadata Types |
|---|---|
| Logic | `ApexClass`, `ApexTrigger`, `Flow`, `ApexPage` |
| OmniStudio | `OmniProcess`, `OmniDataTransform`, `OmniUiCard`, `OmniIntegrationProcedure` |
| Data Model | `CustomObject`, `CustomField`, `ValidationRule`, `RecordType` |
| Security | `PermissionSet`, `Profile`, `SharingRule`, `CustomPermission` |
| Configuration | `CustomMetadata`, `CustomSetting`, `NamedCredential`, `ExternalService` |
| UI | `LightningComponentBundle`, `FlexiPage`, `CustomTab` |
