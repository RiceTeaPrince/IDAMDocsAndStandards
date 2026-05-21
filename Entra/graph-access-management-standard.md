# Microsoft Graph API Access Management Standard

**Document type:** Technical security standard
**Audience:** ICT, Cyber Security, Identity & Access Management (IDAM), Cloud Platform teams
**Aligns to:** ASD Essential Eight (Restrict Administrative Privileges), ASD Information Security Manual (ISM)
**Target maturity:** Essential Eight Maturity Level 2 (ML2)

---

## 1. Purpose

This standard defines how human and non-human identities are granted, used, audited, and removed when interacting with the Microsoft Graph API. It exists because Graph is the primary administrative control plane for the agency's Microsoft 365 and Entra ID tenants, and an over-permissioned Graph identity is functionally equivalent to a Global Administrator.

The standard is written to be implementable in stages. Each stage corresponds to an Essential Eight maturity level for the **Restrict Administrative Privileges** mitigation strategy, mapped through to specific Graph controls.

## 2. Scope

This standard applies to:

- All interactive use of Microsoft Graph (PowerShell, Graph Explorer, Graph CLI, ad hoc scripts, ISE/VS Code sessions, browser-based admin portals that call Graph on the user's behalf).
- All non-interactive use of Microsoft Graph (app registrations, managed identities, daemon services, scheduled automation, CI/CD pipelines).
- All workforce personas who require Graph access, including but not limited to: M365 platform engineers, IDAM engineers, security operations, service desk, automation engineers, and external auditors.

This standard does **not** cover end-user consumption of Microsoft 365 applications (Outlook, Teams, SharePoint), which call Graph implicitly under the user's own delegated context with standard user scopes.

## 3. Principles

The standard is built on six principles. Every control below is justified against at least one of these.

1. **Least privilege by scope, not by role.** Access is granted to the smallest set of Graph permissions that satisfies the task, not the broadest role that contains them.
2. **Delegated over application.** Where a human is performing the action, the action is performed under that human's identity using delegated permissions. Application permissions are an exception requiring justification.
3. **Just-in-time, not standing.** Privileged Graph access is activated for a bounded window and revoked automatically, not granted permanently.
4. **Identifiable actor.** Every Graph action must be attributable to a named human or a named workload identity that has a named human owner.
5. **Defence in depth on credentials.** Service principals are protected by certificate or workload-identity-federation credentials in preference to client secrets, and secrets where unavoidable are short-lived and stored in a managed vault.
6. **Continuous revalidation.** Access that is not used is access that should not exist. Inactivity, role change, and termination are treated as signals to remove access.

## 4. Role model

Graph access is granted via **role-assignable Entra ID security groups**, never directly to user objects, and never directly via Graph API permission assignment to a user. Groups are PIM-enabled where the privilege warrants it.

The agency operates four standing functional role families. Each family has multiple tiered groups so that a member only ever holds the tier they currently need.

| Role family | Purpose | Example Graph scopes (delegated) | Typical Entra role(s) backing it |
|---|---|---|---|
| **M365 Collaboration** | Provisioning and lifecycle of Teams, SharePoint sites, M365 Groups, Planner | `Group.ReadWrite.All`, `Sites.FullControl.All`, `TeamSettings.ReadWrite.All`, `Channel.Create` | Teams Administrator, SharePoint Administrator |
| **IDAM — User Lifecycle** | Create, update, disable user accounts; assign licences; reset credentials for non-privileged users | `User.ReadWrite.All`, `Directory.ReadWrite.All` (tiered), `UserAuthenticationMethod.ReadWrite.All` | User Administrator, Authentication Administrator |
| **IDAM — Access Governance** | Group membership management, access package configuration, access reviews | `Group.ReadWrite.All`, `EntitlementManagement.ReadWrite.All`, `AccessReview.ReadWrite.All` | Identity Governance Administrator, Groups Administrator |
| **IDAM — Privileged** | Management of privileged roles, conditional access, app registrations, directory-wide configuration | `RoleManagement.ReadWrite.Directory`, `Policy.ReadWrite.ConditionalAccess`, `Application.ReadWrite.All` | Privileged Role Administrator, Conditional Access Administrator, Application Administrator |

Each role family is split into **Tier 1 (read)**, **Tier 2 (read/write for routine operations)**, and **Tier 3 (privileged write)** groups, e.g. `sg-graph-idam-userlifecycle-read`, `sg-graph-idam-userlifecycle-write`, `sg-graph-idam-userlifecycle-privileged`. Membership of Tier 2 and Tier 3 is always PIM-eligible, never PIM-active by default.

The IDAM — Privileged family is the only one that may be backed by Entra ID directory roles that include `RoleManagement.ReadWrite.Directory` or equivalent. It is treated as a Tier 0 control-plane role for the purposes of administrative tiering.

## 5. Identity types and when each is permitted

| Identity type | Use case | Permitted? |
|---|---|---|
| Named admin account (`adm-firstname.lastname`) with delegated Graph permissions via PIM-activated group | Default for all interactive Graph use | **Required** |
| Shared admin account | Any use | **Prohibited** |
| User's standard productivity account with admin Graph scopes | Any administrative Graph use | **Prohibited** |
| Managed identity (system- or user-assigned) | Azure-hosted automation calling Graph | **Preferred** for automation |
| App registration with certificate credential | Automation outside Azure, or where managed identity is not available | **Permitted with justification** |
| App registration with client secret | Automation where certificate is not feasible | **Permitted only as exception**, secret lifetime ≤ 180 days, stored in approved vault |
| App registration with federated credential (workload identity federation) | CI/CD pipelines (GitHub Actions, Azure DevOps) calling Graph | **Preferred** for pipelines |

## 6. Maturity stages

The standard defines three progressive stages. **Stage 2 is the agency's target state** and corresponds to Essential Eight ML2. Stage 3 is documented for forward planning but is not currently mandated.

A control is considered "met" only when it is both implemented technically and evidenced operationally (runbook, ticket history, configuration export, or logged report).

### 6.1 Stage 1 — Baseline (Essential Eight ML1)

**Outcome:** Privileged Graph access is identifiable, separated from standard user identity, and gated by MFA. No standing Global Administrator equivalence outside the break-glass account set.

| # | Control | ISM / E8 alignment |
|---|---|---|
| S1-01 | All interactive Graph administration is performed from named admin accounts (`adm-*`). Shared accounts are prohibited. | E8 RAP ML1; ISM-1175, ISM-1247 |
| S1-02 | Admin accounts are separated from the user's standard productivity account, with distinct UPN and credential. | E8 RAP ML1; ISM-1175 |
| S1-03 | All admin accounts are enforced with phishing-resistant MFA (FIDO2 or Windows Hello for Business) via Conditional Access. | E8 MFA ML1+; ISM-1504, ISM-1679 |
| S1-04 | Two break-glass Global Administrator accounts exist, are excluded from Conditional Access blocking policies, use unique strong passphrases stored in a sealed vault, and are alerted on sign-in. | ISM-1685, ISM-1387 |
| S1-05 | Graph permissions for each role family are granted to **role-assignable security groups**, not to individual user objects. | E8 RAP ML1; ISM-1175 |
| S1-06 | A documented register exists listing each role family, the Graph permissions it grants, the Entra roles it confers, and the named owner of the group. | ISM-1525 |
| S1-07 | App registrations are inventoried. Each has a documented business owner, a technical owner, and a stated purpose. Orphaned app registrations (no owner) are blocked from acquiring new permissions. | ISM-1525, ISM-1873 |
| S1-08 | Tenant-wide user consent for applications is restricted to verified publishers and low-risk permissions only. Admin consent workflow is enabled. | ISM-1873 |
| S1-09 | Unified Audit Log is enabled at the tenant level, and Entra ID sign-in and audit logs are forwarded to the agency SIEM with a minimum retention of 12 months. | E8 RAP ML1; ISM-0859, ISM-1537 |
| S1-10 | Admin accounts cannot access the public internet, web mail, or non-administrative web services from the same session used for Graph administration. | E8 RAP ML1; ISM-1380 |

### 6.2 Stage 2 — Target state (Essential Eight ML2)

**Outcome:** Privileged Graph access is just-in-time, scoped to the minimum permissions required for the task, time-bounded, justified, approved where appropriate, and revalidated on a schedule. Service principals are governed equivalently to humans. **This is the agency's mandated state.**

Stage 2 includes everything in Stage 1, plus:

| # | Control | ISM / E8 alignment |
|---|---|---|
| S2-01 | All Tier 2 and Tier 3 Graph role groups are PIM-eligible. No human holds standing membership of a write or privileged group. | E8 RAP ML2; ISM-1175, ISM-1647 |
| S2-02 | PIM activation requires: justification text, ticket reference, MFA re-authentication at activation, and a maximum activation window of 8 hours. | E8 RAP ML2; ISM-1647, ISM-1504 |
| S2-03 | Tier 3 (privileged) activation additionally requires approval from a second nominated approver from a different team. | E8 RAP ML2; ISM-1647 |
| S2-04 | Activation, approval, denial, and expiry events for every Graph-relevant role group are written to the SIEM and reviewed weekly by Cyber Security. | E8 RAP ML2; ISM-0859, ISM-1228 |
| S2-05 | Graph permissions assigned to each role group are reviewed against actual use every six months. Permissions used in fewer than 5% of activations are removed unless justified in writing. | E8 RAP ML2; ISM-1525 |
| S2-06 | Admin accounts and Graph role group memberships are revalidated at least annually via an Entra ID Access Review. Revalidations are recorded and tracked to closure. | E8 RAP ML2; ISM-1647, ISM-1525 |
| S2-07 | Admin accounts that have not signed in for 45 days are automatically disabled. Re-enablement requires a ticket and manager attestation. | E8 RAP ML2; ISM-1648 |
| S2-08 | App registrations follow the same lifecycle: every app registration with Graph permissions is reviewed at least annually by its named business and technical owners. Unowned or unused registrations are disabled then deleted. | E8 RAP ML2; ISM-1525, ISM-1873 |
| S2-09 | Application permissions are only granted where delegated permissions cannot meet the requirement. Each grant of an application permission is documented with the reason delegated was not viable. | ISM-1873 |
| S2-10 | Client secrets on app registrations have a maximum lifetime of 180 days, are rotated automatically where the consuming workload supports it, and are stored in Azure Key Vault or the agency's approved secret management platform. Certificate or federated credentials are mandated for any new app registration. | ISM-1873, ISM-1417 |
| S2-11 | All administrative Graph use occurs from a privileged access workstation (PAW) or Azure Virtual Desktop (AVD) administrative environment. Graph admin sessions originating from standard user endpoints are blocked by Conditional Access (device filter on `deviceTrustType` and compliance state). | E8 RAP ML2; ISM-1380, ISM-1657 |
| S2-12 | Graph-relevant audit events (PIM activation, directory role assignment, app registration permission change, consent grant) generate SIEM detections with documented response runbooks. | E8 RAP ML2; ISM-1228, ISM-0859 |
| S2-13 | Use of high-blast-radius Graph scopes (`Directory.ReadWrite.All`, `RoleManagement.ReadWrite.Directory`, `Application.ReadWrite.All`, `AppRoleAssignment.ReadWrite.All`) generates a near-real-time alert with required triage SLA. | ISM-1228, ISM-0859 |
| S2-14 | A documented joiner/mover/leaver process exists that removes Graph role group eligibility within one business day of a role change or termination. | E8 RAP ML2; ISM-0430, ISM-1647 |

### 6.3 Stage 3 — Forward state (Essential Eight ML3)

**Outcome:** Stage 2, plus continuous monitoring, automated revocation on signal, and demonstrable least privilege at the per-action level. Documented here for forward planning. **Not currently mandated.**

| # | Control | ISM / E8 alignment |
|---|---|---|
| S3-01 | Graph activity is continuously analysed against a per-identity behavioural baseline. Anomalous Graph calls (new endpoint, new tenant target, atypical volume) trigger session revocation. | E8 RAP ML3; ISM-1228 |
| S3-02 | Per-action approval workflows are in place for the highest-risk Graph operations (e.g. creating an app registration, granting admin consent, assigning a directory role), independent of any standing PIM activation. | E8 RAP ML3 |
| S3-03 | App registration permissions are managed declaratively (Infrastructure as Code) with peer review and automated drift detection. Manual permission changes in the portal are alerted and reversed. | ISM-1525, ISM-1873 |
| S3-04 | Token lifetimes for Graph admin sessions are reduced (Conditional Access sign-in frequency ≤ 4 hours) and continuous access evaluation is enforced for revocation propagation in under 15 minutes. | E8 RAP ML3; ISM-1504 |
| S3-05 | All non-human Graph identities are managed identities or federated credentials; client secrets are prohibited for new workloads and being phased out for existing. | ISM-1873, ISM-1417 |

## 7. Per-team implementation patterns

The following section translates the role model into team-specific patterns. Each pattern is the **expected default**; deviations require documented exception.

### 7.1 M365 Collaboration team

The team's day-to-day work — creating Teams, provisioning SharePoint sites, managing M365 group membership — is performed under **delegated permissions** through PIM-activated group membership.

- Routine work (provisioning a Team, adding a channel, updating site settings) is performed under `sg-graph-m365-collab-write`, activated through PIM with an 8-hour maximum window and a ticket reference.
- Read-only investigation (looking up site ownership, group properties) is performed under `sg-graph-m365-collab-read`, which is also PIM-eligible but with auto-approval and no second approver required.
- The team does **not** need `Sites.FullControl.All` for routine provisioning; `Sites.Manage.All` or `Sites.ReadWrite.All` is sufficient and is the default permission on the write group.
- Bulk provisioning that exceeds Graph throttling limits when performed interactively is executed through a named **app registration** (`app-m365-bulk-provisioning`) with application permissions, run from a controlled jump host. The app registration is owned by the team lead and reviewed annually.

### 7.2 IDAM — User Lifecycle

- Creating, updating, and disabling users is performed under `sg-graph-idam-userlifecycle-write`, PIM-activated, with `User.ReadWrite.All` and `Directory.ReadWrite.All` as the bound scopes.
- Authentication method management (resetting MFA, registering a security key on behalf of a user) is gated separately through `sg-graph-idam-authmethods-write` which carries `UserAuthenticationMethod.ReadWrite.All`. This is a Tier 3 activation requiring secondary approval, because it can be used to bypass MFA on any non-privileged user.
- License assignment is performed under the same write group but is the preferred candidate for automation: the team's HR-driven joiner workflow uses a managed identity (`mi-hr-joiner-automation`) with `User.ReadWrite.All` application permission. The managed identity's activity is reviewed monthly against the HR feed.

### 7.3 IDAM — Access Governance

- Group membership changes for non-privileged groups are routine Tier 2 activity under `sg-graph-idam-accessgov-write`.
- **Membership changes for any role-assignable group** (i.e. groups that confer Graph or Entra role) are treated as Tier 3 and require secondary approval. This is enforced both procedurally and through PIM approval configuration on those specific groups.
- Access package configuration is performed under `sg-graph-idam-accessgov-write` with `EntitlementManagement.ReadWrite.All`. Production access package changes go through change management, not directly through Graph.
- Access reviews are configured under the same group; the reviews themselves run autonomously once configured.

### 7.4 IDAM — Privileged

- This is the agency's Tier 0 control-plane group. Membership is restricted to four named individuals and one delegated alternate.
- Activation requires: justification, ticket reference, secondary approval from outside the team (Cyber Security manager or CISO delegate), and a maximum window of 4 hours.
- All activations of this group generate a high-severity SIEM alert with mandatory after-the-fact review, regardless of whether the activity itself was anomalous.
- Members of this group must not also be members of the M365 Collaboration or IDAM — User Lifecycle write groups. Separation of duties is enforced through Entra ID role group conflict configuration.

## 8. App registration and service principal governance

For every app registration that calls Graph:

1. A registration record is created in the IT service management platform capturing: business owner (named human), technical owner (named human), purpose, requested permissions with justification per permission, data classification of the data the app will access, and review date.
2. Permissions are requested in writing and approved by the IDAM — Privileged team. The approver verifies that delegated permissions cannot meet the requirement before approving application permissions.
3. Credentials are issued in the following order of preference: managed identity → federated credential → certificate → client secret. The rationale for using a less-preferred type is documented.
4. The app registration is tagged in Entra ID with its owner email and review date, enabling automated reporting of upcoming reviews and orphaned registrations.
5. Annual review is the responsibility of the business and technical owners. The review attests that the app still exists, the permissions are still required, and the credentials are still in use. Failed reviews disable the credential within five business days; the registration is deleted after a further 30 days unless reinstated.
6. Permission changes are subject to the same approval as initial issuance. Adding a Graph permission to an existing app registration is **not** a routine change.

## 9. Auditability and attribution

Every Graph action must be attributable. This is achieved at three layers:

- **Identity layer.** No shared accounts; app registrations have named owners; managed identities have a documented owning resource.
- **Activation layer.** PIM activation records who activated which group, when, for how long, with what justification, and (for Tier 3) who approved.
- **Action layer.** Graph calls land in Unified Audit Log and Entra audit log with the actor's UPN (or app ID), the target object, the operation, and the result. These are forwarded to the SIEM and retained for at least 12 months. ISM-aligned retention beyond 12 months follows the agency's records management policy.

Quarterly, the Cyber Security team produces a Graph access attribution report covering: most active admin identities by call volume, most-used high-risk scopes, app registrations that called Graph in the period, and any Graph actions for which actor attribution could not be resolved. The last category is treated as a defect and remediated.

## 10. Exceptions

Exceptions to this standard are raised through the agency's standard exception process and require:

- Documented business justification.
- Compensating controls.
- Time-bound expiry (maximum 12 months, renewable with re-review).
- Approval from the CISO delegate.
- Recording in the exceptions register, which is reviewed quarterly.

Common exception categories and the expected compensating controls:

| Exception | Compensating control |
|---|---|
| Standing membership of a write group (no PIM) | Enhanced logging, daily activity review, time-bound to ≤ 90 days |
| Client secret instead of certificate or federated credential | Secret lifetime ≤ 90 days (not 180), stored in vault, rotation automated, additional alerting on use |
| Application permission where delegated is feasible | Scoped to specific target objects via `Application` resource restrictions where available, additional logging |
| Admin session from non-PAW endpoint | Time-bound, alerted, restricted to read-only scopes only |

## 11. Roles and responsibilities

| Role | Responsibility |
|---|---|
| **Standard owner** (Cyber Security manager) | Maintains this standard, schedules reviews, approves changes. |
| **IDAM — Privileged team** | Operates PIM configuration, approves Tier 3 activations, approves app registration permissions, owns the role group register. |
| **Cyber Security operations** | Monitors Graph SIEM detections, runs quarterly attribution report, investigates anomalies, manages exceptions register. |
| **Role family owners** (M365 Collaboration lead, IDAM leads) | Own membership of their family's groups, run their family's access reviews, attest annually. |
| **App registration business and technical owners** | Run annual app reviews, request permission changes, manage credentials. |
| **All admin account holders** | Use named admin accounts only, use PAW for Graph admin, raise tickets for activation, do not share credentials. |

## 12. Review

This standard is reviewed annually, or sooner if:

- An Essential Eight or ISM control referenced in this document changes.
- A Graph permission scope referenced in this document is deprecated or replaced by Microsoft.
- A security incident involving Graph access reveals a gap in this standard.

---

## Appendix A — Mapping summary

| Stage | Essential Eight RAP maturity | Headline controls |
|---|---|---|
| Stage 1 | ML1 | Named admin accounts, MFA on admin, break-glass, group-based assignment, unified audit logging |
| **Stage 2** | **ML2 (target)** | **PIM JIT activation, secondary approval for Tier 3, annual revalidation, 45-day inactivity disable, PAW enforcement, app registration governance, high-risk scope alerting** |
| Stage 3 | ML3 | Behavioural baselining, per-action approval, IaC permissions, reduced token lifetimes, secret-free service principals |

## Appendix B — Sample group naming convention

```
sg-graph-<family>-<tier>
    family: m365collab | idam-userlifecycle | idam-authmethods |
            idam-accessgov | idam-privileged
    tier:   read | write | privileged
```

Examples:

- `sg-graph-m365collab-write`
- `sg-graph-idam-userlifecycle-write`
- `sg-graph-idam-authmethods-write`
- `sg-graph-idam-accessgov-write`
- `sg-graph-idam-privileged`

## Appendix C — High-risk Graph permissions register

The following delegated and application permissions are treated as high-risk and trigger the alerting requirements in S2-13. The list is reviewed every six months against Microsoft's published least-privilege guidance.

- `Directory.ReadWrite.All`
- `Directory.AccessAsUser.All` (delegated)
- `RoleManagement.ReadWrite.Directory`
- `Application.ReadWrite.All`
- `AppRoleAssignment.ReadWrite.All`
- `Policy.ReadWrite.ConditionalAccess`
- `User.ReadWrite.All` (when held by an application identity)
- `Group.ReadWrite.All` (when held by an application identity that can target role-assignable groups)
- `UserAuthenticationMethod.ReadWrite.All`
- `PrivilegedAccess.ReadWrite.AzureAD`
- `Sites.FullControl.All` (when held by an application identity)
