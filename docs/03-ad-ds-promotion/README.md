

03 ad ds promotion readme · MD
# Phase 03 — Install AD DS + Promote DC01 to Domain Controller
 
**Status:** 🟢 Done
**Host(s) involved:** DC01
**Date:** July 2026
 
## Objective
 
Install the Active Directory Domain Services role on DC01 and promote it to
be the first domain controller of a brand-new forest (`company.local`),
turning DC01 from a standalone server into the identity and authentication
backbone for the rest of the lab.
 
## Prerequisites
 
- Phase 02 complete: Server 2022 (Desktop Experience) installed, static IP
  `192.168.10.10` set on the LabNet adapter with DNS pointed at itself,
  hostname set to `DC01`
## Steps
 
1. Attempted to install the AD DS role via Server Manager's "Add Roles and
   Features" wizard — on the first pass, mistakenly selected **Active
   Directory Certificate Services (AD CS)** instead of **Active Directory
   Domain Services (AD DS)**, since the two sit close together in the role
   list. Caught this via `Get-WindowsFeature`, which showed `AD-Certificate`
   installed but `AD-Domain-Services` still `Available`.
2. Removed AD CS cleanly via Server Manager → Remove Roles and Features
   (also removed the dependent Certification Authority sub-role), verified
   removal with `Get-WindowsFeature -Name AD-Certificate` showing
   `Available` again.
3. Installed the correct role via PowerShell, to eliminate any further
   checkbox ambiguity:
```powershell
   Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```
   Verified with `Get-WindowsFeature -Name AD-Domain-Services` →
   `Installed`.
4. Promoted DC01 to a domain controller via the AD DS Configuration Wizard
   (Server Manager notification flag → "Promote this server to a domain
   controller"):
   - Deployment: **Add a new forest**
   - Root domain name: `company.local`
   - Forest/Domain functional level: Windows Server 2016 (default)
   - DNS Server: installed alongside promotion (checked, default)
   - NetBIOS name: `COMPANY` (auto-filled, accepted)
   - DSRM password set and stored separately for disaster-recovery use
   - Ignored the expected "DNS delegation cannot be created" warning —
     correct behavior on an isolated lab domain with no parent DNS zone
5. Wizard completed prerequisite checks (warnings only, no errors),
   installed AD DS/DNS, and rebooted automatically.
6. Logged in post-reboot as `COMPANY\Administrator` (same password as the
   original local Administrator account — carried over automatically).
## Verification
 
```powershell
Get-ADDomain
```
Confirmed `DNSRoot: company.local`, `DistinguishedName: DC=company,DC=local`,
`NetBIOSName: COMPANY`, forest/domain correctly self-referencing.
 
```powershell
Get-Service NTDS, DNS, Netlogon
```
All three: **Running**.
 
```powershell
dcdiag
```
All core health tests passed: `Advertising`, `NetLogons`, `MachineAccount`,
`KnowsOfRoleHolders`, `RidManager`, `Replications`, all partition tests
(`ForestDnsZones`, `DomainDnsZones`, `Schema`, `Configuration`, `company`),
and enterprise tests (`LocatorCheck`, `Intersite`).
 
Two tests reported failures — investigated and explained below rather than
ignored, since understanding *why* a warning is safe is the actual skill
being demonstrated here.
 
```powershell
nslookup company.local
```
Resolved successfully, but returned three addresses (LabNet IP, NAT IP, and
an IPv6 address) since AD registered every network adapter on DC01, not
just the intended LabNet one. Flagged for cleanup in Phase 04.
 
## Troubleshooting Notes
 
- **Wrong role installed initially (AD CS instead of AD DS).** Root cause:
  visually similar checkbox labels in the Server Roles list. Resolution:
  removed the incorrect role via Server Manager, then installed the
  correct role via PowerShell (`Install-WindowsFeature -Name
  AD-Domain-Services`) to remove ambiguity for any future role installs.
- **`dcdiag` reported `DFSREvent` as failed** — SYSVOL replication (DFSR)
  logged initialization warnings during promotion. This is expected and
  benign on a single-DC forest: DFSR health checks are primarily meant to
  catch replication problems *between multiple* domain controllers: with
  only one DC, there is nothing to replicate to yet. No action needed
  unless a second DC is added later and this test still fails.
- **`dcdiag` reported `SystemLog` as failed** — investigated the underlying
  Event Log entries rather than dismissing the test result. Breakdown:
  - Several "unexpected shutdown" events, caused by VirtualBox's Power
    Off being used during earlier troubleshooting (AD CS mix-up, disk
    size issue) instead of a clean in-guest shutdown. Not an AD fault —
    changed practice going forward to always shut down from inside
    Windows.
  - Repeated NTP failures resolving `time.windows.com` — expected on an
    isolated lab network, since DC01's DNS server has no route to
    resolve public internet hostnames by default. Not a defect; a direct
    consequence of the intentional network isolation from Phase 01/02.
  - WinRM "not listening" warnings — WinRM (remote PowerShell management)
    isn't enabled by default and isn't required for AD DS to function;
    irrelevant until remote management is set up in a later phase.
- **DNS registered all of DC01's adapters, not just LabNet** — noted for
  resolution in Phase 04, where DNS zone/record cleanup happens properly
  rather than as an afterthought here.
## Screenshots / Evidence
 
Recommended shots, in the order a reader would want to see them (tells the
promotion story chronologically, and doubles as your own evidence trail):
 
1. `01-addcs-mixup-get-windowsfeature.png` — the `Get-WindowsFeature`
   output showing AD-Certificate installed instead of AD-Domain-Services.
   *Worth including precisely because it's a mistake* — it shows you can
   diagnose and self-correct, which reads as more credible to a reviewer
   than a suspiciously perfect walkthrough.
   ![alt text](<../../screenshots/AD-Domain-Services Installed sucessfully .png>)

2. `02-addc-role-installed.png` — `Get-WindowsFeature -Name
   AD-Domain-Services` showing `Installed`, after the fix.
   ![alt text](<../../screenshots/AD-Domain-Services Installed sucessfully .png>)

3. `03-dcpromo-wizard-deployment-config.png` — the "Add a new forest /
   company.local" wizard screen.
   ![alt text](<../../screenshots/promoting DC01 to Domain COntroller ..png>)

4. `04-dcpromo-prerequisites-check.png` — the prerequisites screen showing
   warnings-only (no red errors) right before clicking Install.
   ![alt text](<../../screenshots/promoting DC01 to Domain COntroller installation in progress ..png>)

5. `05-get-addomain-output.png` — full `Get-ADDomain` output.
![alt text](<../../screenshots/Get-ADDomain output.png>)

6. `07-nslookup-companylocal.png` — the `nslookup` result showing the
   multi-adapter registration issue, referenced forward into Phase 04.
   ![alt text](<../../screenshots/nslookup company.local result.png>)

## Next Phase
 
[04 - DNS Configuration](../04-dns-config/README.md)
 
