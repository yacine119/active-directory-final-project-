# Enterprise Active Directory Lab — enterprise.local

A hands-on lab that simulates a small corporate environment: a Domain Controller, a hardened **Windows Server Core file server**, a structured OU tree for two locations, role-based security groups, departmental SMB shares with least-privilege permissions, a security-baseline GPO, mapped network drives, and Windows LAPS — all built in VMware Workstation.

**Domain:** `enterprise.local` · **Network:** internal `192.168.1.0/24`

![Network diagram](docs/network-diagram.svg)

## Objectives

- Design a realistic OU structure for two locations (HQ-TLM and QQ-ORN)
- Deploy a Windows Server Core file server and manage it remotely
- Implement role-based access control with security groups
- Apply NTFS + share permissions following least privilege
- Harden the environment with a security-baseline GPO (NTLMv2, SMB, Kerberos, auditing, firewall, brute-force protection)
- Deploy Windows LAPS

## Environment

| Name | Role | OS | IP |
|---|---|---|---|
| `SRV00` | Domain Controller (AD DS, DNS, Global Catalog) | Windows Server 2025 Datacenter (Evaluation) | `192.168.1.2` |
| `FS01` | File server (Server Core) | Windows Server 2025 Standard (Evaluation) | DHCP |
| Client | Domain workstation | Windows 11 | DHCP |
| Client | Legacy workstation | Windows 7 | DHCP |

Forest and domain functional level: **Windows Server 2025** (screenshot 02). Host: VMware Workstation.

## OU Structure

```
enterprise.local
├── Admin
│   ├── Admin-Users
│   ├── Admin-Groups
│   └── Service-Accounts
├── Users.01
│   ├── HQ-TLM        → IT-users, HR-users, Finance-users, Sales-users, Management-users
│   └── QQ-ORN        → IT-users, HR-users, Finance-users, Sales-users, Management-users
├── Computers.01
│   ├── HQ-TLM
│   └── QQ-ORN
├── Groups
│   ├── Role-Groups   → HQ-TLM, QQ-ORN
│   └── Distribution-groups
├── Servers
│   ├── File-servers
│   ├── App-Servers
│   └── Web-servers
├── Domain Controllers
└── Disabled-Accounts
```

See screenshot 05 for the live view in *Active Directory Users and Computers*.

## Security Groups (Role-Groups)

| Group | Purpose |
|---|---|
| `GG-IT-Users` / `GG-IT-Managers` | IT employees / IT managers |
| `GG-HR-Users` / `GG-HR-Managers` | HR employees / HR managers |
| `GG-Finance-Users` / `GG-Finance-Managers` | Finance employees / Finance managers |
| `GG-Sales-Users` / `GG-Sales-Managers` | Sales employees / Sales managers |
| `GG-Management-Users` | Management department |

## File Server (FS01)

**Disk layout**

| Drive | Label | Contents |
|---|---|---|
| `C:` | System | Windows Server 2025 Core (60 GB disk) |
| `E:` | Data | `Departments\{IT, HR, Finance, Sales, Management}`, `Projects`, `Public` (40 GB disk) |
| `F:` | backup | `Daily`, `Weekly`, `Monthly`, `SystemImages` (25 GB disk) |

**SMB shares**

| Share | Path | Access |
|---|---|---|
| `IT` | `E:\Departments\IT` | IT Users + Managers |
| `HR` | `E:\Departments\HR` | HR Users + Managers |
| `Finance` | `E:\Departments\Finance` | Finance Users + Managers |
| `Sales` | `E:\Departments\Sales` | Sales Users + Managers |
| `Management` | `E:\Departments\Management` | Management Users |
| `Public` | `E:\Public` | Domain Users (Read) |

**Permissions matrix (least privilege)**

| Folder | IT Users | IT Mgrs | HR Users | HR Mgrs | Domain Admins | Domain Users |
|---|---|---|---|---|---|---|
| IT | Read | Modify | – | – | Full | – |
| HR | – | – | Read | Modify | Full | – |
| Public | – | – | – | – | Full | Read |

The Finance, Sales and Management shares use the same group-based model (their own Users/Managers groups, Domain Admins full control, nobody else). NTFS inheritance was removed on the department folders for strict control, and access is granted only through the role groups above.

**Notes**

- Shares don't appear under *Network* in File Explorer. That's normal for Server Core, so access is by UNC path: `\\FS01\ShareName`.
- Drive mappings are deployed with Group Policy Preferences.

## Security Baseline GPO

One **Security Baseline** GPO, linked at the domain root, hardens the domain. Settings shown in the screenshots:

| Area | Setting |
|---|---|
| Authentication | LAN Manager authentication level: *Send NTLMv2 response only. Refuse LM & NTLM* |
| Kerberos | Maximum lifetime for service ticket 720 minutes; user ticket 12 hours; renewal 10 days; user logon restrictions enforced |
| SMB | Mandate minimum SMB version enabled (Lanman Server) |
| Auditing | Advanced audit policy: credential validation, Kerberos authentication and service ticket operations, other account-logon events |
| Brute-force protection | IP blocking time set to 15 minutes |
| Services | Print Spooler disabled |
| Firewall (Domain profile) | On; inbound blocked by default, outbound allowed; logging of dropped packets and successful connections |
| User rights | Local logon, RDP logon and network-logon denial restricted to specific accounts |
| LAPS | Windows LAPS policy: password encryption, password backup directory, password age limit, post-authentication actions |

## Challenge & Solution

**Problem:** Clients received *"Accessing the resource has been disallowed"* when connecting to file shares.

**Root cause:** The Security Baseline GPO forced SMB signing (`RequireSecuritySignature = True`).

**Solution:** Changed the policy to *Digitally sign communications (if client agrees)* instead of *always*, which restored client compatibility while keeping signing negotiated. Screenshot 12 shows the policy and its description.

## Testing Performed

- Domain join and secure channel verification (screenshot 09)
- User logon from Windows 11 and Windows 7
- Access testing with normal users vs. managers on the departmental shares
- Mapped network drives validation
- Windows LAPS password retrieval
- NTFS and share permissions verification

## Screenshots

All files are in [`/screenshots`](screenshots), numbered in build order.

### Build

**01 — Installing AD DS, Group Policy Management and the AD tools on the DC**
![01](screenshots/01-roles-installation.png)

**02 — AD DS Configuration Wizard: forest/domain functional level Windows Server 2025, DNS and Global Catalog, DSRM password (masked)**
![02](screenshots/02-adds-configuration-wizard.png)

**03 — Ethernet1 properties on the DC VM (second NIC)**
![03](screenshots/03-dc-second-nic.png)

**04 — SConfig on the Server Core file server (before rename and domain join)**
![04](screenshots/04-fs01-sconfig.png)

**05 — OU structure in Active Directory Users and Computers**
![05](screenshots/05-ou-structure.png)

### File server

**06 — PowerShell on FS01: creating the department folders with `New-Item`, `Get-Volume`, `tree`**
![06](screenshots/06-fs01-folder-creation.png)

**07 — `tree` of `E:\` and `F:\`, plus `Get-Disk`**
![07](screenshots/07-fs01-volume-tree.png)

**08 — `Get-Volume` and the backup volume layout (Daily / Weekly / Monthly / SystemImages)**
![08](screenshots/08-fs01-backup-volume.png)

**09 — Domain connectivity: `Test-ComputerSecureChannel`, DNS resolution and `Test-NetConnection` to SRV00**
![09](screenshots/09-domain-connectivity-test.png)

### Security Baseline GPO

**10 — Kerberos policy: maximum lifetime for service ticket 720 minutes**
![10](screenshots/10-gpo-kerberos-policy.png)

**11 — LAN Manager authentication level: NTLMv2 only, refuse LM & NTLM**
![11](screenshots/11-gpo-lan-manager-auth.png)

**12 — SMB signing policy (the setting behind the share-access error)**
![12](screenshots/12-gpo-smb-signing.png)

**13 — Lanman Server: mandate the minimum version of SMB (enabled)**
![13](screenshots/13-gpo-min-smb-version.png)

**14 — User rights assignment**
![14](screenshots/14-gpo-user-rights.png)

**15 — Advanced audit policy: Account Logon**
![15](screenshots/15-gpo-advanced-audit.png)

**16 — Brute-force protection blocking time: 15 minutes**
![16](screenshots/16-gpo-bruteforce-protection.png)

**17 — System services: Print Spooler disabled**
![17](screenshots/17-gpo-system-services.png)

**18 — Windows Defender Firewall overview (Domain profile)**
![18](screenshots/18-gpo-firewall-overview.png)

**19 — Firewall Domain profile: on, inbound blocked, outbound allowed**
![19](screenshots/19-gpo-firewall-domain-profile.png)

**20 — Firewall Domain profile: notification and rule-merging settings**
![20](screenshots/20-gpo-firewall-customize.png)

**21 — Firewall logging: dropped packets and successful connections**
![21](screenshots/21-gpo-firewall-logging.png)

### LAPS

**22 — Windows LAPS policy: post-authentication actions**
![22](screenshots/22-gpo-laps-post-auth.png)

## Skills Demonstrated

- Active Directory design and administration (multi-location OU model)
- Group Policy creation and management, including a security baseline
- Windows Server Core administration with PowerShell
- NTFS and SMB permissions (least privilege), role-based access control
- Windows LAPS deployment
- Troubleshooting network and permission issues (SMB signing)

## Known Limitations / What I'd Do Differently

- **Single domain controller.** One DC means no redundancy; a second DC and Sites & Services would be the next step.
- **Role model.** Permissions are assigned directly to global groups (`GG-*`). The AGDLP pattern (accounts → global groups → domain local groups → permissions) scales better across locations.
- **Windows 7 client.** Windows 7 is out of support and can't use Windows LAPS; it only makes sense as an isolated legacy-compatibility test client.
- **Permissions matrix.** Only IT, HR and Public are tabulated in detail; Finance, Sales and Management follow the same model.

## Future Improvements

- [ ] Second domain controller for redundancy
- [ ] DFS Namespace for the shares
- [ ] Centralized logging
- [ ] AppLocker / WDAC
- [ ] Hybrid join with Microsoft Entra ID

## License

MIT — see [LICENSE](LICENSE).

---

**Author:** Bouaiad Mohamed Yacine · **Project type:** Enterprise Active Directory home lab · **Status:** Completed
