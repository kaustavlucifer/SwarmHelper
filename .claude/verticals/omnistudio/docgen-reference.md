# DocGen Troubleshooting Reference

## Two Flavors of DocGen

| | Vlocity DocGen | OmniStudio DocGen |
|---|---|---|
| Package | vlocity_cmt / vlocity_ins | OmniStudio package |
| Web Templates | Supported | Not supported |
| License check | Setup → Document Generation Settings (if present = OmniStudio DocGen licensed) |

Some orgs have both (customer migrated from Vlocity Custom DataModel to Standard DataModel).

---

## Common Errors

| Error | Likely Cause |
|---|---|
| `Workspace Not found` | DocGen setup incomplete |
| `List has no rows for assignment to sObject` | Template or DR misconfiguration |
| `Validation Error: Exception Occurred` | Setup issue |
| `Input object is not a Key value Pair` | Incorrect input structure to template |

Most issues occur when Setup is not properly done.

---

## Setup References

- Vlocity DocGen setup: internal documentation
- OmniStudio DocGen setup: https://help.salesforce.com/s/articleView?id=ind.doc_gen_select_templates_for_clm_or_foundation_docgen.htm&type=5

---

## Debugging Trick — Stubbing tokenDataMap

For Docx Document Generation, stub token data from the customer org to a Demo org without replicating all customer data:

1. Capture the `tokenDataMap` from Console Logs while generating the document in the customer org
2. In the Demo org, add a **Set Value** step in the OmniScript and paste the entire captured `DataJSON`
3. Set the `useTemplateDRExtract` flag to **false** under generation options
4. Configure the Template Transform and copy the output path to the input path

This eliminates the need to replicate custom sObjects, DataRaptors, etc. in the Demo org.

> Similar stubbing method applies to OmniScripts — stub data at initial steps to replicate issues at a later step directly.
