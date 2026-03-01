# Active Directory Homelab — ApaseForge Domain

A fully functional Windows Server Active Directory environment built from scratch in a virtualized homelab. This project covers network design and planning, DNS and DHCP configuration, domain deployment, and user/group management — documented end-to-end with screenshots.

---

## Network Design

> For the full technical breakdown — including DHCP scope design decisions, lease validation, and the DC02 high availability plan — see [docs/network-architecture.md](docs/network-architecture.md).


The network was planned using an RFC 1918 private address space before any machines were configured. The goal was to create a structured, scalable IP scheme that mirrors how a real enterprise network is segmented.

**Subnet:** `192.168.123.0/24` — Subnet Mask: `255.255.255.0`
**Domain:** `activedirectory.apase1.com`

### IP Address Plan

| Range | Purpose | Notes |
|---|---|---|
| `.1 – .5` | Routers / Default Gateway | `.1` or `.254` as primary gateway; `.1–.5` reserved for VRRP failover across multiple routers |
| `.6 – .10` | Network devices | Switches, access points, and other infrastructure |
| `.11 – .20` | LAN services | NAS, backup servers, monitoring |
| `.21 – .30` | Servers | DC01 at `.21` — statically assigned |
| `.31 – .254` | DHCP clients | 224 addresses allocated dynamically to workstations and other clients |

> **Why this structure?** Reserving low addresses for infrastructure and servers makes firewall rules and ACLs predictable. The large dynamic pool (.31–.254) ensures room to grow without reorganising the scheme.

### Server Assignment

| Host | IP | Role |
|---|---|---|
| DC01 | `192.168.123.21` | Domain Controller, DNS, DHCP |

### DNS

DNS is handled by DC01, which acts as the authoritative nameserver for the internal domain. All clients (Windows and Linux) point to DC01 as their primary DNS server, ensuring internal hostname resolution across the domain.

---

## Homelab Environment

> The homelab was built on VMware using the `192.168.228.0/24` subnet (assigned by the hypervisor). The IP addressing plan above reflects the designed production-equivalent scheme.

| Machine | OS | Role | IP (Lab) |
|---|---|---|---|
| DC01 | Windows Server 2019 | Domain Controller, DNS, DHCP | 192.168.228.21 (static) |
| WS01 | Windows 11 | Domain-joined workstation | 192.168.228.131 (DHCP) |
| UB01 | Ubuntu Desktop | Linux client for cross-platform testing | 192.168.228.132 (DHCP) |

- **Domain:** `ad.apase1.com`
- **Hypervisor:** VMware (local homelab)

---

## What Was Built

### Phase 1 — Network Baseline & DHCP
Verified initial connectivity on DC01 before static IP assignment. Configured a DHCP scope (`ApaseForge-Client-Scope`) on DC01 to serve client IPs to WS01 and UB01. Validated leases with `ipconfig /all` on WS01, and confirmed Layer 3 connectivity between all three machines using ICMP ping tests in both directions.

![Step 01](screenshots/01-DC01-Initial-IPConfig-Before-Static-IP.png)
![Step 02](screenshots/02-DC01-DHCP-New-Scope-Wizard-ApaseForge-Client-Scope.png)
![Step 04](screenshots/04-WS01-IPConfig-All-DHCP-Validation-DC01-DNS.png)
![Step 05](screenshots/05-WS01-Ping-UB01-Layer3-Connectivity-Success.png)
![Step 06](screenshots/06-UB01-Ping-WS01-Bidirectional-ICMP-Success.png)
![Step 07](screenshots/07-DC01-ARP-and-Ping-UB01-192.168.228.132-ICMP-Success.png)

---

### Phase 2 — DNS Configuration
Assigned DC01 a static IP (`192.168.228.21`) and configured it as the primary DNS server. Created a Forward Lookup Zone (`ad.apase1.com`) and a Reverse Lookup Zone (`228.168.192.in-addr.arpa`) in DNS Manager. Added an A record for DC01 with an associated PTR record. Verified forward and reverse resolution using `nslookup` from DC01, and tested DNS resolution by hostname from WS01 and UB01.

![Step 08](screenshots/08-DC01-DNS-Reverse-Lookup-Zone-Wizard-192.168.228.png)
![Step 09](screenshots/09-DC01-DNS-Manager-Forward-and-Reverse-Zones-Configured.png)
![Step 10](screenshots/10-DC01-Static-IP-IPv4-Properties-192.168.228.21.png)
![Step 11](screenshots/11-DC01-DNS-A-Record-Added-dc01.ad.apase1.com.png)
![Step 12](screenshots/12-DC01-DNS-PTR-Record-and-NSLookup-Forward-Reverse-Verified.png)
![Step 14](screenshots/14-WS01-DNS-Ping-dc01.ad.apase1.com-Resolution-Success.png)
![Step 15](screenshots/15-WS01-DNS-Ping-router.ad.apase1.com-Resolution-Success.png)
![Step 16](screenshots/16-UB01-DNS-Ping-Internal-AD-Hostnames-Resolution-Success.png)

---

### Phase 3 — Active Directory Domain Services
Installed the AD DS role on DC01 via Add Roles and Features. Promoted DC01 to a Domain Controller and created a new forest with the domain `ad.apase1.com`. Confirmed domain authentication by logging in as `AD\Administrator` and running `whoami` to verify domain context.

![Step 17](screenshots/17-DC01-ADDS-Role-Installation-Progress.png)
![Step 18](screenshots/18-DC01-Domain-Login-AD-Administrator.png)
![Step 19](screenshots/19-DC01-Whoami-Confirms-AD-Domain-Administrator.png)

---

### Phase 4 — Users, Groups & Domain Join
Opened Active Directory Users and Computers (ADUC) and explored the default Users container. Created three custom Security Groups — `Claude AI`, `Cursor AI`, and `Gemini AI` — to simulate a department/role-based access structure. Created a new domain user `Cole Eden` (`ceden@ad.apase1.com`) and added them to the `Claude AI` group. Joined WS01 to the domain, confirmed it appeared in the Computers OU in ADUC, and logged into WS01 as the domain user `ceden` — verified via Task Manager.

![Step 20](screenshots/20-DC01-ADUC-Users-Container-Default-Groups.png)
![Step 21](screenshots/21-DC01-ADUC-Custom-AI-Security-Groups-Claude-Cursor-Gemini.png)
![Step 22](screenshots/22-DC01-ADUC-New-User-Cole-Eden-ceden-ad.apase1.com.png)
![Step 23](screenshots/23-DC01-ADUC-Cole-Eden-Added-to-Claude-AI-Group.png)
![Step 24](screenshots/24-DC01-ADUC-Computers-WS01-Domain-Joined.png)
![Step 25](screenshots/25-WS01-Task-Manager-ceden-Domain-User-Session-Active.png)

---

## Screenshot Index

| # | File | What it shows |
|---|------|---------------|
| 01 | `01-DC01-Initial-IPConfig-Before-Static-IP` | DC01 on DHCP before being configured |
| 02 | `02-DC01-DHCP-New-Scope-Wizard-ApaseForge-Client-Scope` | Creating the DHCP scope |
| 03 | `03-WS01-IPConfig-DHCP-Lease-192.168.228.31` | WS01 gets lease from new scope |
| 04 | `04-WS01-IPConfig-All-DHCP-Validation-DC01-DNS` | Full DHCP validation with all details |
| 05 | `05-WS01-Ping-UB01-Layer3-Connectivity-Success` | WS01 → UB01 ping success |
| 06 | `06-UB01-Ping-WS01-Bidirectional-ICMP-Success` | UB01 → WS01 bidirectional confirmation |
| 07 | `07-DC01-ARP-and-Ping-UB01-192.168.228.132-ICMP-Success` | DC01 pinging UB01 |
| 08 | `08-DC01-DNS-Reverse-Lookup-Zone-Wizard-192.168.228` | Creating reverse DNS zone |
| 09 | `09-DC01-DNS-Manager-Forward-and-Reverse-Zones-Configured` | Both zones visible in DNS Manager |
| 10 | `10-DC01-Static-IP-IPv4-Properties-192.168.228.21` | DC01 static IP set |
| 11 | `11-DC01-DNS-A-Record-Added-dc01.ad.apase1.com` | A record created for DC01 |
| 12 | `12-DC01-DNS-PTR-Record-and-NSLookup-Forward-Reverse-Verified` | nslookup confirms forward/reverse |
| 13 | `13-WS01-IPConfig-All-DHCP-192.168.228.131` | WS01 DHCP lease check |
| 14 | `14-WS01-DNS-Ping-dc01.ad.apase1.com-Resolution-Success` | WS01 resolves DC01 by name |
| 15 | `15-WS01-DNS-Ping-router.ad.apase1.com-Resolution-Success` | WS01 resolves router by name |
| 16 | `16-UB01-DNS-Ping-Internal-AD-Hostnames-Resolution-Success` | UB01 resolves internal AD names |
| 17 | `17-DC01-ADDS-Role-Installation-Progress` | ADDS role being installed |
| 18 | `18-DC01-Domain-Login-AD-Administrator` | First domain login screen |
| 19 | `19-DC01-Whoami-Confirms-AD-Domain-Administrator` | `whoami` confirms domain context |
| 20 | `20-DC01-ADUC-Users-Container-Default-Groups` | ADUC default groups visible |
| 21 | `21-DC01-ADUC-Custom-AI-Security-Groups-Claude-Cursor-Gemini` | Custom groups created |
| 22 | `22-DC01-ADUC-New-User-Cole-Eden-ceden-ad.apase1.com` | New user created |
| 23 | `23-DC01-ADUC-Cole-Eden-Added-to-Claude-AI-Group` | User added to group |
| 24 | `24-DC01-ADUC-Computers-WS01-Domain-Joined` | WS01 appears in Computers OU |
| 25 | `25-WS01-Task-Manager-ceden-Domain-User-Session-Active` | Domain user logged into WS01 |

---

## Skills Demonstrated

- Network design and IP address planning (RFC 1918, VRRP, segmentation)
- Windows Server 2019 administration
- Active Directory Domain Services (AD DS) — forest and domain creation
- DNS — forward and reverse lookup zones, A and PTR records, nslookup verification
- DHCP — scope creation, lease validation, cross-platform client support
- Active Directory Users and Computers (ADUC) — OUs, users, security groups
- Domain join — workstation enrollment and domain user login
- Cross-platform networking — Windows and Linux (Ubuntu) in the same domain environment
- Network troubleshooting — ipconfig, ping, ARP, nslookup
