# Graph API Access Management Standard — Implementation Guide

**Companion to:** Microsoft Graph API Access Management Standard
**Audience:** Engineers implementing the standard (IDAM, M365 platform, Cyber Security)
**Format:** One section per control. Each section gives prerequisites, implementation steps (portal + Graph PowerShell side by side where applicable), and verification.

---

## How to use this document

Controls are grouped by stage. Implement stages in order — most Stage 2 controls assume Stage 1 controls are already in place. Each control lists its explicit prerequisites at the top.

The portal and PowerShell paths are presented side by side because some controls are easier to inspect in the portal and easier to enforce/audit via PowerShell. Use whichever fits your team's habits; the end state is the same.

Throughout, `Connect-MgGraph` is assumed to have been run with the appropriate scopes. Each control lists the minimum scopes needed for its specific operations.

---

# Stage 1 — Baseline (Essential Eight ML1)

The baseline establishes identity hygiene, MFA, and logging. It is the foundation Stage 2 builds on; without it, the PIM, revalidation, and inactivity controls in Stage 2 either can't be configured or produce meaningless results.

---

## S1-01 — Named admin accounts for all interactive Graph use

**What this control requires:** All interactive Graph administration is performed from named admin accounts (`adm-*`). Shared accounts are prohibited.

**Prerequisites:** None (this is the foundation).

**Implementation steps:**

1. Define the admin account naming convention. The agency standard is `adm-firstname.lastname@<tenant>` (or `adm-firstname.lastname@<adminsubdomain>` if a dedicated admin domain is in use).
2. Identify every individual who currently performs Graph administration. For each, ensure an `adm-*` account exists.
3. Block all existing shared admin accounts from interactive sign-in. Do not delete them yet — they may still be referenced as owners; reassign ownership first.
4. Communicate the change with a cut-over date. After that date, admin Graph sessions from non-`adm-*` UPNs are blocked by Conditional Access (see S2-11 for the device-side enforcement; for the UPN-side enforcement, use a CA policy targeting "Microsoft Graph" as the cloud app and requiring the user be a member of the `sg-admin-accounts` group).

**Portal:**
- Entra admin centre → Identity → Users → New user → User name `adm-firstname.lastname`, give a non-mailbox UPN suffix if you use one, do not assign any licences beyond what's strictly needed (typically none — admin accounts don't need Office licences).

**Graph PowerShell:**
```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All"
New-MgUser -DisplayName "Admin - Jane Smith" `
           -UserPrincipalName "adm-jane.smith@contoso.gov.au" `
           -MailNickname "adm-jane.smith" `
           -AccountEnabled `
           -PasswordProfile @{ ForceChangePasswordNextSignIn = $true; Password = (New-RandomPassword) }
```

**Verification:**
- Run a query for any account in any privileged Entra role group that does not match the `adm-*` UPN pattern. The expected result is zero (after grace period).
  ```powershell
  Get-MgDirectoryRole | ForEach-Object {
      Get-MgDirectoryRoleMember -DirectoryRoleId $_.Id -All |
        Where-Object { $_.AdditionalProperties.userPrincipalName -notlike "adm-*" }
  }
  ```
- Evidence: a screenshot of the above returning empty, plus the IDAM register showing the mapping between standard and admin accounts.

---

## S1-02 — Admin accounts separated from standard productivity accounts

**What this control requires:** Admin accounts use distinct UPN and credentials from the user's standard productivity account.

**Prerequisites:** S1-01.

**Implementation steps:**

1. Confirm every admin account holder also has a separate, non-privileged standard account for email, Teams, document work, and general productivity.
2. Document the pairing in the IDAM register (standard account ↔ admin account ↔ owning manager).
3. Ensure the admin account has no mailbox or, if one is required for receipt of PIM and access-review notifications, that it forwards to the standard account and outbound mail is blocked.
4. Set Conditional Access to block the admin account from accessing Office 365 services other than the Microsoft Graph and Entra admin centre apps (see S1-10).

**Portal:**
- Entra admin centre → Protection → Conditional Access → New policy → Users: `sg-admin-accounts` → Cloud apps: Office 365 (exclude Microsoft Graph, Microsoft Azure Management, Microsoft Admin Portals) → Grant: Block access.

**Graph PowerShell:**
```powershell
# Inspect to confirm
Get-MgUser -UserId "adm-jane.smith@contoso.gov.au" -Property Mail,AssignedLicenses,UserPrincipalName
```

**Verification:**
- Quarterly extract: every member of any admin group has a matching standard account in the register. Mismatches investigated within 5 business days.

---

## S1-03 — Phishing-resistant MFA on all admin accounts

**What this control requires:** FIDO2 or Windows Hello for Business enforced for admin accounts via Conditional Access.

**Prerequisites:** S1-01.

**Implementation steps:**

1. Define an authentication strength policy that allows only FIDO2 security keys and Windows Hello for Business.
2. Issue FIDO2 keys to every admin account holder. Register at least one backup key per user.
3. Create a Conditional Access policy targeting the `sg-admin-accounts` group, all cloud apps, requiring that authentication strength.
4. Run the policy in report-only mode for one week; review the sign-in logs for impact, then enable.

**Portal:**
- Entra → Protection → Authentication methods → Authentication strengths → New custom strength → "Phishing-resistant admin" → FIDO2, Windows Hello for Business.
- Entra → Protection → Conditional Access → New policy → Users: `sg-admin-accounts` → Cloud apps: All → Grant: Require authentication strength → "Phishing-resistant admin".

**Graph PowerShell:**
```powershell
Connect-MgGraph -Scopes "Policy.ReadWrite.AuthenticationMethod","Policy.ReadWrite.ConditionalAccess"
# Inspect existing CA policies
Get-MgIdentityConditionalAccessPolicy | Where-Object { $_.DisplayName -like "*admin*MFA*" }
# Inspect authentication strength policies
Get-MgPolicyAuthenticationStrengthPolicy | Select-Object DisplayName,Id,RequirementsSatisfied
```

**Verification:**
- Sign-in log query: every interactive sign-in by an `adm-*` account in the last 30 days satisfies a phishing-resistant authentication strength. Any exception is investigated.
- Evidence: export of the Conditional Access policy JSON; sign-in log filtered by `userPrincipalName startsWith "adm-"` and `authenticationDetails.authenticationMethod`.

---

## S1-04 — Break-glass Global Administrator accounts

**What this control requires:** Two break-glass GA accounts excluded from blocking CA policies, with unique strong passphrases in a sealed vault, and sign-in alerting.

**Prerequisites:** None.

**Implementation steps:**

1. Create two cloud-only Global Administrator accounts with non-personal UPNs (e.g. `breakglass-01@contoso.onmicrosoft.com`, `breakglass-02@contoso.onmicrosoft.com`). Use the `.onmicrosoft.com` domain so they are unaffected by custom domain federation outages.
2. Generate 32-character random passphrases. Store one in the agency's physical safe, the other in a sealed envelope held by Cyber Security management. Record opening events.
3. Exclude both accounts from **every** Conditional Access policy that could lock them out (MFA, location restrictions, device compliance, risk-based). Excluding them from MFA is acceptable for break-glass; the protection is the sealed credential, not MFA.
4. Configure a high-severity SIEM alert on any sign-in event for these two accounts. Triage SLA: 15 minutes, 24/7.
5. Test the break-glass procedure quarterly. Document the test.

**Portal:**
- Entra → Identity → Users → New user → cloud-only, GA role assigned directly (not via group — break-glass should not depend on group resolution).
- Entra → Protection → Conditional Access → Edit each policy → Users → Exclude → add both accounts.

**Graph PowerShell:**
```powershell
# List CA policies that DO NOT exclude break-glass accounts (defect)
$breakGlassIds = @("<bg1-id>","<bg2-id>")
Get-MgIdentityConditionalAccessPolicy -All | Where-Object {
    $policy = $_
    $excluded = $policy.Conditions.Users.ExcludeUsers
    -not ($breakGlassIds | ForEach-Object { $excluded -contains $_ } | Where-Object { $_ -eq $false }).Count -eq 0
} | Select DisplayName,Id
```

**Verification:**
- A run of the script above returns zero rows.
- Quarterly break-glass test report on file with Cyber Security.
- SIEM alert rule exported and stored as configuration evidence.

---

## S1-05 — Role-assignable security groups for Graph permissions

**What this control requires:** All Graph permissions assigned via role-assignable security groups, not directly to users.

**Prerequisites:** S1-01.

**Implementation steps:**

1. For each role family in the standard (M365 Collaboration, IDAM User Lifecycle, IDAM Access Governance, IDAM Privileged), create the tier groups per the naming convention in Appendix B of the standard.
2. Each group is created **role-assignable** (`isAssignableToRole = true`). This setting cannot be changed later.
3. Assign Entra directory roles to each group, not to individuals.
4. Migrate any existing direct role assignments by adding the user to the corresponding group and then removing the direct assignment.

**Portal:**
- Entra → Identity → Groups → New group → Security → Set "Microsoft Entra roles can be assigned to the group" = Yes.

**Graph PowerShell:**
```powershell
Connect-MgGraph -Scopes "Group.ReadWrite.All","RoleManagement.ReadWrite.Directory"
$group = New-MgGroup -DisplayName "sg-graph-idam-userlifecycle-write" `
                     -Description "IDAM User Lifecycle write tier (User Administrator)" `
                     -MailEnabled:$false -SecurityEnabled `
                     -MailNickName "sg-graph-idam-userlifecycle-write" `
                     -IsAssignableToRole:$true

# Find the User Administrator role definition
$role = Get-MgRoleManagementDirectoryRoleDefinition -Filter "displayName eq 'User Administrator'"

# Assign the role to the group (this makes the group eligible later under PIM in S2-01)
New-MgRoleManagementDirectoryRoleAssignment `
  -PrincipalId $group.Id `
  -RoleDefinitionId $role.Id `
  -DirectoryScopeId "/"
```

> Note: The role assignment above is a permanent active assignment. In Stage 2 (S2-01), you will convert these to PIM-eligible assignments. For Stage 1 it is acceptable to have these as active; the goal at this stage is only to ensure no individuals hold direct role assignments.

**Verification:**
- `Get-MgRoleManagementDirectoryRoleAssignment -All` filtered to assignments whose `PrincipalId` resolves to a user (not a group). Expected result after migration: only the two break-glass accounts.
- Evidence: export of the role assignment list with principal type column.

---

## S1-06 — Role group register

**What this control requires:** A documented register listing each role family, the Graph permissions granted, the Entra roles conferred, and the named group owner.

**Prerequisites:** S1-05.

**Implementation steps:**

1. Create a register (SharePoint list or equivalent) with columns: Group name, Group ID (object ID from Entra), Role family, Tier (read / write / privileged), Entra roles assigned, Graph permissions inferred from those roles, Group owner (named human), Reviewer, Last reviewed, Next review.
2. Populate from the groups created in S1-05.
3. Assign a named human owner to every group in Entra (group owner field), matching the register.
4. Make the register the source of truth for S2-05 and S2-06 reviews.

**Portal:**
- Entra → Groups → select group → Owners → Add owners.

**Graph PowerShell:**
```powershell
# Export current state for the register
Get-MgGroup -Filter "startsWith(displayName,'sg-graph-')" -All |
  ForEach-Object {
    [pscustomobject]@{
      DisplayName = $_.DisplayName
      Id          = $_.Id
      IsAssignableToRole = $_.IsAssignableToRole
      Owners      = (Get-MgGroupOwner -GroupId $_.Id -All).AdditionalProperties.userPrincipalName -join "; "
    }
  } | Export-Csv role-group-register-snapshot.csv -NoTypeInformation
```

**Verification:**
- Every group whose name starts with `sg-graph-` has at least one named owner and a register entry. Run the export above and reconcile against the register.

---

## S1-07 — App registration inventory

**What this control requires:** Every app registration is inventoried with business owner, technical owner, and purpose. Orphans are blocked from acquiring new permissions.

**Prerequisites:** None.

**Implementation steps:**

1. Export the current list of app registrations from Entra.
2. For each, identify business and technical owner. Anything that cannot be attributed becomes an "orphan candidate" and is flagged.
3. Record in the agency's ITSM CMDB (or equivalent) with: app display name, app (client) ID, object ID, purpose, business owner, technical owner, requested permissions, credential type, next review date.
4. For orphan candidates, send a 30-day claim notice to known former owners and the team distribution lists. If unclaimed after 30 days, disable the service principal (set `accountEnabled = false`) and remove all credentials. Delete after a further 60 days.
5. Implement a policy in your app provisioning workflow that requires owner attestation before any new permission grant is approved.

**Portal:**
- Entra → Identity → Applications → App registrations → All applications.

**Graph PowerShell:**
```powershell
Connect-MgGraph -Scopes "Application.Read.All"
Get-MgApplication -All | ForEach-Object {
    $app = $_
    $owners = Get-MgApplicationOwner -ApplicationId $app.Id -All
    [pscustomobject]@{
        DisplayName       = $app.DisplayName
        AppId             = $app.AppId
        ObjectId          = $app.Id
        OwnerCount        = $owners.Count
        OwnerUPNs         = ($owners.AdditionalProperties.userPrincipalName) -join "; "
        SignInAudience    = $app.SignInAudience
        CreatedDateTime   = $app.CreatedDateTime
    }
} | Export-Csv app-registration-inventory.csv -NoTypeInformation
```

**Verification:**
- The inventory CSV cross-referenced against the CMDB. Every row has at least one owner and a CMDB entry. Orphan list separately tracked to closure.

---

## S1-08 — Restrict tenant-wide user consent and enable admin consent workflow

**What this control requires:** Users cannot consent to apps requesting risky permissions on their own behalf. Admin consent workflow is enabled.

**Prerequisites:** None.

**Implementation steps:**

1. Set the user consent policy to "Allow user consent for apps from verified publishers, for selected permissions" — typically just `User.Read`, `offline_access`, `openid`, `profile`, `email`.
2. Enable the admin consent workflow with a designated reviewer group (the IDAM — Privileged team).
3. Communicate to staff that consent requests will be routed via ITSM ticket.
4. Block group owner consent for groups (a related vector).

**Portal:**
- Entra → Identity → Enterprise applications → Consent and permissions → User consent settings → choose the "Allow for verified publishers, for selected permissions" option, then configure permission classifications.
- Same page → Admin consent settings → Enable, set reviewers.

**Graph PowerShell:**
```powershell
Connect-MgGraph -Scopes "Policy.ReadWrite.Authorization"
# Inspect current settings
Get-MgPolicyAuthorizationPolicy | Select-Object DefaultUserRolePermissions

# Set user consent restriction
Update-MgPolicyAuthorizationPolicy -DefaultUserRolePermissions @{
    PermissionGrantPoliciesAssigned = @(
        "ManagePermissionGrantsForSelf.microsoft-user-default-low"
    )
}
```

**Verification:**
- A test standard user account attempts to consent to a non-classified-as-low permission and is blocked, with the admin consent workflow triggered. Test recorded with ticket reference.

---

## S1-09 — Unified Audit Log and SIEM forwarding

**What this control requires:** UAL enabled, sign-in and audit logs forwarded to SIEM, 12-month retention minimum.

**Prerequisites:** None.

**Implementation steps:**

1. Confirm Unified Audit Log is enabled tenant-wide (it is on by default for new tenants; older tenants may need to be confirmed).
2. Configure a diagnostic setting on the Entra tenant to send: AuditLogs, SignInLogs, NonInteractiveUserSignInLogs, ServicePrincipalSignInLogs, ManagedIdentitySignInLogs, ProvisioningLogs, and RiskyUsers to Log Analytics or directly to the SIEM via Event Hub.
3. Confirm the SIEM ingests these and applies a 12-month retention policy.
4. Build a heartbeat detection: alert if any of these log categories stops flowing for more than 30 minutes.

**Portal:**
- Purview compliance portal → Audit → confirm enabled.
- Entra → Monitoring → Diagnostic settings → Add diagnostic setting.

**Graph PowerShell:**
```powershell
# UAL status via Exchange Online (separate module)
# Sign-in log forwarding cannot be configured purely from Graph PowerShell; use Az PowerShell:
# Get-AzDiagnosticSetting -ResourceId "/providers/microsoft.aadiam"
```

**Verification:**
- Run a known event (e.g. add then remove a test user from a non-privileged group) and confirm the event appears in the SIEM within the expected ingestion delay (typically under 15 minutes).
- Retention policy on the SIEM data store shows ≥ 12 months.

---

## S1-10 — Admin accounts cannot access internet/email/web services from admin session

**What this control requires:** Admin accounts are blocked from accessing the public internet, web mail, and non-administrative web services from the session used for Graph admin.

**Prerequisites:** S1-01, S1-02.

**Implementation steps:**

1. The admin account itself has no mailbox and no Office licence (set in S1-02).
2. Conditional Access policy: admin accounts blocked from Office 365 cloud apps except the ones needed for Graph admin (Microsoft Graph, Microsoft Azure Management, Microsoft Admin Portals, Office 365 Exchange Online only if mail forwarding for PIM notifications is required).
3. On the PAW or AVD admin environment (which lands in S2-11), browser is restricted by network egress policy to administrative endpoints only (Microsoft cloud, Entra portal, package mirrors for tooling).
4. Where staff need to do "look up an article" or check non-admin webmail during a session, they do that from their standard productivity account on their standard endpoint.

**Portal:**
- Entra → Conditional Access → policy as described in S1-02 implementation step 4.

**Verification:**
- A test sign-in by an admin account to (e.g.) Outlook on the web is blocked by CA.
- Network egress logs from the admin environment show no traffic to non-allowlisted FQDNs.

---

# Stage 2 — Target state (Essential Eight ML2)

Stage 2 turns the structural baseline into operating discipline: just-in-time activation, second approval for the highest-risk tiers, scheduled revalidation, automated removal on inactivity, and equivalent governance for service principals.

**All Stage 2 controls assume Stage 1 controls are implemented.** Where a Stage 2 control has additional Stage 2 dependencies, they are listed explicitly.

---

## S2-01 — PIM-eligible assignment for Tier 2 and Tier 3 groups

**What this control requires:** Tier 2 and Tier 3 Graph role groups are PIM-eligible. No standing membership for write/privileged groups.

**Prerequisites:** S1-05 (groups exist), S1-06 (register exists), Entra ID P2 licensing on all admin account holders.

**Implementation steps:**

1. For each Tier 2 and Tier 3 group identified in the register, convert the role assignment from active to eligible. In practice this means: the **group → role** assignment can remain active, but the **user → group** membership is the PIM-eligible target.
2. Onboard the group to PIM for Groups (each role-assignable group is automatically eligible for PIM management; explicit "onboarding" only required for non-role-assignable groups managed by PIM).
3. For each current standing member: remove direct membership, create a PIM eligible assignment for the same user against the same group with no expiry (or a finite eligibility window if turnover is expected).
4. Configure activation settings per the standard (see S2-02 for activation requirements, S2-03 for approval requirements).
5. Communicate the change to affected staff with a cut-over date and a quick-reference card on how to activate.

**Portal:**
- Entra → Identity Governance → Privileged Identity Management → Groups → select group → Assignments → Add assignments → Eligible → select users → set assignment type Eligible, max duration.

**Graph PowerShell:**
```powershell
Connect-MgGraph -Scopes "RoleManagement.ReadWrite.Directory","PrivilegedAccess.ReadWrite.AzureADGroup"

# Create an eligible PIM assignment for a user to a role-assignable group
$params = @{
    accessId      = "member"
    principalId   = "<user-object-id>"
    groupId       = "<group-object-id>"
    action        = "AdminAssign"
    scheduleInfo  = @{
        startDateTime = (Get-Date).ToUniversalTime()
        expiration    = @{ type = "noExpiration" }
    }
    justification = "Approved per change CHG0012345 — standing eligibility for IDAM team"
}
New-MgIdentityGovernancePrivilegedAccessGroupEligibilityScheduleRequest -BodyParameter $params
```

**Verification:**
- Query for any active (not eligible) member of a Tier 2 or Tier 3 group, excluding break-glass and emergency exception accounts: expected result zero.
  ```powershell
  Get-MgIdentityGovernancePrivilegedAccessGroupAssignmentSchedule -Filter "groupId eq '<id>'"
  ```
- Evidence: PIM "Assignments" tab for each group shows users in the "Eligible" column, "Active" column empty.

---

## S2-02 — PIM activation requirements (justification, ticket, MFA, ≤ 8h window)

**What this control requires:** Activation requires justification, ticket reference, MFA at activation, and max activation window of 8 hours.

**Prerequisites:** S2-01.

**Implementation steps:**

1. For each Tier 2 group: open PIM → Groups → select group → Settings → Member role → set:
   - **Activation maximum duration:** 8 hours
   - **Require justification on activation:** Yes
   - **Require ticket information on activation:** Yes
   - **Require Microsoft Entra Conditional Access authentication context** OR **Require MFA on activation:** Yes (authentication context is preferred and is what enforces the phishing-resistant strength from S1-03)
2. For each Tier 3 group, use the same settings as a baseline. Additional approval requirements are configured in S2-03.
3. Create an authentication context (e.g. `c1: PIM activation`) and bind a CA policy requiring the phishing-resistant authentication strength to that context. Set PIM to require that context on activation.

**Portal:**
- Entra → PIM → Groups → group → Settings → Member → Edit → set the four flags above and save.
- Entra → Conditional Access → Authentication contexts → New → "PIM activation".
- Entra → Conditional Access → New policy targeting the auth context, grant requires authentication strength = phishing-resistant.

**Graph PowerShell:**
```powershell
# Inspect current activation policy on a PIM-managed group role
Get-MgPolicyRoleManagementPolicyAssignment -Filter "scopeId eq '<group-id>' and scopeType eq 'Group'"
# Then walk the rules:
$policyId = "<from-above>"
Get-MgPolicyRoleManagementPolicyRule -UnifiedRoleManagementPolicyId $policyId
# Update rules with Update-MgPolicyRoleManagementPolicyRule
```

> Most teams find the portal far easier for PIM rule configuration than PowerShell; the PowerShell route is suitable for export and drift detection (which becomes useful at Stage 3, S3-03).

**Verification:**
- Export the PIM rules for every Tier 2/3 group and assert: activation max duration ≤ 8h, justification required, ticket required, MFA/auth context required.
- Sample an activation event in the audit log and confirm the justification text and ticket number are present and non-empty.

---

## S2-03 — Secondary approval for Tier 3 activations

**What this control requires:** Tier 3 (privileged) activation additionally requires approval from a second nominated approver from a different team.

**Prerequisites:** S2-02.

**Implementation steps:**

1. For each Tier 3 group, configure the PIM activation policy to require approval. Set approvers to a named group containing approvers from outside the requesting team (typically the Cyber Security manager and CISO delegate).
2. Document the approval roster, on-call rotation, and out-of-hours SLA for approvals.
3. Set the activation request notification to also email Cyber Security operations so they are aware in real time.

**Portal:**
- Entra → PIM → Groups → group → Settings → Member → Edit → enable "Require approval to activate" → choose approvers.

**Verification:**
- Test activation: a Tier 3 group activation request enters Pending Approval state and cannot complete without an approver action. Test recorded.
- Audit log: every Tier 3 activation event has a corresponding approval event with `approver` populated.

---

## S2-04 — SIEM forwarding and weekly review of PIM events

**What this control requires:** Activation, approval, denial, expiry events for every Graph role group are written to SIEM and reviewed weekly.

**Prerequisites:** S1-09, S2-01.

**Implementation steps:**

1. Confirm PIM events are in scope of the Entra audit log diagnostic setting from S1-09 — they are part of AuditLogs.
2. Build a SIEM dashboard or scheduled report covering, for the previous 7 days: activations by user/group, denied activations, approval response times, activations outside business hours.
3. Cyber Security analyst reviews the dashboard weekly. Each review is recorded in the team's runbook log with name of reviewer, date, and any issues identified.

**Portal:**
- Entra → Monitoring → Audit logs → filter by Category = RoleManagement, Service = PIM.

**Verification:**
- Weekly review log exists with at least 50 of the previous 52 weeks completed (allowing for leave).
- Sample of activation events present in SIEM within ingestion SLA.

---

## S2-05 — Six-monthly Graph permission review against actual use

**What this control requires:** Permissions assigned to each role group are reviewed against actual use every six months; permissions used in fewer than 5% of activations are removed unless justified.

**Prerequisites:** S1-09, S2-01, S2-04.

**Implementation steps:**

1. For each role group, list the Graph permissions the group confers (derived from the Entra roles it holds).
2. For the previous six months, query the Graph API call audit data (via the SIEM, or via Entra ID workbooks if MS Graph activity logs are enabled) and compute, per permission, the percentage of activations during which that permission was actually exercised.
3. For any permission below the 5% threshold, schedule a removal change. The role family owner may submit a written justification to retain (e.g. permission is rarely used but is critical for incident response).
4. Removal is implemented by changing the Entra role assigned to the group (e.g. switching from a more permissive built-in role to a narrower one, or a custom role), or by splitting the group into a separate tier-specific group.
5. Record the review outcome in the role group register (S1-06).

**Portal:**
- Entra → Monitoring → Workbooks → Microsoft Graph activity logs (requires Graph activity logs to be enabled).

**Graph PowerShell:**
```powershell
# Enable Microsoft Graph activity logs (one-time, via diagnostic settings — not Graph PowerShell)
# Then query via Log Analytics:
#   MicrosoftGraphActivityLogs
#   | where TimeGenerated > ago(180d)
#   | where UserPrincipalName startswith "adm-"
#   | summarize calls=count() by RequestUri, ResponsePermissions
```

> Microsoft Graph activity logs (the per-call API audit) is a separate, opt-in log stream from the standard sign-in and audit logs. Enable it as part of S1-09 if not already; it is required for the analysis in this control.

**Verification:**
- Review record for each role group dated within the last 6 months. Each record shows the permissions analysed, usage data, and decisions taken.

---

## S2-06 — Annual revalidation of admin accounts and group memberships

**What this control requires:** Admin accounts and Graph role group memberships are revalidated at least annually via an Entra Access Review. Revalidations recorded and tracked to closure.

**Prerequisites:** S1-05, S2-01.

**Implementation steps:**

1. Configure an Entra Access Review for each role-assignable group, recurring annually.
2. Reviewer: group owner (from the register, S1-06) is the default; manager of the reviewed user is also acceptable.
3. Settings: reviewer must provide justification; auto-apply results; "if reviewer doesn't respond" set to "remove access" (this is the stronger setting and matches the ML2 expectation of revoking unused privilege).
4. Run a corresponding review for the membership of the `sg-admin-accounts` group — i.e. "does this person still need an admin account at all?"
5. Track open reviews to closure. Any review not completed in 30 days escalates to the group owner's manager.

**Portal:**
- Entra → Identity Governance → Access reviews → New access review → choose Groups, select target groups, configure as above.

**Graph PowerShell:**
```powershell
Connect-MgGraph -Scopes "AccessReview.ReadWrite.All"
# Inspect existing reviews
Get-MgIdentityGovernanceAccessReviewDefinition -All | Select DisplayName,Status,Scope
```

**Verification:**
- For every role-assignable group: an Access Review schedule exists, last instance completed within the last 13 months, and the completion record is on file.
- Aggregate report: "% of groups with current revalidation" should be 100%.

---

## S2-07 — 45-day inactivity disable of admin accounts

**What this control requires:** Admin accounts not signed in for 45 days are automatically disabled.

**Prerequisites:** S1-01, S1-09.

**Implementation steps:**

1. Build a scheduled job (Azure Automation runbook, Logic App, or equivalent) that runs daily and:
   - Queries `signInActivity.lastSignInDateTime` for every member of `sg-admin-accounts`.
   - Identifies accounts where the most recent interactive sign-in is more than 45 days ago.
   - Disables those accounts (`Update-MgUser -AccountEnabled:$false`).
   - Records the action in the SIEM and notifies the account holder's manager.
2. Re-enablement requires a service desk ticket and manager attestation. Build a runbook for this.
3. Exclude the two break-glass accounts from the inactivity job — they are expected to be inactive and are protected by other means (S1-04).

**Portal:**
- Manual fallback: Entra → Identity → Users → filter on last sign-in > 45 days, members of `sg-admin-accounts`.

**Graph PowerShell:**
```powershell
Connect-MgGraph -Scopes "AuditLog.Read.All","User.ReadWrite.All"
$cutoff = (Get-Date).AddDays(-45)
$members = Get-MgGroupMember -GroupId "<sg-admin-accounts-id>" -All
foreach ($m in $members) {
    if ($m.Id -in $breakGlassIds) { continue }
    $u = Get-MgUser -UserId $m.Id -Property SignInActivity,UserPrincipalName,AccountEnabled
    if ($u.SignInActivity.LastSignInDateTime -lt $cutoff -and $u.AccountEnabled) {
        Update-MgUser -UserId $u.Id -AccountEnabled:$false
        # log to SIEM via custom event…
    }
}
```

**Verification:**
- Audit log shows the automation has run daily for the last 30 days.
- Sample inactive account: confirm it was disabled within 1 day of crossing the 45-day threshold.

---

## S2-08 — Annual review of app registrations with Graph permissions

**What this control requires:** Every app registration with Graph permissions is reviewed annually by its named business and technical owners.

**Prerequisites:** S1-07 (inventory exists).

**Implementation steps:**

1. For each app registration in the inventory, set a review date in the CMDB.
2. Build a workflow (Power Automate, ServiceNow, or equivalent) that:
   - 30 days before review date: notifies the business and technical owner with a review form.
   - The form asks: is the app still in use, are the permissions still required, are the credentials still in use, who is the current owner.
   - On completion: updates the CMDB, schedules next review.
   - On non-completion within 30 days of review date: notifies Cyber Security and disables the app within 5 business days.
3. Disabled registrations are retained for 30 days then deleted. Owners can request reinstatement during that window with a justification.

**Portal:**
- Entra → Identity → Applications → App registrations → select app → set custom security attribute or tag for `ReviewDate`, `BusinessOwner`, `TechnicalOwner`.

**Graph PowerShell:**
```powershell
# Disable an app registration's service principal
Update-MgServicePrincipal -ServicePrincipalId "<id>" -AccountEnabled:$false
# Delete after retention
Remove-MgApplication -ApplicationId "<id>"
```

**Verification:**
- Inventory report shows every app has `LastReviewed` within the last 13 months.
- Workflow run history shows escalations triggered for overdue reviews.

---

## S2-09 — Application permissions justified vs delegated

**What this control requires:** Application permissions only granted where delegated cannot meet the requirement; each grant documented with rationale.

**Prerequisites:** S1-07, S1-08.

**Implementation steps:**

1. Update the app registration request form (in the ITSM) to require, for each requested permission, an answer to: "Why can this not be satisfied by a delegated permission running under a service account?"
2. The IDAM — Privileged team reviews each request. Acceptable rationales:
   - No interactive user is present (true daemon).
   - Permission is not available as delegated (e.g. some Graph endpoints are app-only).
   - Volume is incompatible with delegated throttling.
3. The rationale is recorded against the app registration in the CMDB and is reviewed annually under S2-08.
4. Existing app registrations with application permissions are retro-reviewed within 6 months of this control going live.

**Verification:**
- Sample: pick 10 app registrations with application permissions; for each, the CMDB record contains the rationale text.

---

## S2-10 — Credential lifetime, rotation, and storage

**What this control requires:** Client secrets ≤ 180 days lifetime, rotated automatically where possible, stored in approved vault. Certificates or federated credentials mandated for new app registrations.

**Prerequisites:** S1-07, an approved secret management platform (Azure Key Vault or equivalent).

**Implementation steps:**

1. For new app registrations: refuse client secret credentials in the standard workflow. Approved credential types in order: managed identity → federated credential → certificate. Document any exception per S2-09 logic.
2. For existing app registrations: inventory existing client secrets and their expiry. Anything > 180 days lifetime is non-compliant — replace at next rotation.
3. Where the consuming workload supports it (e.g. apps deployed to Azure Functions reading Key Vault), wire up automated rotation:
   - Key Vault stores the secret reference.
   - An automation rotates the secret on the app registration and updates Key Vault.
   - The consuming workload reads the new value at next call (or restarts).
4. Where automated rotation isn't feasible, document the manual rotation procedure and schedule it.

**Portal:**
- Entra → App registrations → app → Certificates & secrets → review expiry, add federated credentials.

**Graph PowerShell:**
```powershell
# Inventory existing client secrets
Get-MgApplication -All | ForEach-Object {
    $app = $_
    foreach ($pwd in $app.PasswordCredentials) {
        $lifetime = ($pwd.EndDateTime - $pwd.StartDateTime).Days
        if ($lifetime -gt 180) {
            [pscustomobject]@{
                App = $app.DisplayName
                AppId = $app.AppId
                SecretId = $pwd.KeyId
                LifetimeDays = $lifetime
                Expires = $pwd.EndDateTime
            }
        }
    }
} | Export-Csv overlong-secrets.csv -NoTypeInformation

# Add a federated credential (workload identity federation) for a GitHub Actions pipeline
$fed = @{
    name        = "github-actions-deploy"
    issuer      = "https://token.actions.githubusercontent.com"
    subject     = "repo:contoso/myrepo:ref:refs/heads/main"
    audiences   = @("api://AzureADTokenExchange")
    description = "GitHub Actions pipeline for myrepo main branch"
}
New-MgApplicationFederatedIdentityCredential -ApplicationId "<app-object-id>" -BodyParameter $fed
```

**Verification:**
- `overlong-secrets.csv` produced quarterly is empty (or contains only documented exceptions).
- Sample new app registration created in the last quarter uses certificate or federated credential.

---

## S2-11 — PAW/AVD enforcement for Graph admin sessions

**What this control requires:** Administrative Graph use occurs from PAW or AVD admin environment. Other endpoints blocked by Conditional Access.

**Prerequisites:** S1-01, S1-03, PAW or AVD admin environment deployed.

**Implementation steps:**

1. Tag PAW endpoints and AVD admin session hosts in Intune with a custom device attribute (e.g. `deviceCategory = "PAW"`), or assert compliance via a dedicated compliance policy.
2. Create a Conditional Access policy:
   - Users: `sg-admin-accounts`
   - Cloud apps: Microsoft Graph, Microsoft Azure Management, Microsoft Admin Portals
   - Conditions → Device platforms: All
   - Conditions → Device filter: include only devices matching the PAW tag OR compliance policy
   - Grant: Require compliant device AND require the phishing-resistant authentication strength
3. Run in report-only for 1–2 weeks; expect to see attempted admin sign-ins from standard devices appear — these are the cases the policy will block once enforced.
4. Coordinate cut-over with team leads. Have a fallback documented (break-glass via S1-04 if PAW environment is unavailable).

**Portal:**
- Entra → Conditional Access → New policy → as above.

**Verification:**
- Sign-in log query: every successful sign-in by an admin account to Microsoft Graph in the last 30 days originates from a compliant PAW device. Exceptions documented under the exceptions register (S10).

---

## S2-12 — SIEM detections and runbooks for Graph-relevant events

**What this control requires:** Audit events for PIM activation, directory role assignment, app registration permission change, consent grant generate SIEM detections with documented runbooks.

**Prerequisites:** S1-09.

**Implementation steps:**

1. Define detection rules in the SIEM for each event class:
   - PIM activation (especially Tier 3, outside business hours, by users with no recent activations)
   - Direct directory role assignment to a user (i.e. bypassing the group model — should not happen post-S1-05)
   - Application permission added to any app registration
   - Admin consent granted on behalf of the tenant
   - New app registration created
   - New federated credential added to existing app registration
   - Service principal credential rolled to an unexpected location (e.g. new client secret created on a federated-credential-only app)
2. For each detection, write a response runbook: triage steps, expected legitimate cases, escalation path, containment actions.
3. Tune over the first 30 days to reduce false positives.

**Verification:**
- Each detection rule has an associated runbook document with last-reviewed date within 12 months.
- Tabletop exercise quarterly on at least one of these scenarios.

---

## S2-13 — Near-real-time alert on high-blast-radius scopes

**What this control requires:** Use of `Directory.ReadWrite.All`, `RoleManagement.ReadWrite.Directory`, `Application.ReadWrite.All`, `AppRoleAssignment.ReadWrite.All` generates an alert with documented triage SLA.

**Prerequisites:** S1-09, Microsoft Graph activity logs enabled (see note under S2-05).

**Implementation steps:**

1. Build a SIEM detection on the Microsoft Graph activity log stream filtered to API calls where the call was authorised by one of the listed permissions (the `ResponsePermissions` field on the activity log).
2. Triage SLA: 30 minutes during business hours, 4 hours otherwise. Define on-call rotation.
3. Suppress known-good identities (e.g. specific managed identities that perform legitimate bulk operations) via an allowlist with documented justification, reviewed quarterly.
4. Update Appendix C of the standard whenever the list of high-risk scopes changes.

**Verification:**
- A test call using one of the listed permissions generates an alert within 5 minutes.
- Quarterly review of the allowlist is on file.

---

## S2-14 — Joiner/mover/leaver removes Graph eligibility within one business day

**What this control requires:** Documented JML process removes Graph role group eligibility within 1 business day of role change or termination.

**Prerequisites:** S1-05, S2-01.

**Implementation steps:**

1. Integrate the agency's HR system (or equivalent system of record) with the IDAM provisioning workflow such that role-change and termination events feed into a daily reconciliation job.
2. The job:
   - Identifies users whose role no longer maps to their current Graph role group eligibilities.
   - Removes PIM eligible assignments for the groups they should no longer be eligible for.
   - Disables the admin account on termination (and removes from `sg-admin-accounts`).
   - Records the action and notifies the manager and IDAM team.
3. For high-risk leavers (privileged role holders, terminated for cause), implement an immediate manual procedure that bypasses the daily job — Cyber Security operations can remove eligibility within minutes via PIM admin actions.
4. Validate quarterly: pick 5 leavers from the last quarter and confirm eligibility was removed within the SLA.

**Graph PowerShell:**
```powershell
# Remove a PIM eligible group assignment
$params = @{
    accessId      = "member"
    principalId   = "<user-id>"
    groupId       = "<group-id>"
    action        = "AdminRemove"
    justification = "Leaver — terminated DD-MM-YYYY"
}
New-MgIdentityGovernancePrivilegedAccessGroupEligibilityScheduleRequest -BodyParameter $params
```

**Verification:**
- Quarterly leaver sample: 100% had eligibility removed within 1 business day.
- Job run history shows daily reconciliation has not missed > 2 days in any quarter.

---

# Stage 3 — Forward state (Essential Eight ML3)

Stage 3 is documented for forward planning. Each control below assumes the corresponding Stage 1 and Stage 2 controls are operating. These are non-trivial uplifts and should be planned as named projects rather than business-as-usual tasks.

---

## S3-01 — Behavioural baselining and session revocation

**What this control requires:** Per-identity behavioural baseline; anomalous Graph activity triggers session revocation.

**Prerequisites:** S2-12, S2-13, Microsoft Graph activity logs enabled, an analytics tooling capable of per-identity baselining (Sentinel Behavioural Analytics, a UEBA, or custom on top of Log Analytics).

**Implementation steps:**

1. Establish per-identity baseline over a 30–60 day learning window: typical Graph endpoints called, typical call volume, typical sessions per day, typical source IPs.
2. Build detections on departures from baseline: new endpoint called by this identity, > 3× standard deviation in volume, call from a new geography.
3. Wire detections to automated response: revoke the user's refresh tokens via `Revoke-MgUserSignInSession`, force re-authentication, page on-call.
4. Tune thresholds to keep false positives manageable. Plan for a high false-positive rate at the start; this control is operationally expensive.

**Graph PowerShell:**
```powershell
# Revoke all refresh tokens for a user (forces re-auth on next call)
Revoke-MgUserSignInSession -UserId "<user-id>"
```

**Verification:**
- Detection produces an alert within minutes of an injected anomalous call.
- Quarterly review of false positive rate and tuning actions.

---

## S3-02 — Per-action approval for highest-risk operations

**What this control requires:** Per-action approval workflows for highest-risk Graph operations, independent of standing PIM activation.

**Prerequisites:** S2-02, S2-03, an automation/orchestration platform (e.g. ServiceNow with workflow, Power Platform, custom).

**Implementation steps:**

1. Identify the operations that warrant per-action approval beyond PIM activation. Typical list: creating an app registration, granting admin consent, assigning a directory role to a user (not via a group, which is already prohibited per S1-05), modifying a Conditional Access policy.
2. Block these operations in the portal/Graph by removing the underlying permission from standing roles. Provide an alternative path: an automation that, on approval, performs the operation under a system identity with the elevated permission, and logs the actor, approver, and operation.
3. Each approval is recorded against the requesting actor.

**Verification:**
- Attempt to perform one of the listed operations directly: fails with a permission error.
- Same operation via the approval workflow succeeds and produces an audit record.

---

## S3-03 — Declarative app registration permissions (IaC)

**What this control requires:** App registration permissions managed declaratively in IaC with peer review and drift detection.

**Prerequisites:** S2-08, S2-09, S2-10.

**Implementation steps:**

1. Adopt an IaC tool capable of representing app registrations and their permissions (Terraform with the AzureAD provider, Bicep, or a custom GitOps approach).
2. Migrate existing app registrations into the IaC repository. Each change is reviewed via pull request before being applied by the pipeline.
3. Build a drift detector that compares the declared state in the repository against the actual state in Entra daily. Any drift generates an alert; manual changes in the portal are reverted within a defined SLA (or merged into the repository if the manual change is later determined to be the desired state).

**Verification:**
- All app registrations represented in the repo.
- Drift detector run history shows zero unresolved drift older than 1 business day.

---

## S3-04 — Reduced token lifetimes and continuous access evaluation

**What this control requires:** Conditional Access sign-in frequency ≤ 4 hours for admin sessions; CAE enforced for under-15-minute revocation propagation.

**Prerequisites:** S2-11.

**Implementation steps:**

1. Conditional Access policy targeting `sg-admin-accounts` and Microsoft Graph / Azure Management / Admin Portals: session controls → sign-in frequency = 4 hours, persistent browser session = never.
2. Confirm Continuous Access Evaluation is enabled (it is by default on most modern tenants; verify and document).
3. Test revocation propagation: revoke a token, attempt a Graph call from an existing session, confirm it fails within 15 minutes.

**Verification:**
- CA policy export shows the sign-in frequency setting.
- CAE test result recorded.

---

## S3-05 — Eliminate client secrets for non-human Graph identities

**What this control requires:** All non-human Graph identities use managed identities or federated credentials; client secrets prohibited for new workloads, phased out for existing.

**Prerequisites:** S2-10.

**Implementation steps:**

1. Block new client secret creation on app registrations via a policy/automation. Where the team has no control over secret creation, build a detection that flags creation events and either auto-removes them or pages.
2. Migrate existing client-secret-authenticated apps. Order of preference: managed identity (if hosted in Azure) → federated credential (if hosted in a federation-compatible platform like GitHub or Azure DevOps) → certificate as fallback.
3. The agency's reporting includes a "% of app registrations with client secrets" metric, decreasing monthly toward zero.

**Verification:**
- Quarterly report shows the metric.
- Zero new client secrets created in the last quarter for app registrations subject to this control.

---

# Appendix — Quick-reference matrix

| Control | Stage | Prerequisites |
|---|---|---|
| S1-01 | 1 | — |
| S1-02 | 1 | S1-01 |
| S1-03 | 1 | S1-01 |
| S1-04 | 1 | — |
| S1-05 | 1 | S1-01 |
| S1-06 | 1 | S1-05 |
| S1-07 | 1 | — |
| S1-08 | 1 | — |
| S1-09 | 1 | — |
| S1-10 | 1 | S1-01, S1-02 |
| S2-01 | 2 | S1-05, S1-06 (+ Entra ID P2) |
| S2-02 | 2 | S2-01 |
| S2-03 | 2 | S2-02 |
| S2-04 | 2 | S1-09, S2-01 |
| S2-05 | 2 | S1-09, S2-01, S2-04 |
| S2-06 | 2 | S1-05, S2-01 |
| S2-07 | 2 | S1-01, S1-09 |
| S2-08 | 2 | S1-07 |
| S2-09 | 2 | S1-07, S1-08 |
| S2-10 | 2 | S1-07 (+ vault platform) |
| S2-11 | 2 | S1-01, S1-03 (+ PAW/AVD) |
| S2-12 | 2 | S1-09 |
| S2-13 | 2 | S1-09 (+ Graph activity logs) |
| S2-14 | 2 | S1-05, S2-01 |
| S3-01 | 3 | S2-12, S2-13 (+ UEBA) |
| S3-02 | 3 | S2-02, S2-03 (+ orchestration) |
| S3-03 | 3 | S2-08, S2-09, S2-10 (+ IaC tooling) |
| S3-04 | 3 | S2-11 |
| S3-05 | 3 | S2-10 |
