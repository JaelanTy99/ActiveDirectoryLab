# Lab 1 — Windows Server AD DS Domain (Foundation)

A from-scratch Active Directory Domain Services build on a KVM/QEMU lab environment, hosted on a Fedora Linux desktop. Covers domain promotion, a security-tiered OU design, a documented naming convention, security groups, bulk user provisioning, a proper offboarding workflow, and Group Policy enforcement.

**Domain:** `lab.jaebeasty.local` | **NetBIOS:** `JAEBEASTY` | **DC:** `DC01` (Windows Server 2022 Standard Evaluation)

## Why this lab

I use Active Directory daily at work — Okta, Entra ID, and on-prem AD all assume a directory structure someone else already built. This lab exists so that when an interviewer asks *why* access sprawl happens, I can point to a structure I designed myself and explain the tradeoffs, not just recite terminology.

> **Interview line:** *"I designed the OU and group structure myself, so I understand why access sprawl happens — the structure causes it."*

## Documentation

This README is the summary. Full step-by-step detail, including every issue hit and how it was diagnosed, lives in three phase docs:

| Doc | Covers |
|---|---|
| [`docs/01-vm-setup.md`](docs/01-vm-setup.md) | Fedora host setup, KVM/QEMU, ISO acquisition, isolated lab network, VM creation, Windows install, static IP |
| [`docs/02-ad-setup.md`](docs/02-ad-setup.md) | Forest promotion, DNS troubleshooting, OU design, naming convention, security groups, bulk user creation, offboarding script |
| [`docs/03-gpo-policies.md`](docs/03-gpo-policies.md) | All three GPOs, including a real GPO precedence bug caught and fixed |
| [`docs/naming-convention.md`](docs/naming-convention.md) | Full naming standard for OUs, groups, users, service accounts, and computers |

## OU Design

```
lab.jaebeasty.local
└── OU=JAEBEASTY
    ├── OU=Tier 0 - Domain Control        ← Domain Admins & DCs only
    ├── OU=Tier 1 - Infrastructure
    │   ├── OU=Service Accounts
    │   └── OU=Servers
    ├── OU=Locations                      ← drives GPO scope
    │   ├── OU=HQ-Charlotte  (Users, Workstations)
    │   ├── OU=Site-Raleigh  (Users, Workstations)
    │   └── OU=Site-Columbia (Users, Workstations)
    ├── OU=Departments                    ← intentionally empty, see below
    ├── OU=Groups
    │   └── OU=Security Groups
    └── OU=Disabled Users                 ← offboarding parking lot, never delete
```

**Key design decision:** location and department are deliberately kept on separate axes. Location (OU placement) determines what *policy* applies via GPO. Department (security group membership) determines what *resources* a user can access. Forcing both onto the OU tree at once is a common real-world cause of sprawl — it duplicates department structure under every site and makes GPO inheritance ambiguous. This split is also what lets the offboarding script cleanly strip access (group membership) independently of relocating the object (OU move).

This combines Microsoft's administrative tier model (privilege isolation between Tier 0/1/2) with a location layer, rather than a flat department-only structure — chosen specifically because tiering is the direct, textbook answer to "how do you prevent lateral privilege escalation," which is what most access-sprawl problems actually are.

## Naming Convention (summary)

| Object | Pattern | Example |
|---|---|---|
| OU | Title Case, full words | `Site-Columbia` |
| Security group | `SG-<Scope>-<Resource>-<AccessLevel>` | `SG-Sales-CRM-Access` |
| User | `firstname.lastname` | `jaelan.anderson` |
| Service account | `svc-<category>-<name>` | `svc-app-veeam-backup` |
| Computer | `<SiteCode>-<Type>-<###>` | `HQC-WKS-001` |

Full rationale for every rule in [`docs/naming-convention.md`](docs/naming-convention.md).

## Scripts

| Script | Purpose |
|---|---|
| [`scripts/New-OUStructure.ps1`](scripts/New-OUStructure.ps1) | Builds the full OU tree above. Idempotent. |
| [`scripts/New-SecurityGroups.ps1`](scripts/New-SecurityGroups.ps1) | Builds 10 security groups per the naming convention. Idempotent. |
| [`scripts/New-BulkUsers.ps1`](scripts/New-BulkUsers.ps1) | Reads `scripts/data/users.csv`, creates 50 users in the correct site OU, sets temp passwords, assigns department groups, logs every action. |
| [`scripts/Invoke-UserOffboarding.ps1`](scripts/Invoke-UserOffboarding.ps1) | Single-user offboarding: disable → reset password → strip groups → move to Disabled Users → log. Refuses to run against `svc-*` accounts without `-Force`. |

## Group Policy

| GPO | Linked at | Setting |
|---|---|---|
| `GPO-PasswordPolicy` | Domain root (required — see note) | 14-char min, 90-day max age, 24-password history, complexity enabled |
| `GPO-ScreenLockTimeout` | `OU=JAEBEASTY` | 15-minute inactivity lock |
| `GPO-DisableRemovableMedia` | `OU=JAEBEASTY` | Deny all removable storage access |

Password policy **must** link at the domain root — Windows only honors domain account password policy from a GPO at that scope, regardless of what other OUs it's linked to. Full detail, including a real precedence bug where the policy was linked correctly but silently losing to `Default Domain Policy`, is in [`docs/03-gpo-policies.md`](docs/03-gpo-policies.md).

## Issues Encountered (the actual interview material)

1. **Multi-homed DC DNS registration** — the DC auto-registered its NAT-facing IP into DNS alongside the correct lab IP, because Windows Dynamic DNS registers every DNS-enabled adapter by default. Diagnosed by comparing `nslookup` output against the live zone data directly, found *two separate* stale records (hostname record and zone apex record are different objects), and fixed by disabling registration on the NAT adapter plus manually removing the leftover records.
2. **Duplicate CN collision during bulk creation** — 49/50 users created cleanly; the 50th failed because two people shared a display name and landed in the same OU. sAMAccountName uniqueness is domain-wide; CN uniqueness is scoped per-OU. Two different rules, easy to conflate.
3. **GPO precedence** — a correctly-configured, correctly-linked password policy GPO was silently losing to `Default Domain Policy` because of link order. `net accounts` was the tool that caught it — the GPO console showed everything as "fine."
4. **Wrong GPO link scope** — two GPOs were initially linked at the domain root instead of the intended OU. Not broken, but inconsistent with the documented design; corrected before it could cause unintended inheritance later.

## Environment

- **Host:** Fedora Linux 44
- **Hypervisor:** KVM/QEMU via `virt-install`
- **Guest OS:** Windows Server 2022 Standard Evaluation (Desktop Experience)
- **Network:** Isolated `lab-net` (192.168.56.0/24, no DHCP/forward) + libvirt `default` NAT for internet/activation
