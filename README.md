# Enterprise Hybrid Identity Lifecycle & Directory Synchronization Lab

## Project Overview
This project demonstrates the implementation of a production-grade hybrid identity architecture, bridging an on-premises Active Directory Domain Services (AD DS) infrastructure with a Microsoft 365 / Microsoft Entra ID cloud tenant. 

Designed to mirror strict financial sector security baselines (such as those utilized by Charles Schwab), this architecture enforces data minimization via scoped Organizational Unit (OU) filtering, utilizes automated lifecycle scripting, and implements bidirectional password mechanics via Password Hash Synchronization (PHS) and Password Writeback.

## Architectural Design
The architecture establishes a secure synchronization pipeline between a local Windows Server 2022 Domain Controller and Microsoft Entra ID via the Microsoft Entra Connect engine.

* **Source of Truth:** On-Premises Active Directory (`sc300lab.com`)
* **Target Directory:** Microsoft Entra ID Tenant
* **Identity Provisioning:** Automated via Parameterized PowerShell
* **Synchronization Scope:** Restricted exclusively to an isolated Security/Sync OU

---

## Phase 1: Automated On-Premises "Joiner" Provisioning
To align with corporate identity standards and eliminate manual administrative overhead, a parameterized PowerShell script was developed to handle the bulk ingestion of standard and specialized organizational personas into a dedicated `Sync_Users` OU.

The script dynamically appends the alternative routable User Principal Name (UPN) suffix to ensure zero mapping collisions upon cloud ingestion.

```powershell
# [Paste your working PowerShell provisioning script here]
