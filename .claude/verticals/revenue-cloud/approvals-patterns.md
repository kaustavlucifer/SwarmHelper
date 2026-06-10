## A. Triage & Classification

### Quick Diagnostic Questions

Ask the customer **in this order** to narrow to a pattern:

1. **What is the exact error message?** (copy-paste from UI or debug log)
2. **When does the error occur?** (flow save/deploy, Submit for Approval, Preview, during approval, email)
3. **Which object is being approved?** (Quote, Contract, Order, Custom)
4. **Is it Production or Sandbox?** (preview issues are more common in sandbox)
5. **Is the flow type "Autolaunched Approval Orchestration" or "Record-Triggered"?**
6. **Is the approver a System Admin or a non-admin user?**
7. **Does the org have Revenue Cloud Advanced license provisioned?** (check BT)

---

### Triage Decision Tree

```
Customer reports Advanced Approvals issue
│
├── ERROR AT FLOW SAVE / DEPLOY?
│   ├── "doesn't have an active Advanced Approvals license"
│   │   └── → Pattern 1: Background Step License Error
│   └── "Autolaunched Approval Orchestration" flow type not visible
│       └── → Pattern 1: Background Step License Error
│
├── ERROR AT APPROVAL PREVIEW?
│   ├── "Something went wrong while previewing the approval workflow"
│   │   └── → Pattern 2: Approval Preview Generic Error
│   ├── "Invalid assignee [ID], the given assignee doesn't exist"
│   │   └── → Pattern 3: Preview Shows "No Queue" / Invalid Assignee
│   └── "ActionInput__RecordId" / wrong data type passing Quote ID
│       └── → Pattern 2: Approval Preview Generic Error
│
├── ERROR AT SUBMIT FOR APPROVAL?
│   ├── "No applicable approval process was found"
│   │   └── → Pattern 4: Submit for Approval – No Process Found
│   └── "RCA - Quote Create and Update" process failed / unhandled fault
│       └── → Pattern 5: Unhandled Fault During Submission (Non-Admin)
│
├── SMART APPROVALS NOT AUTO-APPROVING?
│   ├── Auto-approve criteria not firing / approval always created
│   │   └── → Pattern 6: Smart Approvals Not Auto-Approving
│   └── Error when creating ApprovalWorkItem during auto-approval
│       └── → Pattern 7: Auto-Approval Audit Trail Error
│
├── APPROVAL STUCK / STATUS NOT UPDATING?
│   ├── ApprovalWorkItem Approved but ApprovalSubmission stays "In Progress"
│   │   └── → Pattern 8: Approval Submission Stuck in Progress
│   └── Approver sees "Unhandled Fault" when clicking Approve
│       └── → Pattern 5: Unhandled Fault During Submission (Non-Admin)
│
├── EMAILS NOT SENDING?
│   ├── No email triggered on submission/approval
│   │   └── → Pattern 9: Approval Alert Emails Not Sending
│   └── Wrong merge fields / email template not matching
│       └── → Pattern 9: Approval Alert Emails Not Sending
│
├── APEX ACTION ERROR?
│   ├── "Looks like you don't have access to this action"
│   │   └── → Pattern 10: ApprovalPreviousRelatedRecordDetails Access Error
│   └── getPreviousRelaRecDetails invocable action failing
│       └── → Pattern 10: ApprovalPreviousRelatedRecordDetails Access Error
│
├── QUOTE NOT LOCKING DURING APPROVAL?
│   ├── Quote/line items editable after Submit for Approval
│   │   └── → Pattern 15: Quote Locking Not Working During Approval
│   └── Approval Status changes to "Withdrawn" unexpectedly
│       └── → Pattern 15: Quote Locking Not Working During Approval
│
├── RECALL APPROVAL ISSUES?
│   ├── "Recall Approval Submission" action not in Flow Builder list
│   │   └── → Pattern 16: Recall Approval Submission Issues
│   ├── Recall fails / comments not saved / email not sent
│   │   └── → Pattern 16: Recall Approval Submission Issues
│   └── ApprovalSubmission status not updating to "Recalled"
│       └── → Pattern 16: Recall Approval Submission Issues
│
├── DUPLICATE APPROVAL RECORDS?
│   ├── Multiple ApprovalSubmissions for one click / 50+ emails
│   │   └── → Pattern 17: Duplicate Approval Records Created
│   └── Multiple ApprovalWorkItems sent to same approver
│       └── → Pattern 17: Duplicate Approval Records Created
│
├── PLATFORM LICENSE VISIBILITY?
│   ├── Platform license users cannot see Approval Work Items
│   │   └── → Pattern 18: Platform License Users Cannot See Approval History
│   └── Approval History tab not visible despite permission sets assigned
│       └── → Pattern 18: Platform License Users Cannot See Approval History
│
└── DATA CLEANUP REQUEST?
    └── Cannot delete orphaned ApprovalWorkItems
        └── → Pattern 11: Orphaned ApprovalWorkItem Deletion
```

---

## B. Known Issue Patterns

---

### Pattern 1: Background Step License Error (Cannot Save Orchestration Flow)

**Frequency:** High (~8 cases)  
**Severity:** Sev 2–3  
**TTR Impact:** Low — typically resolved same day via BT perm enable

**Symptoms:**
- Error on saving/deploying an orchestration flow: `"You can't save this orchestration because the "Background Step" Advanced Approvals feature is used in "[step_name]" step, but this org doesn't have an active Advanced Approvals license."`
- The flow type `"Autolaunched Approval Orchestration (No Trigger)"` is not visible in Flow Builder
- Occurs in both Production and Sandbox

**Root Cause:**
⚠️ **Important nuance (confirmed by product team in #rev-nova-public — Ramya Sukhavasi):**
- If **Advanced Approvals IS enabled**: the `Standard Approvals: Background steps` (`UseBackgroundSteps`) perm is **NOT required** — Background Steps work automatically
- If **Advanced Approvals is NOT enabled**: `UseBackgroundSteps` IS required (and the error message is misleading — it says "Advanced Approvals license" but actually just needs this perm)

So the error `"doesn't have an active Advanced Approvals license"` has **two different root causes depending on the org state**:
1. Org has no Advanced Approvals enabled at all → enable Advanced Approvals in BT first
2. Org has non-AA orchestration flows using Background Steps → enable `UseBackgroundSteps` perm

**Resolution Steps:**
1. Check BT for the impacted org: is `Advanced Approvals Enabled` checked?
2. **If Advanced Approvals is NOT enabled:**
   - Verify the customer has a valid Revenue Cloud Advanced (or Revenue Cloud Growth) license
   - Enable in BT: `Advanced Approvals Enabled`, `Advanced Approvals: Org Pref: Dynamic Approval Notifications`, `Revenue Cloud Advanced Approvals`
   - After enabling, Background Steps work — `UseBackgroundSteps` not needed
3. **If Advanced Approvals IS enabled but error persists:**
   - Enable `Standard Approvals: Background steps` (`UseBackgroundSteps`) in BT as a fallback
4. Have the customer re-open Flow Builder and confirm the `Autolaunched Approval Orchestration (No Trigger)` type is now visible.
5. Have the customer save/deploy the flow again.

**Verification:**
Customer can save the orchestration flow without errors and the Background Step is retained.

**Workarounds (from #support-rev-rlm-global-swarm-help, #chat_hc_channel):**
- If `"Match Production Licenses"` was tried and failed in Sandbox, it means the Production org itself doesn't have the perm — fix Production first, then sandbox refresh/match.
- For sandboxes: enable `UseBackgroundSteps` directly on the sandbox BT instance; do not wait for license propagation.
- Related case reference: 471182376, 471687279, 472853782

**Related Documentation:**
- [Advanced Approvals Overview](https://help.salesforce.com/s/articleView?id=ind.approvals_advanced_approvals.htm&type=5)
- [Advanced Approvals Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.revenue_lifecycle_management_dev_guide.meta/revenue_lifecycle_management_dev_guide/advanced_approvals_overview.htm)

**Escalation Criteria:**
- Escalate if BT shows license is provisioned and `UseBackgroundSteps` is already enabled but error persists.
- Escalate if the customer has Revenue Cloud Advanced entitlement but BT shows no RCA features enabled at all (provisioning failure).

---

### Pattern 2: Approval Preview Generic Error ("Something Went Wrong")

**Frequency:** Very High (~20 cases)  
**Severity:** Sev 3–4  
**TTR Impact:** Medium — 1–5 days depending on root cause

**Symptoms:**
- Clicking "Preview" on the Approval Workflow component opens the modal but immediately shows: `"Something went wrong while previewing the approval workflow. Try again later, and if the issue persists, contact your Salesforce admin for help."`
- No approval details render; modal is blank with just the error
- Clicking "Preview" button → `ActionInput__RecordId` error: variable is not of the correct data type
- More common in Sandbox than Production

**Root Cause:**
Multiple root causes under this same error surface:
1. The "Select the Record to Approve" input in the flow is misconfigured — a text variable is being used for a Record ID that expects a specific SObject type.
2. The flow configuration is receiving a string/text variable instead of a Quote record variable for `ActionInput__RecordId`.
3. Debug mode with rollback enabled causes asynchronous Slack Notification steps to fail with: `"Because this orchestration is being debugged with rollback enabled and the 'Slack_Notification' step is processed asynchronously, the step must be run as a simulated step."`

**Resolution Steps:**
1. Navigate to the Approval Orchestration flow in Flow Builder.
2. Locate the "Select the Record to Approve" step input configuration.
3. Verify the variable mapped to `ActionInput__RecordId` is of the correct SObject type (e.g., Quote record variable), not a text/string variable.
4. If the customer is debugging: disable rollback in debug mode when the flow includes async steps (like Slack notifications).
5. Test by navigating to a Quote record → Approvals tab → Preview. Confirm the modal renders approval details.

**Verification:**
Preview modal shows full approval routing tree without errors.

**Workarounds (from #support-rev-rlm-global-swarm-help):**
- When debugging flows with async steps, switch to debug mode **without** rollback to avoid the async step simulation error.
- Refer to the internal troubleshooting guide: https://docs.google.com/document/d/162E8MZZGShjoy37MqI9XyAXAZvXnkMbExPvuYizU-uo (RLM Advanced Approvals Smart Approvals guide — requires Salesforce internal access)

**Escalation Criteria:**
- Escalate if input variable types are correct and error still persists in Production.
- Escalate if the error is intermittent and org-wide (may be infrastructure-level).

---

### Pattern 3: Preview Shows ":x: No Queue" / "Invalid Assignee" Error

**Frequency:** High (~8 cases)  
**Severity:** Sev 3–4  
**TTR Impact:** Low — configuration fix

**Symptoms:**
- Approval Workflow Preview component shows `:x: No Queue` for work items that should be assigned to a queue
- Error in preview: `"Failed to process preview data: Invalid assignee [Queue_ID], the given assignee doesn't exist"`
- The Queue ID displayed is valid and has active members

**Root Cause:**
The flow's approval step is configured with a Queue **ID** (15 or 18-char Salesforce ID) as the assignee, but the Advanced Approvals engine expects the Queue's **API Name** (not ID) or the **Username** for user assignees. The preview engine cannot resolve the Queue by its ID in this context.

**Resolution Steps:**
1. Open the Approval Orchestration flow in Flow Builder.
2. Locate the approval step(s) configured with queue assignees.
3. Verify the assignee field is using the Queue's **API Name** (e.g., `Queue_API_Name`), not the Queue ID (`00G...`).
4. Similarly, for user assignees: use **Username** (e.g., `user@company.com`), not the User ID (`005...`).
5. Save and activate the flow.
6. Re-test via Preview — queue work items should now display correctly.

**Verification:**
Preview shows queue name correctly without `:x: No Queue` indicators.

**Workarounds (from #support-rev-rlm-global-swarm-help, ~Mar 2024):**
- If queue is correctly referenced by API Name but still shows No Queue, verify the queue has active members and is not archived.
- Behavior is expected during Preview if the queue has 0 active members — this is a known limitation of the Preview component.

**Escalation Criteria:**
- Escalate if Queue API Name is correctly set and queue has active members but the issue persists.

---

### Pattern 4: Submit for Approval – "No Applicable Approval Process Found"

**Frequency:** Medium  
**Severity:** Sev 2–3  
**TTR Impact:** Medium — 1–3 days

**Symptoms:**
- Clicking "Submit for Approval" on a Quote or Contract shows: `"We can't process your Submit For Approval request because of the following error: No applicable approval process was found."`
- Approval process exists and is active
- More common on Contract objects

**Root Cause:**
1. The approval process **entry criteria** does not match the current record state (e.g., Contract Status is not "Draft" at the time of submission).
2. For Contract: the associated contract document version is not checked in — error becomes: `"We can't process your Submit For Approval request because the associated contract document version isn't checked in."`
3. The approval orchestration flow is not correctly linked to the object/record type.

**Resolution Steps:**
1. Review the approval process entry criteria in Setup → Approval Processes.
2. Compare the current record's field values against the entry criteria — specifically Status, record type, and any custom criteria fields.
3. For Contract: verify a document version is checked in before submitting for approval.
4. Check if an active approval process exists for the object (Quote vs Contract vs Order).
5. Confirm the flow is active and that the entry criteria match the record at submission time.
6. Use Flow Debug to trace if/why the orchestration process is not being triggered.

**Verification:**
Customer can successfully click "Submit for Approval" and an ApprovalSubmission record is created with status "In Progress".

**Related Documentation:**
- [Contract Approval Workflow](https://help.salesforce.com/s/articleView?id=ind.sf_contracts_contract_approval_workflow.htm&type=5)

**Escalation Criteria:**
- Escalate if entry criteria match and flow is active but submission still fails.

---

### Pattern 5: Unhandled Fault When Non-Admin User Approves

**Frequency:** Medium  
**Severity:** Sev 2  
**TTR Impact:** Medium — 2–5 days

**Symptoms:**
- Non-System Administrator approvers see `"Unhandled Fault"` error when clicking Approve on an ApprovalWorkItem
- System Administrators can approve successfully
- Quote status does not update after the non-admin attempts to approve
- Customer has created a custom "RC Approval Permission Set" but issue persists

**Root Cause:**
Non-admin users lack the necessary **object-level and field-level permissions** on the Advanced Approvals objects (`ApprovalSubmission`, `ApprovalWorkItem`, `ApprovalOrchestration`). The Revenue Cloud Advanced Approvals objects require explicit FLS and CRUD permissions that are not included in standard profiles.

**Resolution Steps:**
1. Identify the non-admin user's Profile and Permission Sets.
2. Check FLS and Object permissions on:
   - `ApprovalSubmission` — Read, Edit
   - `ApprovalWorkItem` — Read, Edit
   - Key fields: `Status`, `OwnerId`, `ApprovalSubmissionId`
3. Create or update a Permission Set to grant:
   - `ApprovalSubmission`: Read, Edit
   - `ApprovalWorkItem`: Read, Edit, required field access
4. Assign the permission set to all approver users.
5. Test with a non-admin approver.

**Verification:**
Non-admin approver can approve the record without error; ApprovalWorkItem status updates to "Approved" and parent ApprovalSubmission advances.

**Workarounds (from #support-rev-rlm-global-swarm-help, ~Aug 2024):**
- Temporarily granting System Admin profile to the approver confirms it's a permissions issue, not a logic issue.

**Escalation Criteria:**
- Escalate if permissions are correct but fault persists — may require debug log analysis for the specific Apex class throwing the fault.

---

### Pattern 6: Smart Approvals Not Auto-Approving

**Frequency:** Very High (~22 cases)  
**Severity:** Sev 2–3  
**TTR Impact:** Medium — 3–7 days

**Symptoms:**
- Smart Approvals (auto-approval logic) criteria are met, but approval is NOT auto-approved — a manual ApprovalWorkItem is created instead
- `varAutoApprove = true` in flow logic but approval still goes to approver
- After re-submitting after "Requires Revision", Smart Approval does not trigger

**Root Cause:**
Multiple root causes:
1. **Missing FLS** on ApprovalSubmission object for Smart Approvals fields: `IsEligibleForSmartApproval`, `SmartApprvlBasisSubmissionId`, `IsSmartApprovalRun` — without these, the engine cannot mark the record as eligible.
2. **Conflicting flow logic** — Multiple Decision elements placed above the Approval Step interfere with Smart Approvals evaluation. The criteria on the Approval Step must exactly match the Decision criteria above it.
3. **Competing flows** — Multiple flows triggering on the same record (e.g., `Approval Orchestration Record Trigger HE` and a custom flow) can conflict; disable duplicates.
4. **Group/User ID vs API Name** — the flow is passing a User/Group **ID** (e.g., `005...`) instead of **Username** or **Group API Name** to the assignee field.

**Resolution Steps:**
1. Grant FLS on `ApprovalSubmission` for fields: `IsEligibleForSmartApproval`, `SmartApprvlBasisSubmissionId`, `IsSmartApprovalRun` to the running user's profile/permission set.
2. In the flow, ensure the Approval Step's criteria **exactly matches** any Decision element criteria above it. Remove redundant Decision elements if possible.
3. Check for duplicate flows on the same object — disable all but the primary approval orchestration flow.
4. Verify assignee references use **Username** or **Group API Name**, not IDs.
5. Debug the flow step-by-step using Flow Debug with `varAutoApprove` variable traced.
6. Refer to the internal Smart Approvals troubleshooting guide (Salesforce internal access required): https://docs.google.com/document/d/162E8MZZGShjoy37MqI9XyAXAZvXnkMbExPvuYizU-uo

**Verification:**
Submit a record meeting the auto-approve criteria. ApprovalSubmission is created with `IsSmartApprovalRun = true` and no manual ApprovalWorkItem is created.

**Related Documentation:**
- [Define Custom Logic for Auto-Approvals](https://help.salesforce.com/s/articleView?id=ind.approvals_define_custom_logic_auto_approvals.htm&type=5)

**Escalation Criteria:**
- Escalate if all FLS is granted, no conflicting flows, correct API Names, but Smart Approval still doesn't trigger — may be a platform bug.

---

### Pattern 7: Auto-Approval Audit Trail – Error Creating ApprovalWorkItem

**Frequency:** Low–Medium  
**Severity:** Sev 2  
**TTR Impact:** Medium — 3–5 days

**Symptoms:**
- Customer wants quotes to auto-approve **and** create an ApprovalWorkItem for audit trail purposes
- `varAutoApprove = true` fires correctly, but an error occurs when the flow tries to **create an ApprovalWorkItem** during the auto-approval step
- Error is thrown regardless of which user or input is used
- The approval status is created as "Assigned" instead of "Approved" — not completing the auto-approval

**Root Cause:**
The Advanced Approvals engine does not support creating a manual ApprovalWorkItem during an auto-approval flow step simultaneously — the auto-approval path and the manual work item creation path are mutually exclusive in the orchestration design. The platform prevents creating an ApprovalWorkItem in the same step context as the auto-approve signal.

**Resolution Steps:**
1. Separate the audit trail creation from the auto-approval step: use a **subsequent Background Step** (after the auto-approve step completes) to create the ApprovalWorkItem record, not inline.
2. If creating an audit record is mandatory at submission time, create a **custom object** record (not an ApprovalWorkItem) in a parallel step to log the auto-approval event.
3. Debug the specific flow version causing the issue using recordId from the customer's org and trace the step where the fault occurs.

**Verification:**
Quote is auto-approved (ApprovalSubmission status = Approved), and a separate audit record is created in the next Background Step.

**Escalation Criteria:**
- Escalate to engineering via `#rlm-office-hours` if this is a platform architectural limitation the customer needs a workaround for — engineering can confirm supportability.

---

### Pattern 8: Approval Submission Stuck "In Progress" After Work Item Approved

**Frequency:** High (~12 cases)  
**Severity:** Sev 2  
**TTR Impact:** Medium — 3–7 days

**Symptoms:**
- Approver clicks Approve on an ApprovalWorkItem; the work item status changes to "Approved"
- However, the parent **ApprovalSubmission remains in "In Progress"** status — does not advance
- Quote status does not update
- Issue is consistent (not intermittent)
- Customer is using Flow Orchestration-based approval (not old process builder)

**Root Cause:**
The ApprovalSubmission's status is driven by the orchestration flow's completion logic. Common root causes:
1. A subsequent step in the flow orchestration (after the approval step) has an error, blocking completion.
2. The flow's evaluation criteria for moving past the approval step is not met (e.g., `All Approvers Must Approve` but only one approved out of multiple required).
3. A Background Step downstream of the Approval Step is failing silently.

**Resolution Steps:**
1. Navigate to the failing ApprovalSubmission record and note the current stage/step name.
2. Open the Flow Orchestration's run history in Flow Debug → find the matching orchestration instance.
3. Identify which step after the approval is blocking (look for faulted or paused steps).
4. Check for any Background Steps that may be failing — run debug logs for the Automation Process User.
5. Verify the "Completion Criteria" on the Approval Step — ensure the approver count logic is correct.
6. If all approvers have approved and it's still stuck: check if a post-approval Background Step (e.g., a record update) is failing and blocking the orchestration.

**Verification:**
After fixing the blocking step, ApprovalSubmission advances to "Approved" and the parent record (Quote) status updates accordingly.

**Escalation Criteria:**
- Escalate if the orchestration instance shows all steps completed successfully but ApprovalSubmission is still "In Progress" — potential platform defect.

---

### Pattern 9: Approval Alert Emails Not Sending / Wrong Merge Fields

**Frequency:** Very High (~22 cases combined)  
**Severity:** Sev 3  
**TTR Impact:** Medium — 3–5 days

**Symptoms:**
- No email is triggered when a quote is submitted for approval or when an approval decision is made
- Emails are sent but merge fields are blank or wrong (e.g., Quote fields not populating)
- Custom email template works when `Related Entity Type = Quote` but not when set to `ApprovalWorkItem` or `ApprovalSubmission`

**Root Cause:**
The **Approval Alert Content Definition's Related Entity Type must match the entity type of the flow it is configured on**. Specifically:
- If the flow is configured on a Quote and the Related Entity Type = Quote → merge fields from Quote work correctly
- If Related Entity Type is set to `ApprovalWorkItem` or `ApprovalSubmission`, the email template cannot resolve Quote-level merge fields
- The system uses the Related Entity Type to determine which record's fields are available as merge fields

**Resolution Steps:**
1. Navigate to Setup → Approval Alert Content Definitions.
2. Verify the `Related Entity Type` on the email template matches the object the approval flow is running on.
   - For Quote approvals: set `Related Entity Type = Quote`
   - For Contract approvals: set `Related Entity Type = Contract`
3. Rebuild the merge field references in the email template using the correct entity.
4. If the customer needs both Quote and ApprovalWorkItem fields: create **two separate Approval Alert Content Definitions** — one per entity type — and use them at different steps.
5. Verify email deliverability is not blocked at the org level.

**Verification:**
Submit a record for approval and verify email is received with correct merge field values.

**Related Documentation:**
- Approval Alert Content Definitions setup guide in the RLM Developer documentation

**Escalation Criteria:**
- Escalate if entity type matches and emails are still not sending — check Apex email limits and org email deliverability settings.

---

### Pattern 10: ApprovalPreviousRelatedRecordDetails – "You Don't Have Access to This Action"

**Frequency:** Low  
**Severity:** Sev 3  
**TTR Impact:** Medium — 2–4 days

**Symptoms:**
- Error in flow debug: `"ApprovalPreviousRelatedRecordDetails.InvocableActionException: Looks like you don't have access to this action. Your Salesforce admin can help with that."`
- Occurs when using the `getPreviousRelaRecDetails` Apex invocable action for Smart Approvals auto-approval logic
- Customer is following the Salesforce Help doc for custom auto-approval logic

**Root Cause:**
The `getPreviousRelaRecDetails` invocable action (from the `ApprovalPreviousRelatedRecordDetails` Apex class) **must be called from a Background Step run by the Automation Process User**. When called from a standard user context or from a non-Background Step, the action throws an access exception.

**Resolution Steps:**
1. Confirm the flow step calling `getPreviousRelaRecDetails` is a **Background Step** (not an Interactive Step or an Autolaunched step running as a named user).
2. Verify the Background Step is configured to run as the **Automation Process User** (not the current user or another named user).
3. The invocable action takes 2 inputs:
   - `orchestrationInstanceId` — the current orchestration instance ID (map from the flow context)
   - `steps` — a Text Collection variable containing the step API names for which to fetch previous values
4. The Apex class itself does not need modification — the standard class from the help doc is correct.
5. Re-test by submitting a quote and verifying the Background Step completes without errors.

**Verification:**
Background Step runs successfully and returns previous record field values; Smart Approval auto-approve logic correctly evaluates against historical data.

**Related Documentation:**
- [Define Custom Logic for Auto-Approvals](https://help.salesforce.com/s/articleView?id=ind.approvals_define_custom_logic_auto_approvals.htm&type=5)

**Workarounds (from #support-rev-rlm-global-swarm-help, ~Mar 2025):**
- From Sohag Das: *"The action needs to be called by a Background Step that is run by the Automation Process User. The action takes 2 inputs — one is the current orchestration instance ID and the list of steps (text collection variable) for which we need to fetch the previous values. The Apex class can remain the same as provided in the help article."*

**Escalation Criteria:**
- Escalate if the Background Step is correctly configured as Automation Process User but the action still throws access exceptions.

---

### Pattern 11: Orphaned ApprovalWorkItems – Cannot Delete

**Frequency:** Low  
**Severity:** Sev 4  
**TTR Impact:** Low — process clarification

**Symptoms:**
- Customer requests deletion of `ApprovalWorkItem` records that are "orphaned" (no parent ApprovalSubmission, or parent is in a terminal state)
- Standard UI does not expose a delete button on ApprovalWorkItem records
- Attempting deletion via API returns permission errors

**Root Cause:**
`ApprovalWorkItem` records in Revenue Cloud Advanced Approvals are **system-managed records** that cannot be deleted through the standard UI or by customer admins. They are maintained as audit records tied to the orchestration lifecycle. Deletion requires a support-side data operation or specific platform tooling.

**Resolution Steps:**
1. Confirm the business reason for deletion (data cleanup, GDPR, test data purge).
2. Verify the records are truly orphaned (parent ApprovalSubmission is Deleted or Cancelled).
3. For test/sandbox data: recommend refreshing the sandbox instead of deleting individual records.
4. For Production: escalate — this requires engineering/data team involvement to perform the deletion safely.
5. Do **not** attempt to delete via Apex without engineering approval — cascade effects on ApprovalSubmission records are possible.

**Escalation Criteria:**
- Always escalate to engineering for Production ApprovalWorkItem deletion requests.
- Document the org ID, record IDs, and business justification before escalating.

---

### Pattern 12: Approval Trace Component Slow for Non-Admin Users

**Frequency:** Low–Medium
**Severity:** Sev 3–4
**TTR Impact:** Medium — 3–7 days

**Symptoms:**
- The **Approval Trace** component on a related list loads extremely slowly (30 seconds to 10 minutes) for non-admin users
- System Admins (with Read All / Modify All permissions) experience no delay
- Subsequent loads are faster; first load is consistently slow

**Root Cause:**
The Approval Trace component performs record-level visibility queries that are much more expensive for non-admin users who don't have "View All" permissions. Without sharing-bypass access, the component evaluates sharing rules and record ownership for each ApprovalWorkItem/ApprovalSubmission, resulting in high query cost.

**Resolution Steps:**
1. Verify the issue is specific to non-admin users — confirm an admin user loads it quickly.
2. Grant "View All" on `ApprovalSubmission` and `ApprovalWorkItem` via Permission Set to the affected users.
3. If "View All" is not desired for security reasons, evaluate whether a Custom List View or Report can replace the Approval Trace component for non-admin use.
4. If the issue persists with proper permissions, escalate as a performance bug.

**Verification:**
Non-admin user loads Approval Trace related list within 5 seconds.

**Escalation Criteria:**
- Escalate if "View All" permissions are granted and the component is still slow — likely a platform performance defect.

---

### Pattern 13: Approval Work Item Email Notifications Not Received

**Frequency:** Medium
**Severity:** Sev 3
**TTR Impact:** Low — configuration fix

**Symptoms:**
- Approvers are not receiving email notifications when an ApprovalWorkItem is assigned to them
- Submitters are not receiving status update emails
- Emails worked previously or work in another org

**Root Cause:**
Required **Process Automation Settings** at the org level are not enabled. These are distinct from flow configuration and are often overlooked.

**Resolution Steps:**
1. Go to Setup → Process Automation Settings.
2. Verify the following are **enabled**:
   - `Send Approval Work Item Assignment Emails to Approvers`
   - `Send Approval Submission Status Email Notifications to Submitters`
3. Check the **Org-Wide Email Address** is configured (null org-wide email silently suppresses all approval notification emails).
4. Verify email deliverability is set to "All Email" (Setup → Deliverability).
5. Check if Approval Alert Content Definition is misconfigured — see Pattern 9.

**Verification:**
Submit a test approval and confirm the assigned approver receives an email within 2 minutes.

**Workarounds (from #rev-nova-public — Peter Gillis):**
- Org-Wide Email Address must be set — this is the most commonly missed setting.

**Escalation Criteria:**
- Escalate if all settings are correct and emails are confirmed not in spam — requires Splunk/email delivery log investigation.

---

### Pattern 14: AppvlWorkItemNtfcnEvent – Slack Notifications Not Delivered

**Frequency:** Low
**Severity:** Sev 2–3
**TTR Impact:** Medium — infrastructure-level

**Symptoms:**
- Slack approval notifications are not delivered when an ApprovalWorkItem is assigned
- The approval flow runs successfully with no flow errors
- Splunk shows: `"Failed to start subscriber for topic [/event/AppvlWorkItemNtfcnEvent]"` and `"Topic cache hasn't been initialized yet, individual topic operations are blocked"`

**Root Cause:**
This is a **platform-level infrastructure failure** — the `AppvlWorkItemNtfcnEvent` platform event was published successfully (flow ran fine), but the Conduit/Platform Event subscriber failed to start due to topic cache initialization failure. This is not a flow configuration issue.

**Resolution Steps:**
1. Confirm via Splunk: `index=<env> organizationId=<OrgId> AND "AppvlWorkItemNtfcnEvent"` — look for `"Topic cache hasn't been initialized"`.
2. If intermittent: advise the customer to retry; the topic cache self-heals on re-initialization.
3. If persistent (not self-healing within 1 hour): escalate to the Platform Events / Conduit infrastructure team.

**Splunk Query:**
```
index=coretest organizationId=<OrgId> AND "AppvlWorkItemNtfcnEvent"
```

**Verification:**
Slack notification received after ApprovalWorkItem assignment; no topic cache errors in Splunk.

**Escalation Criteria:**
- Always escalate to platform infrastructure if the topic cache failure is persistent.

---

### Pattern 15: Quote Locking Not Working During Approval

**Frequency:** High (~8 cases)
**Severity:** Sev 2–3
**TTR Impact:** Medium — 2–5 days

**Symptoms:**
- Quote is not locked for editing after "Submit for Approval" is clicked in Revenue Cloud Advanced
- Sales Transaction Editor (line items) remains editable while approval is "In Progress"
- Quote record fields remain editable despite approval being submitted
- Approval Status changes to "Withdrawn" unexpectedly (related: locking not enforced)

**Root Cause:**
Record locking during Advanced Approvals is not automatic like classic Approval Processes. It requires explicit configuration:
1. The lock/unlock actions must be added as **Background Steps** in the orchestration flow (at submission and after approval/rejection)
2. The `Lock Record` and `Unlock Record` flow actions must be explicitly wired into the approval flow
3. Missing `Record Lock` permission for the running user can also prevent locking

**Resolution Steps:**
1. Open the Approval Orchestration flow in Flow Builder.
2. Verify a **Background Step** immediately after the approval trigger calls the `Lock Record` core action on the Quote.
3. Verify Background Steps on Approve/Reject/Recall paths call `Unlock Record` appropriately.
4. Confirm the Automation Process User has the `Lock Records` permission.
5. If using ARM (Approval Rules Module): verify the ARM parallel approval lock configuration under ARM Settings.

**Verification:**
After submitting for approval, attempt to edit the Quote — it should show "This record is locked" and prevent edits.

**Workarounds (from #support-rev-rlm-global-swarm-help):**
- If locking is not possible via flow, a temporary workaround is a validation rule on Quote that prevents edits when `ApprovalStatus__c = 'In Progress'`. Not recommended long-term.

**Escalation Criteria:**
- Escalate if Lock Record action is correctly placed in Background Steps but locking is inconsistent across users.

---

### Pattern 16: Recall Approval Submission Issues

**Frequency:** High (~10 cases)
**Severity:** Sev 2–3
**TTR Impact:** Medium — 2–5 days

**Symptoms:**
- "Recall Approval Submission" action not available in Flow Builder actions list
- Recall fails with error when triggered
- Recall comments are not stored/displayed after recall
- Recall email notification is not sent to approvers
- ApprovalSubmission status does not update to "Recalled" after recall action fires
- `Cannot Reference Recall Approval Submission` error in flow

**Root Cause:**
Multiple causes:
1. The `Cancel Approval Submission` / `Recall Approval Submission` actions are only available in specific flow types and contexts — they cannot be added to all flow types
2. Recall comments field on ApprovalSubmission requires FLS — without it, comments save silently as blank
3. The recall notification email relies on a separate email alert configuration from submission emails

**Resolution Steps:**
1. Verify the Recall action is used in the correct flow type — `Autolaunched Approval Orchestration` or `Record-Triggered Orchestration` only.
2. For `Cancel Approval Submission` not found: the action API name is `cancelApprovalSubmission` — search for it explicitly in the flow action search.
3. Grant FLS on `ApprovalSubmission.RecallComments__c` (or equivalent) to ensure recall comments are saved.
4. For recall emails not triggering: configure a separate Approval Alert Content Definition for the "Recalled" status transition.
5. Test by submitting an approval, then recalling — verify ApprovalSubmission status = "Recalled".

**Verification:**
ApprovalSubmission transitions to "Recalled" status, recall comments are visible on the record, and approvers receive recall notification email.

**Related Documentation:**
- [Recall Approval Submission](https://help.salesforce.com/s/articleView?id=ind.approvals_recall_approval_submission.htm&type=5)

**Escalation Criteria:**
- Escalate if the Recall action is correctly placed but ApprovalSubmission status doesn't update — may be a platform defect.

---

### Pattern 17: Duplicate Approval Records Created

**Frequency:** Medium (~4 cases)
**Severity:** Sev 2
**TTR Impact:** Medium — 3–5 days

**Symptoms:**
- Multiple ApprovalSubmission records created for a single "Submit for Approval" click
- Multiple ApprovalWorkItems sent to the same approver
- 50+ emails sent to approvers on a single submission (related: duplicate ApprovalWorkItems)
- Group membership causes unintended notifications to more users than expected

**Root Cause:**
1. **Competing flows**: Multiple record-trigger flows or approval orchestration flows firing on the same object on the same event (e.g., two flows both triggered by `Quote Status = Submitted`)
2. **Group hierarchy**: Using a Public Group with `Grant Access Using Hierarchies` enabled causes the approval notification to propagate up the manager chain, sending emails to many users
3. **Duplicate flow activations**: Flow was reactivated after a version change without deactivating the old version

**Resolution Steps:**
1. Check for duplicate active flows on the same object/trigger criteria in Setup → Flows.
2. Disable all but the intended active version of the approval orchestration flow.
3. For the "50+ emails" issue: check the Group definition — disable `Grant Access Using Hierarchies` on the Public Group used as approver if unintended hierarchy notifications are occurring.
4. For duplicate ApprovalSubmissions: add a check at submission time (e.g., validation rule or flow decision) to prevent re-submission if an active ApprovalSubmission already exists.

**Verification:**
Exactly one ApprovalSubmission and the expected number of ApprovalWorkItems are created per submission.

**Workarounds (from #support-rev-rlm-global-swarm-help):**
- Disable `Grant Access Using Hierarchies` on Public Groups used as approvers to prevent unintended email cascade.

**Escalation Criteria:**
- Escalate if no duplicate flows exist but duplicate records are still being created.

---

### Pattern 18: Platform License Users Cannot See Approval History / Work Items

**Frequency:** Medium (~4 cases)
**Severity:** Sev 3
**TTR Impact:** Low-Medium

**Symptoms:**
- Users with Platform License (not full Salesforce license) cannot see Approval Work Items on the Quote
- Approval History tab/component not visible to platform license users
- "Users unable to see Approval Work Items" even after permission sets assigned

**Root Cause:**
Advanced Approvals objects (`ApprovalWorkItem`, `ApprovalSubmission`) require a **Salesforce license** or **Revenue Cloud Advanced license** — they are not accessible with Platform licenses. Additionally, the Approval Trace and Approval History components require the license to render.

**Resolution Steps:**
1. Verify the affected users' license type in Setup → Users.
2. If they have Platform licenses: they cannot access ApprovalWorkItem/ApprovalSubmission objects directly — this is a license limitation.
3. For read-only visibility: consider exposing approval data via a **Custom Report** or **Dashboard** that platform license users can view.
4. If full visibility is required: work with the customer's AE to upgrade affected users to the appropriate license.

**Verification:**
Users with appropriate licenses can see Approval Work Items; platform license limitation is communicated to customer.

**Escalation Criteria:**
- Escalate to AE/licensing team if the customer disputes the license requirement.

---

## C. Setup & Configuration Guide

### Prerequisites for Advanced Approvals

| Requirement | Details |
|-------------|---------|
| License | Revenue Cloud Advanced or Revenue Cloud Growth (check in BT → Provisioned Products) |
| Org Permissions (BT) | `Advanced Approvals Enabled`, `Advanced Approvals: Org Pref: Dynamic Approval Notifications`, `Revenue Cloud Advanced Approvals` |
| Background Steps perm | `Standard Approvals: Background steps` (`UseBackgroundSteps`) — **only required if Advanced Approvals is NOT enabled**. When Advanced Approvals IS enabled, Background Steps work without this perm (confirmed: #rev-nova-public, Ramya Sukhavasi) |
| User Permissions | Approvers need Read/Edit on `ApprovalSubmission`, `ApprovalWorkItem` objects |
| FLS (Smart Approvals) | `IsEligibleForSmartApproval`, `SmartApprvlBasisSubmissionId`, `IsSmartApprovalRun` on `ApprovalSubmission` |

### BT Permissions Checklist (Enable in This Order)

1. `Revenue Cloud Advanced Approvals` (top-level)
2. `Advanced Approvals Enabled`
3. `Advanced Approvals: Org Pref: Dynamic Approval Notifications`
4. `Standard Approvals: Background steps` (`UseBackgroundSteps`) — for Background Steps
5. Verify `Autolaunched Approval Orchestration (No Trigger)` flow type appears in Flow Builder

### Common Misconfigurations

| Misconfiguration | Symptom | Fix |
|-----------------|---------|-----|
| Queue ID instead of Queue API Name in flow | "No Queue" in preview, "Invalid assignee" error | Use Queue API Name |
| User ID instead of Username | Assignee not resolved | Use Username (email format) |
| `Related Entity Type` mismatch in Approval Alert | Wrong/blank merge fields in email | Match entity type to flow's object |
| Multiple Decision elements above Approval Step | Smart Approvals not triggering | Match criteria exactly or remove redundant Decisions |
| Missing FLS on ApprovalSubmission | Smart Approvals fields blank | Grant FLS on Smart Approval fields |
| `getPreviousRelaRecDetails` called from non-Background Step | InvocableActionException | Move to Background Step as Automation Process User |
| Competing record-trigger flows | Approval behaves unpredictably | Disable duplicate flows on same object |

---

## D. Licensing & Entitlements

### Decision Flowchart

```
Is "Advanced Approvals Enabled" checked in BT?
│
├── NO → Request feature enablement via BT
│         (requires valid Revenue Cloud Advanced or Growth license)
│
└── YES
    │
    Is "Standard Approvals: Background steps" (UseBackgroundSteps) enabled?
    │
    ├── NO → Customer gets Background Step errors on flow save
    │        → Enable UseBackgroundSteps in BT
    │
    └── YES
        │
        Is Smart Approvals working?
        │
        ├── NO → Check FLS on ApprovalSubmission:
        │        IsEligibleForSmartApproval
        │        SmartApprvlBasisSubmissionId
        │        IsSmartApprovalRun
        │
        └── YES → License is correctly configured
```

### License Editions
Advanced Approvals is available in:
- Enterprise, Unlimited, and Developer Editions of Revenue Cloud
- With the **Revenue Cloud Growth** or **Revenue Cloud Advanced** license where Advanced Approvals is enabled

**Important:** This is **not** the same as CPQ Advanced Approvals (managed package). Revenue Cloud (Core) Advanced Approvals is a native feature built on Flow Orchestration.

---

## E. Documentation Quick-Reference

| Topic | Link |
|-------|------|
| Advanced Approvals Overview | https://help.salesforce.com/s/articleView?id=ind.approvals_advanced_approvals.htm&type=5 |
| Advanced Approvals Developer Guide | https://developer.salesforce.com/docs/atlas.en-us.revenue_lifecycle_management_dev_guide.meta/revenue_lifecycle_management_dev_guide/advanced_approvals_overview.htm |
| Define Custom Logic for Auto-Approvals | https://help.salesforce.com/s/articleView?id=ind.approvals_define_custom_logic_auto_approvals.htm&type=5 |
| Contract Approval Workflow | https://help.salesforce.com/s/articleView?id=ind.sf_contracts_contract_approval_workflow.htm&type=5 |
| Smart Approvals Troubleshooting (Internal) | https://docs.google.com/document/d/162E8MZZGShjoy37MqI9XyAXAZvXnkMbExPvuYizU-uo (Salesforce internal) |
| Smart Approvals Limitations (Internal) | https://docs.google.com/document/d/162E8MZZGShjoy37MqI9XyAXAZvXnkMbExPvuYizU-uo (Limitations tab) |

### Key Design Rules & Gotchas
- Advanced Approvals uses **Flow Orchestration** — not Process Builder or classic Approval Processes
- Flow type must be `Autolaunched Approval Orchestration (No Trigger)` or `Record-Triggered Orchestration`
- Assignees must use **API Names** (Queue API Name, Username) — **never Salesforce IDs**
- `getPreviousRelaRecDetails` Apex action only runs in **Background Steps as Automation Process User**
- Approval Alert Content Definition's `Related Entity Type` must match the flow's primary object
- Smart Approvals fields (`IsEligibleForSmartApproval` etc.) require explicit FLS — not auto-granted
- Multiple competing flows on same object break Smart Approvals evaluation

---

## F. Escalation Paths

### When to Escalate
| Situation | Escalation Path |
|-----------|----------------|
| License correctly provisioned but features still disabled | BT/Provisioning team |
| Correct permissions set but "Unhandled Fault" persists for non-admins | Engineering via #rlm-office-hours |
| ApprovalSubmission stuck In Progress, all steps show complete | Engineering bug — file GUS work item under `RLM - Advanced Approvals` |
| Production ApprovalWorkItem deletion request | Engineering/Data team required |
| Smart Approvals: all config correct, still not auto-approving | Engineering via #rlm-office-hours |
| getPreviousRelaRecDetails fails from Background Step/Automation Process User | Engineering via #rlm-office-hours |

### Before Escalating — Gather This Information
- Org ID (Production and Sandbox if applicable)
- ApprovalSubmission ID, ApprovalWorkItem ID (if relevant)
- Flow API Name and version number
- Debug log for the failing step (captured as Automation Process User and as the customer user)
- Screenshot of BT showing current permission/license state
- Steps to reproduce in a sandbox

### Escalation Channels
- **#support-rev-rlm-global-swarm-help** — primary swarm channel for all RLM/Revenue Cloud Advanced support cases
- **#rlm-office-hours** — engineering office hours for architectural questions and edge cases
- **#rev-nova-public** — product team channel; use for license/provisioning clarifications and feature questions
- **GUS product tag:** `RLM - Advanced Approvals`

---

## G. Tribal Knowledge & Pro Tips

1. **Assignees must be API Names, never IDs** — the #1 root cause of "No Queue" and "Invalid Assignee" preview errors. Always instruct customers to use Queue API Name and Username, not the 18-char Salesforce ID. *(#support-rev-rlm-global-swarm-help, multiple cases, 2024)*

2. **UseBackgroundSteps is a separate BT perm** — even after enabling Revenue Cloud Advanced Approvals, the `UseBackgroundSteps` perm must be explicitly enabled in BT. Not in the standard license provisioning checklist. *(#chat_hc_channel, #support-rev-rlm-global-swarm-help, multiple orgs)*

3. **Smart Approvals FLS is silent** — when `IsEligibleForSmartApproval` FLS is missing, Smart Approvals just silently fails (creates a manual work item). No error surfaces. Always check FLS first when Smart Approvals doesn't trigger. *(#support-rev-rlm-global-swarm-help, Jun 2022)*

4. **getPreviousRelaRecDetails MUST run as Automation Process User in a Background Step** — the help doc shows the Apex code correctly, but doesn't emphasize this context requirement. Running this action from any other context will throw InvocableActionException. *(#support-rev-rlm-global-swarm-help — Sohag Das, Mar 2025)*

5. **Match Production Licenses doesn't always fix sandbox** — if Production itself doesn't have `UseBackgroundSteps`, matching sandbox to Production won't help. Fix Production BT first. *(#chat_hc_channel — Vishwanath Murthy Challa)*

6. **Don't use multiple Decision elements above a single Approval Step** — this breaks Smart Approvals evaluation. The criteria on the Approval Step must exactly match the single Decision element above it. *(#support-rev-rlm-global-swarm-help — Harsh Deep, Dec 2024)*

7. **Approval Alert Content Definition: match entity type to flow object** — if the flow runs on Quote, the Related Entity Type must be Quote. Mixing this with ApprovalWorkItem type causes merge fields to fail silently. *(#support-rev-rlm-global-swarm-help — Shrutika Shamarthi, Sep 2024)*

8. **The internal troubleshooting guide is the go-to reference** — https://docs.google.com/document/d/162E8MZZGShjoy37MqI9XyAXAZvXnkMbExPvuYizU-uo — covers Smart Approvals limitations and setup in detail. *(#support-rev-rlm-global-swarm-help — Vrushabh Mahendrakar, Harsh Deep)*

9. **This is NOT CPQ Advanced Approvals** — clearly distinguish from the managed package. If a customer mentions "CPQ" + "Advanced Approvals", verify which product they're on before troubleshooting. RCA Advanced Approvals is Flow Orchestration-based; CPQ Advanced Approvals is a separate managed package with different objects. *(#support-cross-cloud — Suyash Sinha)*

10. **Debug without rollback for async step flows** — when the flow includes async steps (e.g., Slack notifications via Background Steps), debug with rollback disabled. Otherwise the debugger forces all async steps into "simulated" mode and shows misleading errors. *(#support-rev-rlm-global-swarm-help — Vinay Vura)*

11. **UseBackgroundSteps is ONLY needed when Advanced Approvals is disabled** — the error message `"doesn't have an active Advanced Approvals license"` is misleading. When Advanced Approvals IS enabled, Background Steps work without `UseBackgroundSteps`. When it's NOT enabled, `UseBackgroundSteps` is the fix. Fix order: enable Advanced Approvals first; only add `UseBackgroundSteps` as a fallback for non-AA Background Step usage. *(#rev-nova-public — Ramya Sukhavasi, product team confirmation)*

12. **Process Automation Settings control email delivery** — go-to for "emails not sending" issues: Setup → Process Automation Settings → enable `Send Approval Work Item Assignment Emails to Approvers` and `Send Approval Submission Status Email Notifications to Submitters`. Org-Wide Email Address being null silently suppresses all approval emails. *(#rev-nova-public — Peter Gillis)*

13. **"Manage all pending approval submissions" tab shows "The requested resource does not exist"** — ensure the user has the correct Advanced Approvals permission (either "Advanced Approvals Admin" or "Advanced Approvals User" permission set). Switching to "All Approval Submissions" list view resolves the navigation issue. *(#rev-nova-public — Uditi Arora)*

14. **Approval modal not appearing / loading extremely slowly at Submit** — if the modal to enter request comments doesn't appear or is invisible after clicking "Submit for Approval", this is usually a UI rendering issue under load. Advise customer to refresh the page (2x if needed) and try again in a non-peak window. Persistent occurrences should be captured with browser console logs. *(#rev-nova-public — Burt Demchick)*

15. **Quote locking is NOT automatic in Advanced Approvals** — unlike classic Approval Processes, RCA Advanced Approvals does not auto-lock records. You must explicitly add `Lock Record` Background Steps in the orchestration flow at submission, and `Unlock Record` steps on all exit paths (approve/reject/recall). This is one of the most frequently missed setup steps. *(case data analysis, ~8 cases, 2025–2026)*

---

## H. Active Known Issues

*Connect GUS (`sf org login --alias gus`) and run the query below for live P0/P1/P2 bugs:*

```bash
sf data query \
  --query "SELECT Name, Subject__c, Status__c, Priority__c FROM ADM_Work__c WHERE Product_Tag__r.Name = 'RLM - Advanced Approvals' AND Type__c = 'Bug' AND Status__c IN ('New','Triaged','In Progress') AND Priority__c IN ('P0','P1','P2') ORDER BY Priority__c, CreatedDate DESC LIMIT 20" \
  --target-org gus --json
```

*At time of skill generation (2026-05-27): GUS query returned 0 active P0/P1/P2 bugs under `RLM - Advanced Approvals` tag. Check live before each case.*

---

## I. Code Reference Map

> **CodeSearch not connected** — connect CodeSearch in AI Suite Settings → MCP Servers → CodeSearch to get code-level root cause references for each pattern.

Key classes to search when CodeSearch is available:
| Class/Object | Purpose | Search Query |
|-------------|---------|-------------|
| `ApprovalPreviousRelatedRecordDetails` | Invocable action for Smart Approvals history | `content:"ApprovalPreviousRelatedRecordDetails" lang:apex` |
| `ApprovalSubmission` | Core approval record object | `content:"ApprovalSubmission" lang:apex` |
| `ApprovalWorkItem` | Individual work item per approver | `content:"ApprovalWorkItem" lang:apex` |
| Smart Approvals engine | `IsEligibleForSmartApproval` logic | `content:"IsEligibleForSmartApproval" lang:apex` |
| Background Step context check | Context validation for invocable actions | `content:"UseBackgroundSteps" OR content:"BackgroundStep"` |
