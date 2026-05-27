# Conditional Access Policy — Spreadsheet Analysis Guide

## Overview

The CA naming convention is structured like a database row written as a string. Every hyphen is a column delimiter — the goal is to make that structure explicit in a spreadsheet, then pivot off those columns to generate insights.

```
CA104-Admins-AttackSurfaceReduction-AllApps-AnyPlatform-BlockFromUntrustedCountries
 [1]    [2]          [3]               [4]       [5]                [6]
```

---

## Step 1 — Export Your Policy Data from Entra

You have two options depending on timing.

**Option A — Now (no Graph module installed):**

1. Navigate to **Entra Portal → Protection → Conditional Access → Policies**
2. Click the **Download** button in the top-right
3. This exports a CSV with policy names and states — enough to start

**Option B — Later (once Microsoft.Graph is installed on RDFarm):**

Run the following PowerShell export from RDFarm. This produces a much richer dataset including inclusions, exclusions, grant controls, and apps — and will produce significantly more insightful analysis.

```powershell
Connect-MgGraph -Scopes "Policy.Read.All", "Directory.Read.All"

$policies = Get-MgIdentityConditionalAccessPolicy -All

$report = foreach ($policy in $policies) {
    [PSCustomObject]@{
        DisplayName     = $policy.DisplayName
        State           = $policy.State
        IncludeUsers    = ($policy.Conditions.Users.IncludeUsers -join ", ")
        IncludeGroups   = ($policy.Conditions.Users.IncludeGroups -join ", ")
        ExcludeUsers    = ($policy.Conditions.Users.ExcludeUsers -join ", ")
        ExcludeGroups   = ($policy.Conditions.Users.ExcludeGroups -join ", ")
        IncludeApps     = ($policy.Conditions.Applications.IncludeApplications -join ", ")
        GrantControls   = ($policy.GrantControls.BuiltInControls -join ", ")
        Operator        = $policy.GrantControls.Operator
    }
}

$report | Export-Csv -Path "C:\Reports\CA_Policy_Inventory.csv" -NoTypeInformation
Write-Host "Exported $($report.Count) policies"
```

> **Recommendation:** If you can wait for the Graph module to be installed via ServiceNow CR, do so. The richer export will save you significant manual effort.

---

## Step 2 — Split the Policy Name into Columns

Once your raw data is in Excel:

1. Paste the policy names into **Column A**
2. Select Column A
3. Go to **Data → Text to Columns → Delimited → Hyphen**
4. Split into columns B onwards

This gives you the following column structure:

| Column | Content | Example |
|---|---|---|
| A | Full policy name (raw) | `CA104-Admins-AttackSurfaceReduction-AllApps-AnyPlatform-Block...` |
| B | CA Number | `CA104` |
| C | Persona | `Admins` |
| D | Policy Type | `AttackSurfaceReduction` |
| E | Resource | `AllApps` |
| F | Platform | `AnyPlatform` |
| G | Grant Control | `BlockFromUntrustedCountries` |
| H | Optional Description | *(seventh segment if present)* |
| I | State | `enabled` / `enabledForReportingButNotEnforced` / `disabled` |

---

## Step 3 — Add the Persona Range Column

Add **Column J — Persona Range** using the following formula. This maps the CA number back to its named range bucket, giving you a clean filterable label rather than just a number.

```excel
=IFS(
  VALUE(MID(B2,3,LEN(B2)))<=99,   "Global",
  VALUE(MID(B2,3,LEN(B2)))<=199,  "Admins",
  VALUE(MID(B2,3,LEN(B2)))<=299,  "Internals",
  VALUE(MID(B2,3,LEN(B2)))<=399,  "Externals",
  VALUE(MID(B2,3,LEN(B2)))<=499,  "Guests",
  VALUE(MID(B2,3,LEN(B2)))<=599,  "Guest Admins",
  VALUE(MID(B2,3,LEN(B2)))<=699,  "M365 Service Accounts",
  VALUE(MID(B2,3,LEN(B2)))<=799,  "Azure Service Accounts",
  VALUE(MID(B2,3,LEN(B2)))<=899,  "Corp Service Accounts",
  VALUE(MID(B2,3,LEN(B2)))<=999,  "Workload Identities",
  TRUE,                             "Developer"
)
```

---

## Step 4 — Add Insight Columns

Add the following calculated columns to make the data directly queryable in PivotTables and slicers.

| Column | Label | Formula |
|---|---|---|
| K | Is Enforced? | `=IF(I2="enabled","Yes","No")` |
| L | Is Report-Only? | `=IF(I2="enabledForReportingButNotEnforced","Yes","No")` |
| M | Is a Block Policy? | `=IF(G2="Block","Yes","No")` |
| N | Requires MFA? | `=IF(ISNUMBER(SEARCH("MFA",G2)),"Yes","No")` |
| O | Targets All Apps? | `=IF(E2="AllApps","Yes","No")` |

---

## Step 5 — Build the Insight Sheets

Create a separate sheet for each of the following PivotTables.

### Sheet 2 — Coverage by Persona

- **Rows:** Persona Range (Column J)
- **Columns:** State (Column I)
- **Values:** Count of policy names

Immediately shows which personas have zero enforced policies — that is your gap list.

### Sheet 3 — Policy Type Distribution

- **Rows:** Policy Type (Column D)
- **Columns:** State (Column I)
- **Values:** Count of policy names

Shows whether entire categories of protection (e.g. IdentityProtection, BaseProtection) are unenforced across the environment.

### Sheet 4 — Grant Control Summary

- **Rows:** Grant Control (Column G)
- **Columns:** Persona Range (Column J)
- **Values:** Count of policy names

Answers the question: which personas have Block policies vs MFA policies vs nothing at all?

### Sheet 5 — Report-Only Candidates

- Filter: State = `enabledForReportingButNotEnforced`
- Sort by: Persona Range (Column J)

This becomes your CAB submission planning list. Work through it top to bottom, reviewing sign-in logs for each policy before submitting to CAB.

---

## The Key Output — Coverage Matrix

The single most important view produced by Sheet 2 is a matrix like this. This is what you take to your manager and to CAB.

```
Persona Range          | Enforced | Report-Only | Disabled | COVERED?
-----------------------|----------|-------------|----------|--------------------
Global                 |    ✓     |      ✓      |          | YES
Admins                 |    ✓     |      ✓      |          | YES
Internals              |    ✓     |      ✓      |          | YES
Externals              |          |      ✓      |          | GAP — report-only only
Guests                 |          |             |    ✓     | GAP — nothing enforced
M365 Service Accounts  |    ✓     |             |          | YES
Azure Service Accounts |          |             |          | GAP — no policies at all
Corp Service Accounts  |          |      ✓      |          | GAP — report-only only
Workload Identities    |          |             |          | GAP — no policies at all
Developer              |    ✓     |             |          | YES
```

---

## Next Level — Once Graph Module is Available

Once the richer PowerShell export is available, extend the spreadsheet with additional columns for:

- **Specific group inclusions** — which Entra groups are targeted by each policy
- **Specific group exclusions** — which accounts are explicitly bypassing policies
- **App-level targeting** — whether policies cover specific apps or all cloud apps

This second level of analysis lets you identify accounts that fall through the gaps even within a persona range that appears covered — for example, a service account that is a member of an excluded group and therefore not subject to any enforced policy despite its persona range having coverage.
