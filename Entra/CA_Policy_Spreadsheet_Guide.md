# Conditional Access Policy — Spreadsheet Analysis Guide

## Overview

A well-structured CA naming convention can be treated like a database row written as a string. If each segment of the policy name is separated by a consistent delimiter (e.g. a hyphen), those segments can be split into individual columns and used as the basis for pivot analysis.

The example below illustrates a typical structured naming convention with six segments:

```
CA104-Persona-PolicyType-Resource-Platform-GrantControl
 [1]    [2]       [3]       [4]      [5]         [6]
```

The exact segment names and values will vary depending on your organisation's CA naming standard. Adapt the steps below to match your convention.

---

## Step 1 — Export Your Policy Data from Entra

You have two options depending on timing.

**Option A — Now (no Graph module installed):**

1. Navigate to **Entra Portal → Protection → Conditional Access → Policies**
2. Click the **Download** button in the top-right
3. This exports a CSV with policy names and states — enough to start

**Option B — Later (once Microsoft.Graph is installed on your admin workstation):**

Run the following PowerShell export from your admin workstation. This produces a much richer dataset including inclusions, exclusions, grant controls, and apps — and will produce significantly more insightful analysis.

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

> **Recommendation:** If you can wait for the Graph module to be installed via your change management process, do so. The richer export will save you significant manual effort.

---

## Step 2 — Split the Policy Name into Columns

Once your raw data is in Excel:

1. Paste the policy names into **Column A**
2. Select Column A
3. Go to **Data → Text to Columns → Delimited → Hyphen** (or whichever delimiter your convention uses)
4. Split into columns B onwards

This gives you the following column structure, using a hyphen-delimited six-segment convention as an example:

| Column | Content | Example |
|---|---|---|
| A | Full policy name (raw) | `CA104-Persona-PolicyType-Resource-Platform-GrantControl` |
| B | CA Number | `CA104` |
| C | Persona | `Persona` |
| D | Policy Type | `PolicyType` |
| E | Resource | `Resource` |
| F | Platform | `Platform` |
| G | Grant Control | `GrantControl` |
| H | Optional Description | *(seventh segment if present)* |
| I | State | `enabled` / `enabledForReportingButNotEnforced` / `disabled` |

---

## Step 3 — Add the Persona Range Column

If your CA numbering convention uses numeric ranges to represent different personas (e.g. CA001–CA099 for one group, CA100–CA199 for another), add **Column J — Persona Range** using the following formula pattern.

This maps the CA number back to its named range bucket, giving you a clean filterable label rather than just a number. Replace the threshold values and labels with those defined in your own naming convention.

```excel
=IFS(
  VALUE(MID(B2,3,LEN(B2)))<=99,   "PersonaGroup1",
  VALUE(MID(B2,3,LEN(B2)))<=199,  "PersonaGroup2",
  VALUE(MID(B2,3,LEN(B2)))<=299,  "PersonaGroup3",
  VALUE(MID(B2,3,LEN(B2)))<=399,  "PersonaGroup4",
  VALUE(MID(B2,3,LEN(B2)))<=499,  "PersonaGroup5",
  TRUE,                             "Other"
)
```

> The `MID(B2,3,LEN(B2))` portion strips the `CA` prefix from the policy number before evaluating it numerically. Adjust the prefix length if your convention uses a different prefix.

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

Shows whether entire categories of protection are unenforced across the environment. For example, if your convention includes policy types such as baseline protection or identity protection, this view will reveal if any of those categories exist only in report-only state.

### Sheet 4 — Grant Control Summary

- **Rows:** Grant Control (Column G)
- **Columns:** Persona Range (Column J)
- **Values:** Count of policy names

Answers the question: which personas have block policies vs MFA policies vs nothing at all?

### Sheet 5 — Report-Only Candidates

- Filter: State = `enabledForReportingButNotEnforced`
- Sort by: Persona Range (Column J)

This becomes your change management planning list. Work through it top to bottom, reviewing sign-in logs for each policy before submitting for approval.

---

## The Key Output — Coverage Matrix

The single most important view produced by Sheet 2 is a coverage matrix. Populate it with your own persona range names and the results from your pivot. The goal is to make gaps immediately visible at a glance.

```
Persona Range   | Enforced | Report-Only | Disabled | COVERED?
----------------|----------|-------------|----------|--------------------
PersonaGroup1   |    ✓     |      ✓      |          | YES
PersonaGroup2   |    ✓     |      ✓      |          | YES
PersonaGroup3   |          |      ✓      |          | GAP — report-only only
PersonaGroup4   |          |             |    ✓     | GAP — nothing enforced
PersonaGroup5   |          |             |          | GAP — no policies at all
```

This matrix is what you take to your manager and to your change advisory process. It is clear, defensible, and shows exactly where you are starting from and what needs to move.

---

## Next Level — Once Graph Module is Available

Once the richer PowerShell export is available, extend the spreadsheet with additional columns for:

- **Specific group inclusions** — which Entra groups are targeted by each policy
- **Specific group exclusions** — which accounts are explicitly bypassing policies
- **App-level targeting** — whether policies cover specific apps or all cloud apps

This second level of analysis lets you identify accounts that fall through the gaps even within a persona range that appears covered — for example, a service account that is a member of an excluded group and therefore not subject to any enforced policy despite its persona range having coverage.
