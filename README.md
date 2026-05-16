# Enterprise Hybrid Identity Lifecycle & Directory Synchronization Lab

## Project Overview
This project demonstrates the implementation of a production-grade hybrid identity architecture, bridging an on-premises Active Directory Domain Services (AD DS) infrastructure with a Microsoft 365 / Microsoft Entra ID cloud tenant. 

Designed to mirror strict corporate security baselines, this architecture enforces data minimization via scoped Organizational Unit (OU) filtering, utilizes automated lifecycle scripting, and implements bidirectional password mechanics via Password Hash Synchronization (PHS) and Password Writeback.

## Architectural Design
The architecture establishes a secure synchronization pipeline between a local Windows Server 2022 Domain Controller and Microsoft Entra ID via the Microsoft Entra Connect engine.

* **Source of Truth:** On-Premises Active Directory (`sc300lab.com`)
* **Target Directory:** Microsoft Entra ID Tenant
* **Identity Provisioning:** Automated via Parameterized PowerShell
* **Synchronization Scope:** Restricted exclusively to an isolated Security/Sync OU

---

## Phase 1: Automated On-Premises "Joiner" Provisioning
To align with corporate identity standards and eliminate manual administrative overhead, a parameterized PowerShell script was developed to handle the bulk ingestion of standard and specialized organizational personas into a dedicated `Sync_Users` OU.

The script dynamically appends the alternative routable User Principal Name (UPN) suffix to ensure zero mapping collisions upon cloud ingestion, and checks for account existence to guarantee script idempotency.

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
![PowerShell Provisioning Outputs](01_powershell_user_provisioning.png)
![On-Premises AD Scoped OU Contents](02_active_directory_sync_ou.png)
![AD Domains and Trusts Alternative UPN Configuration](03_alternative_upn_suffix.png)


## Phase 2: Scoped Directory Synchronization (Data Minimization)
Rather than executing a standard "Express" configuration—which risks exporting unroutable service accounts, local computer discovery logs, and default administrative paths to the public cloud—a **Custom Installation** of Microsoft Entra Connect was engineered.
![Scoped OU Filtering Configuration](04_entra_connect_ou_filtering.png)
![Optional Features Writeback Configuration](05_entra_connect_password_writeback.png)

### Security Baseline Configuration:
1. **Authentication Engine:** Password Hash Synchronization (PHS) enabled to safely process cryptographic password representations.
2. **Scoped OU Filtering:** Disabled root domain broad-sync. Configured the engine to strictly evaluate and synchronize the `Sync_Users` OU folder only.
3. **Bidirectional Password Lifecycle:** Enabled **Password Writeback** to support Self-Service Password Reset (SSPR) and maintain password parity between the cloud and the secure local perimeter.

### Verification: Scoped Sync Pipeline Configuration
![Scoped OU Filtering Configuration](04_scoped_ou_filtering.png)
![Optional Features Writeback Configuration](05_optional_features_writeback.png)

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
