# Phase 06 — OU Structure

**Status:** 🟢 Done
**Host(s) involved:** DC01
**Date:** 24-July 2026

## Objective

Design and create an Organizational Unit (OU) hierarchy under `company.local` ![alt text](<../../screenshots/Organizational Unit (OU) hierarchy.png>)
to organize users, computers, and groups by function — establishing the
container structure that Group Policy (Phase 09) and delegation will later
be applied against. AD's built-in default containers (Users, Computers)
cannot have Group Policy linked to them directly, which is why a custom OU
structure is required rather than using the defaults.

## Prerequisites

- Phase 05 complete

## Steps

1. Designed a nested OU structure with a single top-level `Company` OU to
   keep all lab-created objects clearly separated from AD's built-in
   containers (Users, Computers, Domain Controllers, Builtin):
   ```
   company.local
   └── Company
       ├── Admins
       ├── Employees
       │   ├── Sales
       │   └── IT
       ├── Computers
       │   ├── Servers
       │   └── WorkStations
       └── Groups
   ```
2. Created each OU via Active Directory Users and Computers (ADUC) →
   right-click parent → New → Organizational Unit, keeping "Protect
   container from accidental deletion" enabled on every OU.
   ![alt text](<../../screenshots/Creating the OUs (GUI).png>)
   ![alt text](<../../screenshots/Creating OUs .png>)

3. Design reasoning (least-privilege groundwork for later phases):
   - **Admins separated from Employees** — different Group Policy
     (e.g. password/security policy) will need to apply to admin
     accounts than to regular staff; this can't be done if they share
     an OU.
   - **Employees split into department sub-OUs (Sales, IT)** — enables
     department-specific GPOs later in Phase 09.
   - **Computers separated into Servers / WorkStations** — computer
     objects receive their own GPOs (e.g. workstation lock-screen
     policy) independent of user-based policy, so kept in their own
     branch rather than mixed with user OUs.
   - **A dedicated Groups OU** — keeps security groups (created in
     Phase 07) in one predictable location.

## Verification

```powershell
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName
```
![alt text](<../../screenshots/OUs creation confirmation.png>)

Confirmed full nested hierarchy, correctly rooted under
`OU=Company,DC=company,DC=local`, with each child OU's Distinguished Name
reflecting the intended nesting (e.g.
`OU=Sales,OU=Employees,OU=Company,DC=company,DC=local`). AD's own default
`Domain Controllers` OU is untouched, as expected.

## Troubleshooting Notes

No issues encountered — OU creation completed cleanly on the first pass.

## Screenshots / Evidence

Place in `docs/06-ou-structure/`:

1. **`06-01-aduc-ou-tree-expanded.png`** — Active Directory Users and
   Computers with the full `Company` OU tree expanded, showing all
   nested sub-OUs.
   → Place near the top of this doc, right after the Objective section —
   this single screenshot communicates the entire structure visually
   faster than reading the tree diagram in text.
2. **`06-02-new-ou-protect-deletion-dialog.png`** — the New Object -
   Organizational Unit dialog showing "Protect container from accidental
   deletion" checked.
   → Place under step 2, as evidence of following this practice
   deliberately rather than leaving a default unchecked.
3. **`06-03-get-adorganizationalunit-output.png`** — the verification
   command output (already captured as text above; screenshot optional
   here, include only if you want a visual alongside the terminal output).

## Next Phase

[07 - Users & Groups](../07-users-groups/README.md)