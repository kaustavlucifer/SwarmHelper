# VBT (Vlocity Build Tool) — Full Reference

Source: https://github.com/vlocityinc/vlocity_build

---

## What is VBT?

Vlocity Build Tool (VBT) is a command line Node.js tool to export and deploy Vlocity DataPacks in a source-control-friendly YAML format. Primary goal: enable CI/CD for Vlocity Metadata.

---

## Installation

**Requirements:** Node.js v18+. Check with `node -v`.

```bash
# Install or update
npm install --global vlocity

# Verify install
vlocity help
```

> Do NOT clone the repo. Install via npm only.

### Installing a Specific Older Version
```bash
npm install --global vlocity@1.9.1
```

### If Previously Cloned the Repo (unlink first)
```bash
# In the cloned vlocity_build folder:
npm unlink .
```

---

## Authentication

### Option 1 — Salesforce CLI (Recommended)
```bash
# Sandbox
sf org login web --instance-url https://test.salesforce.com --alias MyOrg

# Production
sf org login web --instance-url https://login.salesforce.com --alias MyOrg
```
Then use `-sfdx.username <username_or_alias>` in VBT commands.

### Option 2 — Username & Password (Property File)
Create `build_source.properties`:
```java
sf.username = <Salesforce Username>
sf.password = <Salesforce Password + Security Token>
sf.loginUrl = https://test.salesforce.com
# For proxy:
sf.httpProxy = http://[user:pass@]host[:port]
```
Use with: `vlocity -propertyfile build_source.properties -job job.yaml packExport`

### Option 3 — OAuth
```bash
vlocity packExport -sf.accessToken <token> -sf.instanceUrl <url> -sf.sessionId <id>
```
Also add `oauthConnection: true` to the Job File.

---

## Job File

The Job File defines the project path and settings. Minimal example (`job.yaml`):
```yaml
projectPath: ./vlocity
```

Export OmniStudio DataPacks:
```yaml
projectPath: ./vlocity
queries:
  - OmniScript
  - IntegrationProcedure
  - DataRaptor
  - FlexCard
```

---

## All Commands

### Primary Commands

| Command | Description |
|---|---|
| `packExport` | Export from Salesforce org into DataPack directory |
| `packExportSingle` | Export a single DataPack by Id |
| `packExportAllDefault` | Export ALL default DataPacks |
| `packDeploy` | Deploy all contents of DataPacks directory to org |

### Troubleshooting Commands

| Command | Description |
|---|---|
| `packContinue` | Resume a job that failed/was cancelled mid-run |
| `packRetry` | Retry all deploy errors or re-run all export queries |
| `validateLocalData` | Check for missing/duplicate Global Keys in local files |
| `cleanOrgData` | Find and fix issues in org data; adds missing Global Keys |
| `refreshProject` | Rebuild project folders and resolve missing references |
| `checkStaleObjects` | Ensure all references exist in org or locally before deploy |

### Additional Commands

| Command | Description |
|---|---|
| `packGetDiffsAndDeploy` | Deploy only files modified vs target org |
| `packGetDiffs` | List all diffs between local files and org |
| `packBuildFile` | Build DataPacks directory into a single DataPack file |
| `runJavaScript` | Run a Node.js script on each DataPack |
| `packUpdateSettings` | Refresh DataPacks Settings to current tool version |
| `runApex` | Run anonymous Apex from `-apex` path or `/apex` folder |
| `refreshVlocityBase` | Deploy and activate Base Vlocity DataPacks from package |
| `installVlocityInitial` | Deploy Base + Configuration DataPacks from package |

---

## Command Syntax Examples

```bash
# Export by DataPack type
vlocity -sfdx.username source_org@vlocity.com -job job.yaml packExport -key DataRaptor

# Export single item by type/name
vlocity -sfdx.username source_org@vlocity.com -job job.yaml packExportSingle -type OmniScript -id <Id>

# Deploy
vlocity -sfdx.username target_org@vlocity.com -job job.yaml packDeploy

# Retry errors
vlocity -sfdx.username target_org@vlocity.com -job job.yaml packRetry

# Validate local data, auto-fix keys
vlocity -sfdx.username source_org@vlocity.com -job job.yaml validateLocalData --fixLocalGlobalKeys

# Check diffs then deploy only changed
vlocity -sfdx.username target_org@vlocity.com -job job.yaml packGetDiffsAndDeploy

# Clean org data (adds missing Global Keys)
vlocity -sfdx.username source_org@vlocity.com -job job.yaml cleanOrgData
```

---

## Org to Org Migration — Step by Step

1. Authorize both orgs (CLI or property files)
2. Create Job File with `projectPath` and `queries`
3. Export from source:
   ```bash
   vlocity -sfdx.username source_org@vlocity.com -job EPC.yaml packExport
   vlocity -sfdx.username source_org@vlocity.com -job EPC.yaml packRetry  # if errors
   ```
4. Validate local data:
   ```bash
   vlocity -sfdx.username source_org@vlocity.com -job EPC.yaml validateLocalData
   ```
5. Deploy to target:
   ```bash
   vlocity -sfdx.username target_org@vlocity.com -job EPC.yaml packDeploy
   vlocity -sfdx.username target_org@vlocity.com -job EPC.yaml packRetry  # if errors
   ```

> Run `packRetry` until error count stops decreasing — some errors self-resolve once dependency order is satisfied.

---

## Key Job File Options

| Option | Description | Default |
|---|---|---|
| `activate` | Activate everything after import/deploy | `true` |
| `addSourceKeys` | Generate Global Keys for records missing them | `false` |
| `autoRetryErrors` | Auto-retry after deploy for reference resolution errors | `false` |
| `autoUpdateSettings` | Run `packUpdateSettings` before deploy | `true` |
| `gitCheck` | Use Git hashes to only deploy changed files | `false` |
| `ignoreAllErrors` | Ignore all errors — **not recommended** | `false` |
| `maxDepth` | Max depth of parent/child relationships to export | `-1` (all) |
| `maximumDeployCount` | Max items in a single deploy batch | `1000` |
| `oauthConnection` | Required when using OAuth authentication | `false` |
| `reactivateOmniScriptsWhenEmbeddedTemplateFound` | Re-activate ALL OmniScripts (slow) | `false` |

---

## Log Files

Three log files are generated per command run (in your working directory):

| File | Contents |
|---|---|
| `VlocityBuildLog.yaml` | Summary of what was executed |
| `VlocityBuildErrors.log` | All errors during the job |
| `vlocity-temp/logs/<date>-<time>-<command>.yaml` | Saved copy of the build log |

Run with `--verbose` to include all logging.

---

## Troubleshooting — Common Errors

### `Not Found` during Export
```
Error >> DataRaptor --- GetProducts --- Not Found
Error >> VlocityUITemplate --- ShowProducts --- Not Found
```
**Cause:** A referenced object is inactive or deleted in the source org.
- `VlocityUITemplate Not Found` during OmniScript export — often ignorable if the template lives inside a VF page
- `DataRaptor Not Found` — DataRaptor was deleted or renamed without updating the OmniScript referencing it

---

### `No Match Found` during Deploy
```
Deploy Error >> Product2/02d3feaf... --- iPhone --- No match found for
vlocity_cmt__ProductChildItem__c.vlocity_cmt__ChildProductId__c - vlocity_cmt__GlobalKey__c=db65c1c5...
```
**Cause:** A referenced record exists in source but not in target.
**Fix:** Export and deploy the missing referenced DataPack too.

---

### `SASS Compilation Error`
```
SASS Compilation Error VlocityUITemplate/cpq-total-card
Failed to compile SCSS: @import "cpq-theme-variables";
```
**Fix:** Export the missing VlocityUITemplate:
```bash
vlocity -job job.yaml packExport -key VlocityUITemplate/cpq-theme-variables
```

---

### `No Configuration Found`
```
AttributeCategory/Something -- Error -- No Configuration Found: Attribute Category Migration
```
**Fix:** Run `packUpdateSettings` or add `autoUpdateSettings: true` to job file.

---

### `Duplicate Value Found`
```
Error >> AttributeCategory/Product_Attributes --- duplicate value found: duplicates value on record with id:
```
**Fix:** Update the DisplaySequence on the conflicting Attribute Category in the **target org**. Find duplicates:
```sql
SELECT Id, Name FROM vlocity_cmt__AttributeCategory__c 
WHERE vlocity_cmt__DisplaySequence__c = <value_from_DataPack.json>
```

---

### `Multiple Imported Records will incorrectly create the same Salesforce Record`
**Cause:** Duplicate records in source org (same Matching Key).
**Fix:** Remove duplicates from source org and re-export.

---

### `Some records were not processed`
**Cause:** Configuration mismatch between source and target org.
**Fix:** Run `packUpdateSettings` in both orgs.

---

### Duplicate Global Keys
```
Deploy Error >> Product2/02d3feaf... --- iPhone --- Duplicate Results found for Product2
WHERE vlocity_cmt__GlobalKey__c=02d3feaf... - Related Ids: 01t1I..., 01t1I...
```
**Fix:** Clean up duplicates in the org, then add missing Global Keys:
```bash
vlocity -job job.yaml runJavaScript -js cleanData.js
# or
vlocity -job job.yaml cleanOrgData
```

---

## Global Keys — Important Notes

- Most Vlocity objects use **Global Key** as the unique identifier for upsert
- If 1 match found → record is updated
- If 2+ matches found → deploy error (duplicates must be cleaned)
- Missing Global Keys can be auto-generated: `validateLocalData --fixLocalGlobalKeys` (local) or `cleanOrgData` (org)
- For cross-org migration where both orgs have missing keys, a manual matching strategy may be needed

---

## Auto Compilation of LWC OmniScript and Cards

VBT automatically compiles OmniScript and FlexCard LWC during deploy. For local compilation support (managed package only), set up your SFDX CLI path in the job file. See the GitHub README for details.

---

## New Sandbox Orgs

After installing the Vlocity Managed Package in a new sandbox, run:
```bash
vlocity -sfdx.username sandbox_org@vlocity.com -job job.yaml refreshVlocityBase
```
This deploys and activates the Base Vlocity DataPacks required before deploying custom metadata.
