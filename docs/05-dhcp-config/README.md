# Phase 05 — DHCP Configuration

**Status:** 🟢 Done
**Host(s) involved:** DC01
**Date:** July 2026

## Objective

Install and configure the DHCP Server role on DC01 so future domain clients
(starting with WIN11-PC01 in Phase 08) can automatically receive an IP
address, subnet mask, DNS server, and domain suffix — instead of requiring
manual static configuration on every device.

## Prerequisites

- Phase 04 complete: DNS cleaned up and verified
- Static IP planning already done in Phase 01: DC01 `.10`, SRV01 `.20`
  (planned), WEB01 `.30` (planned), WSUS01 `.40` (planned) — DHCP scope
  range chosen to avoid overlapping any of these

## Steps

### Part A — Installed and authorized the DHCP Server role
1. Installed via Server Manager → Add Roles and Features → **DHCP Server**
   (accepted the DHCP Management Tools dependency).
   ![alt text](<../../screenshots/installing DHCP server role.png>)

2. Completed the post-install **"Complete DHCP configuration"** wizard,
   which performs **authorization** — a domain-specific AD DS security
   requirement: an unauthorized DHCP server will install and appear
   configured but will silently refuse to lease any addresses. This exists
   to prevent rogue DHCP servers from being connected to a real network
   and hijacking traffic — directly relevant to the blue-team layer added
   at the end of this project.
   ![alt text](<../../screenshots/Creating DHCP Scope.png>)
    

### Part B — Created the DHCP scope
1. Planned the address range deliberately to avoid the static IPs already
   reserved for infrastructure (`.10`, `.20`, `.30`, `.40`) — chose
   `192.168.10.100`–`192.168.10.200` for the client-lease range, named
   `LabNet-Client`.
2. Configured scope options during scope creation:
   - **Router (Option 3):** intentionally left blank — LabNet is an
     isolated segment with no gateway/router device.
   - **DNS Domain Name (Option 15):** `company.local`
   - **DNS Servers (Option 6):** `192.168.10.10` (DC01)
   - Lease duration: left at the 8-day default.
3. Activated the scope immediately on creation.
![alt text](<../../screenshots/creating DHCP scope..png>)


## Verification

```powershell
Get-WindowsFeature -Name DHCP
```
`Install State: Installed`

```powershell
Get-DhcpServerInDC
```
Returns `192.168.10.10  dc01.company.local` — confirms DHCP is authorized
in Active Directory, not just installed.
![alt text](<../../screenshots/DHCP Installation confirmation.png>)

```powershell
Get-DhcpServerv4Scope
```
Shows scope `LabNet-Client`, `192.168.10.0/24`, range `192.168.10.100`–
`192.168.10.200`, **State: Active**.

```powershell
Get-DhcpServerv4OptionValue -ScopeId 192.168.10.0
```
Confirms Option 6 (DNS Servers) = `192.168.10.10`, Option 15 (DNS Domain
Name) = `company.local`, Option 51 (Lease Duration) = 8 days, and **no**
Option 3 (Router) — correctly absent for the isolated LabNet segment.

Full client-side lease testing is deferred to Phase 08, once WIN11-PC01
exists and can actually request a lease — nothing to test client-side yet.
![alt text](<../../screenshots/DHCP Test.png>)

## Troubleshooting Notes

- Minor: mistyped the scope ID in the first `Get-DhcpServerv4OptionValue`
  attempt (`192.16.10.0` instead of `192.168.10.0`, missing a digit),
  producing an `ObjectNotFound` error. Immediately recognized as a typo
  from the error message and corrected on retry — noted here mainly as a
  reminder that PowerShell's `ObjectNotFound` errors are often exactly
  what they say: check the input value first before assuming a deeper
  configuration problem.
- No other issues encountered — authorization and scope creation both
  succeeded on the first properly-executed attempt.

## Screenshots / Evidence

Place directly in `docs/05-dhcp-config/`, referenced inline at the point
each step is described:

1. **`05-01-dhcp-role-add-roles-wizard.png`** — Add Roles and Features
   wizard with DHCP Server checked, showing the management tools
   dependency popup.
   → Place under **Part A**, step 1.
2. **`05-02-dhcp-post-install-authorization.png`** — the "Complete DHCP
   configuration" wizard screen showing the authorization step.
   → Place under **Part A**, step 2 — this is the single most important
   screenshot in this phase, since it's proof you understood and
   deliberately completed the authorization step rather than skipping it.
3. **`05-03-new-scope-wizard-range.png`** — the New Scope Wizard's IP
   Address Range screen showing `192.168.10.100`–`192.168.10.200`.
   → Place under **Part B**, step 1–2.
4. **`05-04-new-scope-wizard-dns-option.png`** — the DNS Servers options
   screen showing `192.168.10.10` added.
   → Place under **Part B**, step 2.
5. **`05-05-get-dhcpserverindc-verification.png`** — the
   `Get-DhcpServerInDC` output confirming authorization (already captured
   as text in Verification above — screenshot optional here since the
   text output is short and already clear inline; include only if you
   want a visual alongside the command).
   ![alt text](<../../screenshots/DHCP Test.png>)

## Next Phase

[06 - OU Structure](../06-ou-structure/README.md)