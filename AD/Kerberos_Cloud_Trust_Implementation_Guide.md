# Kerberos Cloud Trust — Implementation Guide

**IDAM Platform | OFFICIAL: Sensitive**
**Environment:** Single Domain / Single Forest — Windows Server 2022 DCs
**Sync Model:** Microsoft Entra Connect Sync (legacy)
**Target Feature:** Windows Hello for Business — Cloud Kerberos Trust
**Endpoint Management:** Co-managed — MECM + Intune
**Entra ID Licensing:** P1 / P2 confirmed

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Active Directory Preparation](#2-active-directory-preparation)
3. [Entra Connect Sync Verification](#3-entra-connect-sync-verification)
4. [Install the AzureADHybridAuthenticationManagement Module](#4-install-the-azureadhybridauthenticationmanagement-module)
5. [Create the Entra Kerberos Server Object](#5-create-the-entra-kerberos-server-object)
6. [Configure Windows Hello for Business Policy](#6-configure-windows-hello-for-business-policy)
7. [Co-Management Workload Configuration](#7-co-management-workload-configuration)
8. [Validation & Testing](#8-validation--testing)
9. [Ongoing Maintenance](#9-ongoing-maintenance)
10. [Troubleshooting](#10-troubleshooting)
11. [Rollback Procedure](#11-rollback-procedure)
12. [Reference & Further Reading](#12-reference--further-reading)

---

## 1. Overview & Architecture

Kerberos Cloud Trust (also referred to as Cloud Kerberos Trust) is the preferred Windows Hello for Business (WHfB) deployment model for hybrid Entra ID environments. It eliminates the need for certificate infrastructure (PKI/ADCS) and removes the legacy Key Trust requirement for read-write Domain Controller access during every authentication.

### 1.1 How It Works

The feature leverages an **Entra Kerberos Server object** — a synthetic read-only domain controller (RODC) representation — created in both Active Directory and Entra ID. The authentication flow is:

1. The device requests a **partial TGT (pTGT)** from Entra ID using the user's WHfB key (PIN or biometric).
2. Entra ID issues the pTGT, signed with the Entra Kerberos Server's private key (held securely in Entra ID).
3. The device presents the pTGT to an on-premises Domain Controller.
4. The DC trusts the Entra Kerberos Server object in AD and exchanges the pTGT for a full Kerberos TGT.
5. The user gains full access to on-premises resources via Kerberos — no certificate required.

### 1.2 Prerequisites Summary

| Component | Requirement | Status |
|---|---|---|
| Domain Controllers | Windows Server 2016 or later (2022 ✓) | Confirmed in scope |
| Domain Functional Level | Windows Server 2016 or later | Verify before proceeding |
| Entra Connect Sync | v2.1.1.0 or later | Check & upgrade if needed |
| Entra ID Licensing | P1 or P2 | Confirmed in scope |
| PowerShell Module | AzureADHybridAuthenticationManagement ≥ 2.2.2 | Installed in Step 4 |
| Device Join State | Hybrid Entra ID Joined or Entra ID Joined | Verify per device |
| Windows Client | Windows 10 21H2+ or Windows 11 | Required for Cloud Trust |
| MFA | Entra ID MFA enabled | Required for WHfB enrolment |

---

## 2. Active Directory Preparation

### 2.1 Verify Domain Functional Level

> **Where to run:** Any Domain Controller or AD admin workstation with the RSAT AD PowerShell module.

```powershell
# Requires: Active Directory PowerShell module (RSAT)

Import-Module ActiveDirectory

# Check Domain Functional Level — must be Windows2016Domain or higher
Get-ADDomain | Select-Object Name, DomainMode

# Check Forest Functional Level
Get-ADForest | Select-Object Name, ForestMode

# Expected output:
#   DomainMode : Windows2016Domain
#   ForestMode : Windows2016Forest
```

> ⚠️ **Warning — Raising Functional Level**
> If `DomainMode` is below `Windows2016Domain` you must raise the domain and forest functional level before proceeding. This operation is **irreversible**. Test in a non-production environment first and execute during a scheduled change window.
> ```powershell
> Set-ADDomainMode -Identity <domain> -DomainMode Windows2016Domain
> Set-ADForestMode -Identity <forest> -ForestMode Windows2016ForestMode
> ```

---

### 2.2 Verify DC Patch Level

All Domain Controllers must have the **January 2022 (or later)** cumulative update applied to support Cloud Kerberos Trust.

> **Where to run:** Admin workstation with remote WMI/WinRM access to DCs, or directly on each DC.

```powershell
# Requires: Remote WMI access or local admin on each DC

$DCs = Get-ADDomainController -Filter * | Select-Object -ExpandProperty HostName

foreach ($DC in $DCs) {
    $patch = Get-HotFix -ComputerName $DC |
             Sort-Object InstalledOn -Descending |
             Select-Object -First 1

    [PSCustomObject]@{
        DomainController = $DC
        LatestPatch      = $patch.HotFixID
        InstalledOn      = $patch.InstalledOn
    }
}
```

---

### 2.3 Active Directory OU & Object Placement

The Entra Kerberos Server objects are created **automatically** by the PowerShell module in Step 5. You do not choose the OU. The table below documents exactly where they land — this is important for GPO scoping, delegation reviews, tiered admin exceptions, and security audits.

| Object | AD Location (Auto-created) | Notes |
|---|---|---|
| `AzureADKerberos` computer object | `CN=AzureADKerberos,OU=Domain Controllers,DC=<domain>,DC=<tld>` | Created by `New-AzureADKerberosServer`. Appears as a pseudo-RODC. |
| `krbtgt_AzureAD` service account | `CN=krbtgt_AzureAD,CN=Users,DC=<domain>,DC=<tld>` | Auto-created. Do NOT move or delete. |
| WHfB user keys (`msDS-KeyCredentialLink`) | On each user object in their existing OU | Attribute written by Entra ID during device registration. No OU change needed. |

> ⚠️ **Warning — Do NOT modify these objects manually**
> Moving `AzureADKerberos` or `krbtgt_AzureAD` out of their default locations, modifying their attributes directly, or including `krbtgt_AzureAD` in standard krbtgt password rotation scripts **will break Cloud Kerberos Trust** and prevent affected users from authenticating with WHfB.
> Add `krbtgt_AzureAD` to the exclusion list of any automated account lifecycle or password rotation tooling immediately after object creation.

---

### 2.4 Delegate Permissions (Least Privilege)

The account used to run the Entra Kerberos Server cmdlets requires:

- **Active Directory:** Domain Admin (DA), or delegated rights to create computer objects in `OU=Domain Controllers` and user objects in `CN=Users`.
- **Entra ID:** Global Administrator or Hybrid Identity Administrator.

> ℹ️ **Australian Government Least-Privilege Note**
> To comply with the ACSC ISM and Essential Eight (Restrict Admin Privileges), use a dedicated break-glass or Tier 0 admin account for this operation. Do not use a day-to-day admin account.
> The **Global Administrator** role should be assigned Just-in-Time (JIT) via Entra ID Privileged Identity Management (PIM), time-boxed to the change window, and revoked immediately after completion. Log the activation as a change record.

---

## 3. Entra Connect Sync Verification

### 3.1 Check Entra Connect Version

Cloud Kerberos Trust requires **Entra Connect Sync version 2.1.1.0 or later**.

> **Where to run:** The Entra Connect Sync server, with local admin rights.

```powershell
# Check installed version
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Azure AD Connect" |
    Select-Object Version

# Minimum required: 2.1.1.0
# Download latest: https://www.microsoft.com/download/details.aspx?id=47594

# After upgrade, verify sync is healthy
Import-Module ADSync
Get-ADSyncScheduler

# Trigger a delta sync to confirm connectivity
Start-ADSyncSyncCycle -PolicyType Delta
```

---

### 3.2 Verify Hybrid Entra ID Join is Configured

Cloud Kerberos Trust requires devices to be **Hybrid Entra ID Joined**. Confirm the Entra Connect Device Options are configured and the Service Connection Point (SCP) is present in AD.

> **Where to run:** Entra Connect Sync server.

```powershell
Import-Module ADSync

# Check device writeback / Hybrid Join configuration
Get-ADSyncGlobalSettingsParameter | Where-Object { $_.Name -like '*Device*' }

# Verify SCP exists in AD (written by Entra Connect wizard)
Get-ADObject -Filter { objectClass -eq "serviceConnectionPoint" } `
    -SearchBase "CN=Configuration,DC=<domain>,DC=<tld>" `
    -Properties keywords |
    Where-Object { $_.keywords -like "*azureADId*" }

# Expected: At least one SCP object returned with your tenant ID in keywords
```

> ℹ️ If no SCP is found, open the Entra Connect wizard and navigate to:
> **Configure device options → Configure Hybrid Azure AD join → Enabled**
> Re-run the wizard and verify SCP creation before proceeding.

---

## 4. Install the AzureADHybridAuthenticationManagement Module

This module provides the cmdlets to create and manage the Entra Kerberos Server object.

> **Where to run:** The Entra Connect Sync server, or a designated admin workstation running Windows Server 2016+ / Windows 10 or 11 that has:
> - Network access to a Domain Controller
> - PowerShell 5.1 or later
> - Internet access (or a configured proxy) to reach Entra ID and PSGallery

```powershell
# Step 1: Ensure NuGet provider is available
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force

# Step 2: Trust PSGallery (if not already trusted)
Set-PSRepository -Name PSGallery -InstallationPolicy Trusted

# Step 3: Install the module
Install-Module -Name AzureADHybridAuthenticationManagement -AllowClobber -Force

# Step 4: Verify installation — expect version 2.2.2 or later
Get-Module -ListAvailable -Name AzureADHybridAuthenticationManagement

# Step 5: Import
Import-Module AzureADHybridAuthenticationManagement

# Step 6: Confirm cmdlets are available
Get-Command -Module AzureADHybridAuthenticationManagement

# Expected cmdlets:
#   Get-AzureADKerberosServer
#   New-AzureADKerberosServer
#   Set-AzureADKerberosServer
#   Remove-AzureADKerberosServer
```

---

## 5. Create the Entra Kerberos Server Object

This is the core step. The cmdlet below creates the `AzureADKerberos` computer object in AD and registers it with Entra ID simultaneously. It requires credentials for **both** AD and Entra ID.

### 5.1 Prepare Variables

> **Where to run:** The Entra Connect Sync server or designated admin workstation (same as Section 4).
> **Requires:** Domain Admin credential AND Global Administrator (or Hybrid Identity Administrator) in Entra ID.

```powershell
Import-Module AzureADHybridAuthenticationManagement

# Collect on-premises Domain Admin credential
$domainCred = Get-Credential -Message "Enter Domain Admin credentials (DOMAIN\AdminAccount)"

# Store your domain DNS name
$domain = (Get-ADDomain).DNSRoot   # e.g. agency.gov.au

# Store your Entra ID admin UPN (used in Option A below)
$cloudAdminUPN = "admin@agency.gov.au"   # Replace with your Global/Hybrid Admin UPN
```

---

### 5.2 Create the Kerberos Server Object

Two options are provided. **Option A is strongly recommended** for Australian Government tenants where MFA is enforced on admin accounts (as required by the ACSC ISM).

#### Option A — MFA-Enabled Tenant (Recommended)

This triggers a browser-based modern authentication prompt for Entra ID sign-in. Complete MFA in the browser window and the command will proceed automatically.

```powershell
# A browser window will open for Entra ID MFA sign-in — complete it to proceed
New-AzureADKerberosServer `
    -Domain $domain `
    -DomainCredential $domainCred `
    -UserPrincipalName $cloudAdminUPN
```

#### Option B — Non-Interactive (Break-Glass / Automation Only)

Use only when Option A is not possible (e.g. a fully automated pipeline using a service principal with no MFA).

```powershell
# Collect Entra ID admin credential (no MFA prompt)
$cloudCred = Get-Credential -Message "Enter Entra ID Admin credential"

New-AzureADKerberosServer `
    -Domain $domain `
    -DomainCredential $domainCred `
    -CloudCredential $cloudCred
```

---

### 5.3 Verify the Kerberos Server Object

```powershell
# Retrieve and inspect the created object
Get-AzureADKerberosServer -Domain $domain -DomainCredential $domainCred

# Expected output:
#   Id                 : <numeric ID>
#   UserAccount        : CN=krbtgt_AzureAD,CN=Users,DC=agency,DC=gov,DC=au
#   ComputerAccount    : CN=AzureADKerberos,OU=Domain Controllers,DC=agency,DC=gov,DC=au
#   DisplayName        : krbtgt_AzureAD
#   DomainDnsName      : agency.gov.au
#   KeyVersion         : 1
#   KeyUpdatedOn       : <timestamp>
#   KeyUpdatedFrom     : <server FQDN>
#   CloudDisplayName   : krbtgt_AzureAD
#   CloudDomainDnsName : agency.gov.au

# Verify the computer object exists in AD
Get-ADComputer -Identity "AzureADKerberos" -Properties * |
    Select-Object Name, DistinguishedName, UserAccountControl

# Verify the service account exists and is enabled
Get-ADUser -Identity "krbtgt_AzureAD" -Properties * |
    Select-Object Name, DistinguishedName, Enabled, PasswordLastSet
```

> ℹ️ **Key Rotation Policy**
> The `krbtgt_AzureAD` key must be rotated at least every **30 days** using `Set-AzureADKerberosServer`. Do **not** rotate this key using standard AD password reset tooling — always use the module. See [Section 9](#9-ongoing-maintenance) for the rotation procedure.

---

## 6. Configure Windows Hello for Business Policy

In a co-managed environment (MECM + Intune), WHfB policy can be delivered via **Intune** (preferred) or **Group Policy (GPO)**. Choose one method — do not apply both to the same device scope.

> ⚠️ **Policy Conflict Risk in Co-Managed Environments**
> Applying WHfB policy from both Intune and GPO to the same device causes conflicts and can result in WHfB provisioning failures. Define a clear workload owner in your co-management configuration **before** applying policy. For new devices, prefer Intune. For legacy AD-joined devices not yet in Intune scope, use GPO.

---

### 6.1 Option A — Intune Policy (Recommended)

#### 6.1.1 Create a Settings Catalog Policy

Navigate to: **Intune Admin Center → Devices → Configuration → Create Policy**
Select: **Windows 10 and later → Settings Catalog**

Configure the following settings (search for each by name in the Settings Catalog):

| Setting | Value | Notes |
|---|---|---|
| Use Windows Hello for Business | Enabled | Core toggle — must be Enabled |
| Use Cloud Trust for On-Premises Authentication | **Enabled** | This is the Cloud Kerberos Trust toggle |
| Use a hardware security device (TPM) | Enabled | TPM 2.0 preferred; TPM 1.2 minimum |
| Use certificate for on-premises authentication | **Disabled** | Disable — Cloud Trust replaces certificate trust |
| Enable PIN Recovery | Enabled | Recommended; requires Intune MDM enrolment |
| Minimum PIN Length | 6 (or per agency policy) | ACSC ISM recommends minimum 6 digits |
| PIN Expiration (days) | 0 or per policy | Set per agency PIN lifecycle policy |

#### 6.1.2 Assignment

- Assign the policy to the **Entra ID group** containing your pilot or production device population.
- Use a **staged ring approach**: Pilot group first → validate → broaden to all Hybrid Joined devices.
- **Exclude** any devices that are not Hybrid Entra ID Joined or Entra Joined (e.g. AD-only servers, DCs).

---

### 6.2 Option B — Group Policy Object (GPO)

#### 6.2.1 Create and Link the GPO

> **Where to run:** Domain Controller or GPMC admin workstation.
> **Requires:** Group Policy Management Console (GPMC), Domain Admin.

```powershell
Import-Module GroupPolicy

# Create a dedicated WHfB GPO — do not add WHfB settings to an existing GPO
$gpoName  = "IDAM-WHfB-CloudKerberosTrust"
$targetOU = "OU=Workstations,OU=Managed,DC=agency,DC=gov,DC=au"  # Adjust to your OU

# Create the GPO
New-GPO -Name $gpoName -Comment "Windows Hello for Business - Cloud Kerberos Trust Policy"

# Link to the target OU
New-GPLink -Name $gpoName -Target $targetOU -LinkEnabled Yes -Enforced No

# Verify the link
Get-GPInheritance -Target $targetOU
```

#### 6.2.2 Configure GPO Settings

Open the GPO in **Group Policy Management Editor** and configure:

| Policy Path | Setting Name | Value |
|---|---|---|
| Computer Configuration → Admin Templates → Windows Components → Windows Hello for Business | Use Windows Hello for Business | **Enabled** |
| (same path) | Use cloud trust for on-premises authentication | **Enabled** |
| (same path) | Use a hardware security device | **Enabled** |
| (same path) | Use certificate for on-premises authentication | **Disabled** |
| (same path) | Configure device unlock factors | Per agency policy (optional) |

> ℹ️ **ADMX Central Store Requirement**
> The **"Use cloud trust for on-premises authentication"** policy setting requires the **Windows 11 or Windows Server 2022 ADMX templates** (or later). If this setting does not appear in your GPMC:
>
> 1. On a Windows 11 device, locate `C:\Windows\PolicyDefinitions\`
> 2. Copy the relevant ADMX/ADML files to your Central Store:
>    `\\<domain>\SYSVOL\<domain>\Policies\PolicyDefinitions\`
> 3. Refresh GPMC — the setting will now be visible.

---

## 7. Co-Management Workload Configuration

In a co-managed environment you must assign the **Device Configuration** workload to Intune when using Option A (Intune policy), or leave it with SCCM when using Option B (GPO). Mixing workload ownership for the same policy area on the same device will cause conflicts.

### 7.1 Set the Device Configuration Workload

Navigate to: **Intune Admin Center → Tenant Administration → Co-management → Properties → Workloads**

| Workload | Recommended Setting | Rationale |
|---|---|---|
| Device Configuration | **Intune** (Pilot or All) | Required if delivering WHfB via Intune Settings Catalog |
| Endpoint Protection | Intune (Pilot or All) | Recommended for Defender / BitLocker alignment |
| Windows Update Policies | Agency decision | Separate concern — align with your patching team |
| Resource Access Policies | Intune | Required for certificate profiles if used in future |

> ℹ️ **Use the Pilot approach**
> Set the Device Configuration workload to **"Pilot Intune"** first, targeting your WHfB pilot Entra ID group. Once the pilot is validated, switch to **"Intune"** for all devices. This prevents unintended policy delivery to production endpoints before the deployment is proven.

---

## 8. Validation & Testing

### 8.1 End-to-End Validation Steps

1. Identify a pilot device that is **Hybrid Entra ID Joined** and has received the WHfB policy (confirm via Intune device status or `gpresult /h`).
2. Sign in with a **standard user account** (not a local or domain admin — admin accounts are excluded from WHfB provisioning by default in most tenant configurations).
3. On first sign-in after policy applies, Windows will prompt the user to set up Windows Hello. Complete PIN enrolment.
4. Sign out and sign back in using the WHfB PIN. The logon should succeed and grant on-premises resource access (e.g. mapped drives, SharePoint on-premises, intranet).
5. Verify on-premises Kerberos tickets are issued:

```powershell
# Run on: The pilot end-user device, as the signed-in standard user (NOT admin)

# Check Kerberos tickets — confirms pTGT exchange succeeded
klist

# Expected: A ticket for krbtgt/<DOMAIN> issued by a DC

# Confirm WHfB credential and join state
dsregcmd /status

# Key fields to verify in output:
#   AzureAdJoined  : YES
#   DomainJoined   : YES
#   NgcSet         : YES   <-- Windows Hello credential registered
```

---

### 8.2 Verify the Entra Kerberos Server Object is Healthy

> **Where to run:** Admin workstation (same as Section 5).

```powershell
Import-Module AzureADHybridAuthenticationManagement

$domain     = (Get-ADDomain).DNSRoot
$domainCred = Get-Credential -Message "Domain Admin credential"

Get-AzureADKerberosServer -Domain $domain -DomainCredential $domainCred

# Validate these fields:
#   KeyVersion         — must be >= 1 (0 indicates creation failed)
#   KeyUpdatedOn       — must reflect creation or last rotation date
#   CloudDomainDnsName — must match your tenant's verified domain

# If KeyVersion is 0 or the object is missing, re-run New-AzureADKerberosServer
```

---

### 8.3 Event Log Validation on Domain Controllers

> **Where to run:** Domain Controllers (directly or via remote PowerShell).

```powershell
# Check for Kerberos TGT issuance events (Event ID 4769)
# Filter for tickets issued after Cloud Trust sign-in attempts

Get-WinEvent -LogName Security -FilterXPath `
    '*[System[EventID=4769] and EventData[Data[@Name="ServiceName"]="krbtgt"]]' `
    -MaxEvents 20 |
    Select-Object TimeCreated, Message

# Check the KDC operational log for Cloud Trust errors
# Event ID 14 indicates a pTGT exchange failure

Get-WinEvent `
    -LogName 'Microsoft-Windows-Kerberos-Key-Distribution-Center/Operational' `
    -MaxEvents 50 |
    Where-Object { $_.LevelDisplayName -in 'Error', 'Warning' } |
    Select-Object TimeCreated, Id, Message
```

---

## 9. Ongoing Maintenance

### 9.1 Key Rotation

Rotate the Entra Kerberos Server key **at least every 30 days**, or immediately following a suspected credential compromise. Always use the PowerShell module — never reset `krbtgt_AzureAD` manually through AD tooling.

> **Where to run:** Admin workstation.
> **Recommended:** Schedule as a recurring task in your change calendar or automation pipeline.

```powershell
Import-Module AzureADHybridAuthenticationManagement

$domain        = (Get-ADDomain).DNSRoot
$domainCred    = Get-Credential -Message "Domain Admin credential"
$cloudAdminUPN = "admin@agency.gov.au"   # Global Admin or Hybrid Identity Admin

# Rotate the Kerberos Server key
Set-AzureADKerberosServer `
    -Domain $domain `
    -DomainCredential $domainCred `
    -UserPrincipalName $cloudAdminUPN `
    -RotateServerKey

# Verify the new key version has incremented
Get-AzureADKerberosServer -Domain $domain -DomainCredential $domainCred |
    Select-Object KeyVersion, KeyUpdatedOn
```

---

### 9.2 Monitoring Checklist

| Check | Frequency | Action if Failing |
|---|---|---|
| `AzureADKerberos` computer object exists in AD | Weekly | Re-run `New-AzureADKerberosServer` |
| `krbtgt_AzureAD` account is enabled | Weekly | Enable via ADUC; investigate cause of disable; add to automation exclusion list |
| Key rotation (KeyVersion incrementing) | Monthly | Run `Set-AzureADKerberosServer -RotateServerKey` |
| Entra Connect Sync health (no sync errors) | Weekly | Check Entra Connect Health in Entra ID portal |
| DC Event ID 14 (KDC errors) | Daily | Investigate DC health, patch level, and object integrity |
| WHfB provisioning success rate (Intune) | Weekly | Review Intune device status report; check for policy conflicts |

---

## 10. Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| WHfB PIN setup prompt never appears | Policy not applied or device not Hybrid Joined | Run `dsregcmd /status` — confirm `AzureAdJoined: YES` and `DomainJoined: YES`. Check Intune policy assignment status or run `gpresult /h`. |
| PIN set up but on-premises resources fail | `AzureADKerberos` object missing or unhealthy | Run `Get-AzureADKerberosServer`. If missing, re-create. Check DC event logs for Event ID 14. |
| "Use cloud trust" setting missing in GPMC | Outdated ADMX templates in Central Store | Copy ADMX/ADML from a Windows 11 device (`C:\Windows\PolicyDefinitions\`) to `\\<domain>\SYSVOL\...\PolicyDefinitions\`. |
| `New-AzureADKerberosServer` fails with authentication error | MFA not handled, or wrong credential type | Use `-UserPrincipalName` (not `-CloudCredential`) to trigger the modern auth browser prompt. |
| `krbtgt_AzureAD` account becomes disabled | Inactive account policy or tiered admin automation script | Add `krbtgt_AzureAD` to the exclusion list of all account disable/expiry automation immediately. |
| Entra Connect Sync not syncing the Kerberos object | Entra Connect version below 2.1.1.0 | Upgrade Entra Connect Sync to the latest version, then re-run `New-AzureADKerberosServer`. |
| WHfB not prompting on a co-managed device | Device Configuration workload still assigned to SCCM | In Intune Co-management settings, set Device Configuration workload to **Intune** (or Pilot Intune). |
| `dsregcmd /status` shows `NgcSet: NO` after policy applied | TPM not present, not enabled in BIOS, or TPM attestation failure | Verify TPM 2.0 is present and enabled in BIOS/UEFI. Check Intune device Hardware page for TPM attestation status. |

---

## 11. Rollback Procedure

> ⚠️ **Warning — Rollback removes WHfB for all affected users**
> Removing the Entra Kerberos Server object will immediately disable Cloud Kerberos Trust. Users who enrolled WHfB with Cloud Trust will fall back to password authentication. Notify your service desk **before** executing this procedure and raise a change record.

> **Where to run:** Admin workstation.
> **Requires:** Domain Admin + Global Administrator (or Hybrid Identity Administrator).

```powershell
Import-Module AzureADHybridAuthenticationManagement

$domain        = (Get-ADDomain).DNSRoot
$domainCred    = Get-Credential -Message "Domain Admin credential"
$cloudAdminUPN = "admin@agency.gov.au"

# Step 1: Remove the Entra Kerberos Server object from both AD and Entra ID
Remove-AzureADKerberosServer `
    -Domain $domain `
    -DomainCredential $domainCred `
    -UserPrincipalName $cloudAdminUPN

# Step 2: Verify removal from AD
Get-ADComputer -Filter { Name -eq "AzureADKerberos" }   # Should return nothing
Get-ADUser    -Filter { Name -eq "krbtgt_AzureAD"    }   # Should return nothing

# Step 3: Revert WHfB policy
#   Intune: Navigate to the WHfB Settings Catalog policy and remove the assignment,
#           or set "Use Windows Hello for Business" to Not Configured.
#   GPO:    Unlink or disable the IDAM-WHfB-CloudKerberosTrust GPO:
#           Set-GPLink -Name "IDAM-WHfB-CloudKerberosTrust" -Target $targetOU -LinkEnabled No

# Step 4: Notify the service desk — users will revert to password authentication
# Step 5: Record the rollback in your ITSM change log
```

---

## 12. Reference & Further Reading

| Resource | URL |
|---|---|
| Microsoft: Cloud Kerberos Trust Deployment Guide | https://learn.microsoft.com/en-us/windows/security/identity-protection/hello-for-business/deploy/hybrid-cloud-kerberos-trust |
| Microsoft: AzureADHybridAuthenticationManagement module | https://learn.microsoft.com/en-us/azure/active-directory/authentication/howto-authentication-passwordless-security-key-on-premises |
| Microsoft: Entra Connect Sync version history | https://learn.microsoft.com/en-us/azure/active-directory/hybrid/reference-connect-version-history |
| Microsoft: Troubleshoot WHfB deployments | https://learn.microsoft.com/en-us/windows/security/identity-protection/hello-for-business/hello-deployment-issues |
| ACSC: Information Security Manual (ISM) | https://www.cyber.gov.au/resources-business-and-government/essential-cyber-security/ism |
| ACSC: Essential Eight — Restrict Admin Privileges | https://www.cyber.gov.au/resources-business-and-government/essential-cyber-security/essential-eight |
| ACSC: Hardening Microsoft Windows | https://www.cyber.gov.au/resources-business-and-government/publications/hardening-microsoft-windows-10-version-21h1-workstations |

---

*IDAM Platform — Entra ID & Active Directory | OFFICIAL: Sensitive*
