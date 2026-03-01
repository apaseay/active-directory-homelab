# Network Architecture Design – Active Directory Lab Environment

> This document covers the full network design, IP addressing strategy, DHCP configuration decisions, and planned high availability extensions for the ApaseForge Active Directory homelab.

---

## Summary

This lab demonstrates the integration of DNS, DHCP, and Active Directory Domain Services in a structured enterprise-style environment. A Windows 11 workstation was joined to the domain, and basic user lifecycle management was performed — including user creation, disabling, and re-enabling an account.

---

## Virtual Machines

### 1. DC01 — Domain Controller

| Property | Detail |
|---|---|
| OS | Windows Server 2019 |
| IP | `192.168.228.21` (static) |
| Role | Primary Domain Controller |

**Responsibilities:**
- Active Directory Domain Services (AD DS)
- DNS Server — authoritative for `ad.apase1.com`
- DHCP Server — managing scope `192.168.228.31 – 192.168.228.254`
- Forest root: `ad.apase1.com`

This is the core of the environment. All domain authentication, name resolution, and IP assignment flows through DC01.

---

### 2. WS01 — Domain-Joined Workstation

| Property | Detail |
|---|---|
| OS | Windows 11 |
| IP | `192.168.228.131` (DHCP) |
| Role | Domain-joined client workstation |

**Responsibilities:**
- Joined to domain `ad.apase1.com`
- Logged in as domain user Cole Eden (`ceden@ad.apase1.com`)
- DNS client pointing to DC01 (`192.168.228.21`)
- Validates domain authentication and DHCP lease delivery

This machine proves that a standard Windows workstation can be enrolled into the domain and authenticate with a domain user account end-to-end.

---

### 3. UB01 — Linux Client (Ubuntu Desktop)

| Property | Detail |
|---|---|
| OS | Ubuntu Desktop |
| IP | `192.168.228.132` (DHCP) |
| Role | Cross-platform DNS and connectivity testing |

**Responsibilities:**
- Internal DNS resolution testing against DC01
- ICMP ping validation for Layer 3 connectivity
- Resolved `router.ad.apase1.com` → `192.168.228.2`
- Resolved `dc01.ad.apase1.com` → `192.168.228.21`

This proves that DNS and network connectivity are not Windows-only — a Linux machine operating in the same subnet can resolve internal AD hostnames correctly, demonstrating a properly integrated network environment.

---

## Addressing Standard

This lab environment follows RFC 1918 private addressing.

| Property | Value |
|---|---|
| Network ID | `192.168.228.0/24` |
| Subnet Mask | `255.255.255.0` |
| Usable Hosts | 254 |
| Gateway | `192.168.228.2` (VMware NAT Router) |

The network operates behind a VMware NAT gateway, simulating an internal enterprise network with outbound internet access through a firewall.

---

## Logical Address Allocation Strategy

To reflect enterprise design principles, the subnet is logically segmented by role rather than using a flat address pool.

### 1. Gateway and Core Routing

| Address | Purpose |
|---|---|
| `192.168.228.2` | Default Gateway (VMware NAT Router) |
| `192.168.228.1 – .5` | Reserved for router redundancy / future VRRP high availability |

---

### 2. Network Infrastructure Devices

Reserved for switches, firewalls, monitoring systems, and core infrastructure services.

**Range:** `192.168.228.6 – 192.168.228.10`

---

### 3. LAN Services Segment

Reserved for internal support services such as monitoring systems, management servers, NAS, backup services, and internal tooling.

**Range:** `192.168.228.11 – 192.168.228.20`

---

### 4. Server Allocation Segment

Dedicated static allocation for critical infrastructure servers.

**Range:** `192.168.228.21 – 192.168.228.30`

| Host | IP | Role |
|---|---|---|
| DC01 | `192.168.228.21` | Primary Domain Controller, DNS, DHCP |
| DC02 | `192.168.228.22` | Secondary Domain Controller – planned redundancy |

---

### 5. Client Dynamic Allocation (DHCP Scope)

Remaining addresses are reserved for dynamic client assignment via DHCP.

**DHCP Scope Range:** `192.168.228.31 – 192.168.228.254`
**Total dynamic pool:** 224 addresses

Clients, workstations, and non-critical systems obtain IP configuration automatically from DC01.

---

## Full Address Map

| Range | Segment |
|---|---|
| `.1 – .5` | Gateway / VRRP redundancy |
| `.6 – .10` | Network infrastructure devices |
| `.11 – .20` | LAN services (NAS, monitoring, backups) |
| `.21 – .30` | Servers (static) |
| `.31 – .254` | DHCP client pool (dynamic) |

---

## Design Rationale

The addressing plan reflects structured enterprise network segmentation principles:

- Static allocation for all infrastructure and servers
- Reserved space for redundancy (VRRP / HA)
- Dedicated DHCP pool for client devices
- Clear separation between critical and non-critical systems

The architecture simulates a production internal Active Directory environment operating behind a NAT firewall with outbound internet access.

---

## DHCP Scope Design Decision

The DHCP scope was intentionally defined as `192.168.228.31 – 192.168.228.254` rather than creating a full `/24` scope and manually excluding infrastructure addresses.

This approach:

- Prevents accidental assignment of infrastructure IPs to clients
- Reduces administrative overhead — no exclusion entries needed
- Reflects structured enterprise subnet planning from the start
- Makes the design intent self-documenting

The infrastructure range (`.1–.30`) is inherently protected by scope boundary rather than exclusion rules.

---

## DHCP Configuration Validation

After configuring DHCP on DC01, Windows 11 (WS01) was set to obtain its IP address automatically. The following sequence occurred:

1. WS01 was configured to obtain an IP address automatically
2. It broadcast a **DHCP Discover** message on the network
3. DC01 (running the DHCP Server role) responded with a **DHCP Offer**
4. An IP from the configured scope (`192.168.228.31`) was offered
5. WS01 accepted — a **DHCP Lease** was assigned

Along with the IP address, DC01 delivered the following DHCP options:

| Option | Value |
|---|---|
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `192.168.228.2` |
| DNS Server | `192.168.228.21` |
| DNS Suffix | `apaseforge.local` |

**Result on WS01 (ipconfig output):**

```
IPv4 Address    → 192.168.228.31
Default Gateway → 192.168.228.2
DNS Suffix      → apaseforge.local
```

No manual IP configuration was performed on WS01. All networking parameters were delivered automatically from DC01 via DHCP, confirming successful scope configuration and DHCP option propagation.

---

## DHCP Active Lease Validation

The DHCP server on DC01 is actively managing IP allocation across the network.

| Client | Hostname | IP Assigned |
|---|---|---|
| Windows 11 Workstation | `WS01.apaseforge.local` | `192.168.228.31` |
| Ubuntu Desktop | `UB01.apaseforge.local` | `192.168.228.132` |

This confirms:

- DHCP scope is operational
- Clients are dynamically receiving IP addresses
- DNS dynamic updates are functioning
- Hostnames are properly registered in the domain

---

## Current Lab State

| Host | IP | Assignment | Role |
|---|---|---|---|
| DC01 | `192.168.228.21` | Static | Domain Controller, DNS, DHCP |
| WS01 | `192.168.228.31` | DHCP | Domain-joined workstation |
| UB01 | `192.168.228.132` | DHCP | Linux client |
| Gateway | `192.168.228.2` | Static | VMware NAT Router |

- **DNS Server:** `192.168.228.21` (DC01)
- **Domain:** `apaseforge.local`

---

## Planned Extensions — DC02 High Availability

The next phase of this lab will introduce **DC02** (`192.168.228.22`) as a secondary Domain Controller to implement redundancy across all critical services.

### DHCP Failover

DHCP failover will be configured between DC01 and DC02 using Windows Server's built-in DHCP Failover feature. This ensures clients continue to receive IP addresses if DC01 becomes unavailable.

**Planned mode:** Hot Standby — DC01 as active, DC02 as standby
**Scope to replicate:** `192.168.228.31 – 192.168.228.254`

Steps:
1. Install DHCP Server role on DC02
2. On DC01, right-click the scope → Configure Failover
3. Add DC02 as the partner server
4. Choose Hot Standby or Load Balance mode
5. Verify lease replication from DC01 to DC02

### DNS Failover

DNS redundancy will be achieved by promoting DC02 as an additional Domain Controller, which automatically replicates the DNS zones from DC01 via Active Directory-integrated zone replication.

Steps:
1. Promote DC02 to Domain Controller (join existing domain `apaseforge.local`)
2. AD-integrated DNS zones replicate automatically
3. Configure clients to use both `192.168.228.21` (primary) and `192.168.228.22` (secondary) as DNS servers
4. Verify resolution from clients using both servers independently

### Goal

Once complete, the lab will demonstrate a fully redundant Active Directory environment — matching the resilience expectations of a production enterprise network.
