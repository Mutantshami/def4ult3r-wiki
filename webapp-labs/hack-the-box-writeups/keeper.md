---
description: 'Difficulty: Easy OS: Linux'
---

# Keeper



## Overview

Keeper is an Easy Linux machine from Hack The Box. The initial foothold involves finding an instance of Best Practical's Request Tracker (RT) running on port 80. By logging in with the default credentials, we find a user profile comment that leaks the SSH password for another employee. After logging in, we find a KeePass database along with a memory dump of the process. Exploiting a known vulnerability in KeePass (CVE-2023-32784) allows us to recover the master password. Inside the database, we find a PuTTY private key for the root user, which we convert to OpenSSH format to log in as root.

## Enumeration

### Nmap Scan

We start with a quick Nmap scan to identify the open ports:

![Nmap scan output - 22/tcp SSH and 80/tcp HTTP](<../../.gitbook/assets/Unknown image (407)>)

The scan shows two open ports: port 22 (SSH) and port 80 (HTTP).

### Web Recon

Visiting the web server on port 80, we see a redirection link to the Request Tracker service at `tickets.keeper.htb/rt/`.

![Request Tracker landing page redirect link](<../../.gitbook/assets/Unknown image (408)>)

After adding `tickets.keeper.htb` to our `/etc/hosts` file, we look up the default credentials for Request Tracker. The default administrative credentials are `root` with the password `password`.

![Google search showing default Request Tracker password](<../../.gitbook/assets/Unknown image (409)>)

Using these credentials, we log in successfully to the Request Tracker dashboard.

![Request Tracker dashboard login success](<../../.gitbook/assets/Unknown image (410)>)

## Foothold

### Administrative Enumeration & Credentials Leak

Within Request Tracker, we navigate to the Admin section to inspect the users.

![RT Admin section menu navigation](<../../.gitbook/assets/Unknown image (411)>)

Under the users list, we find an account for a user named Lise NÃ¸rgaard (`lnorgaard`).

![Finding lnorgaard in RT user list](<../../.gitbook/assets/Unknown image (412)>)

Inspecting her profile, we notice a comment stating that her initial password was set to `Welcome2023!`.

![Lise NÃ¸rgaard profile details and credential leak comment](<../../.gitbook/assets/Unknown image (413)>)

### SSH Access as lnorgaard

We use these credentials to connect to the target machine via SSH:

![SSH login as lnorgaard using Welcome2023!](<../../.gitbook/assets/Unknown image (414)>)

Once logged in, we locate the user flag in the home directory of `lnorgaard`.

![SRetrieving user flag](<../../.gitbook/assets/Unknown image (415)>)

## Privilege Escalation

### KeePass Database & Dump Analysis

Listing the files in the home directory reveals several interesting items: a KeePass database (`passcodes.kdbx`) and a full memory dump (`KeePassDumpFull.dmp`), as well as a zip file `RT30000.zip` containing backup files.

We host these files using Python's built-in HTTP server to download them to our local testing machine:

![Python HTTP server on port 8000 hosting files](<../../.gitbook/assets/Unknown image (416)>)

We download the zip file using `wget`:

![Downloading RT30000.zip via wget](<../../.gitbook/assets/Unknown image (417)>)

Extracting the zip file gives us direct access to `KeePassDumpFull.dmp` and `passcodes.kdbx`.

![Extracting zip file contents](<../../.gitbook/assets/Unknown image (418)>)

### Exploiting KeePass Master Password Dumper (CVE-2023-32784)

A critical vulnerability (CVE-2023-32784) in KeePass 2.x allows recovery of the master password from a process memory dump. The tool reconstructs the password character-by-character.

We run a public password dumper against the memory dump:

![Running dotnet run keepass.dmp to extract password characters](<../../.gitbook/assets/Unknown image (419)>)

The tool successfully dumps candidates for each character position, leaving the first character masked:

![Dumping characters and candidate matches](<../../.gitbook/assets/Unknown image (420)>)

![Dumping character list candidates from dump file](<../../.gitbook/assets/Unknown image (421)>)

The combined output shows the password suffix: `â—dgrÃ¸d med flÃ¸de` where the candidates for the second character are Danish characters like `Ã¸`.

Searching for `dgrÃ¸d med flÃ¸de` on Google reveals the popular Danish dessert/phrase: **rÃ¸dgrÃ¸d med flÃ¸de**.

![Google search for dgrÃ¸d med flÃ¸de showing rÃ¸dgrÃ¸d med flÃ¸de](<../../.gitbook/assets/Unknown image (422)>)

### Unlocking the Database

Using `rÃ¸dgrÃ¸d med flÃ¸de` as the master password, we attempt to unlock the KeePass database `passcodes.kdbx`.

![Entering master password into KeePass database prompt](<../../.gitbook/assets/Unknown image (423)>)

Using `kpcli` or KeePassXC, we can open the database file successfully. Inside the `Network` group, we locate an entry named `keeper.htb (Ticketing Server)`.

![Opening passcodes.kdbx in kpcli and navigating to Network group](<../../.gitbook/assets/Unknown image (424)>)

Inspecting the entry shows a private key in the notes field formatted as a PuTTY private key (PPK).

![Inspecting ticketing server entry notes containing PuTTY private key](<../../.gitbook/assets/Unknown image (425)>)

### Root Access

We save the PPK key block to a file and convert it into an OpenSSH format key using `puttygen`:

`puttygen root.ppk -O private-openssh -o id_rsa`

Once converted, we set the appropriate permissions on the key (`chmod 600 id_rsa`) and log in as the `root` user via SSH using PuTTY or the OpenSSH client:

![Configuring host destination in PuTTY](<../../.gitbook/assets/Unknown image (426)>)

![Selecting the private key for authentication in PuTTY](<../../.gitbook/assets/Unknown image (427)>)

Logging in as root gives us access to read the root flag from `root.txt`:

![Root shell and reading root flag](<../../.gitbook/assets/Unknown image (428)>)

This grants us root administrative access to the machine.
