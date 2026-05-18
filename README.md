# Hybrid Identity & IAM Home Lab

## Overview
This project documents a hands-on IAM lab environment I built while transitioning from IT support into Identity and Access Management (IAM).

The goal of the lab was to gain practical experience with:
- Microsoft Entra ID
- Hybrid Active Directory synchronization
- PowerShell automation
- Conditional Access
- RBAC and Privileged Identity Management (PIM)
- SAML federation
- Microsoft Graph API

The lab environment includes:
- Windows Server 2022 Domain Controller
- On-premises Active Directory (`sc300lab.com`)
- Microsoft Entra ID tenant
- Microsoft Entra Connect synchronization

I used this environment to practice common IAM tasks such as user provisioning, group management, hybrid identity synchronization, MFA enforcement, and access governance.

---

# Phase 1: Automated User Provisioning in Active Directory

## Objective
Create and manage test user accounts in Active Directory using PowerShell instead of manual account creation.

## What I Implemented
- Created a dedicated `Sync_Users` OU for synchronized identities
- Built a PowerShell script to automate user creation
- Configured alternative UPN suffixes for Entra ID synchronization
- Added basic duplicate-account checking before user creation
- Created multiple user types including:
  - standard users
  - admin account
  - vendor account
  - break-glass account

## Screenshots
![PowerShell Provisioning Outputs](01_powershell_user_provisioning.png)

![On-Premises AD Scoped OU Contents](02_active_directory_sync_ou.png)

![AD Domains and Trusts Alternative UPN Configuration](03_alternative_upn_suffix.png)

## What I Learned
- Basic Active Directory user provisioning with PowerShell
- How UPN suffixes affect Entra ID synchronization
- The importance of OU structure in hybrid environments
- How automation can reduce repetitive administrative tasks
<details>
<summary>View PowerShell Script</summary>

```powershell
# 1. Load the Active Directory modules so PowerShell understands AD commands
Import-Module ActiveDirectory

# 2. CONFIGURATION VARIABLES
$TargetOU   = "OU=Sync_Users,DC=sc300lab,DC=com"      # Target OU Distinguished Name
$UPNSuffix  = "yourtenant.onmicrosoft.com"             # Target M365 Domain (Update this!)
$DefaultPass = "SchwabLab2026!"                        # Secure temporary bootstrap password

# 3. CORE ENTERPRISE PERSONA MATRIX (Simulating HR Data Ingestion)
$Roster = @(
    # Standard Personas
    @{ FirstName = "Marcus";    LastName = "Vance";      SamName = "mvance";      Title = "IAM Associate" },
    @{ FirstName = "Elena";     LastName = "Rostova";    SamName = "erostova";    Title = "Security Analyst" },
    @{ FirstName = "David";     LastName = "Kim";        SamName = "dkim";        Title = "Cloud Engineer" },
    
    # Specialized/Privileged Personas
    @{ FirstName = "Alex";      LastName = "Admin";      SamName = "alexadmin";   Title = "Helpdesk Administrator" },
    @{ FirstName = "Vendor";    LastName = "Support";    SamName = "vsupport";    Title = "Third-Party Contractor" },
    @{ FirstName = "Emergency"; LastName = "BreakGlass"; SamName = "breakglass01";Title = "Break-Glass Account Override" }
)

# 4. PROVISIONING ENGINE
foreach ($User in $Roster) {
    $TargetUPN = "$($User.SamName)@$UPNSuffix"
    $SecurePassword = ConvertTo-SecureString $DefaultPass -AsPlainText -Force

    # Structural Check: Ensure account doesn't already exist before creating
    if (-not (Get-ADUser -Filter "SamAccountName -eq '$($User.SamName)'")) {
        
        New-ADUser -Name "$($User.FirstName) $($User.LastName)" `
                   -GivenName $User.FirstName `
                   -Surname $User.LastName `
                   -SamAccountName $User.SamName `
                   -UserPrincipalName $TargetUPN `
                   -Path $TargetOU `
                   -Title $User.Title `
                   -AccountPassword $SecurePassword `
                   -Enabled $true `
                   -ChangePasswordAtLogon $false
        
        Write-Host "Successfully provisioned enterprise persona: $($User.SamName) as $TargetUPN" -ForegroundColor Green
    } else {
        Write-Host "Account identity $($User.SamName) already exists. Skipping creation." -ForegroundColor Yellow
    }
}
```
</details>

# Phase 2: Scoped Directory Synchronization (Entra Connect)

## Objective
Configure Microsoft Entra Connect to sync only selected users from on-premises Active Directory to Microsoft Entra ID.

## What I Implemented
- Installed and configured Microsoft Entra Connect using a custom setup (not express settings)
- Scoped synchronization to only the `Sync_Users` OU
- Enabled Password Hash Synchronization (PHS) for cloud authentication
- Enabled Password Writeback to support self-service password reset (SSPR)
- Prevented full-domain sync to avoid syncing unnecessary or test objects

## Why This Matters
In a real environment, limiting synchronization scope helps reduce risk by ensuring only intended user accounts are synced to the cloud.

## Screenshots
![Scoped OU Filtering Configuration](04_entra_connect_ou_filtering.png)

![Password Writeback Configuration](05_optional_features_writeback.png)


## What I Learned
- How Entra Connect controls identity synchronization between on-prem and cloud
- The difference between Password Hash Sync and Password Writeback
- Why OU filtering is important in hybrid identity environments
- How configuration choices directly affect what identities appear in Microsoft Entra ID
# Phase 3: Identity Lifecycle Testing (Joiner / Mover / Leaver)

## Objective
Validate how user lifecycle changes in Active Directory (joiner, mover, leaver scenarios) are reflected in Microsoft Entra ID through synchronization.

---

## 1. Cloud Sync Verification (Joiner Validation)

## What I Observed
- Verified that on-premises users synchronized to Microsoft Entra ID
- Confirmed accounts showed as “On-premises sync enabled”
- Validated identity visibility in the cloud tenant

![Entra ID User Cloud State](06_entra_cloud_sync_verification.png)

---

## 2. Mover Scenario: Group-Based Access Change

To simulate a role change, I updated user access using an Active Directory security group and triggered a manual sync.

## What I Implemented
- Created a security group in Active Directory
- Added a user to the group
- Triggered a delta sync using Azure AD Connect
- Verified group membership in Microsoft Entra ID

### Screenshots
![PowerShell Forced Delta Sync Run](07_powershell_group_creation_delta_sync.png)

![Hybrid Group Cloud Sync Members Verified](08_hybrid_group_rbac_sync.png)

<details>
<summary>View Script</summary>

```powershell
New-ADGroup -Name "SG-SecurityOperations-Cloud" `
            -GroupScope Global `
            -GroupCategory Security `
            -Path "OU=Sync_Users,DC=sc300lab,DC=com"

Add-ADGroupMember -Identity "SG-SecurityOperations-Cloud" -Members "erostova"

Start-ADSyncSyncCycle -PolicyType Delta
```

</details>

---

## 3. Leaver Scenario: Account Disable and Offboarding

To simulate an offboarding event, I disabled multiple user accounts in Active Directory, moved them to a dedicated inactive OU, and verified synchronization.

### Screenshots
![Leaver Bulk Disable Sync Run](leaver_bulk_disable_sync_run.png)

![AD Disabled Users OU State](ad_disabled_users_ou_state.png)

---

## What I Learned
- How identity changes flow from Active Directory to Microsoft Entra ID
- How synchronization affects user and group updates in hybrid environments
- How group-based access control supports role changes (Mover scenario)
- How account disabling and OU structuring supports offboarding (Leaver scenario)
- How Joiner / Mover / Leaver lifecycle processes are implemented in IAM environments



# Phase 4: Access Governance with Terraform (Entra ID)

## Objective
Use Terraform to create and manage basic Microsoft Entra ID Conditional Access policies instead of configuring them manually in the Azure portal.

---

## What I Implemented
- Used Terraform (AzureAD provider) to manage Entra ID Conditional Access policies
- Created a policy requiring MFA for an administrative user
- Created a second policy restricting access for a vendor account
- Used dynamic user lookup instead of hardcoding object IDs
- Applied configuration using Terraform CLI commands

![terraform apply success](03a_terraform_apply_success.png)
![entra conditional acces](03b_entra_conditional_access.png)
---

## Terraform Configuration
<details>
<summary>Terraform Script</summary>
### Provider Setup
```hcl
terraform {
  required_providers {
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.48.0"
    }
  }
}

provider "azuread" {}
```

---

### Conditional Access Policies
```hcl
variable "tenant_domain" {
  default = "scfun.onmicrosoft.com"
}

data "azuread_user" "admin_user" {
  user_principal_name = "alexadmin@${var.tenant_domain}"
}

data "azuread_user" "vendor_user" {
  user_principal_name = "vsupport@${var.tenant_domain}"
}

resource "azuread_conditional_access_policy" "mfa_admin" {
  display_name = "MFA-For-Admins"
  state        = "enabled"

  conditions {
    users {
      included_users = [data.azuread_user.admin_user.object_id]
    }

    applications {
      included_applications = ["All"]
    }
  }

  grant_controls {
    built_in_controls = ["mfa"]
  }
}

resource "azuread_conditional_access_policy" "vendor_restriction" {
  display_name = "Vendor-Access-Policy"
  state        = "enabled"

  conditions {
    users {
      included_users = [data.azuread_user.vendor_user.object_id]
    }

    applications {
      included_applications = ["All"]
    }
  }

  grant_controls {
    built_in_controls = ["mfa"]
  }
}
</details>
```

---

## Deployment Steps
<details>
<summary>Terraform Script</summary>
- `terraform init` – initialize provider plugins  
- `terraform plan` – review planned changes  
- `terraform apply -auto-approve` – apply configuration  
</details>details>
---

## Issue Encountered & Fix

During deployment, Terraform failed because Microsoft Entra ID had **Security Defaults enabled**, which blocked Conditional Access policy creation.

### Resolution
- Disabled Security Defaults in the Entra admin portal
- Re-ran Terraform deployment successfully
- Policies were applied and became active in the tenant

---

## Verification

- Confirmed Conditional Access policies were created in Entra ID
- Verified MFA requirement applied to admin user
- Verified vendor policy was active and enforced

---

## What I Learned
- How Terraform can be used to manage Entra ID access policies
- How Conditional Access policies enforce MFA and access restrictions
- How API-level restrictions (like Security Defaults) affect automation
- Basic troubleshooting of Microsoft Graph / Entra policy deployment issues
---

# Phase 5: Break-Glass Account Monitoring (Log Query)

## Objective
Monitor sign-in activity for a high-privilege break-glass account using Microsoft Entra sign-in logs.

---

## What I Implemented
- Queried Microsoft Entra ID sign-in logs using Kusto Query Language (KQL)
- Filtered activity for a break-glass account (`breakglass01`)
- Reviewed successful sign-in events for audit visibility

---

## KQL Query Used

```kusto
SigninLogs
| where UserPrincipalName == "breakglass01@scfun.onmicrosoft.com"
| where Status.errorCode == 0
| project TimeGenerated, UserPrincipalName, IPAddress, Location, ClientAppUsed
```

---

## Purpose
Break-glass accounts are highly privileged emergency accounts that are excluded from standard MFA policies. Monitoring their usage is important to ensure any access is intentional and reviewed.

---

## What I Learned
- How to query Entra ID sign-in logs using KQL
- How break-glass accounts are used in enterprise IAM environments
- Basic log filtering and audit visibility concepts

# Phase 6: Basic Compliance Reporting with Microsoft Graph

## Objective
Use Microsoft Graph API with PowerShell to retrieve and review Microsoft Entra ID user data for basic compliance visibility and reporting.

---

## What I Implemented
- Connected to Microsoft Graph using PowerShell (via `az rest`)
- Queried Entra ID user objects from the directory
- Parsed JSON responses into PowerShell objects
- Reviewed identity attributes for basic compliance-style reporting
- Built simple logic to flag specific identity types (break-glass and vendor accounts)

---

## Microsoft Graph Query Script

```powershell
# Query Microsoft Graph for Entra ID users
Write-Host "Querying Microsoft Graph for user data..." -ForegroundColor Cyan

$RawResponse = az rest --method get --url "https://graph.microsoft.com/v1.0/users?`$select=displayName,userPrincipalName,id,assignedLicenses"

# Convert JSON response into PowerShell object
$GraphData = ConvertFrom-Json $RawResponse
$TenantUsers = $GraphData.value

Write-Host "`n=== BASIC IAM COMPLIANCE REPORT ===" -ForegroundColor Yellow
Write-Host "-----------------------------------"

foreach ($User in $TenantUsers) {

    $Status = "COMPLIANT"
    $Notes = ""

    # Basic rule: check break-glass accounts
    if ($User.userPrincipalName -like "*breakglass*" -and 
        ($User.assignedLicenses.Count -eq 0 -or $User.assignedLicenses -eq $null)) {
        $Status = "REVIEW REQUIRED"
        $Notes = "Break-glass account missing expected license assignment"
    }

    # Basic rule: check vendor accounts
    elseif ($User.userPrincipalName -like "*vsupport*") {
        $Status = "REVIEW REQUIRED"
        $Notes = "Vendor account requires periodic access review"
    }

    [PSCustomObject]@{
        Name      = $User.displayName
        UPN       = $User.userPrincipalName
        Status    = $Status
        Notes     = $Notes
    } | Format-List
}
```

---

## What This Demonstrates
- Basic use of Microsoft Graph API for identity data retrieval
- How IAM teams can programmatically review directory users
- Early-stage compliance-style logic using PowerShell
- Understanding of break-glass and vendor identity types

---

## Challenges
- Required working through Microsoft Graph authentication via Azure CLI
- JSON response structure required parsing before use in PowerShell
- Limited permissions affected some data visibility during testing

---

## What I Learned
- How Microsoft Graph API exposes Entra ID directory data
- How PowerShell can be used for identity reporting tasks
- How IAM teams begin automating compliance checks
- Importance of API permissions in identity management systems


# Phase 7: SSO and Group-Based Access (SAML Federation)

## Objective
Demonstrate how Active Directory security groups can be used to control access to a SaaS application using Microsoft Entra ID Single Sign-On (SSO) with SAML.

---

## What I Implemented
- Configured a SAML-based Single Sign-On (SSO) integration with GitHub Enterprise
- Used Microsoft Entra ID as the Identity Provider (IdP)
- Assigned access using an Active Directory security group (`SG-SecurityOperations-Cloud`)
- Verified that group membership controlled application access
- Tested user authentication through a federated login flow

---

## Access Flow Overview

1. User is created in Active Directory  
2. User is added to a security group (`SG-SecurityOperations-Cloud`)  
3. Group is synchronized to Microsoft Entra ID  
4. Entra ID is configured as the Identity Provider (IdP)  
5. User authenticates to GitHub using SAML SSO  
6. Access is granted based on group membership  

---

## Configuration Summary

### Microsoft Entra ID Setup
- Configured GitHub Enterprise as an Enterprise Application
- Enabled SAML-based Single Sign-On
- Mapped user attributes for authentication (UPN-based identity)

### Access Control Model
- No direct user assignment to the application
- Access controlled through security group membership
- Centralized access management through Active Directory

---

## Verification

### Application Assignment via Group
![GitHub Enterprise Group Assignment](lab1-01-entra-github-app-assignment.png)

### SAML Authentication Flow
![SAML SSO Login Verification](lab1-03-github-saml-sso-success.png)

### Successful Federated Login
User successfully authenticated to GitHub using Microsoft Entra ID SSO.

---

## What I Learned
- How SAML SSO enables authentication between Entra ID and SaaS applications
- How security groups simplify SaaS access management
- How identity federation works in a hybrid environment
- How IAM teams reduce manual account provisioning using centralized access controlengine alongside the SAML framework. SCIM uses background API hooks to automatically create, update, and completely de-provision the user's workspace inside GitHub in real-time based on their Entra security group compliance status, ensuring zero stale accounts exist during employee offboarding.
