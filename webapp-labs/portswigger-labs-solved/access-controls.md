# access\_controls\_writeup

> **Topic:** Access Control Vulnerabilities\
> **Platform:** [PortSwigger Web Security Academy](https://portswigger.net/web-security/access-control)\
> **Labs covered:** 13 labs (Lab 1 – Lab 13)

Access control vulnerabilities occur when a web application fails to properly restrict what authenticated or unauthenticated users can do. This writeup walks through 13 hands-on labs covering unprotected admin panels, cookie manipulation, privilege escalation, IDOR, and header-based bypass techniques.

## Lab 1 — Unprotected Admin Panel

### Objective

> This lab has an unprotected admin panel. Solve the lab by deleting the user **carlos**.

### Vulnerability

The admin panel URL is exposed through the `robots.txt` file, which is publicly accessible without any authentication.

### Solution

{% stepper %}
{% step %}
#### Navigate to `robots.txt`

Navigate to `/robots.txt` on the lab domain. The file reveals a disallowed path — which is the admin panel URL.
{% endstep %}

{% step %}
#### Visit the disclosed admin panel path

Visit the disclosed admin panel path directly in the browser.
{% endstep %}

{% step %}
#### Delete `carlos`

From the admin panel, delete the user **carlos** to solve the lab.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
`robots.txt` is a public file — never use it to hide sensitive paths. Security through obscurity is not access control
{% endhint %}

## Lab 2 — Admin Panel Disclosed in JavaScript

### Objective

> Solve the lab by accessing the admin panel and using it to delete the user **carlos**.

### Vulnerability

The admin panel path is hardcoded inside the page's JavaScript source code, making it visible to anyone who inspects the page source.

### Solution

{% stepper %}
{% step %}
#### View the page source

Open the lab and view the page source (Right-click → View Page Source, or use DevTools). Search for the admin path in the JavaScript.
{% endstep %}

{% step %}
#### Visit the admin path

Copy the path and navigate to it directly in the URL bar.
{% endstep %}

{% step %}
#### Delete `carlos`

Access the admin panel and delete the user **carlos**.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Client-side JavaScript is fully visible to end users. Never embed privileged paths, tokens, or internal logic in frontend code.
{% endhint %}

## Lab 3 — Admin Panel via Forgeable Cookie

### Objective

> This lab has an admin panel at `/admin`, which identifies administrators using a **forgeable cookie**. Solve the lab by accessing the admin panel and using it to delete the user **carlos**.

### Vulnerability

The application uses a simple cookie value (e.g., `Admin=false`) to determine whether a user is an administrator. Since the cookie is not cryptographically signed, it can be modified by the user.

### Solution

{% stepper %}
{% step %}
#### Intercept the request and inspect the cookie

Open the lab and intercept the request using Burp Suite (or your browser's DevTools). Observe the cookie in the request.
{% endstep %}

{% step %}
#### Modify the cookie value

Modify the cookie value — change `Admin=false` to `Admin=true` — and forward the request.
{% endstep %}

{% step %}
#### Access `/admin` and delete `carlos`

Access `/admin` with the modified cookie. The admin panel loads, and you can delete **carlos**.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Never use client-controlled cookies to make authorization decisions. Session state that influences access control must be stored server-side and validated on every request.
{% endhint %}

## Lab 4 — Broken Access Control via Role ID Manipulation

### Objective

> This lab has an admin panel at `/admin`, accessible only to logged-in users with a `roleid` of **2**. Solve the lab by accessing the admin panel and using it to delete the user **carlos**.

### Vulnerability

The application accepts user-supplied role data (like `roleid`) in a request body and applies it directly — allowing a normal user to elevate their own privileges.

### Solution

{% stepper %}
{% step %}
#### Log in and use the email update functionality

Log in as **wiener** using the provided credentials (`wiener:peter`). Navigate to the account settings and use the "Update Email" functionality.
{% endstep %}

{% step %}
#### Intercept the request and send it to Repeater

Intercept the email update request with Burp Suite and send it to Repeater.
{% endstep %}

{% step %}
#### Add `roleid: 2`

In the Repeater tab, add `"roleid": 2` to the request body (JSON) and send the modified request.
{% endstep %}

{% step %}
#### Confirm the role change

Confirm the response shows `roleid: 2` has been applied to your account.
{% endstep %}

{% step %}
#### Access the admin panel and delete `carlos`

Navigate to `/admin`. Your account now has admin privileges — delete **carlos** to solve the lab.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Role and privilege assignments must be enforced server-side and must never be accepted from user-controlled input. Validate role data against a trusted source (e.g., the database) rather than the request body.
{% endhint %}

## Lab 5 — Horizontal Privilege Escalation (IDOR via Username)

### Objective

> This lab has a horizontal privilege escalation vulnerability on the user account page.\
> Obtain the API key for the user **carlos** and submit it as the solution.\
> Credentials: `wiener:peter`

### Vulnerability

The account page uses a predictable, user-controlled `id` parameter in the URL to identify which user's data to display. By changing this parameter, an attacker can view another user's account data — including their API key.

### Solution

{% stepper %}
{% step %}
#### Log in and inspect your account page

Log in as **wiener** and navigate to your account page. Notice the URL contains `?id=wiener`.
{% endstep %}

{% step %}
#### Intercept the request and send it to Repeater

Intercept this request using Burp Suite and send it to Repeater.
{% endstep %}

{% step %}
#### Change the `id` parameter to `carlos`

Change the `id` parameter from `wiener` to `carlos` and send the modified request.
{% endstep %}

{% step %}
#### Extract Carlos's API key

Inspect the response. Carlos's account page is returned, including his API key.
{% endstep %}

{% step %}
#### Submit the API key

Copy the API key and submit it as the solution.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
User-specific resources must be protected by server-side session verification, not just URL parameters. Always verify that the currently authenticated user is authorized to access the requested resource.
{% endhint %}

## Lab 6 — Horizontal Privilege Escalation (IDOR via GUID)

### Objective

> This lab has a horizontal privilege escalation vulnerability on the user account page, but identifies users with **GUIDs**.\
> Find the GUID for **carlos**, then submit his API key as the solution.\
> Credentials: `wiener:peter`

### Vulnerability

The application uses GUIDs (Globally Unique Identifiers) as user IDs, making them harder to guess — but the GUIDs are leaked in publicly accessible pages (e.g., blog posts authored by carlos).

### Solution

{% stepper %}
{% step %}
#### Find a blog post authored by Carlos

Browse the lab's blog and find a post authored by **carlos**. Click on his name/profile link.
{% endstep %}

{% step %}
#### Copy Carlos's GUID

Observe that the URL contains carlos's GUID as a query parameter. Copy the GUID value.
{% endstep %}

{% step %}
#### Log in as `wiener`

Log in as **wiener** and navigate to the account page (`?id=wiener-guid`).
{% endstep %}

{% step %}
#### Replace the `id` parameter with Carlos's GUID

Replace the `id` parameter in the URL with carlos's GUID and send the request.
{% endstep %}

{% step %}
#### Submit Carlos's API key

The response returns carlos's account page containing his API key. Submit it as the solution.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Using GUIDs instead of sequential IDs adds some obscurity, but if those GUIDs are leaked anywhere (blog posts, profile pages, metadata), they can be harvested and used for IDOR attacks. GUID-based IDs are not a substitute for proper authorization checks.
{% endhint %}

## Lab 7 — API Key Leaked in Redirect Response Body

### Objective

> This lab contains an access control vulnerability where sensitive information is **leaked in the body of a redirect response**.\
> Obtain the API key for the user **carlos** and submit it as the solution.\
> Credentials: `wiener:peter`

### Vulnerability

When the server redirects an unauthenticated (or wrong-user) request, it sends an HTTP 302 redirect — but the response _body_ still contains the sensitive data from the original page, before the redirect takes effect. Browsers follow the redirect and discard the body, but an intercepting proxy can read it.

### Solution

{% stepper %}
{% step %}
#### Intercept the account page request

Log in as **wiener** and access your account page. Intercept the request in Burp Suite.
{% endstep %}

{% step %}
#### Change the user parameter to `carlos`

Modify the request — change the username/ID parameter to `carlos` — and forward the modified request.
{% endstep %}

{% step %}
#### Inspect the 302 response body

In Burp Suite's Proxy history, examine the response. Even though the server issues a 302 redirect, the **response body** contains carlos's account data, including his API key.
{% endstep %}

{% step %}
#### Submit the API key

Extract the API key from the response body and submit it.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
When performing access control checks, ensure that sensitive data is **never included in redirect response bodies**. The server should return an empty body (or a generic message) alongside any 3xx redirect.
{% endhint %}

## Lab 8 — Insecure Direct Object Reference via Static File URL

### Objective

> This lab stores user chat logs directly on the server's file system, and retrieves them using **static URLs**.\
> Find the password for the user **carlos** and log into their account.

### Vulnerability

Chat transcripts are stored as sequentially numbered static text files (e.g., `2.txt`, `3.txt`). Any user who knows the naming convention can access another user's transcript directly.

### Solution

{% stepper %}
{% step %}
#### Open Live Chat and generate a transcript

Open the **Live Chat** tab and send any message. Then click **View Transcript**.
{% endstep %}

{% step %}
#### Intercept the transcript request

Intercept the transcript download request in Burp Suite. Note the URL structure — the file is served as a numbered `.txt` file.
{% endstep %}

{% step %}
#### Access another transcript

Modify the filename in the URL to `1.txt` (or enumerate other numbers) and send the request.
{% endstep %}

{% step %}
#### Extract Carlos's password

Review the contents of the other user's transcript. Carlos's chat log contains a password mentioned in the conversation.
{% endstep %}

{% step %}
#### Log in as Carlos

Return to the login page and log in as **carlos** using the discovered password.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Files stored on a server with static, predictable URLs are effectively public if there's no access control layer in front of them. Sensitive files should require authentication and authorization to access — never rely solely on obscure filenames.
{% endhint %}

## Lab 9 — Password Exposed in Masked Input (Admin Privilege Escalation)

### Objective

> This lab has a user account page that contains the current user's existing password, **prefilled in a masked input**.\
> Retrieve the administrator's password, then use it to delete the user **carlos**.\
> Credentials: `wiener:peter`

### Vulnerability

The account page prefills the password field with the user's actual password from the server. While the browser masks the field visually (showing `••••••`), the raw HTML — and the HTTP response — contains the plaintext password. By accessing the administrator's account page (via IDOR), the administrator's password is leaked.

### Solution

{% stepper %}
{% step %}
#### Open your account page

Log in as **wiener** and navigate to the account page. Notice the URL contains `?id=wiener`.
{% endstep %}

{% step %}
#### Request the administrator account page

Modify the `id` parameter to `administrator` in Burp Suite's Repeater.
{% endstep %}

{% step %}
#### Read the plaintext password from the response

Inspect the response body. The prefilled password input contains the administrator's actual password in plaintext.
{% endstep %}

{% step %}
#### Log in as administrator

Log in to the application as **administrator** using the retrieved password.
{% endstep %}

{% step %}
#### Delete `carlos`

Navigate to the admin panel and delete the user **carlos**.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Never send sensitive data — especially passwords — to the client, even in hidden or masked form. Prefilling a password field requires the server to transmit the password value in the HTML, where it can be read directly from the response.
{% endhint %}

## Lab 10 — Access Control Bypass via X-Original-URL Header

### Objective

> This website has an unauthenticated admin panel at `/admin`, but a **front-end system blocks external access** to that path.\
> However, the back-end application supports the `X-Original-URL` header.\
> Access the admin panel and delete the user **carlos**.

### Vulnerability

The front-end proxy inspects the request URL and blocks access to `/admin`. However, the back-end framework trusts the `X-Original-URL` header and uses it to route requests — bypassing the front-end restriction entirely.

### Solution

{% stepper %}
{% step %}
#### Confirm `/admin` is blocked

Attempt to navigate to `/admin` directly. Confirm that the request is blocked (403 Forbidden).
{% endstep %}

{% step %}
#### Add `X-Original-URL` and the query parameter

Intercept a request (to any accessible path, e.g., `/`) and add the following two elements in Burp Suite's Repeater:

* Add `X-Original-URL: /admin` as a new HTTP header.
* Append `?username=carlos` to the GET request path.
{% endstep %}

{% step %}
#### Access the admin panel

Send the request. The back-end routes you to `/admin` despite the front-end block.
{% endstep %}

{% step %}
#### Delete `carlos`

Use the same technique with `X-Original-URL: /admin/delete` and `?username=carlos` in the query string to delete the user and solve the lab.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Non-standard headers like `X-Original-URL`, `X-Forwarded-URL`, and `X-Rewrite-URL` can be used to override the URL the back-end processes. Ensure that the back-end does not accept URL-overriding headers from untrusted sources, or strip them at the proxy layer.
{% endhint %}

## Lab 11 — Method-Based Access Control Bypass

### Objective

> This lab implements access controls based partly on the **HTTP method** of requests.\
> Log in as `wiener:peter` and exploit the flawed access controls to promote yourself to become an administrator.

### Vulnerability

The application's access control logic only restricts `POST` requests to certain admin actions (like promoting users). By switching the request method to `GET`, the check is bypassed and the action is still processed by the server.

### Solution

{% stepper %}
{% step %}
#### Intercept the admin upgrade request

Log in as **administrator** (`administrator:admin`) and find the "Upgrade user" functionality. Intercept the upgrade request in Burp Suite — note it is a `POST` request.
{% endstep %}

{% step %}
#### Capture the `wiener` session

Log in as **wiener** and capture the session cookie.
{% endstep %}

{% step %}
#### Modify the request method

Take the admin's upgrade request (for carlos), send it to Repeater, and:

* Replace the session cookie with **wiener's** session cookie.
* Change the `username` parameter to `wiener`.
* Change the HTTP method from `POST` to `GET`.
{% endstep %}

{% step %}
#### Send the request

Send the modified request. The server processes it — wiener is now promoted to administrator.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Access control must be enforced consistently regardless of the HTTP method used. If an action is restricted to admins via `POST`, the same restriction must apply when the same action is invoked via `GET`, `PUT`, or any other method.
{% endhint %}

## Lab 12 — Multi-Step Process with Broken Access Control

### Objective

> This lab has an admin panel with a **flawed multi-step process** for changing a user's role.\
> Log in as `wiener:peter` and exploit the flawed access controls to promote yourself to become an administrator.

### Vulnerability

The admin role-upgrade flow consists of two steps: an initial upgrade request and a confirmation request. Access control is only enforced on the **first** step. The confirmation step assumes that anyone who reaches it must have already passed the authorization check — which is not true.

### Solution

{% stepper %}
{% step %}
#### Intercept both upgrade requests

Log in as **administrator** and use the admin panel to upgrade a user (e.g., carlos). Intercept both requests in Burp Suite:

* The initial upgrade request.
* The confirmation request.
{% endstep %}

{% step %}
#### Capture the `wiener` session

Log in as **wiener** and capture the session cookie.
{% endstep %}

{% step %}
#### Replay the confirmation request with `wiener`'s session

In Burp Repeater, take the **confirmation request** (step 2 of the process), replace the session cookie with wiener's session, and change the `username` to `wiener`. Send the request.
{% endstep %}

{% step %}
#### Confirm the privilege escalation

The server processes the confirmation without re-checking authorization. Wiener is promoted to administrator.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
Authorization must be validated at **every step** of a multi-step process — not just the first. Never assume that reaching a later step implies the user passed earlier access control checks.
{% endhint %}

## Lab 13 — Referer-Based Access Control Bypass

### Objective

> This lab controls access to certain admin functionality based on the **Referer header**.\
> Log in as `wiener:peter` and exploit the flawed access controls to promote yourself to become an administrator.

### Vulnerability

The admin action endpoint checks the `Referer` header to verify that the request came from the admin panel page. Since the `Referer` header is fully controlled by the client, it can be forged.

### Solution

{% stepper %}
{% step %}
#### Capture the admin upgrade request

Log in as **administrator** and navigate to the admin panel. Intercept the "Upgrade user" request for carlos. Note the `Referer` header — it points to `/admin`.
{% endstep %}

{% step %}
#### Capture the `wiener` session

Log in as **wiener** and capture the session cookie.
{% endstep %}

{% step %}
#### Forge the `Referer` header

In Burp Repeater, send the upgrade request with:

* The **Referer** header set to the admin panel URL (e.g., `https://<lab-id>.web-security-academy.net/admin`).
* The session cookie replaced with **wiener's** session.
* The `username` parameter changed to `wiener`.
{% endstep %}

{% step %}
#### Promote `wiener`

The server accepts the request because the `Referer` header matches — wiener is promoted to administrator.
{% endstep %}
{% endstepper %}

{% hint style="info" %}
The `Referer` header is a client-supplied value and cannot be trusted for access control decisions. Any server-side authorization check based solely on the `Referer` header can be trivially bypassed.
{% endhint %}

## Summary Table

| Lab | Vulnerability Type          | Technique Used                            |
| --- | --------------------------- | ----------------------------------------- |
| 1   | Unprotected admin panel     | `robots.txt` disclosure                   |
| 2   | Admin path in source code   | JavaScript source inspection              |
| 3   | Forgeable cookie            | Cookie manipulation (`Admin=true`)        |
| 4   | Broken role assignment      | JSON body parameter injection (`roleid`)  |
| 5   | IDOR — username             | URL parameter manipulation (`?id=carlos`) |
| 6   | IDOR — GUID                 | GUID harvesting from blog posts           |
| 7   | Info leak in redirect body  | Burp proxy intercepts 302 response body   |
| 8   | IDOR — static file URL      | Incrementing filename (`1.txt`)           |
| 9   | Password in prefilled input | IDOR to admin account page                |
| 10  | Header-based URL override   | `X-Original-URL` header injection         |
| 11  | HTTP method-based bypass    | POST → GET method switch                  |
| 12  | Multi-step process flaw     | Skipping to confirmation step             |
| 13  | Referer-based bypass        | Forging the `Referer` header              |

***

## Tools Used

* **Burp Suite** — intercepting and modifying HTTP requests
* **Browser DevTools** — inspecting page source and cookies

## References

* [PortSwigger: Access Control Vulnerabilities](https://portswigger.net/web-security/access-control)
* [OWASP: Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
