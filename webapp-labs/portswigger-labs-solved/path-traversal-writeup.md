# Path Traversal Writeup

**Author:** Ihtisham\
**Topic:** Path Traversal (Directory Traversal)\
**Platform:** PortSwigger Web Security Academy\
**Tools:** Burp Suite, Web Browser

***

## Table of Contents

1. [Lab 1 — File path traversal, simple case](path-traversal-writeup.md#lab-1--file-path-traversal-simple-case)
2. [Lab 2 — File path traversal, traversal sequences blocked with absolute path bypass](path-traversal-writeup.md#lab-2--file-path-traversal-traversal-sequences-blocked-with-absolute-path-bypass)
3. [Lab 3 — File path traversal, traversal sequences stripped non-recursively](path-traversal-writeup.md#lab-3--file-path-traversal-traversal-sequences-stripped-non-recursively)
4. [Lab 4 — File path traversal, traversal sequences stripped with superfluous URL-decode](path-traversal-writeup.md#lab-4--file-path-traversal-traversal-sequences-stripped-with-superfluous-url-decode)
5. [Lab 5 — File path traversal, validation of start of path](path-traversal-writeup.md#lab-5--file-path-traversal-validation-of-start-of-path)
6. [Lab 6 — File path traversal, validation of file extension with null byte bypass](path-traversal-writeup.md#lab-6--file-path-traversal-validation-of-file-extension-with-null-byte-bypass)

***

## Before You Start

### What is Path Traversal?

Path traversal (or directory traversal) is a vulnerability that allows an attacker to read files on the server that they shouldn't have access to.

For example, a website might load images like this:

```
<img src="/loadImage?filename=product1.png">
```

If the server doesn't check the `filename` properly, you can change it to:

```
/loadImage?filename=../../../etc/passwd
```

The `../` means "go up one folder". By chaining enough of these, you can reach the server's root directory (`/`) and read sensitive files like `/etc/passwd` (which contains user account info on Linux).

### The Goal

In all these labs, our goal is the same: retrieve the contents of the `/etc/passwd` file by exploiting the image loading feature.

***

## Lab 1 — File path traversal, simple case

**What's happening:** The server doesn't have any defenses against path traversal.

### Steps

**Step 1:** Log in to the lab and click on any product to view its details. Intercept the request in Burp Suite when the product image loads.

The request will look like this:

```
GET /image?filename=24.jpg HTTP/1.1
```

**Step 2:** Change the filename parameter to include path traversal sequences:

```
GET /image?filename=../../../etc/passwd HTTP/1.1
```

Send the request.

![Burp Suite showing the simple path traversal payload](<../../.gitbook/assets/Unknown image (97)>)

If you don't go back enough directories, the server might say "Not Found".

![Server responding with Not Found](<../../.gitbook/assets/Unknown image (98)>)

Keep adding `../` until you reach the root directory. `../../../etc/passwd` usually works.

![Burp Suite showing the successful retrieval of /etc/passwd](<../../.gitbook/assets/Unknown image (99)>)

The server responds with the contents of the `/etc/passwd` file!

**Step 3:** The lab is solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (100)>)

### Why did this work?

The server took our input (`../../../etc/passwd`) and directly appended it to the image directory path. It allowed the `../` sequences to traverse up to the root directory and read the target file.

***

## Lab 2 — File path traversal, traversal sequences blocked with absolute path bypass

**What's happening:** The server blocks the `../` sequence. But it allows absolute paths.

### Steps

**Step 1:** Intercept the image loading request in Burp Suite.

Since the server blocks `../`, we can try giving it the exact, absolute path to the file from the root of the file system.

**Step 2:** Change the filename to:

```
GET /image?filename=/etc/passwd HTTP/1.1
```

Send the request.

![Burp Suite showing the absolute path payload /etc/passwd](<../../.gitbook/assets/Unknown image (101)>)

The server ignores the base directory entirely and just fetches the absolute path we provided.

**Step 3:** The lab is solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (102)>)

### Why did this work?

The server checked for `../` and blocked it, but it failed to check if the user provided an absolute path starting with `/`.

***

## Lab 3 — File path traversal, traversal sequences stripped non-recursively

**What's happening:** The server tries to remove any `../` it finds in the input.

### Steps

**Step 1:** Intercept the image loading request.

If we send `../`, the server deletes it, leaving nothing. But if we send `....//`, the server deletes the middle `../`, leaving behind... another `../`!

**Step 2:** Change the filename to use nested traversal sequences:

```
GET /image?filename=....//....//....//etc/passwd HTTP/1.1
```

Send the request.

![Burp Suite showing the nested traversal sequence payload](<../../.gitbook/assets/Unknown image (103)>)

The server strips out the inner `../`, and processes the remaining `../../../etc/passwd`. The contents of the file are returned.

**Step 3:** The lab is solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (104)>)

### Why did this work?

The server only removed `../` once (non-recursively). By nesting the sequences like `....//`, the inner part gets deleted, and the outer parts merge to form a new, working `../` sequence.

***

## Lab 4 — File path traversal, traversal sequences stripped with superfluous URL-decode

**What's happening:** The server blocks `../` sequences, but it URL-decodes the input before using it.

### Steps

**Step 1:** Intercept the image loading request.

If we URL encode `../` once, it becomes `%2e%2e%2f`. Web servers often decode this automatically, so the security check might still catch it. To bypass this, we URL-encode it **twice**. The double URL-encoded version of `../` is `%252e%252e%252f`.

**Step 2:** Change the filename to the double URL-encoded payload:

```
GET /image?filename=%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd HTTP/1.1
```

Send the request.

![Burp Suite showing the double URL-encoded payload](<../../.gitbook/assets/Unknown image (105)>)

The front-end server/WAF misses the blocked characters, but the back-end server decodes it fully back to `../../../etc/passwd` and reads the file.

**Step 3:** The lab is solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (106)>)

### Why did this work?

The security filter checked the input before it was fully decoded. By double-encoding our payload, it slipped past the filter, and then the application decoded it again into a dangerous traversal sequence.

***

## Lab 5 — File path traversal, validation of start of path

**What's happening:** The server checks that the filename starts with a specific expected base folder (like `/var/www/images`).

### Steps

**Step 1:** Intercept the image loading request.

The server forces us to start our path with the expected folder name. We can just provide that folder name, and then immediately traverse back out of it!

**Step 2:** Change the filename to include the required base folder followed by traversal sequences:

```
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/1.1
```

Send the request.

![Burp Suite showing the payload starting with the required base folder](<../../.gitbook/assets/Unknown image (107)>)

The server checks if the path starts with `/var/www/images/` — it does! Then it evaluates the rest of the path, traversing up three levels to the root, and reading `/etc/passwd`.

**Step 3:** The lab is solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (108)>)

### Why did this work?

The validation only checked the beginning of the string. It didn't check what happened after the valid starting folder.

***

## Lab 6 — File path traversal, validation of file extension with null byte bypass

**What's happening:** The server checks that the filename ends with an expected file extension, like `.png` or `.jpg`.

### Steps

**Step 1:** Intercept the image loading request.

We need our payload to end with `.png`, but we want to read `/etc/passwd`. We can use a **null byte** (`%00`) to trick the server. A null byte signifies the "end of string" in many programming languages.

**Step 2:** Change the filename to include a null byte before the required extension:

```
GET /image?filename=../../../etc/passwd%00.png HTTP/1.1
```

Send the request.

![Burp Suite showing the null byte payload ending with .png](<../../.gitbook/assets/Unknown image (109)>)

The validation check sees `.png` at the very end of the string and allows it. However, when the server's file system API goes to read the file, it stops reading at the `%00` null byte. It ends up reading `/etc/passwd` and ignoring the `.png` part.

**Step 3:** The lab is solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (110)>)

### Why did this work?

The security check validated the entire string, but the underlying file reading function treated the null byte as the end of the filename. This disconnect allowed us to satisfy the extension check while actually reading a completely different file.

***

## Quick Reference Summary

| Lab | Validation Method        | Bypass Payload                                                                |
| --- | ------------------------ | ----------------------------------------------------------------------------- |
| 1   | None                     | `../../../etc/passwd`                                                         |
| 2   | Blocks `../`             | `/etc/passwd` (Absolute path)                                                 |
| 3   | Removes `../`            | `....//....//....//etc/passwd` (Nested)                                       |
| 4   | Decodes and blocks `../` | `%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd` (Double URL Encode) |
| 5   | Must start with folder   | `/var/www/images/../../../etc/passwd`                                         |
| 6   | Must end with `.png`     | `../../../etc/passwd%00.png` (Null byte)                                      |

***

## How to Prevent Path Traversal

1. **Avoid user input for filenames:** If possible, don't let users supply filenames directly. Use an ID (e.g., `id=123`) mapping to the file instead.
2. **Validate input strictly:** Ensure the filename contains only permitted characters (like alphanumeric characters) and no paths/directories.
3. **Use built-in API functions:** Use functions in your language framework to resolve the canonical path, and then verify that the resolved path starts with your intended base directory.

***

> **Disclaimer:** This writeup is for educational purposes only. All labs were done on PortSwigger's intentionally vulnerable platform. Don't try this on real systems without permission.
