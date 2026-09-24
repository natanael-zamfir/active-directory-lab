# 🪪 Active Directory Identity Lifecycle Lab: Joiner, Leaver, Compromise
In this lab I worked on Active Directory, learning the User Identity Lifecycle for 3 scenarios.

**Scenario 1, onboarding.** A new starter, Grace Hopper, joins Finance as an Analyst. I created her account and two role groups.

**Scenario 2, offboarding.** Grace leaves. I disabled her account, reset the password, removed her groups, added a note, and moved her to a Quarantine organisational unit.

**Scenario 3, suspected compromise.** An alert fires on Kevin Brown in Sales: failed logons, then a success, then changes to his account. I contained it: disable, reset, document, quarantine.

I built the lab in AWS on a `c7i-flex.large` EC2 instance, running Active Directory Domain Services for the domain `cyberlab.local`. I worked through the Active Directory console and checked the Security event logs.

---

## Events I worked through

| Event ID | Meaning |
| --- | --- |
| 4720 | User account created |
| 4722 | User account enabled |
| 4724 | Administrator reset an account's password |
| 4725 | User account disabled |
| 4726 | User account deleted |
| 4728 / 4729 | Member added to / removed from a global security group |
| 4738 | User account changed |
| 4624 | Successful logon |
| 4625 | Failed logon (NTLM and interactive only) |
| 4771 | Kerberos pre-authentication failed |
| 4672 | Special privileges assigned to a new logon |

---

## Setup

Domain controller promoted and verified:

```
DNSRoot        NetBIOSName DomainMode
cyberlab.local CYBERLAB    Windows2016Domain

Name IPv4Address IsGlobalCatalog
DC01 172.31.40.1            True
```

<img width="620" src="https://github.com/user-attachments/assets/dd5f7fd0-8a39-4328-ba2f-afe0c234ba64" />

Promotion (turning the plain Windows server into a domain controller) points DNS at the server itself, so I added the AWS resolver (`169.254.169.253`, the built-in AWS DNS) as a forwarder to reach the internet:

```powershell
Add-DnsServerForwarder -IPAddress 169.254.169.253 -PassThru
```

Auditing had to be turned on first, because Windows does not create these events retroactively:

```
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Special Logon" /success:enable
auditpol /set /subcategory:"Account Lockout" /success:enable /failure:enable
auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable
```

<img width="620" src="https://github.com/user-attachments/assets/4b3c316d-a7b7-4fca-b32e-279377739273" />

Organisational units:

```
CyberLab
├── IT
├── Finance
├── HR
├── Sales
├── ServiceAccounts
└── Quarantine
```

<img width="560" src="https://github.com/user-attachments/assets/81281eb4-4064-41c4-b09f-21a0c16e197c" />

<img width="460" src="https://github.com/user-attachments/assets/7cce61d6-d9b6-4a0c-b98a-b559257cda64" />

---

## Scenario 1: Onboarding

I created two empty groups in the Finance organisational unit to replicate a real life setup, `Finance-Users` and `Finance-ReadOnly`, to later show these being added and removed. Both are Security groups rather than Distribution, since a Distribution group is only a mailing list and cannot be given access to anything.

New user via ADUC (Active Directory Users and Computers), `dsa.msc`:

<img width="560" src="https://github.com/user-attachments/assets/e64cd99c-649b-4b2c-ab68-4c196abc4891" />

<img width="560" src="https://github.com/user-attachments/assets/d8634c2b-0451-4c48-a5eb-2a3022815248" />

Temporary password with **User must change password at next logon** ticked:

<img width="420" src="https://github.com/user-attachments/assets/4f8a6583-2a14-4eb5-ae66-d1f5269ce0e2" />

<img width="560" src="https://github.com/user-attachments/assets/863719b3-3f7f-45c7-97ec-d136eb85edb5" />

Job title and department on the **Organization** tab:

<img width="400" src="https://github.com/user-attachments/assets/2ccbd42e-f43a-478b-bee0-12dc536db5fd" />

<img width="460" src="https://github.com/user-attachments/assets/1d72bfff-467a-4c76-a9e0-b4b6383e35f9" />

I mistyped a group name on purpose to see what **Check Names** does. It resolves the name against the directory before committing, so a typo fails rather than silently creating something:

<img width="420" src="https://github.com/user-attachments/assets/ce2550e6-c7ea-44a6-a80e-87692140a886" />

Final membership:

<img width="440" src="https://github.com/user-attachments/assets/73d8861a-3335-4c52-bf24-c3b69d373ef5" />

### Events

Event Viewer, Security log, filtered on `4720,4722,4728`:

<img width="620" src="https://github.com/user-attachments/assets/922b9799-f692-4c60-ad85-d3b99f00e062" />

4720, account created:

<img width="520" src="https://github.com/user-attachments/assets/b73220d7-d807-405d-a74a-0bd31d063c67" />

4722, account enabled:

<img width="520" src="https://github.com/user-attachments/assets/c779a300-6752-4020-99b0-eb66cad0a497" />

4728, one per group:

<img width="560" src="https://github.com/user-attachments/assets/214753d6-a969-4c9b-bd8b-a6d70d75b1e1" />

<img width="560" src="https://github.com/user-attachments/assets/b1906b5b-c626-464d-86d9-16587e3db210" />

Subject shows who made the change. 4720 then 4722 then 4728 is what a legitimate joiner looks like, and also exactly what an attacker creating a persistence account looks like. The difference is context: who made the change, at what hour, and into which group.

---

## Scenario 2: Offboarding

Order: disable, reset password, remove groups, document, move to Quarantine.

Disable first, because it is instant and stops new authentication (4725):

<img width="560" src="https://github.com/user-attachments/assets/86c5cd07-22cb-4847-85d4-712bda35349d" />

Password reset (4724):

<img width="420" src="https://github.com/user-attachments/assets/f7e4c283-4d22-4897-9ad8-71fb4804dca7" />

Groups removed (4729). `Domain Users` will not remove because it is the primary group, which is expected:

<img width="420" src="https://github.com/user-attachments/assets/a7eaede1-1b81-4997-bf60-2b2a35d3c377" />

Description added so anyone looking at the account later knows why it was disabled:

<img width="420" src="https://github.com/user-attachments/assets/fadf971d-a2fd-4514-9d15-ea8bea9c2abb" />

Moved to Quarantine:

<img width="360" src="https://github.com/user-attachments/assets/8c3d2fb4-f5a1-4b2f-ac07-c0794622e6c5" />

### Why disable instead of delete

The account's SID (Security Identifier) is what actually appears in file permissions, application access lists and every historical log entry. The username is a label on top of it. Delete the account and old Security events stop resolving to a name, permissions become orphaned SIDs, and anything the person owned loses its owner. Disabling cuts access just as fast and keeps all of it.

Events, filtered on `4725,4724,4729,4738`:

<img width="620" src="https://github.com/user-attachments/assets/99610262-084e-4b69-b048-492fcc24e0d9" />

---

## Scenario 3: Suspected account compromise, Kevin Brown

### The assumption

The team monitors alerts in Microsoft Sentinel, which collects the Windows Security log from the domain controller.

An alert fires on `kevin.brown`, a Sales user: a run of failed logons, then a successful one within 3 minutes, then a change to his account shortly after. Around the same time the Sales manager emails to say Kevin's machine has been behaving oddly since he opened an invoice attachment.

### Setting up the account

Kevin Brown created in the Sales organisational unit:

<img width="620" src="https://github.com/user-attachments/assets/7bd4d360-be5b-490a-8e4a-bc5356c3f06d" />

To generate the failed logons I used `net use` with the wrong password:

<img width="620" src="https://github.com/user-attachments/assets/4cefa76b-33ba-466b-ad38-666274f04aff" />

Failures confirmed in the Security log as 4771:

<img width="620" src="https://github.com/user-attachments/assets/b17608ca-05e2-4f3b-87f8-27c774cf3b78" />

Then the correct password for the success, followed by the account change:

<img width="620" src="https://github.com/user-attachments/assets/93acee99-5847-4a5a-8024-2bfcf60c692e" />

```powershell
Set-ADUser kevin.brown -PasswordNeverExpires $true
```

### What the events show

```
4771   Kerberos pre-authentication failed, kevin.brown
4624   Successful logon
4738   Account changed: Don't Expire Password enabled
```

The failures came back as 4771 rather than 4625, because `net use` authenticates over Kerberos. 4625 only covers NTLM and interactive logons, so a brute force rule built on 4625 alone would miss this entirely. Fourteen failures in 45 seconds, then a success two minutes later, then the account change eight seconds after that.

The 4738 is what indicates a compromise rather than a user forgetting their password. Opening it shows the User Account Control value changing from `0x10` to `0x210`, with `'Don't Expire Password' - Enabled`:

<img width="620" src="https://github.com/user-attachments/assets/fb8d9e4c-69a1-4b67-858a-b7bd8c1f61bf" />

That flag is not something a Sales user sets on their own account, and it does one thing for an attacker: it stops the credential they have just stolen from expiring.

### Containment

| Step | Action | Event |
| --- | --- | --- |
| 1 | Disable the account | 4725 |
| 2 | Reset the password | 4724 |
| 3 | Add a description recording what happened | 4738 |
| 4 | Move to the Quarantine organisational unit | - |

**Disable first**, because it is instant and stops new logons.

**Then reset the password**, which kills the stolen credential.

**Then document and quarantine.** Description set to:

```
Account compromised 24.09.26 19:20 - failed logons then success, Don't Expire Password enabled by attacker. Account disabled, password reset. Endpoint investigation ongoing.
```

Not deleted, because the account is evidence.

Containment events:

<img width="620" src="https://github.com/user-attachments/assets/9e110571-120f-49a3-815f-171ae12a0518" />

```
4771   Failed Kerberos authentication
4624   Successful logon
4738   Don't Expire Password enabled
        |  Alert raised in Sentinel, account treated as compromised
4725   Account disabled
4724   Password reset
4738   Description updated
        |  Moved to Quarantine, retained for investigation
```

---

## Summary

Three scenarios through the full identity lifecycle. Two accounts end up in Quarantine: Grace as a leaver, Kevin pending investigation.

<img width="620" src="https://github.com/user-attachments/assets/c45515de-1779-4075-a7df-2d0ba29a91ec" />
