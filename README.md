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

<img width="620" alt="image" src="https://github.com/user-attachments/assets/bff5cb2b-af23-4093-8820-48996089c8eb" />

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

<img width="620" alt="image" src="https://github.com/user-attachments/assets/2c5101f4-25e6-48a6-aef8-28b1723bd92b" />


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


<img width="560" alt="image" src="https://github.com/user-attachments/assets/ec35403e-842b-471f-8076-6da4bd5984b2" />

<img width="460" alt="image" src="https://github.com/user-attachments/assets/f851d16b-510e-410d-a416-3f3955e72d1b" />

---

## Scenario 1: Onboarding

I created two empty groups in the Finance organisational unit to replicate a real life setup, `Finance-Users` and `Finance-ReadOnly`, to later show these being added and removed. Both are Security groups rather than Distribution, since a Distribution group is only a mailing list and cannot be given access to anything.

New user via ADUC (Active Directory Users and Computers), `dsa.msc`:

<img width="560" alt="image" src="https://github.com/user-attachments/assets/f19d24c0-f766-4f8e-848b-05dd0c11d8fd" />


<img width="560" alt="image" src="https://github.com/user-attachments/assets/c931296d-35aa-4930-9e36-ae14579e49ea" />


Temporary password with **User must change password at next logon** ticked:

<img width="420" alt="image" src="https://github.com/user-attachments/assets/9bc8bdd0-c202-45cf-807e-2630e5109d77" />

<img width="560" alt="image" src="https://github.com/user-attachments/assets/8ff3fb6e-aecf-4187-aa3f-2f2a19209ad2" />


Job title and department on the **Organization** tab:

<img width="400" alt="image" src="https://github.com/user-attachments/assets/2e34bd05-67cd-4f4d-b8fa-62e75abdcc2d" />

<img width="460" alt="image" src="https://github.com/user-attachments/assets/e266c625-f389-42d4-b81b-64d95b58b843" />

I mistyped a group name on purpose to see what **Check Names** does. It resolves the name against the directory before committing, so a typo fails rather than silently creating something:

<img width="420" alt="image" src="https://github.com/user-attachments/assets/57f49527-e8ae-4465-a3c1-4ba22f4875e5" />


Final membership:

<img width="440" alt="image" src="https://github.com/user-attachments/assets/2a9ee64b-9cb9-4020-8086-c8d483761622" />


### Events

Event Viewer, Security log, filtered on `4720,4722,4728`:

<img width="620" alt="image" src="https://github.com/user-attachments/assets/359a8dec-ef7c-4634-81a7-899d4061c231" />


4720, account created:

<img width="520" alt="image" src="https://github.com/user-attachments/assets/e10ec6e8-5cc7-44c6-8a8c-bb043c427382" />

4722, account enabled:

<img width="520" alt="image" src="https://github.com/user-attachments/assets/3e52af76-5b64-4129-9294-1110a55d7075" />


4728, one per group:

<img width="560" alt="image" src="https://github.com/user-attachments/assets/c17f0143-8a27-4085-b0d0-26741ace8917" />

<img width="560" alt="image" src="https://github.com/user-attachments/assets/37077a39-7b40-4b81-ab6e-ae50f56ae329" />

Subject shows who made the change. 4720 then 4722 then 4728 is what a legitimate joiner looks like, and also exactly what an attacker creating a persistence account looks like. The difference is context: who made the change, at what hour, and into which group.

---

## Scenario 2: Offboarding

Order: disable, reset password, remove groups, document, move to Quarantine.

Disable first, because it is instant and stops new authentication (4725):

<img width="560" alt="image" src="https://github.com/user-attachments/assets/c33252fb-579c-478b-aecb-39bff34ed47a" />

Password reset (4724):

<img width="420" alt="image" src="https://github.com/user-attachments/assets/a8991129-614d-4e5c-9843-3bdf64245b0a" />


Groups removed (4729). `Domain Users` will not remove because it is the primary group, which is expected:

<img width="420" alt="image" src="https://github.com/user-attachments/assets/6e31c4fd-6e2f-4a28-a95a-3af0bdca8890" />


Description added so anyone looking at the account later knows why it was disabled:

<img width="420" alt="image" src="https://github.com/user-attachments/assets/4c7efbea-3167-4f5c-900d-c4d1cabe87de" />


Moved to Quarantine:

<img width="360" alt="image" src="https://github.com/user-attachments/assets/8efb0b1b-b42a-4557-937b-00ba306dcfa2" />


### Why disable instead of delete

The account's SID (Security Identifier) is what actually appears in file permissions, application access lists and every historical log entry. The username is a label on top of it. Delete the account and old Security events stop resolving to a name, permissions become orphaned SIDs, and anything the person owned loses its owner. Disabling cuts access just as fast and keeps all of it.

Events, filtered on `4725,4724,4729,4738`:

<img width="620" alt="image" src="https://github.com/user-attachments/assets/a7d0b47e-bfe2-4309-a87d-0087f52d7624" />

---

## Scenario 3: Suspected account compromise, Kevin Brown

### The assumption

The team monitors alerts in Microsoft Sentinel, which collects the Windows Security log from the domain controller.

An alert fires on `kevin.brown`, a Sales user: a run of failed logons, then a successful one within 3 minutes, then a change to his account shortly after. Around the same time the Sales manager emails to say Kevin's machine has been behaving oddly since he opened an invoice attachment.

### Setting up the account

Kevin Brown created in the Sales organisational unit:

<img width="560" alt="image" src="https://github.com/user-attachments/assets/5a6eeee9-bf34-46b8-9b85-06918538605c" />


To generate the failed logons I used `net use` with the wrong password:

<img width="560" alt="image" src="https://github.com/user-attachments/assets/33f3b550-14df-42fc-81c1-a8a0bc2bf8b8" />


Failures confirmed in the Security log as 4771:

<img width="620" alt="Screenshot 2026-09-24 194812" src="https://github.com/user-attachments/assets/7ec638bf-1926-42f5-9b20-a8fbbd2060d9" />


Then the correct password for the success, followed by the account change:

<img width="620" alt="image" src="https://github.com/user-attachments/assets/a2aaffe7-c972-4bbe-8a6b-044200be4203" />


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

<img width="620" alt="image" src="https://github.com/user-attachments/assets/91038fbb-00e4-4f0d-97d0-fec28f2d1f8b" />


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

<img width="620" alt="image" src="https://github.com/user-attachments/assets/8076061b-7253-40b8-911a-2979ecdda7ca" />


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

<img width="560" alt="image" src="https://github.com/user-attachments/assets/0416284d-598f-4d04-8258-33f8a61479e4" />
