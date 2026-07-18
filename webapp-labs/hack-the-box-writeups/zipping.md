---
description: >-
  Difficulty: Medium OS: Linux (Ubuntu) Attack Path: Nmap → Symlink ZIP LFI →
  SQL Injection → MySQL file write → RCE as rektsu → sudo stock binary → shared
  library hijack → root
---

# Zipping

## Overview

Zipping is a Medium Linux box with a creative two-phase attack. The web application accepts ZIP uploads containing a PDF but we can abuse this by creating a symlink inside the ZIP that points to system files. This gives us Local File Inclusion to read PHP source code. Inside the source we find a SQL injection vulnerability in the shop which we use to write a PHP file to disk via MySQL. That file becomes our webshell and gives us a shell as rektsu. Finally, rektsu can run a custom binary as root via sudo - and that binary loads a shared library from rektsu home directory which we control. We compile a malicious .so file and get root.

## Enumeration

### Nmap

Scanning with: nmap -T4 -A -v 10.10.11.229

![nmap scan - port 22 ssh and port 80 apache](<../../.gitbook/assets/Unknown image (346)>)

Open ports:

* Port 22: SSH - OpenSSH 9.0p1 Ubuntu
* Port 80: HTTP - Apache httpd 2.4.54 (Ubuntu)

## Web Application - ZIP Upload Feature

The website at http://zipping.htb has a Work With Us page where you upload a ZIP file. The ZIP must contain a PDF resume. The server extracts it and gives you a link to access the uploaded file.

![Work With Us page - ZIP upload with link to uploaded file](<../../.gitbook/assets/Unknown image (347)>)

After upload the server shows the path to your uploaded file at: uploads/hash/filename.pdf

## Phase 1 - Symlink LFI via ZIP Upload

### The Trick

A ZIP file can store symlinks - files that just point somewhere else. When the server unzips and serves the symlink, it reads whatever the symlink points to. This gives us Local File Inclusion.

### Steps

Create a symlink pointing at /etc/passwd, then zip it with the --symlinks flag:

![Creating symlink and zipping - ln -s and zip --symlinks commands](<../../.gitbook/assets/Unknown image (348)>)

The key here is --symlinks flag. Without it, zip would follow the symlink and copy the actual target file. With it, the symlink itself is stored inside the ZIP.

Upload the ZIP on the Work With Us page, then visit the link given after upload:

![Browser showing /etc/passwd contents via symlink LFI - rektsu user visible](<../../.gitbook/assets/Unknown image (349)>)

The server returned the contents of /etc/passwd instead of a PDF. We can see user rektsu with home at /home/rektsu and shell /bin/bash.

Use the same trick to read PHP source files - point the symlink at /var/www/html/shop/functions.php to find the SQL injection.

## Phase 2 - SQL Injection to Webshell

### Finding the SQLi

Reading the PHP source reveals the shop product page builds a raw SQL query using user input without sanitization. The product\_id field in the cart POST request is injectable.

Intercept the add-to-cart request in Burp Suite:

![Burp Suite - shop page intercepted, PHPSESSID cookie visible](<../../.gitbook/assets/Unknown image (350)>)

### Writing a File via MySQL

MySQL has an INTO OUTFILE feature that writes query results to a file on disk. We inject a SELECT that writes a PHP file to a web-accessible location.

The payload is injected into the product\_id parameter. Burp Inspector decodes it:

![Burp Inspector - SQL injection payload writing PHP file decoded](<../../.gitbook/assets/Unknown image (351)>)

The decoded payload writes a small PHP eval stub into /var/lib/mysql/p.php

### Executing the Webshell

The shop uses a page= parameter to include PHP files by path. We point it at our written file and pass a system command via the request parameter:

![Burp - GET request triggering the webshell with bash reverse shell command](<../../.gitbook/assets/Unknown image (352)>)

This triggers our file, which runs the bash reverse shell command back to our listener on port 1234.

## Shell as rektsu

Start the listener before triggering:

```
nc -lvnp 4444
```

We catch a shell as rektsu. Exploring the system we see /tmp has bash, libcounter.c and uploads directories.

![netcat listener - rektsu shell received, exploring /var/www/html/shop and /tmp](<../../.gitbook/assets/Unknown image (353)>)

The web root shows standard PHP shop files. /tmp has our uploaded files plus a libcounter.c file already there.

## Privilege Escalation - Shared Library Hijack

### sudo -l

```
sudo -l
```

Output: User rektsu may run the following commands on zipping: (ALL) NOPASSWD: /usr/bin/stock

rektsu can run /usr/bin/stock as root with no password. Run it:

```
sudo /usr/bin/stock
```

It asks for a password. The password is: St0ckM4nager

After entering the password it shows a stock management menu. It is a compiled binary. Using strings or ltrace on it reveals it tries to load a shared library at:

```
/home/rektsu/.config/libcounter.so
```

That path is inside rektsu home directory - which we fully control.

### Writing the Malicious Shared Library

A shared library with a constructor function runs its code the moment the library is loaded - before main() even starts. So when root runs the binary and it loads our .so file, our code executes as root.

Write a malicious libcounter.c:

```
// gives us a root shell when loaded
#include stdio.h
#include stdlib.h
#include unistd.h

void __attribute__((constructor)) pwn() {
    setuid(0);
    setgid(0);
    system("/bin/bash");
}
```

Compile it and place it in the expected path:

```
gcc -shared -o /home/rektsu/.config/libcounter.so -fPIC libcounter.c
```

![Compiling libcounter.so in /tmp - gcc command and ls showing the file](<../../.gitbook/assets/Unknown image (354)>)

The compile runs from /tmp where libcounter.c exists. The output goes directly to /home/rektsu/.config/libcounter.so

### Getting Root

Now run the binary as sudo:

```
sudo /usr/bin/stock
```

The binary loads our malicious .so - constructor fires - we get a root shell immediately. Navigate to /root and read the flag:

![root shell - cat /root/root.txt showing the flag](<../../.gitbook/assets/Unknown image (354)>)

```
cat /root/root.txt
973c6f1eaaafb587b92924e9c2a05b33
```

Rooted!

## Summary

| Step           | Detail                                                     |
| -------------- | ---------------------------------------------------------- |
| Nmap           | Port 22 SSH + Port 80 Apache                               |
| ZIP Upload     | Symlink attack - LFI of /etc/passwd and PHP source         |
| LFI            | Read shop source code, found SQL injection                 |
| SQLi           | product\_id param into MySQL INTO OUTFILE                  |
| Webshell       | page= param includes written .php file, RCE as rektsu      |
| sudo           | /usr/bin/stock NOPASSWD                                    |
| Library Hijack | libcounter.so in \~/.config loaded by stock binary as root |

***

## Key Takeaways

* ZIP symlink attacks: always test ZIP upload endpoints with symlinks pointing to /etc/passwd or /etc/hosts
* The --symlinks flag is essential - without it zip copies the file contents instead of the link
* Reading PHP source via LFI is far more powerful than blind fuzzing
* MySQL INTO OUTFILE can write arbitrary files to disk if secure\_file\_priv allows it
* sudo shared library hijack: if a SUID or sudo binary loads a .so from a user-controlled path, you get code execution as root
* Constructor functions in C shared libraries run before main - perfect for this technique

&#x20;_Platform: Hack The Box_
