---
description: 'Difficulty: Easy OS: Linux'
---

# Pilgrimage

## Overview

Pilgrimage is an Easy Linux machine that demonstrates the risks of Git directory exposure, outdated third-party image manipulation libraries, and insecure administrative scripting. We start by dumping an exposed Git repository to retrieve the web source code. Analyzing the source code reveals the use of a vulnerable ImageMagick version (CVE-2022-44268) for image shrinking. By exploiting this, we perform an arbitrary file read to extract the application's SQLite database containing user credentials. After logging in as `emily`, we find a root-owned bash script monitoring the uploads folder with `inotifywait` and scanning files with `binwalk`. Exploiting a remote code execution vulnerability in `binwalk` (CVE-2022-4510) grants us a root reverse shell.

## Enumeration

### Nmap Scan

We scan the target using Nmap to identify active services:

![Nmap scan output showing ports 22 and 80 open](<../../.gitbook/assets/Unknown image (429)>)

The scan reveals:

* **Port 22:** OpenSSH 8.4p1 Debian
* **Port 80:** nginx 1.18.0

### Web & Git Enumeration

Navigating to `http://pilgrimage.htb/` displays an image shrinking application:

![Pilgrimage online image shrinker interface](<../../.gitbook/assets/Unknown image (430)>)

We test for Git directory exposure and find that the `.git/` folder is fully accessible:

![Exposed Git directory file list on port 80](<../../.gitbook/assets/Unknown image (431)>)

Using `git-dumper`, we download the repository contents to our local workspace:

![Dumping Git repository via git\_dumper.py](<../../.gitbook/assets/Unknown image (432)>)

The dumped source code includes files like `index.php`, `dashboard.php`, and a local ImageMagick executable named `magick`:

![Files extracted from the dumped Git repository](<../../.gitbook/assets/Unknown image (433)>)

## Foothold

### Expliterating ImageMagick LFI (CVE-2022-44268)

Analyzing the application source shows it calls the `magick` binary to resize uploaded images. This version of ImageMagick is vulnerable to CVE-2022-44268 (Arbitrary File Read). We use an LFI proof-of-concept script to generate a PNG payload designed to read `/etc/passwd`:

![Generating exploit.png payload to read local files](<../../.gitbook/assets/Unknown image (434)>)

![Verification of generated exploit.png payload](<../../.gitbook/assets/Unknown image (435)>)

We upload our malicious PNG to the shrinker tool:

![Uploading exploit.png to the image shrinker](<../../.gitbook/assets/Unknown image (436)>)

Once uploaded, we view and download the shrunken output image:

![Accessing the processed output image](<../../.gitbook/assets/Unknown image (437)>)

![Downloading the output image using wget](<../../.gitbook/assets/Unknown image (438)>)

Using `identify` to print the verbose metadata of the downloaded image reveals a hex-encoded dump in the raw profile type:

![Verbose identification of output image showing hex profile metadata](<../../.gitbook/assets/Unknown image (439)>)

![Hex data chunk extracted from image metadata](<../../.gitbook/assets/Unknown image (440)>)

Using CyberChef to decode the hex block, we confirm the successful LFI and see the `/etc/passwd` file, which reveals the local user `emily`:

![Decoding hex data in CyberChef showing passwd content](<../../.gitbook/assets/Unknown image (441)>)

![CyberChef passwd output showing user emily](<../../.gitbook/assets/Unknown image (442)>)

### Dumping Database Credentials

To find credentials, we repeat the ImageMagick LFI attack to read the application's SQLite database file located at `/var/db/pilgrimage`:

![Generating a payload shami.png to read the SQLite database](<../../.gitbook/assets/Unknown image (443)>)

We upload this payload, download the resulting shrunk image, and extract its hex metadata:

![Downloading SQLite database payload result](<../../.gitbook/assets/Unknown image (444)>)

Decoding this database hex dump in CyberChef reveals the user credentials: `emily` and the password `emilyabigchunkyboi123`:

![CyberChef output showing Emily credentials from SQLite dump](<../../.gitbook/assets/Unknown image (445)>)

![CyberChef database hex decoding detail](<../../.gitbook/assets/Unknown image (446)>)

### SSH Access as emily

We connect to the target via SSH using Emily's credentials:

![SSH login as emily](<../../.gitbook/assets/Unknown image (447)>)

Once logged in, we read `user.txt` in Emily's home directory:

![Retrieving user flag from user.txt](<../../.gitbook/assets/Unknown image (448)>)

***

## Privilege Escalation

### Administrative Process Monitoring

We inspect the system processes and files. We find an administrative monitoring script `/usr/sbin/malwarescan.sh` running as root. This script uses `inotifywait` to watch the `/var/www/pilgrimage.htb/shrunk/` upload folder. When a new file is uploaded, it runs `binwalk` on the file to scan for malware.

The version of `binwalk` installed is v2.3.2, which is vulnerable to a Remote Command Execution bug (CVE-2022-4510) due to insecure path traversal.

![Exploit Database page for Binwalk v2.3.2 RCE (CVE-2022-4510)](<../../.gitbook/assets/Unknown image (449)>)

![Vulnerable PHP source executing magick convert on uploads](<../../.gitbook/assets/Unknown image (450)>)

### Exploiting Binwalk RCE (CVE-2022-4510)

Using a Binwalk exploit script, we generate a malicious PNG file that contains our reverse shell command payload pointing to our listener. We copy the payload image into the `/var/www/pilgrimage.htb/shrunk` directory where it will trigger the root-owned malware scan script:

![Running exploit generator script to trigger binwalk execution](<../../.gitbook/assets/Unknown image (451)>)

As soon as the file is detected in the monitored folder, the root script runs `binwalk` on it, triggering our payload. We receive a connection back on our Netcat listener:

![Root reverse shell connection and reading root.txt](<../../.gitbook/assets/Unknown image (452)>)

This grants us root administrative access to the machine.
