# Enterprise Network Troubleshooting & Diagnostic Field Guide
## Retail Store Network Infrastructure Simulation

---

### 1. Executive Summary & Diagnostic Philosophy

In an enterprise retail environment, network uptime directly governs revenue generation. A 10-minute network outage across Point of Sale (POS) checkout lanes leads to customer abandonments, revenue loss, and inventory discrepancies.

This field guide documents real-world retail IT support troubleshooting workflows using the **OSI 7-Layer Bottom-Up Methodology**. It covers simulated incidents, diagnostic CLI commands, root cause analysis (RCA), and standard operating procedures (SOP) for remediation.

```
       [ Application / Port Issues ]  <-- Layer 7: HTTP/S, MSSQL 1433, RAW 9100
       [ Transport / ACL Drops    ]  <-- Layer 4: TCP SYN drops, Extended ACLs
       [ Network / Subnet Routing ]  <-- Layer 3: Subinterfaces, Gateways, NAT
       [ Data Link / VLAN Issues  ]  <-- Layer 2: 802.1Q Trunks, Port Security, STP
       [ Physical Layer / Cables  ]  <-- Layer 1: Link LEDs, Bad Patch Cords
```

---

### 2. Scenario 1: POS Terminal Lane 04 Cannot Process Payments

#### Incident Ticket
- **Priority**: P1 - High (Revenue Impacting)
- **Reported Issue**: POS Terminal 04 at Checkout Lane 04 displays: *"Error 1004: Payment Gateway Unreachable - Cannot Authorize Card."* Cash transactions work locally, but credit/debit processing fails.

#### Diagnostic Workflow & Command Trace

**Step 1: Check Layer 1 & Physical Interface on Access Switch**
```bash
SW-POS# show interfaces FastEthernet0/4 status
Port      Name               Status       Vlan       Duplex  Speed Type
Fa0/4     POS Lane 04        connected    10         full    100   100BaseTX
```
*Result: Port is physically UP and properly assigned to VLAN 10.*

**Step 2: Verify IP Configuration on POS Terminal**
```cmd
C:\> ipconfig /all
Windows IP Configuration
   IPv4 Address. . . . . . . . . . . : 192.168.10.14
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.10.1
   DNS Servers . . . . . . . . . . . : 192.168.30.10, 8.8.8.8
```
*Result: Valid IP obtained from DHCP pool `POS_POOL`.*

**Step 3: Test Default Gateway & DNS Reachability**
```cmd
C:\> ping 192.168.10.1
Pinging 192.168.10.1 with 32 bytes of data:
Reply from 192.168.10.1: bytes=32 time=1ms TTL=255
Reply from 192.168.10.1: bytes=32 time=1ms TTL=255

C:\> ping 192.168.30.10
Pinging 192.168.30.10 with 32 bytes of data:
Reply from 192.168.30.10: bytes=32 time=2ms TTL=127
```
*Result: Gateway and internal DNS server respond promptly.*

**Step 4: Verify Payment Gateway Resolution & ACL Hits on Edge Router**
```cmd
C:\> nslookup payment.gateway.bank.com
Server:  SRV-AD-DNS.retail.internal
Address: 192.168.30.10

Non-authoritative answer:
Name:    payment.gateway.bank.com
Address: 198.51.100.45
```
*DNS resolves successfully. Let's inspect R1-EDGE access-list matches:*
```bash
R1-EDGE# show access-lists ACL_POS_IN
Extended IP access list ACL_POS_IN
    10 permit tcp 192.168.10.0 0.0.0.255 host 192.168.30.12 eq 1433 (1420 matches)
    20 permit tcp 192.168.10.0 0.0.0.255 host 192.168.30.12 eq 443 (890 matches)
    30 permit udp 192.168.10.0 0.0.0.255 host 192.168.30.10 eq domain (2104 matches)
    40 permit tcp 192.168.10.0 0.0.0.255 host 192.168.30.10 eq domain (12 matches)
    50 permit udp 192.168.10.0 0.0.0.255 host 192.168.30.10 eq 123 (640 matches)
    60 permit udp any eq bootpc any eq bootps (340 matches)
    70 permit tcp 192.168.10.0 0.0.0.255 192.168.40.0 0.0.0.255 eq 9100 (521 matches)
    80 permit tcp 192.168.10.0 0.0.0.255 any eq 443 (0 matches)  <--- EXAMINED: 0 MATCHES
    90 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 (0 matches)
    100 deny ip 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255 (0 matches)
    110 deny ip any any log (84 matches)                          <--- TRAFFIC DROPPED HERE
```
*Inspecting the system logs on R1-EDGE:*
```
%SEC-6-IPACCESSLOGP: list ACL_POS_IN denied tcp 192.168.10.14(49152) -> 198.51.100.45(8443), 1 packet
```

#### Root Cause Analysis (RCA)
The bank payment gateway recently updated its merchant authorization API to listen on non-standard port **TCP 8443** rather than standard 443. The strict perimeter ACL `ACL_POS_IN` only allowed outbound port 443, silently discarding the connection requests at the router interface.

#### Resolution & Verification
Add the updated API port to `ACL_POS_IN`:
```bash
R1-EDGE(config)# ip access-list extended ACL_POS_IN
R1-EDGE(config-ext-nacl)# 85 permit tcp 192.168.10.0 0.0.0.255 any eq 8443
R1-EDGE(config-ext-nacl)# end
R1-EDGE# write memory
```
*Verification: Test card authorization transaction completed on POS 04 with HTTP 200 OK within 350ms.*

---

### 3. Scenario 2: Switchport Err-Disabled (Port Security Violation)

#### Incident Ticket
- **Priority**: P2 - Medium
- **Reported Issue**: Staff workstation in Store Manager Office (Port Fa0/1 on `SW-STAFF`) lost network connectivity. The physical switch port LED turned solid amber.

#### Diagnostic Workflow & Command Trace

**Step 1: Check Interface Status on Switch**
```bash
SW-STAFF# show interfaces FastEthernet0/1 status
Port      Name               Status       Vlan       Duplex  Speed Type
Fa0/1     Store Manager PC   err-disabled 20           auto   auto 100BaseTX
```

**Step 2: Identify Err-Disabled Reason**
```bash
SW-STAFF# show interfaces status err-disabled
Port      Name               Status       Reason               Err-disable Vlans
Fa0/1     Store Manager PC   err-disabled psecure-violation
```

**Step 3: Inspect Port Security Violation Details**
```bash
SW-STAFF# show port-security interface FastEthernet0/1
Port Security              : Enabled
Port Status                : Secure-down
Violation Mode             : Shutdown
Aging Time                 : 0 mins
Aging Type                 : Absolute
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 2
Total MAC Addresses        : 2
Configured MAC Addresses   : 0
Sticky MAC Addresses       : 2
Last Source Address:Vlan   : 0050.56a1.b2c3:20  <--- UNREGISTERED MAC DETECTED
Security Violation Count   : 1
```

#### Root Cause Analysis (RCA)
A visiting district supervisor connected an unmanaged 5-port personal desktop switch to the wall jack in the Store Manager's office to connect both a personal laptop and a test tablet. The switch detected 3 distinct MAC addresses on a port configured with `switchport port-security maximum 2` and `violation shutdown`. The switchport automatically placed itself in `err-disabled` state to protect network integrity.

#### Resolution & Verification
1. Disconnect the unauthorized desktop switch and leave only the authorized workstation connected.
2. Recover the port from `err-disabled` state:
```bash
SW-STAFF(config)# interface FastEthernet0/1
SW-STAFF(config-if)# shutdown
SW-STAFF(config-if)# no shutdown
SW-STAFF(config-if)# end
```
3. Verify normal operational status:
```bash
SW-STAFF# show port-security interface FastEthernet0/1
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Shutdown
```

---

### 4. Scenario 3: Inter-VLAN Routing Failure After Switch Configuration

#### Incident Ticket
- **Priority**: P1 - High
- **Reported Issue**: POS registers are unable to transmit transactions to the retail database server `SRV-RETAIL-DB` (`192.168.30.12`).

#### Diagnostic Workflow & Command Trace

**Step 1: Traceroute from POS Terminal**
```cmd
C:\> tracert 192.168.30.12
Tracing route to SRV-RETAIL-DB.retail.internal [192.168.30.12]
over a maximum of 30 hops:
  1    <1 ms    <1 ms    <1 ms  192.168.10.1 [R1-EDGE]
  2     *        *        *     Request timed out.
  3     *        *        *     Request timed out.
```
*Analysis: The packet reaches R1-EDGE, but fails to reach the destination subnet `192.168.30.0/24`.*

**Step 2: Check Routing Table on R1-EDGE**
```bash
R1-EDGE# show ip route connected
C    192.168.10.0/24 is directly connected, GigabitEthernet0/0/0.10
C    192.168.20.0/24 is directly connected, GigabitEthernet0/0/0.20
C    192.168.30.0/24 is directly connected, GigabitEthernet0/0/0.30
C    192.168.40.0/24 is directly connected, GigabitEthernet0/0/0.40
C    192.168.99.0/24 is directly connected, GigabitEthernet0/0/0.99
```
*Result: Routes are present in the router routing table.*

**Step 3: Check 802.1Q Trunks on SW-CORE**
```bash
SW-CORE# show interfaces trunk
Port        Mode             Encapsulation  Status        Native vlan
Gi0/1       on               802.1q         trunking      666
Gi0/2       on               802.1q         trunking      666
Gi0/3       on               802.1q         trunking      666
Gi0/4       on               802.1q         trunking      666

Port        Vlans allowed on trunk
Gi0/1       10,20,30,40,99
Gi0/2       10,99
Gi0/3       20,40,99
Gi0/4       99              <--- DEFECT FOUND: VLAN 30 IS MISSING FROM ALLOWED LIST
```

#### Root Cause Analysis (RCA)
During prior maintenance, an administrator ran `switchport trunk allowed vlan 99` instead of `switchport trunk allowed vlan add 99` on `Gi0/4` (connecting to `SW-SERVER`). This inadvertently overwritten the allowed list and stripped VLAN 30 from traversing the trunk link to the server farm.

#### Resolution & Verification
Add VLAN 30 back to the allowed list on SW-CORE:
```bash
SW-CORE(config)# interface GigabitEthernet0/4
SW-CORE(config-if)# switchport trunk allowed vlan add 30
SW-CORE(config-if)# end
```
*Verification on POS terminal:*
```cmd
C:\> ping 192.168.30.12
Pinging 192.168.30.12 with 32 bytes of data:
Reply from 192.168.30.12: bytes=32 time=2ms TTL=127
Reply from 192.168.30.12: bytes=32 time=1ms TTL=127
```

---

### 5. Scenario 4: Network Receipt Printers Unreachable Across Checkout Bank

#### Incident Ticket
- **Priority**: P2 - Medium
- **Reported Issue**: Cashiers at Lanes 01–10 report receipts are not printing.

#### Diagnostic Workflow & Command Trace

**Step 1: Ping Printer Gateway from Switch Management SVI**
```bash
SW-CORE# ping 192.168.40.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.40.1, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 1/2/3 ms
```

**Step 2: Ping Receipt Printer 01**
```bash
SW-CORE# ping 192.168.40.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.40.11, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
```

**Step 3: Check MAC Address Table on Access Switch SW-STAFF**
```bash
SW-STAFF# show mac address-table vlan 40
          Mac Address Table
-------------------------------------------
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
  40    0014.2201.2345    DYNAMIC     Fa0/13
```
*Port Fa0/11 (Receipt Printer 01) has no active MAC registered.*

**Step 4: Check Interface FastEthernet0/11 on SW-STAFF**
```bash
SW-STAFF# show interfaces FastEthernet0/11
FastEthernet0/11 is down, line protocol is down (notconnect)
  Hardware is Fast Ethernet, address is 0019.5601.200b
```

#### Root Cause Analysis (RCA)
Inspection of the physical patch panel in IDF 2 revealed that during morning floor cleaning, the patch cable connecting patch panel port 11 to the access switch was partially dislodged.

#### Resolution & Verification
Re-seated RJ-45 patch cable into switchport `Fa0/11`.
```bash
SW-STAFF# show interfaces FastEthernet0/11 status
Port      Name               Status       Vlan       Duplex  Speed Type
Fa0/11    PRN-RECEIPT-01     connected    40         full    100   100BaseTX
```
*Test receipt printed successfully from POS 01.*

---

### 6. Scenario 5: Edge NAT/PAT Translation Saturation

#### Incident Ticket
- **Priority**: P1 - High
- **Reported Issue**: Cloud-based inventory synchronization intermittently drops, and staff report website timeouts during peak afternoon retail hours.

#### Diagnostic Workflow & Command Trace

**Step 1: Check NAT Statistics on R1-EDGE**
```bash
R1-EDGE# show ip nat statistics
Total active translations: 2451 (0 static, 2451 dynamic; 2451 extended)
Outside interfaces:
  GigabitEthernet0/0/1
Inside interfaces:
  GigabitEthernet0/0/0.10, GigabitEthernet0/0/0.20, GigabitEthernet0/0/0.30, GigabitEthernet0/0/0.40
Hits: 1249021  Misses: 1042
CEF Translated packets: 1248000, CEF Punted packets: 1021
Expired translations: 89042
Dynamic mappings:
-- Inside Source
[Id: 1] access-list 1 interface GigabitEthernet0/0/1 ref count 2451
```

**Step 2: Inspect Active NAT Translations for Anomalies**
```bash
R1-EDGE# show ip nat translations | include 192.168.20.75
tcp 203.0.113.2:49152      192.168.20.75:49152      198.51.100.10:80         198.51.100.10:80
tcp 203.0.113.2:49153      192.168.20.75:49153      198.51.100.11:80         198.51.100.11:80
... (1,850 simultaneous sessions from a single host 192.168.20.75) ...
```

#### Root Cause Analysis (RCA)
A staff workstation (`192.168.20.75`) was infected with unauthorized adware/torrent software that opened over 1,800 simultaneous TCP sockets, exhausting the NAT port space allocated on the public IP overload interface and causing timeouts for legitimate transactions.

#### Resolution & Verification
1. Administratively isolate the infected host at the access switch:
```bash
SW-STAFF(config)# interface FastEthernet0/7
SW-STAFF(config-if)# shutdown
SW-STAFF(config-if)# description QUARANTINED - MALWARE INFECTION
```
2. Clear stale NAT translations on R1-EDGE:
```bash
R1-EDGE# clear ip nat translation *
```
3. Verify NAT health:
```bash
R1-EDGE# show ip nat statistics
Total active translations: 120 (Normal operating baseline)
```

---

### 7. Cisco IOS Diagnostic Cheat Sheet

| Command | Objective | Expected Healthy Output |
|---|---|---|
| `show ip interface brief` | Quick check of all interface IP addresses & Layer 1/2 status | All required subinterfaces showing `up / up` |
| `show interfaces trunk` | Check 802.1Q trunking, native VLAN, and allowed VLAN lists | Native VLAN 666, all relevant VLANs allowed and forwarding |
| `show vlan brief` | Verify VLAN existence and access port assignments | All VLANs `active` with correct ports assigned |
| `show ip dhcp binding` | View active DHCP leases assigned to client endpoints | Table of MAC-to-IP leases across pools |
| `show ip dhcp conflict` | Check for duplicate IP detection events | 0 conflicts |
| `show ip nat translations` | View active PAT port translation state table | Active translation mappings with public IP `203.0.113.2` |
| `show access-lists` | View packet hit counters on security filters | Permit/Deny match counters incrementing |
| `show port-security interface <port>` | Inspect MAC address limit and violation counters | `Secure-up`, violation count 0 |
| `show spanning-tree root` | Verify Root Bridge placement | `SW-CORE` is root for VLANs 10, 20, 30, 40, 99 |
