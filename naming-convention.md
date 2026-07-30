# Naming Convention — JAEBEASTY Lab AD DS

This document defines the naming standard for all objects in `lab.jaebeasty.local`.
The goal is that any object's name tells you what it is, where it lives, and how
privileged it is, without needing to open it up in ADUC first.

**Core principle:** abbreviate where an object is typed often (hostnames, logon
names); spell out in full where an object is read rarely but must be unambiguous
(OUs, group descriptions, audit logs).

---

## 1. Organizational Units

**Pattern:** Title Case, full words, no abbreviations.

| Example | Notes |
|---|---|
| `Site-Columbia` | Not `Col` |
| `Tier 0 - Domain Control` | Descriptive enough to read cold in a GPO report |
| `Service Accounts` | Not `SvcAccts` |

**Rationale:** OU names surface in GPO inheritance reports, delegation views, and
audit trails months or years after creation. Nobody types a full distinguished
name by hand day-to-day, so there is no cost to full clarity — and a real cost
to ambiguity when an auditor or a future admin has to guess what `T0-DC` meant.

---

## 2. Security Groups

**Pattern:** `SG-<Scope>-<Resource/Role>-<AccessLevel>`

Scope comes first so that groups sort and cluster alphabetically by *where*
they apply — critical once the environment has 100+ groups and you need to
audit everything scoped to one site or one tier at a glance.

| Example | Meaning |
|---|---|
| `SG-HQCharlotte-FileShare-Read` | Read access to a specific resource, one site |
| `SG-All-VPN-Users` | Cross-site functional group |
| `SG-Tier1-ServerAdmins` | Privileged, tier-scoped |
| `SG-Sales-Salesforce-Access` | Department + application access |

---

## 3. User Accounts (Human)

**Pattern (logon / sAMAccountName):** `firstname.lastname`

| Example |
|---|
| `jaelan.anderson` |

Kept flat and predictable — no site or department encoded in the logon name,
because a person's site or department can change without requiring an account
rename (which breaks logon history, mail flow, and app integrations tied to
the sAMAccountName).

---

## 4. Service Accounts

**Pattern:** `svc-<category>-<name>`

Prefixed (not suffixed) so every service account sorts together as a block in
any alphabetical list — making it trivial to eyeball "is this a human or a
service account" and to scope automation safely.

| Category | Prefix | Example |
|---|---|---|
| Application service account | `svc-app-` | `svc-app-veeam-backup` |
| Scheduled task account | `svc-task-` | `svc-task-nightly-report` |
| Infrastructure/system account | `svc-infra-` | `svc-infra-print-server` |

**Operational rule:** offboarding/cleanup automation must never act on any
account whose sAMAccountName begins with `svc-` without explicit manual
review — disabling a live backup or print service account by accident in an
automated sweep is a preventable outage.

---

## 5. Computer Objects

**Pattern:** `<SiteCode>-<Type>-<###>`

| Site | Code |
|---|---|
| HQ-Charlotte | `HQC` |
| Site-Raleigh | `RAL` |
| Site-Columbia | `COL` |

| Type | Code |
|---|---|
| Workstation | `WKS` |
| Server | `SRV` |

| Example | Meaning |
|---|---|
| `HQC-WKS-001` | Workstation #1 at HQ-Charlotte |
| `RAL-SRV-002` | Server #2 at Site-Raleigh |
| `COL-WKS-014` | Workstation #14 at Site-Columbia |

Abbreviated on purpose — this is the one object type technicians type
constantly (remote sessions, inventory lookups, ticket references), so
brevity has real day-to-day value here in a way it doesn't for OUs.

---

## Design Note: Why Department Isn't in the OU Path

Department (Sales, Service, Parts, IT) is expressed through **security group
membership**, not OU placement. Two objects govern two different things:

- **Location (OU)** → determines what *policy* applies (GPOs: printers,
  timezone, local delegation)
- **Department (security group)** → determines what *resources* a user can
  access (file shares, applications)

Conflating the two into a single OU hierarchy is a common cause of access
sprawl: it forces every department to exist as a sub-OU under every site,
duplicating structure, and it makes GPO inheritance ambiguous when a policy
needs to apply by location but a permission needs to apply by department.
Keeping them on separate axes keeps both scopes clean and independently
auditable.
