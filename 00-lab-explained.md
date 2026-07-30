# Lab 1 Explained — Understanding What You Built and Why

This isn't a repeat of the README. The README documents *what* you did for your portfolio. This document explains *why* each piece works the way it does, so you can defend every decision in an interview without needing to look anything up. Read it slowly. If a section doesn't make sense, that's the section to re-read before moving to Lab 2.

---

## Part 1: The Virtualization Layer

### Why KVM/QEMU instead of VirtualBox?

VirtualBox is a Type 2 hypervisor — it runs as an application on top of your existing OS. KVM turns the Linux kernel itself into a Type 1 (bare-metal) hypervisor, using hardware virtualization extensions (Intel VT-x, which you confirmed with that `grep vmx` check) to run guest VMs at near-native speed. On Fedora specifically, KVM is a first-class citizen — it's what Fedora Workstation ships with virt-manager for — while VirtualBox constantly fights Fedora's Secure Boot module signing and kernel updates. You chose the tool that's *supposed* to be there.

### Why did the VM need two network adapters?

This is the single most important networking concept in the whole lab, so make sure it's solid: **a domain controller needs internet access to activate its Windows evaluation license and pull updates, but it needs to be isolated from your home network for the AD environment itself.** Those are two different, sometimes conflicting needs, so you gave the VM two separate virtual NICs:

- **`lab-net`** — an isolated virtual network with no route to the internet and no DHCP server. This is where the "real" lab lives. DC01 got a static IP here (`192.168.56.10`) because a domain controller's IP address is foundational — DNS records, SRV records, and every domain-joined computer's configuration all point at it. If that IP silently changed via DHCP, the whole domain would break.
- **`default`** (libvirt's built-in NAT network) — gives the VM internet access without exposing your isolated lab segment to anything.

The critical rule you learned the hard way: **give the DC only one default gateway** (on the NAT adapter), and **make the DC point at itself for DNS** (`127.0.0.1` on the lab adapter, since it becomes the DNS server once AD DS installs). Two default gateways on one machine creates a routing conflict — the OS doesn't know which path outbound traffic should take, and you get intermittent, confusing failures that look like they're happening somewhere else entirely.

### Why did Windows Setup show "no drives found"?

The 60GB virtual disk was attached using a **virtio** bus (`bus=virtio` in the `virt-install` command) because virtio disks are dramatically faster than emulating a real SATA/IDE controller — QEMU doesn't have to pretend to be real hardware, it just exposes a paravirtualized interface the guest OS talks to directly. The tradeoff: Windows has **no built-in driver** for that interface. It's not a bug, it's the expected cost of choosing performance over "just works out of the box." That's why you had to load the `viostor` driver mid-install before the disk would even appear.

---

## Part 2: What "Promoting to a Domain Controller" Actually Means

`Install-ADDSForest` did four things at once, and it's worth separating them out because people often use "domain controller," "Active Directory," and "DNS server" interchangeably when they're actually distinct pieces working together:

1. **Installed the Active Directory database (NTDS)** — a specialized database (technically an ESE/JET database, similar lineage to old Exchange databases) that stores every object in your directory: users, computers, groups, OUs. This is what the `NTDS` service you checked is actually running.

2. **Installed and configured DNS** — this is the part people underestimate. **Active Directory cannot function without DNS**, because domain-joined computers don't find the DC by IP address, they find it by querying DNS for special records (`_ldap._tcp.lab.jaebeasty.local`, `_kerberos._tcp.lab.jaebeasty.local`, etc.) that point them to whichever DC can service their request. This is exactly why the multi-homed DNS bug you fixed mattered so much — if a client machine ever received the wrong IP for the domain from a DNS query, it would fail to authenticate, fail to apply GPOs, fail basically everything, and the error messages it produces almost never say "your DNS is wrong."

3. **Set up Kerberos (KDC)** — the actual authentication protocol AD uses. When you log into a domain-joined machine, you're not sending your password to the DC directly; you're getting a ticket from the KDC that proves who you are without repeatedly transmitting your credentials. This is why the `KDC` service exists as its own thing separate from `NTDS`.

4. **Established the forest and domain** — `lab.jaebeasty.local` is the domain; the forest is the security boundary around it (a forest can contain multiple domains, but yours has exactly one, which is normal for a lab or small org). The **FSMO roles** you checked (`Get-ADForest`) — Schema Master, Domain Naming Master, RID Master, PDC Emulator — are five specialized jobs that, in a single-DC environment, all just live on that one DC by default. In a multi-DC environment, these roles get distributed, and knowing which role does what (e.g., the PDC Emulator handles time sync and password change urgency) is a real interview topic once you're managing more than one DC.

### Why did the DSRM password matter so much?

**Directory Services Restore Mode** is a special boot mode for a domain controller that lets you repair or restore the AD database when the DC itself won't boot normally — think of it as AD's equivalent of Windows Safe Mode, except it's specifically for fixing a broken directory database. You set this password once, during promotion, and it's *not* the same as any domain account's password. If you ever lose it and genuinely need DSRM later, recovery is painful. This is exactly why "write it down somewhere real" wasn't a throwaway line — in production, this password is usually stored in a password vault with restricted access, separate from day-to-day admin credentials.

---

## Part 3: OU Design — The Part That's Actually About Security, Not Just Organization

### The tiering concept

Microsoft's Enterprise Access Model (the modern name for what used to be called the "tier model") separates AD objects by **blast radius** — how much damage is possible if that object's credentials are compromised:

- **Tier 0** = things that *are* the domain. Domain Controllers, Domain Admin accounts, anything that could be used to compromise the entire forest if breached.
- **Tier 1** = servers and the service accounts that run on them. Compromising a Tier 1 account can take down a server or an application, but can't (in a properly designed environment) escalate to full domain control.
- **Tier 2** = everyday workstations and standard user accounts. The blast radius of a compromised Tier 2 account should be limited to that one user's own access.

The entire point of tiering is **credential isolation**: a Domain Admin should never log into a Tier 2 workstation, because doing so leaves cached credentials that a compromised workstation could steal and escalate all the way up. This is the actual mechanism behind most real-world "one phished laptop turned into a full domain compromise" breach stories you'll read about. Your OU structure — Tier 0 domain control kept completely separate from Tier 1 infrastructure kept separate from Locations (which house your Tier 2 human users) — is a working (if small-scale) implementation of that principle.

### Why does Location get its own layer, separate from Department?

This was the single design decision I pushed you hardest to understand, so let's nail it down permanently:

- **OU placement determines what *policy* applies.** GPOs link to OUs. If Sales, Service, Parts, and IT all needed different printer mappings depending on which building they're physically in, that's a **location** problem, solved by OU structure.
- **Group membership determines what *resources* a person can access.** Whether someone can open the CRM or the DMS has nothing to do with which building they sit in — it's a **department** problem, solved by security groups.

If you tried to force both into the OU tree at once, you'd end up with something like `OU=Sales,OU=HQ-Charlotte` and `OU=Sales,OU=Site-Raleigh` — duplicating the "Sales" concept under every single site, and making it structurally awkward the moment someone transfers between departments *or* between locations (should they move OUs? Which one?). Keeping department purely as group membership means a person can change departments with zero OU moves and zero GPO re-application — you just change their group memberships.

### Why `ProtectedFromAccidentalDeletion`?

This flag, set on every OU your script created, prevents someone (including future-you, six months from now, tired and moving fast) from accidentally deleting an OU that has hundreds of users nested inside it. In real environments, an accidental OU deletion is a legitimately catastrophic, hard-to-recover incident. This is a genuine production best practice, not lab theater.

---

## Part 4: Naming Conventions — Why This Isn't Just Bureaucracy

A naming convention is a **contract with your future self and every other admin who touches this environment.** Two specific payoffs from your convention design are worth internalizing:

1. **The `svc-` prefix made your offboarding script safer without extra code.** Because you committed to prefixing every service account, your offboarding script could add a single, simple safety check (`if SamAccountName -like "svc-*"`) instead of needing a separate lookup table of "which accounts are service accounts." The naming convention *is* the safety mechanism. This is a genuinely elegant pattern worth remembering: good naming conventions often eliminate the need for separate metadata.

2. **Scope-first group naming (`SG-<Scope>-<Resource>-<Access>`) makes audits possible at scale.** When you eventually have 200 security groups instead of 10, alphabetically sorting by name will cluster everything scoped to one site or one tier together. If you'd named groups resource-first instead (`SG-CRM-Sales-Access`), an audit of "everything Tier 0 can access" would require reading every single group name individually instead of just scrolling to the "Tier0" cluster.

---

## Part 5: Security Groups — The Detail Worth Knowing Cold

You used `-GroupScope Global` for every group in this lab, and it's worth understanding the three group scopes that exist in AD, because this is a near-guaranteed interview question:

- **Global groups** — can contain users from the *same domain only*, but can be granted access to resources in *any* domain in the forest. This is the right default for "who's in this role" groups like the ones you built.
- **Domain Local groups** — can contain users from *any* domain, but can only be granted access to resources *in the same domain*. These are typically used for the actual permission assignment on a resource (e.g., a Domain Local group gets read access to a file share, and Global groups representing roles get nested inside it).
- **Universal groups** — can contain users from any domain and be granted access anywhere in the forest, but membership changes replicate to the entire forest's Global Catalog, which is expensive at scale.

The classic pattern taught in every AD course is **AGDLP**: **A**ccounts go into **G**lobal groups, which go into **D**omain **L**ocal groups, which get **P**ermissions assigned. In a single-domain lab like yours, this distinction matters less practically (everything's already in one domain), but knowing *why* Global was the correct choice — and being able to explain Domain Local and Universal on demand — is exactly the kind of depth that separates "I clicked through a tutorial" from "I understand the directory."

---

## Part 6: Bulk User Creation — The Script Design Choices

### Why check that the target OU exists before creating the user?

Without that guard, if you ever add a new site to the CSV without first running the OU-creation script for it, `New-ADUser` would throw a raw AD exception that looks like a permissions error rather than "this path doesn't exist." Failing with a clear, readable message (`target OU does not exist`) instead of an unhelpful native error is the difference between a five-second fix and a twenty-minute troubleshooting detour six months from now when you've forgotten the script's internals.

### Why is idempotency ("safe to re-run") a design goal at all?

Imagine the script fails halfway through creating 50 users — maybe a network blip, maybe you closed the laptop lid. Without an existence check, re-running the script would either throw 30 duplicate-object errors or, worse, silently create 30 *duplicate* users with slightly different attributes. Checking `if (Get-ADUser -Filter ...)` before creating anything means you can always just re-run the whole script after any failure and trust that it'll pick up exactly where it left off.

### The CN vs. sAMAccountName lesson (the joseph.rodriguez2 bug)

This is worth its own paragraph because it's a genuinely non-obvious AD rule: **an object's sAMAccountName (logon name) must be unique across the *entire domain*, but an object's CN (common name, its display name) only has to be unique within its *own container (OU)*.** Your script correctly made the logon names unique (`joseph.rodriguez` / `joseph.rodriguez2`) but used the same display name for both, and because both people happened to land in the same site OU, AD rejected the second one. This is a real trap that catches experienced admins, not just beginners — the two identifiers look similar but follow completely different uniqueness rules.

---

## Part 7: The Offboarding Script — Order Is the Whole Design

Every step in your offboarding script exists in a specific order for a specific reason, and this is worth being able to explain step-by-step, unprompted, in an interview:

1. **Disable first, always.** If the script crashes after step 1 but before step 5, the account is still disabled — the worst-case outcome is "access was cut off correctly," which is the safe failure mode. If you'd stripped groups first and the script crashed before disabling, you'd have a still-enabled account that happens to have lost some access — a much murkier, harder-to-audit state.

2. **Reset the password to something nobody will ever know.** The account is already disabled, so this isn't about "locking them out" — it's about invalidating any cached credentials sitting on an old laptop, browser, or saved RDP session. Defense in depth: even if something bypasses the disabled flag, the old password is now useless.

3. **Strip group memberships — this is the actual access-sprawl fix.** A disabled account that's still sitting in ten security groups is a live landmine. If anyone ever re-enables it (accidentally, or through a compromised process), it immediately regains every permission it ever had. This step is what your interview line is really about.

4. **Move to Disabled Users** — this removes the account from any location-based GPO scope (it's no longer in the HQ-Charlotte OU, so HQ-Charlotte-specific policies stop applying to it) and gives you a single, obvious place to look for "who's been offboarded."

5. **Log everything, with a timestamp and the operator's identity.** In a real environment, this log is your evidence trail if someone ever asks "when was this person's access removed, and by whom?" — a question that comes up in security audits, compliance reviews, and sometimes legal disputes.

---

## Part 8: Group Policy — The Concept Most People Get Wrong

### GPOs don't "run" — they apply through inheritance, and inheritance has rules

A GPO linked to an OU applies to every user/computer object inside that OU *and every OU nested beneath it*, unless something blocks that inheritance. This is why linking `GPO-ScreenLockTimeout` at `OU=JAEBEASTY` was enough to cover every site underneath it — you didn't need to link it three separate times.

### Why does password policy have to link at the domain root specifically?

This is a genuinely special-cased rule in AD, not a general pattern: **Windows only reads password policy for domain user accounts from GPOs linked at the domain level.** You can link a password-policy GPO to any OU you want, and the GPO console will show it as "linked" with no errors — but it will silently do nothing for domain accounts. (Fine-Grained Password Policies exist as a separate mechanism for setting different password rules for different groups of users, but that's a more advanced tool for a later lab.) This single fact trips up a huge number of people learning AD, because everything else about GPOs behaves the way you'd intuitively expect, and this one exception doesn't.

### The precedence bug, explained properly

You had two GPOs both configuring password policy at the same scope (domain root): the built-in `Default Domain Policy` and your `GPO-PasswordPolicy`. When multiple GPOs at the same link level configure the *same setting* differently, **GPO link order decides the winner** — and counterintuitively, **lower order number wins** (Order 1 beats Order 2). Your custom GPO was sitting at Order 2, so `Default Domain Policy`'s built-in defaults (7-character minimum, 42-day max age) kept winning every time, even though your GPO existed, was correctly linked, and showed no errors anywhere in the GPO console.

The lesson that generalizes far beyond this one bug: **"a GPO is linked" and "a GPO is actually in effect" are two different facts, and only testing the *actual resulting behavior* (in this case, `net accounts`) can tell you which one is true.** The GPMC console will never warn you about a losing precedence battle — it just quietly shows you a GPO that exists and is linked, with no indication that another GPO is beating it. This is exactly the kind of thing that separates people who can build a lab from people who can actually troubleshoot a production environment when something "should" be working but isn't.

---

## Part 9: How This Maps to Your Actual Career Direction

Since your target is IAM specifically, here's the direct translation from what you built to what IAM work actually involves:

| What you built | The IAM concept it demonstrates |
|---|---|
| Tier 0/1/2 OU separation | **Privileged Access Management (PAM)** — isolating high-privilege credentials from everyday use |
| Security groups scoped by role/resource | **Role-Based Access Control (RBAC)** — the foundational pattern behind almost every enterprise entitlement system, including Okta |
| Department accessed via groups, not OU placement | **Separation of duties between identity attributes and access grants** — a person's org-chart position and their system access are related but distinct data |
| The offboarding script's strict order | **Identity lifecycle management** — the formal term for the deprovisioning half of joiner-mover-leaver processes |
| `svc-` naming convention enabling a safety check | **Entitlement governance through metadata** — using naming/tagging conventions to make automated policy enforcement possible without a separate database |
| Multi-homed DNS bug | Not IAM directly, but a genuine "I can troubleshoot a live directory service" credibility signal |

When you're in an interview and someone asks "tell me about a time you designed an access control structure," this lab is a real, specific, defensible answer — not a hypothetical.

---

## Quick Self-Check

Before moving to Lab 2, you should be able to answer all of these without looking anything up:

1. Why does the DC need two network adapters, and why can't both have a default gateway?
2. What are the three group scopes in AD, and when would you use each?
3. Why is DNS not optional for Active Directory to function?
4. What's the difference between sAMAccountName uniqueness and CN uniqueness?
5. Why must a password-policy GPO be linked at the domain root specifically?
6. In the offboarding script, why does "disable" happen before "strip group memberships," not after?
7. What's the difference between "a GPO is linked" and "a GPO is in effect"?
8. Why is department expressed as group membership instead of OU placement in your design?

If any of these feel shaky, that's exactly the section of this document worth re-reading before you start Lab 2.
