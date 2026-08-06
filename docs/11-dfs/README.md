# Phase 11 — DFS (Distributed File System)

**Status:** 🟡 In Progress (Part 1 of 2 — SRV02 built, networked, and
domain-joined; DFS Namespaces and DFS Replication role install/config
pending)
**Host(s) involved:** DC01, SRV01, SRV02
**Date:** August 2026 (Part 1)

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
   ![alt text](<../../screenshots/creating srv02 1.png>)
2. Configured network adapter before first boot: Internal Network
   (`LabNet`), Adapter Type explicitly set to Intel PRO/1000 MT Desktop
   from the start (avoiding the blank-adapter issue hit during Phase 08).
   ![alt text](<../../screenshots/creating srv02 2.png>)
3. Installed Windows Server 2022 Standard Evaluation as **Server Core**
   (same rationale as SRV01 — lighter weight, and clipboard limitations
   make console-based GUI work impractical anyway).
   ![alt text](<../../screenshots/creating srv02 5.png>)
4. At SRV02's console (minimal typing, before remote management was
   available): set static IP via PowerShell directly (applying the
   Phase 10 lesson that `sconfig`'s network wizard fails on a blank
   gateway field), renamed the computer to `SRV02`, and enabled
   `Enable-PSRemoting -Force`.
   ![alt text](<../../screenshots/srv 02 initial config.png>)
   ![alt text](<../../screenshots/srv 02 initial config 2.png>)
5. Switched to remote management from DC01 immediately, rather than
   attempting any further console-based configuration:
   ```powershell
   Set-Item WSMan:\localhost\Client\TrustedHosts -Value "192.168.10.20,192.168.10.21" -Force
   Enter-PSSession -ComputerName 192.168.10.21 -Credential SRV02\Administrator
   ```
   ![alt text](<../../screenshots/access srv02 through DC01.png>)
   
6. Domain-joined SRV02 via the remote session, using `Get-Credential`
   with a pasted password (avoiding the credential-entry issue
   encountered during SRV01's domain join in Phase 10):
   ```powershell
   $cred = Get-Credential
   Add-Computer -DomainName "company.local" -Credential $cred -Restart
   ```
   ![alt text](<../../screenshots/accessing srv02 through DC01.png>)
7. Moved the SRV02 computer object into its OU (run from DC01):
   ```powershell
   Get-ADComputer -Identity "SRV02" | Move-ADObject -TargetPath "OU=Servers,OU=Computers,OU=Company,DC=company,DC=local"
   ```
   ![alt text](<../../screenshots/srv02 verification 2.png>)

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

## Remaining for this phase (Part 2, next session)

- [ ] Install `FS-DFS-Namespace` and `FS-DFS-Replication` roles on both
      SRV01 and SRV02
- [ ] Create a DFS Namespace (e.g. `\\company.local\Files`) with folder
      targets pointing to the real Sales/IT shares
- [ ] Create a DFS Replication group between SRV01 and SRV02 for a
      shared folder, and verify files replicate both directions
- [ ] Test client access via the namespace path from WIN11-PC01

## Screenshots / Evidence

Place in `docs/11-dfs/`:

1. **`11-01-srv02-vm-settings.png`** — VirtualBox VM settings for SRV02.
2. **`11-02-ipconfig-verification.png`** — static IP verification (already
   captured as text above; screenshot optional).
3. **`11-03-adcomputer-ou-verification.png`** — the OU placement
   verification (already captured as text above; screenshot optional).

*(Part 2 screenshots — DFS role install, namespace, replication group,
and client test — to be added once that session is documented.)*

## Next Phase

Complete Part 2 (DFS role installation, namespace, and replication)
before moving to [12 - Print Server](../12-print-server/README.md).