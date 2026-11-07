# Active Directory Bulk User Provisioning Pipeline

A production-grade PowerShell automation pipeline that consumes structured CSV data to rapidly provision multi-departmental user accounts in Active Directory. Features dynamic OU targeting, programmatic object schema compliance, and secure automated generation of randomized temporary passwords.

## Architectural Overview

##  Infrastructure Environment
* **Domain Context:** `corp.local` (Windows Server Functional Domain Mode)
* **Management Tools:** Active Directory Module for Windows PowerShell (RSAT)
* **Automation Scope:** Deployment targeting distinct OUs (`OU=Marketing,DC=corp,DC=local` and `OU=HR Department,DC=corp,DC=local`).

##  Schema Definition (`employees_sample.csv`)

The data pipeline relies on a strict header blueprint layout:

```csv
FirstName,LastName,SamAccountName,UserPrincipalName,Department,Title,OU
James,Smith,jsmith,jsmith@corp.local,Marketing,Marketing Director,"OU=Marketing,DC=corp,DC=local"
Emily,Clark,eclark,eclark@corp.local,HR,HR Director,"OU=HR Department,DC=corp,DC=local"
```

## Automated Deployment Engine

This execution pipeline reads files from `C:\ADImport\employees.csv`, handles dynamic runtime routing, and logs onboarding parameters safely.

```powershell
Import-Module ActiveDirectory; \(csvPath = "C:\ADImport\employees.csv"; \)logPath = "C:\ADImport\temp_passwords.txt"; users = Import-Csv -Path csvPath; foreach (user in users) { Add-Type -AssemblyName System.Web; \$tempPassword = [System.Web.Security.Membership]::GeneratePassword(12, 2); securePassword = ConvertTo-SecureString tempPassword -AsPlainText -Force; targetOU = user.OU; if (user.Department -eq "HR") targetOU = "OU=HR Department,DC=corp,DC=local" } \$userParams = @{ GivenName = user.FirstName; Surname = user.LastName; Name = "(user.FirstName) (user.LastName)"; SamAccountName = user.SamAccountName; UserPrincipalName = user.UserPrincipalName; Department = user.Department; Title = user.Title; Path = targetOU; AccountPassword = securePassword; ChangePasswordAtLogon = true; Enabled = true }; try { if (Get-ADUser -Filter "SamAccountName -eq '(user.SamAccountName)'") { Write-Warning "User (user.SamAccountName) already exists. Skipping." } else { New-ADUser @userParams; Write-Host "Successfully created: (user.SamAccountName)" -ForegroundColor Green; "(user.SamAccountName) temporary password: tempPassword" | Out-File -FilePath logPath -Append } } catch { Write-Error "Failed to create user \$(user.SamAccountName). Error: _" } }; Write-Host "Done! Passwords saved to \$logPath" -ForegroundColor Cyan
```

##  Production Security Compliance
* **Cryptographic Constraints:** Autogenerates high-entropy 12-character alpha-numeric credentials.
* **Identity Hygiene:** Sets `ChangePasswordAtLogon = $true` forcing immediate remediation upon initial domain authentication.
* **Error Resilience:** Embedded `try/catch` runtime logging preventing transaction interruptions from duplicate records.
