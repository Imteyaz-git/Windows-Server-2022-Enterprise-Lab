# Windows Server 2022 Enterprise Lab

A hands-on Active Directory enterprise lab built entirely in VMs, documenting
the design and deployment of a small enterprise network from the ground up —
Active Directory, DNS, DHCP, Group Policy, file services, PKI — followed by a
defensive/blue-team layer (auditing, Sysmon, SIEM log forwarding, hardening,
and a simulated attack + detection writeup).

Built as a learning project transitioning into cybersecurity, with a SOC/blue
team focus. This repo documents the process honestly, including mistakes and
troubleshooting — not just a polished end state.

## Status

🟢 Done | 🟡 In Progress | ⚪ Not Started

| Phase | Status |
|---|---|
| 01 - Planning | 🟢 Done 
| 02 - Server 2022 Install | 🟢 Done 
| 03 - AD DS Install + Promotion | 🟡 In Progress
| 04 - DNS Configuration | ⚪ Not Started |
| 05 - DHCP Configuration | ⚪ Not Started |
| 06 - OU Structure | ⚪ Not Started |
| 07 - Users & Groups | ⚪ Not Started |
| 08 - Client Domain Join | ⚪ Not Started |
| 09 - Group Policy | ⚪ Not Started |
| 10 - File Server / NTFS Permissions | ⚪ Not Started |
| 11 - DFS | ⚪ Not Started |
| 12 - Print Server | ⚪ Not Started |
| 13 - WDS | ⚪ Not Started |
| 14 - WSUS | ⚪ Not Started |
| 15 - IIS | ⚪ Not Started |
| 16 - AD CS (PKI) | ⚪ Not Started |
| 17 - Hyper-V | ⚪ Not Started |
| 18 - Windows Admin Center | ⚪ Not Started |
| 19 - Backup/Restore | ⚪ Not Started |
| 20 - Disaster Recovery | ⚪ Not Started |
| 21 - Enterprise Troubleshooting | ⚪ Not Started |
| Security Monitoring (Blue Team Layer) | ⚪ Not Started |

## Lab Topology

| Host | Role | IP |
|---|---|---|
| DC01 | Domain Controller (AD DS, DNS, DHCP, later AD CS) | 192.168.10.10 |
| SRV01 | File Server (shares, NTFS, DFS) | 192.168.10.20 |
| WIN11-PC01 | Windows 11 domain client | DHCP-assigned |
| WEB01 *(planned)* | IIS Web Server | 192.168.10.30 |
| WSUS01 *(planned)* | Patch Management | 192.168.10.40 |

- **Domain:** company.local
- **Subnet:** 255.255.255.0
- **Gateway:** 192.168.10.1
- **Hypervisor:** VMware Workstation Pro

## Repo Structure

```
docs/                    → one folder per phase, each with its own README
security-monitoring/     → blue-team layer, added after core build is complete
scripts/                 → reusable PowerShell scripts written along the way
screenshots/             → supporting evidence, referenced from phase docs
```

Each phase folder in `docs/` follows the same format: **Objective → Steps →
Verification → Troubleshooting → Screenshots**. See `docs/03-ad-ds-promotion/`
for the template once it's filled in.

## Why this project

Documenting a full enterprise AD build from scratch, then layering security
monitoring and detection on top, to demonstrate both the sysadmin fundamentals
and the blue-team thinking (least privilege, auditing, detection engineering)
relevant to a SOC analyst role — not just "I clicked through some wizards."

## Tools used

VMware Workstation Pro, Windows Server 2022 (Evaluation), Windows 11, and
later: Sysmon, Wazuh/Splunk Free, Kali Linux (for the attack simulation).
