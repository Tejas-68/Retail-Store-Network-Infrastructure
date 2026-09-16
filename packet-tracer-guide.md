# Cisco Packet Tracer Lab Deployment & Recreation Guide
## Retail Store Network Infrastructure Simulation

This guide provides step-by-step instructions for reproducing or inspecting this enterprise network topology inside **Cisco Packet Tracer** (version 8.0 or newer).

---

## 1. Hardware & Device Inventory

To build this lab in Cisco Packet Tracer, place the following devices onto the workspace canvas:

| Device Role | Packet Tracer Device Model | Display Name | Physical Interfaces Used |
|---|---|---|---|
| **Edge Router** | Cisco 2911 (or Cisco 4321 / 2901) | `R1-EDGE` | `Gi0/0` (Trunk), `Gi0/1` (WAN) |
| **ISP Simulator** | Cisco 2911 ISR | `ISP-SIM` | `Gi0/0` (203.0.113.1/30), Server loopback |
| **Core Switch** | Cisco Catalyst 3560-24PS (or 2960) | `SW-CORE` | `Gi0/1`, `Gi0/2`, `Gi0/3`, `Gi0/4` |
| **POS Access Switch** | Cisco Catalyst 2960-24TT | `SW-POS` | `Gi0/1` (Uplink), `Fa0/1 - Fa0/24` |
| **Staff Access Switch** | Cisco Catalyst 2960-24TT | `SW-STAFF` | `Gi0/1` (Uplink), `Fa0/1 - Fa0/24` |
| **Server Access Switch** | Cisco Catalyst 2960-24TT (or 3560) | `SW-SERVER` | `Gi0/1` (Uplink), `Fa0/1 - Fa0/10` |
| **POS Terminals** | PC-PT (x 4 or more) | `POS-01`, `POS-02`, `POS-20` | FastEthernet0 |
| **Staff Workstations** | PC-PT / Laptop (x 3) | `STAFF-PC-01`, `STAFF-PC-02` | FastEthernet0 |
| **Retail Servers** | Server-PT (x 3) | `SRV-AD-DNS`, `SRV-DHCP`, `SRV-RETAIL-DB` | FastEthernet0 |
| **Network Printers** | Printer-PT (x 2) | `PRN-RECEIPT-01`, `PRN-LABEL-01` | FastEthernet0 |

---

## 2. Exact Cabling Interconnect Matrix

Use standard **Copper Straight-Through** cables (or automatic type) to interconnect ports as follows:

```
+---------------+-----------------------+---------------+-----------------------+---------------+
| Source Device | Source Port           | Target Device | Target Port           | Cable Type    |
+---------------+-----------------------+---------------+-----------------------+---------------+
| R1-EDGE       | GigabitEthernet0/0/0  | SW-CORE       | GigabitEthernet0/1    | Straight-Thru |
| R1-EDGE       | GigabitEthernet0/0/1  | ISP-SIM       | GigabitEthernet0/0/0  | Straight-Thru |
| SW-CORE       | GigabitEthernet0/2    | SW-POS        | GigabitEthernet0/1    | Straight-Thru |
| SW-CORE       | GigabitEthernet0/3    | SW-STAFF      | GigabitEthernet0/1    | Straight-Thru |
| SW-CORE       | GigabitEthernet0/4    | SW-SERVER     | GigabitEthernet0/1    | Straight-Thru |
| SW-POS        | FastEthernet0/1       | POS-01        | FastEthernet0         | Straight-Thru |
| SW-POS        | FastEthernet0/2       | POS-02        | FastEthernet0         | Straight-Thru |
| SW-STAFF      | FastEthernet0/1       | STAFF-PC-01   | FastEthernet0         | Straight-Thru |
| SW-STAFF      | FastEthernet0/2       | STAFF-PC-02   | FastEthernet0         | Straight-Thru |
| SW-STAFF      | FastEthernet0/13      | PRN-RECEIPT-01| FastEthernet0         | Straight-Thru |
| SW-STAFF      | FastEthernet0/14      | PRN-LABEL-01  | FastEthernet0         | Straight-Thru |
| SW-SERVER     | FastEthernet0/1       | SRV-AD-DNS    | FastEthernet0         | Straight-Thru |
| SW-SERVER     | FastEthernet0/2       | SRV-DHCP      | FastEthernet0         | Straight-Thru |
| SW-SERVER     | FastEthernet0/3       | SRV-RETAIL-DB | FastEthernet0         | Straight-Thru |
+---------------+-----------------------+---------------+-----------------------+---------------+
```

---

## 3. Configuration Deployment Order

To avoid STP blocking and routing loops during initialization, paste the Cisco IOS configurations in this order:

1. **Step 1: Core Switch (`SW-CORE`)**
   - Click `SW-CORE` > **CLI** tab.
   - Enter `enable` > `configure terminal`.
   - Copy & paste the contents of `configs/SW-CORE-Distribution.cfg`.

2. **Step 2: Access Switches (`SW-POS`, `SW-STAFF`, `SW-SERVER`)**
   - On each access switch, open CLI and paste the respective configuration file (`SW-POS-Access.cfg`, `SW-STAFF-Access.cfg`, `SW-SERVER-Access.cfg`).

3. **Step 3: Edge Router (`R1-EDGE`)**
   - Open `R1-EDGE` > **CLI** tab.
   - Answer `no` if prompted for initial configuration dialog.
   - Enter `enable` > `configure terminal`.
   - Copy & paste the contents of `configs/R1-EDGE-Router.cfg`.

4. **Step 4: Configure ISP Simulator (Optional WAN Internet)**
   - Configure simulated ISP on `ISP-SIM`:
   ```bash
   Router(config)# hostname ISP-SIM
   ISP-SIM(config)# interface GigabitEthernet0/0/0
   ISP-SIM(config-if)# ip address 203.0.113.1 255.255.255.252
   ISP-SIM(config-if)# no shutdown
   ISP-SIM(config)# interface Loopback0
   ISP-SIM(config-if)# ip address 8.8.8.8 255.255.255.255
   ISP-SIM(config-if)# no shutdown
   ```

5. **Step 5: End-Device IP Setup**
   - Open each POS Terminal and Staff PC > **Desktop** > **IP Configuration** > Click **DHCP**.
   - Verify each device obtains an IP in its designated subnet (`192.168.10.x` for POS, `192.168.20.x` for Staff).
   - For Servers and Printers, assign the static IPs listed in `ip-addressing.xlsx`.

---

## 4. Verification Test Cases

Execute the following test matrix to validate end-to-end functionality:

| Test # | Origin Device | Destination Device / Target | Expected Behavior | Justification |
|---|---|---|---|---|
| **TEST 01** | `POS-01` (`192.168.10.11`) | `192.168.10.1` (Gateway) | **PASS** (Reply < 1ms) | POS subinterface routing valid |
| **TEST 02** | `POS-01` (`192.168.10.11`) | `192.168.30.12` (Database) | **PASS** (Permitted TCP/ICMP) | POS must query retail ERP database |
| **TEST 03** | `POS-01` (`192.168.10.11`) | `192.168.20.51` (Staff PC) | **FAIL (BLOCKED)** | PCI-DSS isolation: POS cannot talk to Staff |
| **TEST 04** | `STAFF-PC-01` (`192.168.20.51`)| `192.168.10.11` (POS) | **FAIL (BLOCKED)** | ACL 102 drops Staff inbound to POS |
| **TEST 05** | `STAFF-PC-01` (`192.168.20.51`)| `192.168.40.11` (Receipt Printer)| **PASS** | Staff office printing permitted |
| **TEST 06** | `PRN-RECEIPT-01` | `192.168.10.11` (POS) | **FAIL (BLOCKED)** | ACL 103 prevents printer from initiating inbound to POS |
| **TEST 07** | `STAFF-PC-01` (`192.168.20.51`)| `8.8.8.8` (Public Internet) | **PASS** (NAT Overload) | NAT overload translates through `203.0.113.2` |
