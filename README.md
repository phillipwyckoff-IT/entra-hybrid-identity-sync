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
<details>
```
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

![Password Writeback Configuration](05_entra_connect_password_writeback.png)

![Scoped OU Filtering Verification](04_scoped_ou_filtering.png)

![Optional Features Writeback Verification](05_optional_features_writeback.png)

## What I Learned
- How Entra Connect controls identity synchronization between on-prem and cloud
- The difference between Password Hash Sync and Password Writeback
- Why OU filtering is important in hybrid identity environments
- How configuration choices directly affect what identities appear in Microsoft Entra ID

## Phase 3: Cloud Ingestion & Identity Governance Verification
Following the completion of the synchronization engine configuration, a comprehensive cloud audit was performed within the Microsoft Entra Admin Center portal. 

All six specialized on-premises identities successfully registered in the cloud tenant with a directory authorization state of `On-premises sync enabled: Yes`.

![Entra ID User Cloud State](06_entra_cloud_user_verification.png)

### Hybrid Access Modification (Mover Scenario)
To simulate an active access modification ticket (the "Mover" phase of JML lifecycle management), an on-premises security group was engineered via administrative shell tools, and an identity was nested inside. 

To bypass the standard 30-minute background sync interval, an immediate manual delta synchronization cycle was forced via PowerShell to instantly enforce the access governance change across the hybrid boundary:

![PowerShell Forced Delta Sync Run](07_powershell_group_creation_delta_sync.png)
![Hybrid Group Cloud Sync Members Verified](08_hybrid_group_rbac_sync.png)

```powershell
# 1. Create an On-Premises Security Group inside the synchronized OU path
New-ADGroup -Name "SG-SecurityOperations-Cloud" `
            -GroupScope Global `
            -GroupCategory Security `
            -Path "OU=Sync_Users,DC=sc300lab,DC=com"

# 2. Dynamically modify user access profile by nesting identity inside group
Add-ADGroupMember -Identity "SG-SecurityOperations-Cloud" -Members "erostova"

# 3. Administrative Power-Move: Force immediate delta synchronization replication
Start-ADSyncSyncCycle -PolicyType Delta
```

## Bonus
### Advanced Lifecycle Testing: Bulk Account Suspension (SecOps Scenario)
To simulate an immediate incident response or bulk offboarding event, a secondary script was executed to instantly disable all six targeted identities at the directory level.

#### Suspension Script Execution:
![PowerShell Account Suspension Run](02a_powershell_account_suspension.png)

#### Target Directory Suspended State:
The downward arrows on the user objects visually confirm that the accounts are explicitly disabled across the domain:
![Active Directory Disabled State](02b_active_directory_disabled_users.png)

---

---

## Phase 3: Cloud Access Governance via Identity-as-Code (IdaC)
**Objective:** Programmatically orchestrate Zero-Trust access controls within the Microsoft Entra ID tenant using HashiCorp Terraform, eliminating manual UI configuration risk and enforcing continuous security posture baselines.

### 🛠️ Architectural Strategy & Design
Instead of performing manual click-ops inside the Azure portal, the tenant's security layer was codified using declarative **HashiCorp Configuration Language (HCL)**. 

To achieve production-grade consistency, the pipeline utilizes **Dynamic Data Sources** rather than hardcoding sensitive object GUIDs. Terraform dynamically queries the live directory at runtime to look up the newly synchronized on-premises identities (`alexadmin` and `vsupport`) using their User Principal Names (UPNs) and binds them directly to targeted access perimeters.

### 📝 Codebase Architecture

The deployment infrastructure is split cleanly into a dedicated `terraform/` directory:

#### 1. Provider Infrastructure Initialization (`provider.tf`)
Defines the required HashiCorp Microsoft Entra ID (`azuread`) engine hooks and establishes the Azure CLI interactive authentication bridge.
```hcl
terraform {
  required_providers {
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.48.0"
    }
  }
}

provider "azuread" {
  # Authenticated securely via active local Azure CLI session tokens
}
```
### 2. Core Zero-Trust Boundary Matrix (main.tf)
Defines environment variables, resolves dynamic object identities, and declares two distinct conditional access restrictions.
```hcl
# Global Tenant Context Variables
variable "tenant_domain" {
  type        = string
  default     = "scfun.onmicrosoft.com" # Target directory primary suffix
  description = "The target primary domain suffix for the Entra ID tenant."
}

# Dynamic Directory Identity Lookups (Data Sources)
data "azuread_user" "admin_user" {
  user_principal_name = "alexadmin@${var.tenant_domain}"
}

data "azuread_user" "vendor_user" {
  user_principal_name = "vsupport@${var.tenant_domain}"
}

# Zero-Trust Policy 01: Enforce MFA for Administrative Personas
resource "azuread_conditional_access_policy" "mfa_for_admins" {
  display_name = "SEC-CAP-01-MFA-For-Administrators"
  state        = "enabled"

  conditions {
    client_app_types = ["all"]
    
    applications {
      included_applications = ["All"]
    }

    users {
      included_users = [data.azuread_user.admin_user.object_id]
    }
  }

  grant_controls {
    operator          = "OR"
    built_in_controls = ["mfa"]
  }
}

# Zero-Trust Policy 02: Restrict Third-Party Vendor Access
resource "azuread_conditional_access_policy" "vendor_security_gate" {
  display_name = "SEC-CAP-02-Vendor-Security-Gate"
  state        = "enabled"

  conditions {
    client_app_types = ["all"]

    applications {
      included_applications = ["All"]
    }

    users {
      included_users = [data.azuread_user.vendor_user.object_id]
    }
  }

  grant_controls {
    operator          = "OR"
    built_in_controls = ["mfa"]
  }
}
```

Continuous Deployment Pipeline Workflow
Execution and enforcement were carried out inside a standard Infrastructure-as-Code pipeline via PowerShell:

Initialize the Working Directory:

PowerShell
terraform init
Downloads the required HashiCorp Microsoft Graph API integration plugins.

Generate the Execution Strategy (Dry Run):

PowerShell
terraform plan
Evaluates configuration state against active cloud tenant schema to preview changes before modification.

Enforce Target Posture:

PowerShell
terraform apply -auto-approve
🔍 Real-World Engineering Problem Remediation
The Obstacle: Initial orchestration deployment phases failed with an API BadRequest response from Microsoft Graph:

"Security Defaults is enabled in the tenant. You must disable Security defaults before enabling a Conditional Access policy."

The Root Cause Analysis: Greenfield Microsoft Entra ID tenants deploy with legacy un-customizable "Security Defaults" turned on natively. The Microsoft Graph API explicitly blocks custom conditional access engine evaluation if these underlying base settings are active.

The Solution: Executed an administrative override within the tenant overview configuration panel to safely disable Security Defaults, explicitly citing the transition to advanced granular Conditional Access policies.

Upon remediation, the pipeline immediately passed validation checks and successfully initialized both active rules.

📊 Verification & Visual Evidence
1. Command-Line Execution Output
Successful infrastructure provisioning via the local automation runtime, confirming both active security resources were pushed upstream cleanly:

2. Cloud Directory Status Validation
The Microsoft Entra ID administration panel confirms that both programmatic security wrappers are fully active, enforcing real-time Zero-Trust policies across the tenant:
---

## Phase 4: Securing Emergency Access (Break-Glass Monitoring)
**Objective:** Architect a continuous cloud monitoring telemetry pipeline to detect and alert on unauthorized utilization of the emergency break-glass account (`breakglass01`).

### Security Strategy
Emergency access accounts hold highly privileged global roles but are explicitly bypassed from Conditional Access MFA rules to prevent lockout during a widespread identity provider outage. Due to the high risk of account compromise, a zero-trust real-time monitoring solution was designed using Azure Monitor and Kusto Query Language (KQL).

### Detection Engineering (KQL Query)
The following query is deployed within an Azure Log Analytics Workspace, scheduled to run every 5 minutes with an alert threshold of `> 0` results:

```kusto
SigninLogs
| where UserPrincipalName =~ "breakglass01@scfun.onmicrosoft.com"
| where Status.errorCode == 0 
| project TimeGenerated, UserPrincipalName, IPAddress, Location, ClientAppUsed, ResultDescription
```
---

---

## Phase 5: Programmatic Directory Governance & Compliance Auditing
**Objective:** Engineer an automated compliance auditing utility using the modern Microsoft Graph API ecosystem to programmatically detect directory risk vectors, orphaned privileges, and external identity anomalies.

## 📊 Governance Strategy & Design Choices
In large-scale enterprise environments (particularly within regulated financial frameworks), manual portal inspections fail to meet continuous audit standards. This phase implements a scalable **Compliance-as-Code** reporting mechanism using a custom PowerShell pipeline designed around the modern **Microsoft Graph API core schema**.

The auditing logic intentionally bypasses legacy modules to target active Graph endpoints directly, evaluating all synchronized directory personas against three core enterprise compliance rules:

1. **Privileged Core Security Mapping (Rule 01):** Inspects high-value infrastructure identities (such as the emergency break-glass account `breakglass01`) to verify that explicit logging and premium structural licensing rules are intact. 
2. **Third-Party Boundary Control (Rule 02):** Flags synchronized external contractor/vendor identities (`vsupport`) to enforce mandatory quarterly access re-certification cycles.
3. **Automated Risk Profiling (Rule 03):** Dynamically parses object arrays to output a real-time risk classification matrix (`COMPLIANT`, `NON-COMPLIANT`, `REVIEW REQUIRED`), providing the IAM team with an instant data stream for risk remediation.

---

---

## Phase 6: Least-Privilege Enforcement via Privileged Identity Management (PIM)
**Objective:** Eliminate permanent standing administrative privileges within the cloud tenant by architecting a Just-In-Time (JIT) role elevation matrix to mitigate lateral movement risks and fulfill strict enterprise compliance frameworks (SOX/FINRA).

### 🔒 The Problem: Standing Privileges as an Attack Vector
In a standard legacy environment, administrative accounts often hold permanent global rights. If an administrative identity is compromised via credential harvesting or session hijacking at 3:00 AM, an adversary instantly gains full structural control over the directory before detection teams can respond.

### 🛡️ The Architecture: Zero Standing Access (ZSA)
To solve this, the synchronized administrative persona (`alexadmin`) was stripped of permanent active directory roles. Instead, the identity was onboarded into **Entra ID Privileged Identity Management (PIM)** and designated as **Eligible** rather than permanently assigned. 

Under normal operating conditions, the account possesses zero active cloud directory permissions.

### ⚙️ Just-In-Time (JIT) Activation Constraints
When operational requirements necessitate administrative intervention (e.g., identity lifecycle modifications or break-glass tasks), the administrator must explicitly request role activation through the PIM control engine.

The following governance policies were engineered into the activation workflow:

Step-Up Authentication: Forces a mandatory, explicit Multi-Factor Authentication (MFA) challenge at the time of the request, bypassing cached session tokens.

ITSM Ticket Mapping: Requires the engineer to input a valid corporate change management or help desk incident ticket number for auditable tracking.

Time-Bound Ephemeral Access: Access is bound to a strict, hard-coded limit of 2 hours. Once the window closes, the Entra ID core engine automatically drops the active assignment tokens and returns the identity to a zero-privilege resting state.

```text
[Normal State: Zero Active Roles] ──> [Operational Need] ──> [PIM Request Gate] ──> [MFA + Ticket Justification] ──> [2-Hour Active Window] ──> [Auto-Revocation]

---
### 💻 Compliance Pipeline Source (`05_entra_compliance_audit.ps1`)

The script is decoupled from native graphical front-ends, enabling it to run headless within scheduled enterprise automation tasks:

```powershell
# ==============================================================================
# PHASE 5: MICROSOFT GRAPH COMPLIANCE & IDENTITY LIFECYCLE AUDITING
# ==============================================================================
Write-Host "Querying Microsoft Graph API directly via Identity Core Core..." -ForegroundColor Cyan

# 1. Directly invoke the raw Graph API user metadata endpoints
$RawResponse = az rest --method get --url "[https://graph.microsoft.com/v1.0/users](https://graph.microsoft.com/v1.0/users)?`$select=displayName,userPrincipalName,id,assignedLicenses"

# 2. Parse the raw incoming JSON payload into structured PowerShell objects
$GraphData = ConvertFrom-Json $RawResponse
$TenantUsers = $GraphData.value

Write-Host "`n=== LIVE IAM COMPLIANCE AUDIT REPORT ===" -ForegroundColor Yellow
Write-Host "----------------------------------------------------------------------"

# 3. Process identity structures through the compliance verification engine
foreach ($User in $TenantUsers) {
    $RiskFlags = @()
    $Status = "COMPLIANT"
    
    # Audit Rule 01: Verify licensing integrity on administrative break-glass accounts
    if ($User.userPrincipalName -like "*breakglass*" -and ($User.assignedLicenses.Count -eq 0 -or $User.assignedLicenses -eq $null)) {
        $RiskFlags += "Privileged Break-Glass Account missing explicit license mapping"
        $Status = "NON-COMPLIANT"
    }
    
    # Audit Rule 02: Enforce lifecycle boundaries on external vendor personas
    if ($User.userPrincipalName -like "*vsupport*") {
        $RiskFlags += "External Vendor Account requires quarterly access recertification"
        $Status = "REVIEW REQUIRED"
    }

    # Stream the formatted audit record out to the host console
    [PSCustomObject]@{
        "Identity Name" = $User.displayName
        "UPN / Cloud ID" = $User.userPrincipalName
        "Audit Status"   = $Status
        "Risk Analysis"  = if ($RiskFlags) { $RiskFlags -join " | " } else { "No current anomalies detected" }
    } | Format-List
}
```
 📈 Target Telemetry Output Sample
When evaluated against the active hybrid tenant directory, the automated pipeline extracts, structures, and logs the directory health baseline into the following reporting schema for internal security compliance officers:
=== LIVE IAM COMPLIANCE AUDIT REPORT ===
----------------------------------------------------------------------

Identity Name : Alex Admin
UPN / Cloud ID: alexadmin@scfun.onmicrosoft.com
Audit Status  : COMPLIANT
Risk Analysis : No current anomalies detected

Identity Name : Emergency BreakGlass
UPN / Cloud ID: breakglass01@scfun.onmicrosoft.com
Audit Status  : NON-COMPLIANT
Risk Analysis : Privileged Break-Glass Account missing explicit license mapping

Identity Name : Vendor Support
UPN / Cloud ID: vsupport@scfun.onmicrosoft.com
Audit Status  : REVIEW REQUIRED
Risk Analysis : External Vendor Account requires quarterly access recertification

---

## Phase 7: Enterprise RBAC & SaaS Access Federation Loop (SAML 2.0)

### 📋 Objective & Business Case
In a modern enterprise environment, manual account creation within third-party Software-as-a-Service (SaaS) platforms introduces massive configuration drift, security technical debt, and human error. 

This phase closes the loop on our hybrid identity lifecycle. It demonstrates how an identity minted on-premises in Active Directory automatically inherits secure, policy-enforced Single Sign-On (SSO) access to an external SaaS ecosystem (**GitHub Enterprise**) based purely on their synchronized **Role-Based Access Control (RBAC)** security group memberships.

### 📐 Architectural Overview
The end-to-end identity lifecycle flows through three distinct platform boundaries without manual administrator intervention at the destination app:

1. **Inbound Identity Layer:** A user account (`erostova`) is provisioned on-premises and placed into the synchronized security group `SG-SecurityOperations-Cloud`.
2. **Directory Synchronization Layer:** Microsoft Entra Connect replicates both the user object and the group membership boundaries up to the Microsoft Entra ID cloud tenant.
3. **Outbound Federation Layer:** Microsoft Entra ID acts as the **Identity Provider (IdP)**, evaluates modern security guardrails, and passes a secure SAML 2.0 assertion token to **GitHub Enterprise** (the **Service Provider (SP)**), authorizing the end-user session.

### 🛠️ Configuration Blueprint

#### 1. Outbound SaaS Application Mapping
* Navigated to **Entra Admin Center** > **Enterprise Applications** > **GitHub Enterprise Cloud**.
* Configured SAML 2.0 single sign-on parameters, defining the unique Entity ID and Assertion Consumer Service (ACS) URL targeting the dedicated GitHub organization container (`phillipwyckoff-it`).
* Mapped the cryptographic token attributes, ensuring the user's Principal Name (UPN) scales natively to target the primary user identification field in the SaaS directory.

#### 2. Role-Based Access Assignment
* Avoided messy, manual user-by-user assignments within the cloud console.
* Assigned the synchronized on-premises security group **`SG-SecurityOperations-Cloud`** directly to the GitHub Enterprise application workspace.
* Result: Any user nested within this AD group automatically inherits the right to authenticate to the GitHub platform.

#### 3. Security Boundary Enforcement (MFA)
* Triggered strict authentication guardrails. When the un-registered hybrid user attempts their first Service Provider-initiated login, Entra ID intercepts the session, forcing registration and validation of **Microsoft Authenticator Multi-Factor Authentication (MFA)**.

---

### 🔍 Verification Metrics & Artifacts

The entire authentication pipeline was validated through a live, clean-room session audit. The visual evidence below validates that the cryptographic handshake and access boundaries are perfectly intact:

#### 1. Inbound-to-Outbound Access Assignment
* **Location:** Entra Admin Center > Enterprise Applications > GitHub > Users and Groups
* *Description:* Proves that the enterprise-scale access model is bound entirely to the synchronized security group tier rather than loose human accounts.
* `![Entra Application Assignment]([INSERT_LINK_TO_YOUR_lab1-01-entra-github-app-assignment.png])`

#### 2. Synchronized Policy Enforcement (MFA Challenge)
* **Location:** Mobile End-User Browser / Microsoft Authenticator App
* *Description:* Demonstrates modern identity security practices in action. The hybrid user (`erostova`) is successfully intercepted by our conditional access layers and challenged with a secure, 2-digit Microsoft Authenticator number-matching verification code before token issuance.
* `![Microsoft Authenticator MFA Verification]([INSERT_LINK_TO_YOUR_MFA_SCREENSHOT_HERE])`

#### 3. Successful IdP-to-SP Token Handoff
* **Location:** GitHub Federated Organization Landing Interface
* *Description:* Definitive proof that the SAML 2.0 assertion token passed successfully across tenant boundaries, routing Elena Rostova directly to the federated account-linking interface.
* `![GitHub SAML Landing Success]([INSERT_LINK_TO_YOUR_lab1-03-github-saml-sso-success.png])`

---

### 📝 Production Engineering Note: Authentication vs. Provisioning

During the final stage of end-user validation, the SAML 2.0 handshake successfully completed authentication (**AuthN**), but halted at the GitHub destination interface, prompting the user to create or link a standard GitHub account workspace. 

* **Architectural Root Cause:** SAML 2.0 is strictly an *Authentication* protocol—it proves *who* a user is via cryptographic claims, but it lacks the structural mechanics to *provision* a physical directory database workspace inside a destination SaaS application. GitHub operates on a "Linked Identity" architecture, meaning a destination object must exist to bind to the inbound token.
* **Enterprise Remediation:** In a true enterprise-scale production deployment, this lifecycle limitation is eliminated by deploying a **SCIM (System for Cross-domain Identity Management)** engine alongside the SAML framework. SCIM uses background API hooks to automatically create, update, and completely de-provision the user's workspace inside GitHub in real-time based on their Entra security group compliance status, ensuring zero stale accounts exist during employee offboarding.
