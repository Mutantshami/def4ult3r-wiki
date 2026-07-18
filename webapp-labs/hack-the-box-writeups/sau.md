---
description: 'Difficulty: Easy OS: Linux'
---

# Sau

## Overview

Sau is an Easy Linux machine that involves exploiting a Server-Side Request Forgery (SSRF) in Request Baskets, chain-loading it to exploit a Command Injection vulnerability in a localized Maltrail instance, and abusing passwordless sudo permissions on systemctl to achieve root command execution.

## Enumeration

### Nmap Scan

We begin by scanning the target host with Nmap:

![Nmap scan output showing ports 22, 80 (filtered), and 55555 open](<../../.gitbook/assets/Unknown image (453)>)

The scan reveals:

* **Port 22:** OpenSSH 8.4p1 Debian
* **Port 80:** filtered HTTP (Nginx/Apache/etc. running locally)
* **Port 55555:** open, hosting an unknown web application

### Request Baskets (Port 55555)

Visiting the web page on port 55555 shows a Request Baskets dashboard running version 1.2.1:

![Request Baskets interface running version 1.2.1](<../../.gitbook/assets/Unknown image (454)>)

This version is vulnerable to CVE-2023-27163, a Server-Side Request Forgery (SSRF) vulnerability. This allows us to proxy external traffic through the basket server to reach internal network resourcesâ€”specifically, the filtered port 80.

## Foothold

### Exploiting Request Baskets SSRF (CVE-2023-27163)

We prepare a proof-of-concept shell script to automate the creation of a basket with custom routing rules:

![Exploit directory containing the CVE-2023-27163 PoC](<../../.gitbook/assets/Unknown image (455)>)

We execute the script to configure a new proxy basket `bjwryw` pointing internally to the localhost:

![Running the SSRF exploit to create the proxy basket](<../../.gitbook/assets/Unknown image (456)>)

We authorize access to our basket settings:

![Token verification prompt to authorize access to basket bwycct](<../../.gitbook/assets/Unknown image (457)>)

Inside the basket configuration settings, we set the **Forward URL** to `http://127.0.0.1:80/` (the target's local port 80) and check the **Proxy Response** option. This forces the basket to relay target responses back to us:

![Configuring proxy basket settings to point to target loopback on port 80](<../../.gitbook/assets/Unknown image (458)>)

![Proxy basket successfully created](<../../.gitbook/assets/Unknown image (459)>)

Now, accessing the basket URL `http://10.10.11.224:55555/bjwryw` forwards us directly to the hidden service on port 80, which is running **Maltrail v0.53**:

![Maltrail v0.53 interface loaded via the SSRF proxy basket](<../../.gitbook/assets/Unknown image (460)>)

## Exploitation

### Maltrail v0.53 Command Injection

Maltrail v0.53 is vulnerable to Command Injection via the `username` parameter on its login interface. We use a Python exploit script to automate the injection and execute a reverse shell:

![Executing the Maltrail v0.53 exploit payload](<../../.gitbook/assets/Unknown image (461)>)

We set up our Netcat listener and successfully capture the incoming connection:

![Receiving initial reverse shell connection on our listener](<../../.gitbook/assets/Unknown image (462)>)

We verify our user identity as `puma` and retrieve the user flag from `/home/puma/user.txt`:

![Verifying shell user identity and reading user.txt](<../../.gitbook/assets/Unknown image (463)>)

## Privilege Escalation

### Checking Sudo Privileges

We run `sudo -l` to check the current user's passwordless execution rights:

![sudo -l output showing sudo rights for systemctl status trail.service](<../../.gitbook/assets/Unknown image (464)>)

The output shows that the user `puma` can run `/usr/bin/systemctl status trail.service` as `root` without a password.

### Systemctl Pager Escape to Root

Since `systemctl status` invokes a pager (usually `less` or `more`) to display service logs when the terminal is not fully configured, we can escape the pager program to run arbitrary commands.

We execute the command:

```bash
sudo /usr/bin/systemctl status trail.service
```

Once inside the pager view, we type `!/bin/bash` and press Enter to escape:

![Escaping systemctl status pager to spawn a root bash shell](<../../.gitbook/assets/Unknown image (465)>)

The pager executes our command as the calling user (`root`), dropping us into a root command line session.

We navigate to the root directory and retrieve the root flag:

![Navigating to root home and reading root.txt](<../../.gitbook/assets/Unknown image (466)>)

We have successfully compromised the Sau machine!

![Hack The Box completion screen for Sau](<../../.gitbook/assets/Unknown image (467)>)
