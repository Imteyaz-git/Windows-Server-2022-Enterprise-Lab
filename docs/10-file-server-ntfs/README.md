# Phase 10 — File Server / NTFS Permissions (SRV01)

**Status:** 🟡 In Progress (Part 1 of 2 — VM built and networked; domain
join and file shares pending)
**Host(s) involved:** SRV01
**Date:** 29-July 2026 (Part 1)

## Objective

Build SRV01 as a Windows Server 2022 Server Core file server, join it to
`company.local`, and configure department file shares with NTFS
permissions scoped to the `GG-Sales` and `GG-IT` security groups created
in Phase 07 — demonstrating the layered Share + NTFS permission model and
least-privilege access control.

## Prerequisites

- Phase 07 complete: `GG-Sales`, `GG-IT` security groups exist
- Phase 06 complete: `Company\Computers\Servers` OU exists

## Steps completed today (Part 1)

1. Created the SRV01 VM in VirtualBox: 2048 MB RAM, 2 vCPU, 60 GB
   dynamically-allocated VDI, storage at `C:\Lab\VMs`, "Skip Unattended
   Installation" checked.
   ![alt text](<../../screenshots/srv01 1 installation .png>)
   ![alt text](<../../screenshots/srv01 initial config 3.png>)

2. Configured network adapter before first boot: Internal Network
   (`LabNet`), Adapter Type explicitly set to Intel PRO/1000 MT Desktop
   (learned from the Phase 08 troubleshooting — don't leave this blank).
   ![alt text](<../../screenshots/srv01 3.png>)

3. Installed Windows Server 2022 **Standard Evaluation** (Server Core,
   no Desktop Experience) — chosen deliberately this time for lighter
   RAM usage and additional PowerShell practice, since file server
   management is commonly done via CLI/remote tools anyway.
   ![alt text](<../../screenshots/srv inst 1.png>)
   ![alt text](<../../screenshots/srv insta 2.png>)
   ![alt text](<../../screenshots/srv insta 3.png>)
   ![alt text](<../../screenshots/srv insta 4.png>)
   ![alt text](<../../screenshots/srv insta 5.png>)
   ![alt text](<../../screenshots/srv insta 6.png>)
   ![alt text](<../../screenshots/srv insta 7.png>)
   


4. Renamed the computer to `SRV01` via `sconfig` → Option 2, rebooted.
   ![alt text](<../../screenshots/srv config 2.png>) 

5. Configured static networking via PowerShell instead of `sconfig`'s
   Option 8 network wizard, due to a wizard bug (see Troubleshooting
   Notes below):
   - IP: `192.168.10.20` / `255.255.255.0`
   - No default gateway (intentional — isolated LabNet segment)
   - DNS: `192.168.10.10` (DC01 — SRV01 is a domain member, not a DC, so
     it points to DC01 for name resolution rather than itself)

     ![alt text](<../../screenshots/srv config 1.png>)
     ![alt text](<../../screenshots/srv01 net setup 1.png>)
     ![alt text](<../../screenshots/srv 01 net set 2.png>)
     ![alt text](<../../screenshots/srv01 net set 3.png>)

## Verification (Part 1)

```powershell
ipconfig /all
```
Confirmed: IPv4 `192.168.10.20`, subnet `255.255.255.0`, no default
gateway, DNS server `192.168.10.10`.

```powershell
ping 192.168.10.10
```
![alt text](<../../screenshots/srv01 net set 3.png>)
100% success — confirms basic LabNet connectivity to DC01 before
attempting domain join.

## Troubleshooting Notes

- **`sconfig`'s Option 8 network wizard cancels the entire operation if
  the Gateway field is left blank**, rather than accepting "no gateway"
  as a valid input — a known usability quirk of the text-menu tool.
  Worked around by configuring the static IP directly via PowerShell
  instead, which correctly supports omitting a gateway simply by not
  passing that parameter at all, with no awkward blank-field handling
  required.
- Minor: the `New-NetIPAddress` cmdlet did not behave as expected on
  first attempt; the address was successfully applied on a subsequent
  attempt. Confirmed final state was correct via `ipconfig /all` and a
  successful ping to DC01, rather than trusting the cmdlet's own
  behavior in isolation.
- Clipboard sharing not yet working on this VM (Guest Additions not
  installed yet) — deferred to the next session, noted so it isn't
  mistaken for a networking problem.

## Remaining for Part 2 (next session)

- [ ] Install Guest Additions + enable clipboard sharing
- [ ] Domain-join SRV01 to `company.local` (`sconfig` Option 1)
- [ ] Move the SRV01 computer object into `Company\Computers\Servers` OU
- [ ] Create `C:\Shares\Sales` and `C:\Shares\IT` folders
- [ ] Create SMB shares scoped to `GG-Sales` and `GG-IT`
- [ ] Set NTFS permissions (Modify, inheritance reset) per group
- [ ] Test access as `Rizwan` (Sales) and `Arman` (IT) — confirm each can
      only access their own department's share

## Screenshots / Evidence

Place in `docs/10-file-server-ntfs/`:

1. **`10-01-srv01-vm-settings.png`** — VirtualBox VM settings showing
   RAM/CPU/network configuration.
   → Place under **Part 1, step 1-2**.
2. **`10-02-ipconfig-static-verification.png`** — `ipconfig /all` output
   confirming static IP/DNS.
   → Place under **Verification (Part 1)**.

*(Part 2 screenshots to be added once that session is documented.)*

## Next Phase

Continue Phase 10, Part 2 (domain join, shares, NTFS permissions) before
moving to [11 - DFS](../11-dfs/README.md).