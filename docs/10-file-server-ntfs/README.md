# Phase 10 — File Server / NTFS Permissions (SRV01)

**Status:** 🟢 Done
**Host(s) involved:** DC01, SRV01, WIN11-PC01
**Date:** July–August 2026

## Objective

Build SRV01 as a Windows Server 2022 Server Core file server, join it to
`company.local`, and configure department file shares with NTFS
permissions scoped to the `GG-Sales` and `GG-IT` security groups created
in Phase 07 — demonstrating the layered Share + NTFS permission model and
least-privilege access control.

## Prerequisites

- Phase 07 complete: `GG-Sales`, `GG-IT` security groups exist
- Phase 06 complete: `Company\Computers\Servers` OU exists

## Steps

### Part 1 — VM creation and initial networking (previous session)
Created the SRV01 VM (2048 MB RAM, 2 vCPU, 60 GB dynamically-allocated
VDI), installed Windows Server 2022 Standard Evaluation as **Server
Core** (chosen deliberately over Desktop Experience for lighter RAM
usage and additional PowerShell practice), renamed to `SRV01` via
`sconfig`, and configured static networking (`192.168.10.20`, no
gateway, DNS `192.168.10.10`) via PowerShell after `sconfig`'s network
wizard proved unable to handle a blank gateway field correctly.

### Part 2 — Guest Additions limitation discovered, pivoted to remote management
Attempted to install Guest Additions and enable clipboard sharing
directly on SRV01's console, following the same process used
successfully on DC01 and WIN11-PC01. This did not work, and investigation
revealed why: clipboard sharing depends on **VBoxTray**, a system-tray
component that requires a desktop shell (Explorer) to run — which Server
Core does not have by design. This is not a fixable configuration issue;
it's an inherent limitation of the Server Core installation option.

**Resolution — pivoted to the realistic, production-appropriate approach:**
manage SRV01 remotely via PowerShell from DC01, rather than working at
SRV01's own console. This is genuinely how Server Core machines are
operated day-to-day in real environments.
```powershell
# On SRV01 (one-time, at the console):
Enable-PSRemoting -Force

# On DC01:
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "192.168.10.20" -Force
Enter-PSSession -ComputerName 192.168.10.20 -Credential SRV01\Administrator
```

### Part 3 — Domain join (via remote session)
```powershell
Add-Computer -DomainName "company.local" -Credential COMPANY\Administrator -Restart
```
First attempt failed with `The user name or password is incorrect` —
traced to a credential-entry issue at the interactive prompt, not an
actual wrong password. Resolved by using `Get-Credential` to capture
credentials explicitly (and pasting the password rather than typing it,
to eliminate transcription error) before passing them to `Add-Computer`.

After the join and reboot, moved the computer object into its OU (run
from DC01, since SRV01/Server Core does not have AD management tools
installed):
```powershell
Get-ADComputer -Identity "SRV01" | Move-ADObject -TargetPath "OU=Servers,OU=Computers,OU=Company,DC=company,DC=local"
```

### Part 4 — Created shares and NTFS permissions (via remote session)
```powershell
New-Item -Path "C:\Shares\Sales" -ItemType Directory
New-Item -Path "C:\Shares\IT" -ItemType Directory

New-SmbShare -Name "Sales" -Path "C:\Shares\Sales" -FullAccess "COMPANY\GG-Sales"
New-SmbShare -Name "IT" -Path "C:\Shares\IT" -FullAccess "COMPANY\GG-IT"

icacls "C:\Shares\Sales" /grant "COMPANY\GG-Sales:(OI)(CI)M" /inheritance:r
icacls "C:\Shares\IT" /grant "COMPANY\GG-IT:(OI)(CI)M" /inheritance:r
```

Design reasoning: Share-level permissions were deliberately kept
generous (Full Access to the relevant group) — the real, fine-grained
restriction happens at the **NTFS** layer via `icacls`, avoiding the
need to maintain two separate, potentially conflicting permission
systems for the same folder. `/inheritance:r` strips the folder's
default inherited permissions, which would otherwise silently allow
broader built-in groups access despite the explicit grant; `(OI)(CI)M`
grants Modify (not Full Control) so the group can read/write/delete
files but not alter permissions or take ownership.

## Verification

```powershell
hostname
```
(via remote session) Confirmed `SRV01`.

```powershell
Get-ADComputer -Identity "SRV01" | Select-Object Name, DistinguishedName
```
(on DC01) Confirmed `CN=SRV01,OU=Servers,OU=Computers,OU=Company,DC=company,DC=local`.

```powershell
icacls "C:\Shares\Sales"
icacls "C:\Shares\IT"
```
Confirmed each folder shows only its intended group —
`COMPANY\GG-Sales:(OI)(CI)(M)` on Sales, `COMPANY\GG-IT:(OI)(CI)(M)` on
IT — with no leftover broad inherited permissions.

```powershell
Get-SmbShare
```
[SCREENSHOT/OUTPUT PLACEHOLDER — confirm both Sales and IT shares listed
with correct paths]

### Part 5 — Live access testing as domain users (WIN11-PC01)

Tested actual access as both department users, from WIN11-PC01, using
their normal (non-administrative) domain login sessions.

**Important finding during testing:** `net use \\SRV01\<share>` succeeded
for **both** users against **both** shares, regardless of group
membership — initially appeared to be a permission misconfiguration.
Investigation (checking `Get-SmbShareAccess` directly on SRV01, which
confirmed only the intended group had share-level access) revealed this
is expected Windows behavior: `net use` only establishes a drive mapping
and does not by itself enforce the access check — the real permission
enforcement (both share and NTFS layers) happens at the point of an
actual file operation (listing, reading, writing). This meant the test
plan needed a follow-up "list/create a file" step to be a valid proof of
restriction, not just a successful `net use` connection.

**As Rizwan (Sales):**
```powershell
net use S: \\SRV01\Sales        # succeeds
Get-ChildItem S:\                # succeeds
New-Item -Path "S:\test-rizwan.txt" -ItemType File -Value "Created by Rizwan"  # succeeds

net use I: \\SRV01\IT           # succeeds (connection only)
Get-ChildItem I:\                # ACCESS DENIED
New-Item -Path "I:\test-rizwan-on-it.txt" -ItemType File  # ACCESS DENIED
```

**As Arman (IT):**
```powershell
net use I: \\SRV01\IT           # succeeds
Get-ChildItem I:\                # succeeds (empty)
New-Item -Path "I:\test-arman.txt" -ItemType File -Value "Created by Arman"  # succeeds

net use S: \\SRV01\Sales        # succeeds (connection only)
Get-ChildItem S:\                # ACCESS DENIED
```

Both results mirror each other exactly, confirming symmetric, correctly
enforced isolation between departments.

## Remaining for this phase

None — all parts complete.

## Troubleshooting Notes

- **Clipboard sharing does not work on Server Core**, regardless of how
  many times Guest Additions is installed — root cause is architectural
  (VBoxTray requires a desktop shell that Server Core doesn't have), not
  a misconfiguration. Recognized this after the second failed attempt
  rather than continuing to retry the same fix, and pivoted to remote
  PowerShell management instead — which is both a practical workaround
  and a more realistic representation of how Server Core is actually
  administered in production environments.
- `Add-Computer` domain join failed on the first attempt with a
  misleading-sounding credential error; root cause was an issue at the
  interactive credential prompt rather than the password itself being
  wrong. Resolved using `Get-Credential` with a pasted (not typed)
  password to remove transcription risk. Worth remembering for future
  troubleshooting: an "incorrect username or password" error from
  `Add-Computer` isn't always proof the stored password is wrong — the
  prompt itself can be the source of the error.
- **`net use` connection succeeding is not proof of actual access** — this
  was the key finding of Part 5. When Rizwan connected to `\\SRV01\IT`,
  the connection itself succeeded even though he has no membership in
  `GG-IT`. Windows permits the drive mapping to establish for any
  authenticated domain user; the real share + NTFS permission check only
  triggers on an actual file operation (`Get-ChildItem`, `New-Item`,
  etc.). Verified this wasn't a misconfiguration by checking
  `Get-SmbShareAccess` directly on SRV01, which confirmed only the
  intended group held share-level access — then confirmed the real
  enforcement point by testing an actual file operation, which correctly
  denied access. Lesson carried forward: any future access-control
  testing in this lab should always test an actual read/write operation,
  not just connection success, since connection success alone can give a
  false impression of unrestricted access.

## Screenshots / Evidence

Place in `docs/10-file-server-ntfs/`:

1. **`10-01-srv01-vm-settings.png`** — VirtualBox VM settings (from
   Part 1).
2. **`10-02-ipconfig-static-verification.png`** — static IP verification
   (from Part 1).
3. **`10-03-remote-pssession-connect.png`** — the `Enter-PSSession`
   connection into SRV01 from DC01, showing the `[192.168.10.20]:`
   prompt.
   → Place under **Part 2** — good evidence of the remote-management
   pivot.
4. **`10-04-domain-join-success.png`** — successful `Add-Computer`
   output after the credential fix.
   → Place under **Part 3**.
5. **`10-05-icacls-verification.png`** — the `icacls` output for both
   Sales and IT folders (already captured as text above; screenshot
   optional).
   → Place under **Part 4** / Verification.
6.  `10-06-rizwan-sales-access-full.png` — the
   full Rizwan-on-WIN11-PC01 PowerShell output showing Sales
   connect/list/create all succeeding.
   ![alt text](<../../screenshots/riz test .png>)
   → Place under **Part 5**.
7. [SCREENSHOT PLACEHOLDER] `10-07-rizwan-it-denied.png` — the
   `Get-ChildItem I:\` / `New-Item` Access Denied errors for Rizwan on
   the IT share.
   ![alt text](<../../screenshots/riz test it.png>)
   → Place under **Part 5** — this is the key "proof of enforcement"
   screenshot.
8.  `10-08-arman-it-access-full.png` — mirrored
   success for Arman on the IT share.
   ![alt text](<../../screenshots/arman it test.png>)
   → Place under **Part 5**.
9.  `10-09-arman-sales-denied.png` — the Access
   Denied error for Arman on the Sales share.
   ![alt text](<../../screenshots/arman it test.png>)
   → Place under **Part 5**.

## Next Phase

[11 - DFS](../11-dfs/README.md)