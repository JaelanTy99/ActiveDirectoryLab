# Phase 3 — Group Policy Objects

Three GPOs required by the lab spec: password policy, screen lock timeout, disable removable media. Built via GPMC (`gpmc.msc`), not scripted — GPO settings are deeply nested in the console tree, and walking through the GUI is what actually teaches (and documents) *where* each setting lives.

## GPO 1 — Password Policy

**Linked at the domain root** (`lab.jaebeasty.local`), not a sub-OU. This is a hard AD requirement, not a preference: domain account password policy is only honored when set via a GPO linked at the domain level (or via Fine-Grained Password Policies for exceptions) — linking a password-policy GPO to any other OU silently does nothing for domain user accounts.

Path: `Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy`

| Setting | Value |
|---|---|
| Enforce password history | 24 passwords |
| Maximum password age | 90 days |
| Minimum password age | 1 day |
| Minimum password length | 14 characters |
| Password must meet complexity requirements | Enabled |
| Store passwords using reversible encryption | Disabled |

### Issue encountered — GPO precedence

After linking and running `gpupdate /force`, `net accounts` still showed the **out-of-the-box defaults** (`Minimum password length: 7`, `Maximum password age: 42`) rather than the configured values. The GPO existed, was correctly linked, and had no errors — but it wasn't actually winning.

Root cause: `Get-GPInheritance` showed two GPOs linked at the domain root — `Default Domain Policy` at **Order: 1** and `GPO-PasswordPolicy` at **Order: 2**. In GPO link ordering, **lower order number = higher precedence**. `Default Domain Policy` ships with its own explicit password settings pre-configured, and those were winning the conflict against the newly-created GPO.

This is a well-known but easy-to-miss trap: a GPO being *linked* is not the same as a GPO being *in effect* when multiple GPOs at the same scope configure the same setting.

Fix — reorder the link to give the custom GPO priority:

```powershell
Set-GPLink -Name "GPO-PasswordPolicy" -Target "DC=lab,DC=jaebeasty,DC=local" -Order 1
gpupdate /force
net accounts
```

Confirmed after the fix:

```
Minimum password length:        14
Maximum password age (days):    90
Length of password history:     24
```

## GPO 2 — Screen Lock Timeout

**Linked at `OU=JAEBEASTY`** (corrected after an initial mistake — see below).

Path: `Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options`

Setting: **"Interactive logon: Machine inactivity limit"** → `900` seconds (15 minutes)

Chosen over the older screensaver-based lock timeout because it enforces at the session level directly rather than depending on a screensaver being enabled, which users can and do disable.

## GPO 3 — Disable Removable Media

**Linked at `OU=JAEBEASTY`** (corrected after an initial mistake — see below).

Path: `Computer Configuration → Policies → Administrative Templates → System → Removable Storage Access`

Setting: **"All Removable Storage classes: Deny all access"** → Enabled

## Issue encountered — wrong link target

Both GPO 2 and GPO 3 were initially created and linked at the **domain root** instead of `OU=JAEBEASTY` as the design called for. Not functionally broken (domain root is a parent scope, so `JAEBEASTY` still inherits it) but inconsistent with the documented design — and a real problem waiting to happen the moment an unrelated top-level OU is added later, since it would inherit these policies too.

Corrected:

```powershell
Remove-GPLink -Name "GPO-ScreenLockTimeout" -Target "DC=lab,DC=jaebeasty,DC=local"
New-GPLink -Name "GPO-ScreenLockTimeout" -Target "OU=JAEBEASTY,DC=lab,DC=jaebeasty,DC=local"

Remove-GPLink -Name "GPO-DisableRemovableMedia" -Target "DC=lab,DC=jaebeasty,DC=local"
New-GPLink -Name "GPO-DisableRemovableMedia" -Target "OU=JAEBEASTY,DC=lab,DC=jaebeasty,DC=local"
```

## Final Verification

```powershell
Get-GPO -All | Select-Object DisplayName, Id, CreationTime
Get-GPInheritance -Target "DC=lab,DC=jaebeasty,DC=local" | Select-Object -ExpandProperty GpoLinks
Get-GPInheritance -Target "OU=JAEBEASTY,DC=lab,DC=jaebeasty,DC=local" | Select-Object -ExpandProperty GpoLinks
net accounts
```

Confirmed: `GPO-PasswordPolicy` linked and winning at domain root; `GPO-ScreenLockTimeout` and `GPO-DisableRemovableMedia` correctly linked at `OU=JAEBEASTY`; `net accounts` reflects the configured password policy values, not defaults.
