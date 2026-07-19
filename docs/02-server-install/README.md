# Phase 02 — Windows Server 2022 Install (DC01)

**Status:** 🟢 Done
**Host(s) involved:** DC01
**Date:** 19 July 2026

## Objective

Install Windows Server 2022 (Standard Evaluation, Desktop Experience) as the
base OS for DC01, configure static networking on the isolated lab segment,
and set the correct hostname — establishing the foundation that Active
Directory will be promoted on top of in Phase 03.

## Prerequisites

- Phase 01 (Planning) complete
- `SERVER_EVAL_x64FRE_en-us.iso` downloaded and stored at `C:\Lab\ISOs\`
- VirtualBox installed on host

## Environment

- **Hypervisor:** Oracle VirtualBox (not VMware, per available tooling)
- **Host machine:** Windows, 8 GB RAM
- **VM specs:** 2048 MB RAM, 2 vCPU, 60 GB dynamically-allocated VDI
- **Install edition:** Windows Server 2022 Standard Evaluation (Desktop
  Experience) — GUI chosen deliberately for a first build, accepting the
  higher RAM footprint vs. Server Core
- **Networking:** two virtual NICs
  - Adapter 1 → Internal Network (`LabNet`) → static `192.168.10.10`
  - Adapter 2 → NAT → DHCP-assigned, outbound internet only

## Steps

1. Created the DC01 VM in VirtualBox: name `DC01`, storage folder set
   explicitly to `C:\Lab\VMs` (not the default profile path), 2048 MB RAM,
   2 vCPU, 60 GB dynamically-allocated VDI, ISO attached at creation.
2. Configured both network adapters (Internal Network "LabNet" + NAT)
   *before* first boot.
3. Ran through interactive Windows Setup: language/time/keyboard →
   edition selection (Standard Evaluation, Desktop Experience) → custom
   install → selected the virtual disk → installation completed.
4. Set local Administrator account password on first boot (OOBE screen).
5. Identified the two NICs by their auto-assigned IPs (`10.0.2.15` =
   NAT, `169.254.x.x` autoconfig = LabNet), then configured LabNet with:
   - IP: `192.168.10.10`
   - Subnet: `255.255.255.0`
   - Gateway: none (isolated segment)
   - DNS: `192.168.10.10` (self — required ahead of Phase 03's AD DS promotion)
6. Renamed both adapters in Network Connections (`LabNet`, `NAT`) for clarity.
7. Renamed the computer from the default `WIN-XXXXXXX` to `DC01` via
   Server Manager → Local Server → System Properties, then rebooted.

## Verification

```powershell
hostname
ipconfig /all
```
Confirmed:
- Hostname returns `DC01`
- LabNet adapter shows `192.168.10.10` / `255.255.255.0` / DNS `192.168.10.10`
- NAT adapter shows a `10.0.2.x` DHCP-assigned address

## Troubleshooting Notes

- **VirtualBox's Unattended Installation silently auto-installed Server
  Core** the first time the VM was created, because an ISO was attached
  during the New VM wizard with the "Skip Unattended Installation" option
  left unchecked. This skipped every interactive setup screen (including
  the edition picker) and dropped straight into `sconfig` post-install —
  which looked like a broken/unexpected screen at first. Fixed by deleting
  the VM entirely (Remove → Delete all files) and recreating it with
  **Skip Unattended Installation checked**, which restored the normal
  interactive Setup flow and allowed explicitly choosing the Desktop
  Experience edition.
- **Virtual disk was created at 25 GB instead of the intended 60 GB** on
  a later attempt — caught during the disk-selection screen in Setup
  before installing. Fixed by cancelling Setup, removing the undersized
  disk attachment, deleting it via Virtual Media Manager, and creating a
  fresh 60 GB dynamically-allocated VDI before restarting installation.
- Ctrl+Alt+Del at the Windows lock screen goes to the host OS, not the
  VM, by default — used VirtualBox's **Input → Keyboard → Insert
  Ctrl-Alt-Del** menu option instead.

## Screenshots / Evidence

- [ ] Screenshot: VirtualBox VM settings — Network tab (both adapters)
![alt text](<../../screenshots/VirtualBox VM settings — Network tab.png>)
![alt text](<../../screenshots/virtual box network setting adapter 2.png>)

- [ ] Screenshot: `ipconfig /all` output showing static LabNet IP
![alt text](<../../screenshots/ip configuration result.png>)

- [ ] Screenshot: Server Manager dashboard post-rename showing hostname `DC01`
![alt text](<../../screenshots/Server Manager dashboard post-rename showing hostname `DC01`.png>)

## Next Phase

[03 - AD DS Install + Promotion](../03-ad-ds-promotion/README.md)