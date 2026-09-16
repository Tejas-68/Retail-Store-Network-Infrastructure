# Retail Store Network Infrastructure Simulation
### Enterprise Multi-VLAN Segmentation, Router-on-a-Stick, NAT/PAT, and PCI-DSS Security Controls

[![Network Architecture](https://img.shields.io/badge/Architecture-Hierarchical_2--Tier-blue.svg)](#network-architecture)
[![Cisco IOS](https://img.shields.io/badge/Cisco_IOS-15.4_%2F_15.0-navy.svg)](configs/)
[![Packet Tracer](https://img.shields.io/badge/Simulation-Cisco_Packet_Tracer-005073.svg)](packet-tracer-guide.md)
[![Security Standard](https://img.shields.io/badge/Compliance-PCI--DSS_Aligned-green.svg)](#security-architecture--acls)
[![License](https://img.shields.io/badge/License-MIT-lightgrey.svg)](LICENSE)

---

> [!NOTE]
> **Independent Project Disclaimer**: This repository contains an independent networking lab and architecture simulation designed for technical demonstration and portfolio evaluation. All topologies, IP schemes, VLAN designs, and configurations are simulated and do not represent the proprietary network architecture, production systems, or internal configurations of Lulu Group International or any specific employer.

---

## 1. Project Overview

Modern retail hypermarkets and department stores rely on highly available, strictly segmented network infrastructures. Point of Sale (POS) terminals processing card transactions, back-office operations, inventory databases, and network peripherals must operate seamlessly on a unified physical infrastructure without compromising security or regulatory compliance.

This project delivers a **full-lifecycle network infrastructure simulation** for a modern mid-sized retail store. Built using Cisco IOS switching and routing technologies, the design addresses the dual requirements of **operational resilience** and **rigorous security segmentation** (aligning with PCI-DSS 4.0 standards for Cardholder Data Environments).

### Core Technical Competencies Demonstrated
- **Hierarchical Network Design**: 2-Tier distribution and access switching architecture with 802.1Q trunking.
- **VLAN Segmentation & Micro-Perimeters**: Complete traffic isolation between POS lanes, administrative staff, servers, and network printers.
- **Inter-VLAN Routing**: Router-on-a-Stick (RoaS) subinterfaces with 802.1Q encapsulation on Cisco ISR.
- **Dynamic NAT / PAT (Port Address Translation)**: Single-IP public overload allowing internal segments internet connectivity.
- **Centralized DHCP Services**: Custom DHCP pools with lease optimization, excluded static ranges, and DNS parameters.
- **Enterprise Access Control Lists (ACLs)**: Stateful perimeter policies blocking untrusted cross-VLAN traffic and restricting printer outbound access.
- **Layer 2 Hardening**: Port Security with sticky MAC addressing, Rapid-PVST+ Root Bridge tuning, Spanning-Tree PortFast, BPDU Guard, and unrouted Blackhole Native VLAN (VLAN 666) to neutralize VLAN-hopping exploits.

---

## 2. Network Topology

The architecture follows a modular hierarchical design. Core distribution switch `SW-CORE` aggregates access switches across the retail floor and connects upstream to Edge Router `R1-EDGE` via an 802.1Q trunk.

![Retail Store Network Topology](network-topology.png)

### High-Level Topology Flow
```
                           +------------------------+
                           |     Internet Cloud     |
                           +------------------------+
                                       |
                           (203.0.113.0/30 WAN Uplink)
                                       |
                           +------------------------+
                           |  R1-EDGE Gateway Router|  <-- DHCP Server, NAT/PAT,
                           |    (Cisco 2911 ISR)    |      Inter-VLAN & Security ACLs
                           +------------------------+
                                       | (802.1Q Trunk: VLAN 10,20,30,40,99)
                           +------------------------+
                           |        SW-CORE         |  <-- Rapid-PVST+ Primary Root
                           | (Catalyst 3560 Switch) |      Native VLAN 666
                           +------------------------+
                            /          |          \
           +---------------+           |           +---------------+
           |                           |                           |
  (Trunk: VLAN 10,99)         (Trunk: VLAN 20,40,99)       (Trunk: VLAN 30,99)
           |                           |                           |
+--------------------+      +--------------------+      +--------------------+
|       SW-POS       |      |      SW-STAFF      |      |     SW-SERVER      |
|  (Catalyst 2960)   |      |  (Catalyst 2960)   |      |  (Catalyst 3750)   |
+--------------------+      +--------------------+      +--------------------+
          |                           |                           |
+--------------------+      +--------------------+      +--------------------+
|  POS Terminals     |      | Staff Workstations |      | Retail Data Center |
|  Lanes 01 - 40     |      | (Back-Office Admin)|      | (Active Directory, |
|  [VLAN 10: .10.0]  |      | [VLAN 20: .20.0]   |      |  ERP/DB, DNS/NTP)  |
|  PCI-DSS CDE Zone  |      | Corporate Segment  |      | [VLAN 30: .30.0]   |
+--------------------+      +--------------------+      +--------------------+
                                      |
                            +--------------------+
                            | Network Printers   |
                            | (Receipt / Label)  |
                            | [VLAN 40: .40.0]   |
                            +--------------------+
```

---

## 3. IP Addressing & Subnet Allocation Plan

The internal network leverages the RFC 1918 Class C private range `192.168.0.0/16`, subdivided into predictable `/24` subnets per functional department:

| VLAN ID | VLAN Name | Subnet (CIDR) | Subnet Mask | Default Gateway | Usable Host Range | Broadcast IP | Allocation Method | Purpose / Security Zone |
|:---:|:---|:---|:---|:---|:---|:---|:---|:---|
| **10** | `POS-TERMINALS` | `192.168.10.0/24` | `255.255.255.0` | `192.168.10.1` | `192.168.10.2 - .254` | `192.168.10.255` | DHCP Reservation | Cardholder Data Environment (CDE) |
| **20** | `STAFF-WORKSTATIONS`| `192.168.20.0/24` | `255.255.255.0` | `192.168.20.1` | `192.168.20.2 - .254` | `192.168.20.255` | Dynamic DHCP | Back-Office Workstations & Admin |
| **30** | `SERVER-FARM` | `192.168.30.0/24` | `255.255.255.0` | `192.168.30.1` | `192.168.30.2 - .254` | `192.168.30.255` | Static Assignment | Active Directory, DNS, ERP & POS DB |
| **40** | `NETWORK-PRINTERS` | `192.168.40.0/24` | `255.255.255.0` | `192.168.40.1` | `192.168.40.2 - .254` | `192.168.40.255` | Static / DHCP | Thermal Receipt & Barcode Printers |
| **99** | `MANAGEMENT-SVI` | `192.168.99.0/24` | `255.255.255.0` | `192.168.99.1` | `192.168.99.2 - .254` | `192.168.99.255` | Static Only | Out-Of-Band Switch/Router Management |
| **666**| `NATIVE-BLACKHOLE` | *Unrouted* | *N/A* | *N/A* | *N/A* | *N/A* | None | Native VLAN Blackhole (Anti-VLAN Hopping)|
| **WAN**| `ISP-UPLINK` | `203.0.113.0/30` | `255.255.255.252`| `203.0.113.1` | `203.0.113.2` | `203.0.113.3` | Static Public | Internet Gateway Link to ISP |

> 📊 A complete multi-tab spreadsheet with device inventories, switchport allocations, and ACL matrices is available in [`ip-addressing.xlsx`](ip-addressing.xlsx) (and [`ip-addressing.csv`](ip-addressing.csv)).

---

## 4. Security Architecture & ACLs

In retail environments, customer payment card security is governed by **PCI-DSS (Payment Card Industry Data Security Standard)**. A critical requirement is **Requirement 1: Install and Maintain Network Security Controls**, which mandates that the Cardholder Data Environment (CDE) must be strictly isolated from general office and guest traffic.

```
       [ VLAN 10: POS ]  <--- STRICT ISOLATION (No cross-talk) ---> [ VLAN 20: Staff ]
              |                                                               |
     (Permit Port 1433/443 only)                                    (Permit Web/SMB)
              v                                                               v
       [ VLAN 30: Server Farm ] <---------------------------------------------+
```

### Access Control Matrix Summary

| Origin Subnet | Destination | Protocol / Ports | Action | Rationale |
|---|---|---|:---:|---|
| **VLAN 10 (POS)** | VLAN 30 (`192.168.30.12`) | TCP 1433 (MSSQL), TCP 443 (ERP) | **PERMIT** | Transaction processing & live inventory queries |
| **VLAN 10 (POS)** | VLAN 30 (`192.168.30.10`) | UDP 53 (DNS), UDP 123 (NTP) | **PERMIT** | Network timing & domain name resolution |
| **VLAN 10 (POS)** | VLAN 40 (Printers) | TCP 9100 (RAW), TCP 515 (LPR) | **PERMIT** | Sending receipt print jobs to customer printers |
| **VLAN 10 (POS)** | Public Internet | TCP 443 (HTTPS) | **PERMIT** | Encrypted payment authorization with bank gateway |
| **VLAN 10 (POS)** | **VLAN 20 (Staff)** | Any IP | **DENY** | **PCI-DSS Rule**: Prevent POS from accessing untrusted staff |
| **VLAN 20 (Staff)** | **VLAN 10 (POS)** | Any IP | **DENY** | **PCI-DSS Rule**: Staff PCs can never initiate into POS registers |
| **VLAN 20 (Staff)** | VLAN 30 (Servers) | TCP 445 (SMB), 443 (HTTPS) | **PERMIT** | Department file shares and web management portals |
| **VLAN 20 (Staff)** | Public Internet | HTTP, HTTPS, DNS | **PERMIT** | Corporate internet, email, and SaaS access |
| **VLAN 40 (Printers)**| **VLAN 10 (POS)** | Any IP | **DENY** | Printers must never initiate connections into POS systems |
| **VLAN 40 (Printers)**| **Public Internet** | Any IP | **DENY** | Prevent compromised IoT devices from outbound command & control |

---

## 5. Key Configuration Highlights

### A. Router-on-a-Stick 802.1Q Subinterfaces (`R1-EDGE`)
Subinterfaces terminate 802.1Q tags on a single physical link, enabling cost-effective, centralized routing between VLANs:
```cisco
interface GigabitEthernet0/0/0.10
 description Gateway for VLAN 10 [POS Terminals]
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 ip access-group ACL_POS_IN in
 ip nat inside
```

### B. Dynamic NAT Overload (PAT)
Translates all internal subnets to the single public IP assigned to the WAN interface `GigabitEthernet0/0/1`:
```cisco
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 permit 192.168.20.0 0.0.0.255
access-list 1 permit 192.168.30.0 0.0.0.255

ip nat inside source list 1 interface GigabitEthernet0/0/1 overload
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

### C. Layer 2 Switchport Hardening (`SW-POS` & `SW-STAFF`)
Prevents rogue machine attachment, MAC flooding attacks, and unauthorized network taps:
```cisco
interface range FastEthernet0/1 - 24
 description POS Register Connection - VLAN 10
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### D. Native VLAN Hopping Mitigation
By reassigning the untagged native VLAN to an unused blackhole ID (`VLAN 666`) and stripping VLAN 1 from all trunk links, 802.1Q double-tagging attacks are effectively mitigated:
```cisco
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 666
 switchport trunk allowed vlan 10,20,30,40,99
 switchport nonegotiate
```

---

## 6. Verification & Operational Evidence

### 1. Interface & Subinterface Status (`show ip interface brief`)
```
Interface                  IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0/0       unassigned      YES manual up                    up      
GigabitEthernet0/0/0.10    192.168.10.1    YES manual up                    up      
GigabitEthernet0/0/0.20    192.168.20.1    YES manual up                    up      
GigabitEthernet0/0/0.30    192.168.30.1    YES manual up                    up      
GigabitEthernet0/0/0.40    192.168.40.1    YES manual up                    up      
GigabitEthernet0/0/0.99    192.168.99.1    YES manual up                    up      
GigabitEthernet0/0/1       203.0.113.2     YES manual up                    up      
```

### 2. Active DHCP Lease Verification (`show ip dhcp binding`)
```
IP address       Client-ID/Hardware address   Lease expiration        Type
192.168.10.11    0050.7966.6801               Sep 17 2026 11:30 AM    Automatic
192.168.10.12    0050.7966.6802               Sep 17 2026 11:30 AM    Automatic
192.168.20.51    000c.29d1.34a8               Sep 18 2026 11:30 AM    Automatic
```

### 3. Active NAT / PAT Session Table (`show ip nat translations`)
```
Pro Inside global          Inside local           Outside local          Outside global
tcp 203.0.113.2:49152      192.168.10.11:49152    198.51.100.45:443      198.51.100.45:443
tcp 203.0.113.2:49153      192.168.20.51:50123    142.250.190.46:443     142.250.190.46:443
udp 203.0.113.2:5353       192.168.20.51:5353     8.8.8.8:53             8.8.8.8:53
```

### 4. PCI-DSS Isolation Test (POS to Staff blocked)
```cmd
C:\> ping 192.168.20.51
Pinging 192.168.20.51 with 32 bytes of data:
Destination host unreachable.
Destination host unreachable.

R1-EDGE# show access-lists ACL_POS_IN
    90 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 log (340 matches)
```

---

## 7. Repository Structure

```
Retail-Store-Network-Infrastructure/
├── README.md                      # Primary project documentation & architecture guide
├── network-topology.png           # High-resolution visual network topology diagram
├── ip-addressing.xlsx             # Formatted 4-sheet IP addressing & port allocation workbook
├── ip-addressing.csv              # Plaintext CSV export of the subnet scheme
├── router-configuration.txt       # Annotated Cisco IOS running-config for Edge Router
├── switch-configuration.txt       # Annotated Cisco IOS running-configs for all 4 switches
├── vlan-configuration.txt         # Standalone VLAN database & 802.1Q trunking scripts
├── troubleshooting.md             # Real-world retail IT support & network incident scenarios
├── packet-tracer-guide.md         # Step-by-step Cisco Packet Tracer recreation walkthrough
└── configs/                       # Modular Cisco IOS production config files
    ├── R1-EDGE-Router.cfg         # Edge Gateway Router (DHCP, NAT, ACLs, RoaS)
    ├── SW-CORE-Distribution.cfg   # Core/Distribution Switch (Rapid-PVST+ Root, Trunks)
    ├── SW-POS-Access.cfg          # POS Access Switch (Port Security, BPDU Guard)
    ├── SW-STAFF-Access.cfg        # Staff & Printer Access Switch
    └── SW-SERVER-Access.cfg       # Server Farm Access Switch
```

---

## 8. How to Test & Deploy

1. Open **Cisco Packet Tracer** (v8.0+ recommended).
2. Follow the port interconnect cabling table in [`packet-tracer-guide.md`](packet-tracer-guide.md).
3. Paste configuration files from the [`configs/`](configs/) directory in order:
   - `SW-CORE-Distribution.cfg`
   - `SW-POS-Access.cfg`, `SW-STAFF-Access.cfg`, `SW-SERVER-Access.cfg`
   - `R1-EDGE-Router.cfg`
4. Set end-device NICs to DHCP and verify address acquisition.
5. Execute the test matrix documented in Section 4 of [`packet-tracer-guide.md`](packet-tracer-guide.md).

---

## 9. Real-World Troubleshooting Guide

For hands-on IT support and network operations scenarios, refer to [`troubleshooting.md`](troubleshooting.md), which documents realistic troubleshooting tickets including:
- **Scenario 1**: POS Terminal Lane 04 payment gateway authorization failures (ACL & DNS analysis).
- **Scenario 2**: Workstation switchport placed in `err-disabled` state due to port security violations.
- **Scenario 3**: Inter-VLAN routing failure caused by trunk allowed list misconfiguration.
- **Scenario 4**: Network receipt printer communication drops due to physical layer dislodgment.
- **Scenario 5**: Edge NAT overload table exhaustion caused by rogue host background traffic.

---

## 10. Author & Contact

**Tejas N. C.**  
- **GitHub**: [@Tejas-68](https://github.com/Tejas-68)  
- **Focus**: IT Support, Enterprise Networking, Infrastructure Operations & Cybersecurity
