# Active Directory Help Desk Lab on AWS

A fake company network built on AWS: a Windows Server domain controller, a domain-joined employee PC, and 5 common help desk tickets worked and documented in Jira.

**Stack:** AWS EC2 · Windows Server 2025 · Active Directory · DNS · PowerShell · Jira

```
My Mac → RDP (my IP only) → CORPDOMAINCTRL (Domain Controller) → corp.local
                                        ↑
                          HR-DESKTOP01 (employee PC)
```

## 1. Build the domain controller

Launched a Windows Server on EC2 (`t3.small`, RDP locked to my IP), then used PowerShell to install Active Directory and create the `corp.local` domain.

```
Rename-Computer -NewName "CORPDOMAINCTRL" -Restart
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "corp.local" -InstallDns
```

![Instance running](Screenshot1.jpg)
![Connected over RDP](Screenshot2.jpg)
![AD installed](Screenshot3.jpg)
![Domain login](Screenshot4.jpg)
![Domain verified](Screenshot5.jpg)

## 2. Set up the company

Created departments (OUs), one employee in each, and a security group per department. Everyone starts as a regular user with no admin rights.

| Department | Employee | Group |
|---|---|---|
| HR | Tyler Anderson | HR-Staff |
| IT | Madison Brooks | IT-Staff |
| Sales | Daniel Rivera | Sales-Staff |

```
New-ADOrganizationalUnit -Name "HR"
New-ADUser -Name "Tyler Anderson" -SamAccountName "tanderson" -Path "OU=HR,DC=corp,DC=local" -AccountPassword (Read-Host -AsSecureString) -Enabled $true
New-ADGroup -Name "HR-Staff" -GroupScope Global -Path "OU=HR,DC=corp,DC=local"
Add-ADGroupMember -Identity "HR-Staff" -Members "tanderson"
```

![Departments](Screenshot6.jpg)
![Employees](Screenshot7.jpg)
![Groups](Screenshot8.jpg)

## 3. Join an employee PC to the domain

Launched a second instance, allowed traffic between the two inside my VPC (`172.31.0.0/16`), pointed the PC's DNS at the domain controller, and joined it to the domain. Gave HR-Staff Remote Desktop access, then logged in as Tyler.

```
Set-DnsClientServerAddress -InterfaceIndex (Get-NetAdapter).ifIndex -ServerAddresses 172.31.3.229
Add-Computer -DomainName "corp.local" -NewName "HR-DESKTOP01" -Credential CORP\Administrator -Restart
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "CORP\HR-Staff"
```

![Both instances](Screenshot9.jpg)
![DNS test](Screenshot10.jpg)
![Domain joined](Screenshot11.jpg)
![Logged in as Tyler](Screenshot12.jpg)

## 4. Help desk tickets

Each ticket was logged in Jira, worked, verified, and closed.

### Ticket 1: Password reset
Tyler forgot his password. Reset it in AD and confirmed he could log in.

```
Set-ADAccountPassword -Identity tanderson -Reset -NewPassword (Read-Host -AsSecureString)
```

![Ticket opened](Screenshot13.jpg)
![Password reset](Screenshot14.jpg)
![Ticket closed](Screenshot15.jpg)

### Ticket 2: Account lockout
Set a lockout policy (5 tries, 15 min), locked Tyler out with bad passwords, then confirmed and unlocked him.

```
Set-ADDefaultDomainPasswordPolicy -Identity corp.local -LockoutThreshold 5 -LockoutDuration 00:15:00 -LockoutObservationWindow 00:15:00
Unlock-ADAccount -Identity tanderson
```

![Lockout error](Screenshot16.jpg)
![Unlocked](Screenshot17.jpg)
![Ticket closed](Screenshot18.jpg)

### Ticket 3: New hire
Created Jasmine Carter in Sales with her title, a temp password she has to change at first login, and Sales-Staff membership.

![New user created](Screenshot19.jpg)
![Ticket closed](Screenshot20.jpg)

### Ticket 4: Access request (denied)
Daniel in Sales asked for access to the HR shared folder. Checked the share (HR-Staff only) and his groups (Sales only). HR files are confidential and his role doesn't need them, so I denied it and pointed him to manager approval.

![Access check](Screenshot21.jpg)
![Ticket closed](Screenshot22.jpg)

### Ticket 5: Offboarding
Madison left the company. Disabled her account, removed her from IT-Staff, added a note, and moved her to a Disabled Users OU. Disabled instead of deleted so the account is kept for records.

![Offboarded](Screenshot23.jpg)
![Ticket closed](Screenshot24.jpg)

![All tickets done](Screenshot25.jpg)

## Issues I ran into

- **"Install-WindowsFeature not recognized":** I had opened the wrong PowerShell. Switched to Windows PowerShell as admin.
- **RDP error 0x204:** while adding the VPC rule, my RDP rule got overwritten. Re-added RDP from my IP.
- **Public IP changes:** every stop/start gives a new IP, so I download a fresh RDP file each session.

---

Built by [Jordan Drake](https://github.com/jdrakegit) · [LinkedIn](https://www.linkedin.com/in/jordan-drake-a95471397)
