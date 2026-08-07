# Phase 11 — DFS (Distributed File System)

**Status:** 🟢 Done
**Host(s) involved:** DC01, SRV01, SRV02
**Date:** August 2026

## Objective

Build a second file server (SRV02) so DFS can be demonstrated properly —
both **DFS Namespaces** (a unified virtual path abstracting the real
location of shared folders) and **DFS Replication** (keeping identical
copies of a folder synced across multiple servers for redundancy).

## Why a second file server was necessary

DFS has two genuinely separate components. DFS Namespaces has real value
even with a single file server. **DFS Replication requires at least two
file servers to demonstrate meaningfully** — setting it up against only
one server would not prove anything real. Rather than skip DFS
Replication entirely (as was done with the DNS forwarder in Phase 04, due
to an environment constraint outside the lab's control), a second file
server (SRV02) was built specifically to allow DFS Replication to be
demonstrated properly, since building a VM is fully within the lab's
control unlike Phase 04's host networking issue.

## Prerequisites

- Phase 10 complete: SRV01 built, domain-joined, remote PowerShell
  management pattern established (Server Core has no usable clipboard —
  see Phase 10 troubleshooting notes — so SRV02 was built applying that
  lesson from the start)
- `Company\Computers\Servers` OU exists (Phase 06)

## Steps completed today (Part 1)

1. Created the SRV02 VM in VirtualBox: 2048 MB RAM, 2 vCPU, 60 GB
   dynamically-allocated VDI, storage at `C:\Lab\VMs`, "Skip Unattended
   Installation" checked.
2. Configured network adapter before first boot: Internal Network
   (`LabNet`), Adapter Type explicitly set to Intel PRO/1000 MT Desktop
   from the start (avoiding the blank-adapter issue hit during Phase 08).
3. Installed Windows Server 2022 Standard Evaluation as **Server Core**
   (same rationale as SRV01 — lighter weight, and clipboard limitations
   make console-based GUI work impractical anyway).
4. At SRV02's console (minimal typing, before remote management was
   available): set static IP via PowerShell directly (applying the
   Phase 10 lesson that `sconfig`'s network wizard fails on a blank
   gateway field), renamed the computer to `SRV02`, and enabled
   `Enable-PSRemoting -Force`.
5. Switched to remote management from DC01 immediately, rather than
   attempting any further console-based configuration:
   ```powershell
   Set-Item WSMan:\localhost\Client\TrustedHosts -Value "192.168.10.20,192.168.10.21" -Force
   Enter-PSSession -ComputerName 192.168.10.21 -Credential SRV02\Administrator
   ```
6. Domain-joined SRV02 via the remote session, using `Get-Credential`
   with a pasted password (avoiding the credential-entry issue
   encountered during SRV01's domain join in Phase 10):
   ```powershell
   $cred = Get-Credential
   Add-Computer -DomainName "company.local" -Credential $cred -Restart
   ```
7. Moved the SRV02 computer object into its OU (run from DC01):
   ```powershell
   Get-ADComputer -Identity "SRV02" | Move-ADObject -TargetPath "OU=Servers,OU=Computers,OU=Company,DC=company,DC=local"
   ```

## Verification (Part 1)

```powershell
hostname
```
(via remote session into SRV02) Confirmed `SRV02`.

```powershell
ipconfig /all
```
Confirmed: IPv4 `192.168.10.21`, subnet `255.255.255.0`, no default
gateway, DNS server `192.168.10.10`.

```powershell
Get-ADComputer -Identity "SRV02" | Select-Object Name, DistinguishedName
```
(on DC01) Confirmed
`CN=SRV02,OU=Servers,OU=Computers,OU=Company,DC=company,DC=local`.

## Troubleshooting Notes

No issues encountered this session — every lesson learned during SRV01's
build in Phase 10 (adapter type must be explicitly set, `sconfig`'s
network wizard fails on a blank gateway, Server Core has no usable
clipboard so remote management should be established immediately,
credential prompts should use pasted passwords) was applied proactively
from the start, resulting in a clean build with no repeated mistakes.

## Steps (Part 2 — DFS role installation, Namespace, and Replication)

8. Installed `FS-DFS-Namespace` and `FS-DFS-Replication` roles on both
   SRV01 and SRV02 via remote PowerShell sessions from DC01.
   ![alt text](<../../screenshots/installing dfs on srv02.png>)
   ![alt text](<../../screenshots/install dfs sucess srv01.png>)
   ![alt text](<../../screenshots/dfs management on dc01.png>)
   ![alt text](<../../screenshots/dfs namespace 1 on dc01.png>)
   ![alt text](<../../screenshots/dfs namespace 2.png>)
   ![alt text](<../../screenshots/dfs namespace 3.png>)
   ![alt text](<../../screenshots/dfs namespace 4.png>)
   ![alt text](<../../screenshots/dfs replication 5.png>)
   ![alt text](<../../screenshots/dfs namespace confirmation.png>)
9. Installed the `RSAT-DFS-Mgmt-Con` management tools on **DC01**
   separately — the server roles alone do not add the DFS Management
   console to a machine that isn't itself running the DFS roles.
   ![alt text](<../../screenshots/dfs man console on dc01 installed sucess.png>)
10. Created a domain-based DFS Namespace `\\company.local\Files`, hosted
    on DC01 (namespace servers do not need to be file servers
    themselves).
11. Added a folder inside the namespace (`Sales`) with a folder target
    pointing to `\\SRV01\Sales` — after resolving a significant
    multi-layered access issue (see Troubleshooting Notes).
    ![alt text](<../../screenshots/add sales folder to namespace .png>)
    ![alt text](<../../screenshots/allowing dc01 to manage sales folder for dfs management.png>)
12. Created a matching empty folder `C:\Shares\Sales` on SRV02, then
    built a DFS Replication Group (`Sales-Replication`) via DFS
    Management: Multipurpose type, both SRV01 and SRV02 as members,
    Full Mesh topology, SRV01 set as Primary Member (initial source of
    truth), replicating `C:\Shares\Sales`.
    ![alt text](<../../screenshots/dfs replication 1.png>)
    ![alt text](<../../screenshots/dfs replication 3.png>)
    ![alt text](<../../screenshots/dfs replication 4.png>)
    ![alt text](<../../screenshots/dfs replication 5.png>)

13. Verified replication by creating a new file on SRV01 and confirming
    it appeared automatically on SRV02 within roughly a minute, with no
    manual copy step.
    ![alt text](<../../screenshots/dfs replication verified 7.png>)

## Verification

```powershell
Get-WindowsFeature -Name FS-DFS-Namespace, FS-DFS-Replication -ComputerName SRV01
Get-WindowsFeature -Name FS-DFS-Namespace, FS-DFS-Replication -ComputerName SRV02
```
Both roles `Installed` on both servers.

```powershell
Get-DfsnRoot -Path "\\company.local\Files"
```
`Type: Domain V2`, `State: Online`.

```powershell
Get-DfsnFolder -Path "\\company.local\Files\Sales"
```
`State: Online`, correctly targeting `\\SRV01\Sales`.

```powershell
Get-DfsReplicationGroup
```
`Sales-Replication`, `State: Normal`.

```powershell
Get-DfsrMember -GroupName "Sales-Replication"
```
Both SRV01 and SRV02 listed, `State: Normal`, each showing 1 inbound / 1
outbound connection (correct for a 2-member full mesh).

**Live replication test:**
```powershell
New-Item -Path "C:\Shares\Sales\replication-test.txt" -ItemType File -Value "Created on SRV01"
```
(on SRV01) — then, roughly a minute later, on SRV02:
```powershell
Get-ChildItem "C:\Shares\Sales"
```
Confirmed `replication-test.txt` present on SRV02, automatically
replicated with no manual copy. Notably, `test-rizwan.txt` (an older
file from Phase 10's access testing, pre-dating the replication group's
creation) was also present on SRV02 — confirming DFS Replication's
**initial sync** correctly captured pre-existing content, not just
changes made after the group was created.

## Troubleshooting Notes

- **DFS Management console did not appear in Server Manager → Tools on
  DC01** after installing the DFS roles on SRV01/SRV02. Root cause: the
  management console/tools are a separate installable component
  (`RSAT-DFS-Mgmt-Con`) from the actual server roles, and must be
  installed specifically on the machine used to *manage* DFS, not just
  the machines hosting the DFS shares. Resolved with
  `Install-WindowsFeature -Name RSAT-DFS-Mgmt-Con` on DC01.

- **Adding `\\SRV01\Sales` as a folder target repeatedly failed with
  "Access to the path is denied,"** despite `\\SRV01\Sales` genuinely
  existing and being reachable. This was the most involved
  troubleshooting sequence in the project so far — documented in full
  since the eventual root cause was not the first, second, or even third
  theory tested:
  1. **First theory — NTFS permissions:** `COMPANY\Administrator` (the
     account performing the DFS setup) is not a member of `GG-Sales`,
     which held the only NTFS grant on the folder from Phase 10.
     Added `COMPANY\Domain Admins:(OI)(CI)F` via `icacls`. Confirmed via
     `icacls` output that the grant was correctly applied — but the
     error persisted.
  2. **Second theory — stale Kerberos ticket:** suspected the current
     session's cached authentication token didn't yet reflect the
     `Domain Admins` membership. Ran `klist purge` and reconnected;
     `Get-ChildItem "C:\Shares\Sales"` then succeeded **when run inside
     a PowerShell remote session directly on SRV01** — appearing to
     confirm this theory. However, the same access from **DC01** (both
     via PowerShell and File Explorer, after a full interactive
     logoff/logon to guarantee an entirely fresh token) still failed —
     showing the Kerberos-ticket theory, while plausible, was not
     actually the root cause; the earlier apparent "fix" coincided with,
     but was not caused by, the ticket purge.
  3. **Root cause, found via direct evidence rather than further
     inference:** checked `Get-SmbShareAccess -Name "Sales"` directly
     and found the **share-level** permission (configured in Phase 10)
     still granted access to `GG-Sales` only — `Domain Admins` had only
     ever been added at the **NTFS** layer, never at the **share**
     layer. Per the "most restrictive wins" rule governing how Share and
     NTFS permissions combine, the share layer's lack of a `Domain
     Admins` entry was independently sufficient to block access,
     regardless of how correctly NTFS was configured. Fixed with:
     ```powershell
     Grant-SmbShareAccess -Name "Sales" -AccountName "COMPANY\Domain Admins" -AccessRight Full -Force
     ```
     Access succeeded immediately afterward, from DC01, with no further
     changes needed.
  - **Lesson:** when troubleshooting access issues, actual evidence
    (querying the real, current state of each permission layer
    directly) should be checked *before* trusting a plausible-sounding
    theory (like Kerberos caching) that happens to correlate with a
    change in behavior. The Kerberos ticket purge and logoff/logon were
    reasonable, standard troubleshooting steps, but they were not
    actually diagnostic on their own — checking `Get-SmbShareAccess`
    directly was.

## Screenshots / Evidence

Place in `docs/11-dfs/`:

1. **`11-01-srv02-vm-settings.png`** — VirtualBox VM settings for SRV02
   (Part 1).
2. **`11-02-ipconfig-verification.png`** — static IP verification
   (Part 1; text already captured above, screenshot optional).
3. **`11-03-adcomputer-ou-verification.png`** — OU placement verification
   (Part 1; text already captured above, screenshot optional).
4. **`11-04-dfs-roles-installed.png`** — `Get-WindowsFeature` output
   confirming DFS roles on both servers.
   → Place under **Part 2, step 8**.
5. **`11-05-new-namespace-wizard.png`** — the New Namespace wizard,
   domain-based namespace type selected.
   → Place under **step 10**.
6. **`11-06-smbshareaccess-before-after.png`** — the `Get-SmbShareAccess`
   output before and after the `Grant-SmbShareAccess` fix (the key
   diagnostic evidence for the access-denied saga).
   → Place under **Troubleshooting Notes, item 3** — this is the most
   important screenshot in the phase, since it's the actual proof of
   the real root cause versus the ruled-out theories.
7. **`11-07-replication-group-wizard.png`** — the New Replication Group
   wizard, showing both SRV01 and SRV02 as members.
   → Place under **step 12**.
8. **`11-08-replication-test-both-servers.png`** — side-by-side or
   sequential `Get-ChildItem` output from SRV01 and SRV02 showing
   `replication-test.txt` present on both.
   → Place under **Verification, live replication test** — the definitive
   "it works" screenshot for the whole phase.

## Next Phase

[12 - Print Server](../12-print-server/README.md)