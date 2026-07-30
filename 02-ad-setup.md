# Phase 2 — AD DS Domain, OU Structure, Groups & Users

## 1. Domain Controller Promotion

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

```powershell
Install-ADDSForest `
  -DomainName "lab.jaebeasty.local" `
  -DomainNetbiosName "JAEBEASTY" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "Enter DSRM Password")
```

`WinThreshold` is the internal codename for the 2016-level functional mode baseline — correct and expected for Server 2022, not a typo.

**Issue encountered:** first promotion attempt failed prerequisite validation — the DSRM password didn't meet complexity requirements (needs upper/lower/number/symbol). Two other prerequisite *warnings* appeared and were correctly ignored as expected-in-a-lab noise:
- "at least one physical network adapter does not have static IP" — expected, the NAT adapter is intentionally DHCP
- "a delegation for this DNS server cannot be created" — expected for any isolated forest root with no parent zone to delegate from

Promotion succeeded on retry with a compliant DSRM password; the server rebooted automatically at the end, confirming success.

*Screenshot: [`01-dsrm-password-complexity-failure.png`](../screenshots/02-ad-setup/01-dsrm-password-complexity-failure.png)*

## 2. Post-Promotion Verification

```powershell
Get-ADDomain
Get-ADForest
Get-Service NTDS, DNS, Netlogon, KDC | Format-Table Name, Status, StartType
```

All four core services (`NTDS`, `DNS`, `Netlogon`, `KDC`) confirmed `Running`. FSMO roles (PDC Emulator, RID Master, Schema Master, Domain Naming Master) all correctly point to `DC01.lab.jaebeasty.local`, as expected for a single-DC forest.

**Issue encountered — multi-homed DNS registration:**

```
> nslookup lab.jaebeasty.local
Addresses:  192.168.56.10
            192.168.122.79
```

The domain name resolved to **two** addresses — the correct lab-net IP *and* the NAT adapter's DHCP address. Root cause: Windows Dynamic DNS registers every IP on every DNS-enabled adapter by default, not just the "intended" one. This is a real, well-documented gotcha for any dual-homed DC, and left unfixed it can cause clients to intermittently receive an unreachable address for the domain.

Fix — stop the NAT adapter from registering into DNS at all:

```powershell
Set-DnsClient -InterfaceAlias "Ethernet" -RegisterThisConnectionsAddress $false
ipconfig /flushdns
ipconfig /registerdns
```

This cleared the `DC01` A record's duplicate, but a **second, separate stale record** remained — the zone's apex/root record (`@`), which AD DS also auto-creates and which is distinct from the hostname record:

```powershell
Get-DnsServerResourceRecord -ZoneName "lab.jaebeasty.local" -RRType A -Name "@"
Remove-DnsServerResourceRecord -ZoneName "lab.jaebeasty.local" -RRType A -Name "@" -RecordData "192.168.122.79" -Force
```

Confirmed clean afterward via `nslookup lab.jaebeasty.local 192.168.56.10` querying the DC directly (bypassing any client-side resolver cache).

*Screenshots: [`02-dns-duplicate-address-issue.png`](../screenshots/02-ad-setup/02-dns-duplicate-address-issue.png), [`03-dns-duplicate-resolved.png`](../screenshots/02-ad-setup/03-dns-duplicate-resolved.png)*

## 3. OU Structure

Design combines Microsoft's administrative tier model (privilege isolation) with a location layer (GPO scoping):

```
lab.jaebeasty.local
└── OU=JAEBEASTY
    ├── OU=Tier 0 - Domain Control
    ├── OU=Tier 1 - Infrastructure
    │   ├── OU=Service Accounts
    │   └── OU=Servers
    ├── OU=Locations
    │   ├── OU=HQ-Charlotte  → Users, Workstations
    │   ├── OU=Site-Raleigh  → Users, Workstations
    │   └── OU=Site-Columbia → Users, Workstations
    ├── OU=Departments        (deliberately empty — see design note)
    ├── OU=Groups
    │   └── OU=Security Groups
    └── OU=Disabled Users
```

**Design note — why Departments has no child OUs:** department (Sales, Service, Parts, IT) is expressed via **security group membership**, not OU placement. Location determines what *policy* applies (GPOs); department determines what *resources* a user can access. Conflating both onto one axis is a common real-world cause of access sprawl — it forces duplicate department sub-OUs under every site, and makes GPO inheritance ambiguous.

Built via `scripts/New-OUStructure.ps1` — idempotent, `-ProtectedFromAccidentalDeletion` set on every OU.

*Screenshots: [`04-ou-structure-build-complete.png`](../screenshots/02-ad-setup/04-ou-structure-build-complete.png), [`07-aduc-final-ou-tree.png`](../screenshots/02-ad-setup/07-aduc-final-ou-tree.png)*

## 4. Naming Convention

Full convention documented separately in `docs/naming-convention.md`. Summary:

| Object type | Pattern | Example |
|---|---|---|
| OU | Title Case, full words | `Site-Columbia` |
| Security group | `SG-<Scope>-<Resource>-<AccessLevel>` | `SG-Sales-CRM-Access` |
| User (human) | `firstname.lastname` | `jaelan.anderson` |
| Service account | `svc-<category>-<name>` | `svc-app-veeam-backup` |
| Computer | `<SiteCode>-<Type>-<###>` | `HQC-WKS-001` |

## 5. Security Groups

10 groups built via `scripts/New-SecurityGroups.ps1`, all landing in a single flat `Security Groups` OU — organization comes from the naming convention itself, not OU nesting.

*Screenshot: [`05-security-groups-created.png`](../screenshots/02-ad-setup/05-security-groups-created.png)*

## 6. Bulk User Creation (50 users)

`scripts/New-BulkUsers.ps1` reads `scripts/data/users.csv` and, per row:
1. Creates the user in the correct site-based OU
2. Sets a temp password, forces change at next logon
3. Adds the user to a department-mapped security group
4. Logs every action to a timestamped file

**Issue encountered — duplicate CN collision:** 49 of 50 users created cleanly; one failed:

```
[FAIL] joseph.rodriguez2: An attempt was made to add an object to the
directory with a name that is already in use
```

Root cause: the CSV generator correctly made the **logon name** (`sAMAccountName`) unique across two people both named Joseph Rodriguez (`joseph.rodriguez` / `joseph.rodriguez2`), but the script's `New-ADUser -Name` parameter used the plain display name (`"Joseph Rodriguez"`) for both — and both landed in the same OU (`Site-Columbia`). AD requires the **CN** to be unique per-OU; sAMAccountName uniqueness is domain-wide, but CN uniqueness is scoped per-container. These are two different uniqueness rules that are easy to conflate.

Fixed by manually creating the second account with a distinguishing CN (`"Joseph Rodriguez (2)"`) while preserving the intended logon name. Verified final count:

```powershell
(Get-ADUser -Filter * -SearchBase "OU=JAEBEASTY,DC=lab,DC=jaebeasty,DC=local").Count
# 50
```

*Screenshot: [`06-bulk-users-49-of-50-created.png`](../screenshots/02-ad-setup/06-bulk-users-49-of-50-created.png) — shows the original 49/50 run before the manual fix above*

## 7. Offboarding Script

`scripts/Invoke-UserOffboarding.ps1` — takes a single `-SamAccountName` parameter and performs, **in strict order**:

1. **Disable** the account — immediate access cutoff, done first so a mid-script failure still leaves the worst-case state as "access is off," not something less safe
2. **Reset password** to a random, discarded value — invalidates cached/saved credentials on old devices
3. **Strip all group memberships** (except `Domain Users`, the primary group, which can't be removed via `Remove-ADGroupMember`) — the actual sprawl-prevention step
4. **Move to Disabled Users OU** — removes it from location-based GPO scope, centralizes "who's offboarded" into one place
5. **Log every action** with timestamp and operator identity

Includes a safety guard: refuses to run against any account matching the `svc-*` naming pattern without an explicit `-Force` override, since this script is built for human offboarding — the naming convention is what makes this check possible without a separate lookup table.

Tested successfully against `donald.clark`:

```
STEP 1/5 [OK] Account disabled
STEP 2/5 [OK] Password reset to random value (not recorded)
STEP 3/5 [OK] Removed from group: SG-Sales-CRM-Access
STEP 4/5 [OK] Moved to OU=Disabled Users,OU=JAEBEASTY,...
STEP 5/5 [OK] Final state - Enabled: False | Groups remaining: 0
```

*Screenshot: [`08-offboarding-script-successful-run.png`](../screenshots/02-ad-setup/08-offboarding-script-successful-run.png)*
