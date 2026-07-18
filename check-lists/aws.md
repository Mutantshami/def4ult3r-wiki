---
description: Checklist for AWS Pentesting
---

# AWS

## Checklist to run through every engagement

**Before you touch anything**

* [ ] Written authorization / scope confirmed
* [ ] Know exactly which AWS account(s) and services are in scope

**Setup**

* [ ] Credentials loaded under a dedicated CLI profile
* [ ] Ran `get-caller-identity` to confirm who you are

**Enumeration**

* [ ] Listed all users, groups, roles, policies
* [ ] Checked attached + inline policies for anything interesting
* [ ] Read the actual JSON of policies you're suspicious of
* [ ] Enumerated EC2, S3, RDS, Lambda — whatever's in scope
* [ ] Ran `iam__enum_permissions` in Pacu as a cross-check

**Access & Escalation**

* [ ] Identified any entry point (SSRF/RCE → metadata service)
* [ ] Loaded any temporary credentials obtained
* [ ] Ran `iam__privesc_scan` in Pacu, or manually checked for `iam:Attach*` / `iam:Put*` / `iam:PassRole`
* [ ] Attempted escalation only within scope
* [ ] Confirmed the escalation with `get-caller-identity` / `list-attached-role-policies`

**Cleanup — don't skip this**

* [ ] Detached any policy you attached during testing
* [ ] Deleted any temp users, roles, or access keys you created
* [ ] Account is back to how it was before you started

**Reporting**

* [ ] Every command + output documented
* [ ] Clear before/after showing the privilege escalation path
* [ ] Remediation advice included (least privilege, MFA, disable IMDSv1, rotate keys, etc.)
