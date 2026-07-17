# Phase 01 — Planning

**Status:** 🟢 Done
**Host(s) involved:** N/A (design/documentation phase, no VMs built yet)
**Date:** July 2026

## Objective

Design the lab topology, domain structure, and repo layout before touching
any VM software. The goal is to know exactly what's being built and why
before installation begins — avoiding rebuilds caused by wrong domain names,
bad IP planning, or an undocumented, ad-hoc repo structure.

## Prerequisites

None — this is the first phase.

## Steps

1. Defined the lab topology and roles for each machine:
   - **DC01** (192.168.10.10) — Domain Controller: AD DS, DNS, DHCP, later AD CS (PKI)
   - **SRV01** (192.168.10.20) — File Server: shares, NTFS permissions, DFS
   - **WIN11-PC01** — Windows 11 client, domain-joined via DHCP
   - Planned for later: **WEB01** (192.168.10.30, IIS), **WSUS01** (192.168.10.40, patching)

2. Chose domain name `company.local` and subnet `192.168.10.0/24`
   (gateway `192.168.10.1`) — an internal, non-routable lab network isolated
   from the host's real network via VMware's virtual networking.

3. Defined the full build order across 21 phases, from base server install
   through Active Directory, networking services, file/print services,
   PKI, and infrastructure management tools — followed by a separate
   blue-team/security-monitoring layer added after the core build.

4. Set up the GitHub repository (`Windows-Server-2022-Enterprise-Lab`) with:
   - A consistent `docs/NN-phase-name/` folder structure, one folder per
     build phase, each following the same documentation template
     (Objective → Steps → Verification → Troubleshooting → Screenshots)
   - A custom `.gitignore` to prevent VM disk images, ISOs, and snapshot
     files from ever being committed (these are multi-GB and shouldn't be
     version-controlled or redistributed)
   - A `security-monitoring/` folder reserved for the blue-team layer,
     to be built once DC01/SRV01/WIN11-PC01 are running
   - `scripts/` and `screenshots/` folders for reusable PowerShell and
     supporting evidence

5. Downloaded installation media:
   - `SERVER_EVAL_x64FRE_en-us.iso` (Windows Server 2022 Evaluation, from
     the official Microsoft Evaluation Center)
   - `Win11_25H2_EnglishInternational_x64_v2.iso` (Windows 11, official
     Microsoft download)

## Verification

- Repo structure confirmed by browsing the GitHub repo directly:
  `docs/` contains all 21 phase folders, each with a placeholder README.
- ISO filenames cross-checked against known official Microsoft Evaluation
  Center naming conventions.
- (To do before Phase 02) Verify ISO integrity with `Get-FileHash` against
  Microsoft's published checksums, and confirm free disk space is
  sufficient for the VMs planned (Server 2022 + Win11 + future hosts).

## Troubleshooting Notes

- Initial repo setup hit a few git/PowerShell environment issues, not
  planning mistakes as such, but worth noting since they're common
  first-time gotchas:
  - `git add .` failed with `fatal: not a git repository` when run from
    the wrong working directory (`C:\Projects` instead of the cloned repo
    folder) — fixed by `cd`-ing into the correct path.
  - Git flagged `detected dubious ownership` on the cloned folder, likely
    because it was initially cloned/created under a different Windows
    session context than the one now accessing it — fixed with
    `git config --global --add safe.directory <path>`.
  - `.gitignore` is a hidden dotfile and didn't copy over cleanly from
    Windows Explorer initially — recreated directly via PowerShell
    heredoc instead.

## Screenshots / Evidence

- N/A for this phase (design/documentation only, no GUI steps to capture).
  Repo structure itself is the evidence — see the live GitHub repo tree.

## Next Phase

[02 - Server 2022 Install](../02-server-install/README.md)
