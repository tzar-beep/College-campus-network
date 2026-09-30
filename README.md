# Design and Implementation of Secure Wireless Network for a University Campus
**Course**: Computer Networks (University Project & Viva Guide)  
**File Saved As**: `C:/Users/vaibhav/Cisco Packet Tracer 9.0.1/saves/Campus_Secure_Wireless_Network.pkt`  
**Canvas Diagram**: `C:/Users/vaibhav/.packetsmith/MCP-Packet-Tracer/projects/campus_secure_network.png`  

---

## 1. Executive Summary & Architecture Overview

This project implements a hierarchically structured, secure campus network engineered specifically around the core Computer Networks academic syllabus. Rather than building an over-engineered network, this design adheres strictly to academic clarity, fundamental correctness, and straightforward explainability during viva examinations.

### High-Level Topology Architecture
```
                         +-----------------------+
                         |   ISP-Router (WAN)    |
                         |  203.0.113.1 / 8.8.8.8|
                         +-----------+-----------+
                                     | Serial0/0/0 (PPP, Point-to-Point)
                                     | 203.0.113.0/30 (OSPF Area 0 + Default Route)
                         +-----------+-----------+
                         |   Edge-Router (GW)    |
                         |   NAT/PAT + Inter-VLAN|
                         +-----------+-----------+
                                     | GigabitEthernet0/0 (802.1Q Trunk)
                         +-----------+-----------+
                         |       Core-SW         |------+ FastEthernet0/24 (VLAN 50)
                         |   Catalyst 2960-24TT  |      | [ Univ-Server: 192.168.10.250 ]
                         +---+---------------+---+      | (DNS: 53, HTTP: 80, FTP: 21)
                             |               |
             +---------------+               +---------------+
             | Gi0/2 (Trunk)                 | Fa0/1 (Trunk) | Fa0/2 (Trunk)
     +-------+-------+               +-------+-------+       +-------+-------+
     |  Student-SW   |               |  Faculty-SW   |       |   Admin-SW    |
     | (VLAN 10, 40) |               |   (VLAN 20)   |       |   (VLAN 30)   |
     +---+---+---+---+               +---+-------+---+       +---+-------+---+
         |   |   |   |                   |       |               |       |
      Fa0/1 Fa0/2|  Fa0/4             Fa0/1   Fa0/2           Fa0/1   Fa0/2
     PC1  PC2  Fa0/3 Guest-AP         PC      AP             PC1     PC2
               Student-AP  |                   |             (Port-Security)
                   |       |                   |
            Student-Laptop |            Faculty-Laptop
                     Guest-Laptop
```

---

## 2. Hardware Inventory (18 Devices)

| Device Name | PT Model | Layer / Role | Primary Function |
| :--- | :--- | :--- | :--- |
| **ISP-Router** | Cisco 2911 | WAN / Core | External ISP gateway running OSPF Area 0 and hosting public service `8.8.8.8`. |
| **Edge-Router** | Cisco 2911 | Campus Edge | Router-on-a-stick inter-VLAN routing, PAT overload, DHCP pools, and department ACLs. |
| **Core-SW** | Cisco 2960-24TT | Distribution / Core | Central campus 802.1Q trunk aggregator and server access switch. |
| **Student-SW** | Cisco 2960-24TT | Access Layer | Access layer switch providing connectivity to student workstations and student/guest APs. |
| **Faculty-SW** | Cisco 2960-24TT | Access Layer | Access layer switch isolating faculty desktop and faculty wireless traffic. |
| **Admin-SW** | Cisco 2960-24TT | Access Layer | Hardened access switch with sticky port-security for administrative staff. |
| **Univ-Server** | Server-PT | Application Layer | Central university multi-service server providing DNS, HTTP, and FTP. |
| **Student-AP** | AccessPoint-PT | Wireless Access | Student wireless access point mapped to VLAN 10. |
| **Faculty-AP** | AccessPoint-PT | Wireless Access | Faculty wireless access point mapped to VLAN 20. |
| **Guest-AP** | AccessPoint-PT | Wireless Access | Isolated guest wireless access point mapped to VLAN 40. |
| **Student-PC1** | PC-PT | End Host | Student lab desktop computer in VLAN 10. |
| **Student-PC2** | PC-PT | End Host | Student lab desktop computer in VLAN 10. |
| **Student-Laptop**| Laptop-PT | End Host (WiFi)| Wireless student laptop associated with Student-AP in VLAN 10. |
| **Faculty-PC** | PC-PT | End Host | Faculty office workstation in VLAN 20. |
| **Faculty-Laptop**| Laptop-PT | End Host (WiFi)| Wireless faculty laptop associated with Faculty-AP in VLAN 20. |
| **Admin-PC1** | PC-PT | End Host | Administrative workstation protected by port-security in VLAN 30. |
| **Admin-PC2** | PC-PT | End Host | Secondary administrative workstation in VLAN 30. |
| **Guest-Laptop** | Laptop-PT | End Host (WiFi)| Visitor laptop in VLAN 40 restricted by ACL. |

---

## 3. Logical Segmentation: VLAN Design Table

| VLAN ID | Name | Purpose | Gateway (Edge-Router Sub-interface) | Switch Access Ports |
| :---: | :--- | :--- | :--- | :--- |
| **10** | `STUDENTS` | Student labs and student WiFi | `192.168.10.1` (`Gi0/0.10`) | `Student-SW` Fa0/1, Fa0/2, Fa0/3 |
| **20** | `FACULTY` | Professors and research staff | `192.168.10.129` (`Gi0/0.20`) | `Faculty-SW` Fa0/1, Fa0/2 |
| **30** | `ADMIN` | University finance & administration | `192.168.10.193` (`Gi0/0.30`) | `Admin-SW` Fa0/1, Fa0/2 |
| **40** | `GUEST` | Visitors & temporary wireless users | `192.168.10.225` (`Gi0/0.40`) | `Student-SW` Fa0/4 |
| **50** | `SERVER` | University Data Center / Services | `192.168.10.241` (`Gi0/0.50`) | `Core-SW` Fa0/24 |

---

## 4. IPv4 Variable Length Subnet Masking (VLSM) Table

**Base Network**: `192.168.10.0/24` (256 addresses total, partitioned efficiently without waste)

| Subnet Purpose | Network Address | Prefix / Mask | Usable Host Range | Gateway | Broadcast |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Students (VLAN 10)** | `192.168.10.0` | `/25` (`255.255.255.128`) | `192.168.10.1` – `192.168.10.126` | `192.168.10.1` | `192.168.10.127` |
| **Faculty (VLAN 20)** | `192.168.10.128`| `/26` (`255.255.255.192`) | `192.168.10.129` – `192.168.10.190` | `192.168.10.129` | `192.168.10.191` |
| **Admin (VLAN 30)** | `192.168.10.192`| `/27` (`255.255.255.224`) | `192.168.10.193` – `192.168.10.222` | `192.168.10.193` | `192.168.10.223` |
| **Guest (VLAN 40)** | `192.168.10.224`| `/28` (`255.255.255.240`) | `192.168.10.225` – `192.168.10.238` | `192.168.10.225` | `192.168.10.239` |
| **Server (VLAN 50)** | `192.168.10.240`| `/28` (`255.255.255.240`) | `192.168.10.241` – `192.168.10.254` | `192.168.10.241` | `192.168.10.255` |
| **WAN Serial Link** | `203.0.113.0` | `/30` (`255.255.255.252`) | `203.0.113.1` – `203.0.113.2` | `203.0.113.1` | `203.0.113.3` |

### Static Host IP Allocation Summary
- `Student-PC1`: `192.168.10.11/25` (GW: `192.168.10.1`, DNS: `192.168.10.250`)
- `Student-PC2`: `192.168.10.12/25` (GW: `192.168.10.1`, DNS: `192.168.10.250`)
- `Student-Laptop`: `192.168.10.13/25` (GW: `192.168.10.1`, DNS: `192.168.10.250`)
- `Faculty-PC`: `192.168.10.130/26` (GW: `192.168.10.129`, DNS: `192.168.10.250`)
- `Faculty-Laptop`: `192.168.10.131/26` (GW: `192.168.10.129`, DNS: `192.168.10.250`)
- `Admin-PC1`: `192.168.10.194/27` (GW: `192.168.10.193`, DNS: `192.168.10.250`)
- `Admin-PC2`: `192.168.10.195/27` (GW: `192.168.10.193`, DNS: `192.168.10.250`)
- `Guest-Laptop`: `192.168.10.226/28` (GW: `192.168.10.225`, DNS: `192.168.10.250`)
- `Univ-Server`: `192.168.10.250/28` (GW: `192.168.10.241`, DNS: `192.168.10.250`)

---

## 5. Routing Architecture

### 1. Default Routing (Campus Edge to ISP)
- Configured on `Edge-Router`:
  ```cisco
  ip route 0.0.0.0 0.0.0.0 203.0.113.1
  ```
- **Viva Concept**: A default route matches all destination IP packets that do not have a specific match in the routing table (Gateway of Last Resort). In a single-homed campus, directing unknown traffic upstream to the ISP is optimal because the campus router does not need the entire Internet routing table.

### 2. Single-Area OSPF (Area 0 on WAN Link)
- Configured on `Edge-Router` and `ISP-Router`:
  ```cisco
  ! Edge-Router
  router ospf 1
   router-id 1.1.1.1
   network 203.0.113.0 0.0.0.3 area 0
   default-information originate

  ! ISP-Router
  router ospf 1
   router-id 2.2.2.2
   network 203.0.113.0 0.0.0.3 area 0
   network 8.8.8.8 0.0.0.0 area 0
  ```
- **Live Status Verified**: `State: FULL/-` between `Edge-Router` (`1.1.1.1`) and `ISP-Router` (`2.2.2.2`).
- **Route Installed**: `Edge-Router` learns `8.8.8.8/32 [110/65] via 203.0.113.1` over `Serial0/0/0`.

---

## 6. WAN & Data Link Layer (Unit 4 Alignment)

- **Transmission Media**: Serial V.35 DCE/DTE simulation cable.
- **Layer 2 Encapsulation**: Point-to-Point Protocol (**PPP**).
- **Clock Rate**: `64000` bps on ISP-Router DCE interface.
- **Viva Distinction (PPP vs HDLC)**:
  - Cisco HDLC is proprietary and lacks standardized authentication.
  - PPP is an open standard (RFC 1661) featuring LCP (Link Control Protocol) for link establishment, NCP (Network Control Protocol) for multi-protocol encapsulation, and optional PAP/CHAP authentication.

---

## 7. Network Address Translation (NAT / PAT)

- **Type**: Port Address Translation (NAT Overload).
- **Inside Interfaces**: `GigabitEthernet0/0.10`, `0.20`, `0.30`, `0.40`, `0.50`.
- **Outside Interface**: `Serial0/0/0` (`203.0.113.2`).
- **Configuration**:
  ```cisco
  access-list 1 permit 192.168.10.0 0.0.0.255
  ip nat inside source list 1 interface Serial0/0/0 overload
  ```
- **Live Translation Verification**:
  ```
  Edge-Router#show ip nat translations
  Pro Inside global      Inside local       Outside local      Outside global
  icmp 203.0.113.2:20    192.168.10.11:20   8.8.8.8:20         8.8.8.8:20
  ```
- **Viva Concept**: Private addresses (RFC 1918) are non-routable on the public Internet. PAT dynamically translates multiple internal source IP/Port combinations into a single public routable IP using unique Layer 4 port identifiers.

---

## 8. Application Layer Services & Protocol Matrix (Unit 5 Alignment)

| Service | Protocol | Transport Protocol | Default Port | Academic Role / Demonstration |
| :--- | :---: | :---: | :---: | :--- |
| **DNS** | Domain Name System | **UDP** | **53** | Resolves `univ.edu`, `www.univ.edu`, and `ftp.univ.edu` to `192.168.10.250`. |
| **HTTP** | HyperText Transfer | **TCP** | **80** | Hosts the University Campus Secure Portal web page. |
| **FTP** | File Transfer Protocol | **TCP** | **21** (Control) / 20 (Data)| Demonstrates stateful client-server file exchange using credentials (`cisco`/`cisco`). |
| **Telnet**| Remote Virtual Terminal | **TCP** | **23** | Demonstrates unencrypted CLI remote device management for syllabus evaluation. |
| **SSH** | Secure Shell | **TCP** | **22** | Recommended cryptographic alternative to Telnet in production networks. |

---

## 9. Security Implementations (ACLs, Hardening & Port-Security)

### 1. Extended Access Control Lists (ACLs)
- **Student Restriction (`ACL-STUDENT` applied inbound on `Gi0/0.10`)**:
  ```cisco
  ip access-list extended ACL-STUDENT
   deny ip 192.168.10.0 0.0.0.127 192.168.10.192 0.0.0.31
   permit ip any any
  ```
  *Result*: Student workstations can access the Server, Internet, and Faculty resources, but packets destined for the Admin subnet (`192.168.10.192/27`) are immediately dropped with `Destination host unreachable`.

- **Guest Isolation (`ACL-GUEST` applied inbound on `Gi0/0.40`)**:
  ```cisco
  ip access-list extended ACL-GUEST
   permit udp 192.168.10.224 0.0.0.15 host 192.168.10.250 eq 53
   permit tcp 192.168.10.224 0.0.0.15 host 192.168.10.250 eq 80
   deny ip 192.168.10.224 0.0.0.15 192.168.10.0 0.0.0.255
   permit ip 192.168.10.224 0.0.0.15 any
  ```
  *Result*: Visitors can only query DNS and browse HTTP on the server, plus access the external WAN/Internet. Any internal lateral movement is denied.

### 2. Layer 2 Port Security
- Configured on `Admin-SW` interface `FastEthernet0/1`:
  ```cisco
  interface FastEthernet0/1
   switchport mode access
   switchport access vlan 30
   switchport port-security
   switchport port-security maximum 1
   switchport port-security violation shutdown
   switchport port-security mac-address sticky
  ```
  *Live State*: Learned sticky MAC `0007.EC37.5BDE` for `Admin-PC1`. If an attacker replaces `Admin-PC1` with an unauthorized rogue device, the port immediately transitions to the `err-disabled` (shutdown) state.

### 3. Device Hardening Baseline
- **Encrypted Privileged Passwords**: `enable secret cisco123` across all routers and switches.
- **Service Password Encryption**: `service password-encryption` to obfuscate plaintext passwords in running configs.
- **Legal Warning Banners**: `banner motd #...#` on console and terminal sessions.
- **Local User Authentication**: `username admin privilege 15 secret admin123`.

---

## 10. Actual Verification & Test Results Matrix

All tests were executed through the Packet Tracer engine:

| Test Case | Source Device | Target / Resource | Expected Outcome | Actual Result | Status |
| :---: | :--- | :--- | :--- | :--- | :---: |
| **1** | Student-PC1 | Gateway (`192.168.10.1`) | Ping 4/4 replies (0% loss) | `Packets: Sent=4, Received=4, Lost=0 (0% loss)` | **PASS** |
| **2** | Student-PC1 | Univ-Server (`192.168.10.250`)| Inter-VLAN Ping (0% loss) | `Packets: Sent=4, Received=4, Lost=0 (0% loss)` | **PASS** |
| **3** | Student-PC1 | Admin-PC1 (`192.168.10.194`) | **BLOCKED** by `ACL-STUDENT` | `Reply from 192.168.10.1: Destination host unreachable` | **PASS** |
| **4** | Admin-PC1 | Univ-Server (`192.168.10.250`)| Full Reachability (0% loss) | `Packets: Sent=4, Received=4, Lost=0 (0% loss)` | **PASS** |
| **5** | Faculty-PC | Univ-Server (`192.168.10.250`)| Full Reachability (0% loss) | `Packets: Sent=4, Received=4, Lost=0 (0% loss)` | **PASS** |
| **6** | Guest-Laptop | Faculty-PC (`192.168.10.130`) | **BLOCKED** by `ACL-GUEST` | `Packets: Sent=4, Received=0, Lost=4 (100% loss)` | **PASS** |
| **7** | Student-PC1 | DNS Query (`nslookup univ.edu`)| Resolves to `192.168.10.250` | `Name: univ.edu, Address: 192.168.10.250` | **PASS** |
| **8** | Student-PC1 | FTP Session (`ftp 192.168.10.250`)| User login & directory listing| `230 User logged in. 226 Transfer complete.` | **PASS** |
| **9** | Student-PC1 | Telnet Session (`192.168.10.1`)| Login prompt on TCP 23 | Authenticated, executed `show ip int br`, exited | **PASS** |
| **10**| Edge-Router | ISP-Router Serial (`203.0.113.1`)| Point-to-Point WAN ping | `Success rate is 100 percent (5/5)` | **PASS** |
| **11**| Edge-Router | ISP Loopback (`8.8.8.8`) | OSPF Route Ping (0% loss) | `Success rate is 100 percent (5/5)` | **PASS** |
| **12**| Student-PC1 | Internet Host (`8.8.8.8`) | **NAT/PAT Outbound Access** | `Packets: Sent=4, Received=4, Lost=0 (0% loss)` | **PASS** |
| **13**| Edge-Router | NAT Translation Table | Dynamic PAT Entry created | `icmp 203.0.113.2:20 -> 8.8.8.8:20` verified | **PASS** |
| **14**| Edge-Router | OSPF Neighbor Adjacency | Formed neighbor with ISP | `Neighbor ID 203.0.113.1: State FULL/-` | **PASS** |
| **15**| Admin-SW | Port-Security on Fa0/1 | Sticky MAC learned | `Status: Secure-up, Sticky MAC: 0007.EC37.5BDE` | **PASS** |
| **16**| Entire Topology| `pt_health_check` | 0 down links, 0 IP conflicts | `healthy: true, summary: Topología saludable` | **PASS** |

---

## 11. Viva Questions & Model Answers

### Q1: Why did you use Router-on-a-Stick instead of an L3 switch for Inter-VLAN routing?
**Answer**: Router-on-a-Stick using IEEE 802.1Q encapsulation clearly demonstrates fundamental Layer 2 and Layer 3 interactions: trunking, VLAN tags, and sub-interfaces. It allows applying security policies (Extended ACLs) and NAT centrally at the edge router while keeping the core switch simple.

### Q2: What is the purpose of VLSM in this project?
**Answer**: Traditional classful subnetting wastes IP addresses by giving every subnet the same size. In our project, Students require a large subnet (`/25` with 126 usable hosts), Faculty needs medium capacity (`/26` with 62 hosts), Admin needs fewer (`/27` with 30 hosts), and Guests/Servers need only 14 hosts (`/28`). Using VLSM, the entire campus fits neatly inside a single `/24` private block without wasting address space.

### Q3: How does NAT Overload (PAT) differentiate between multiple student PCs accessing the same public IP?
**Answer**: PAT appends unique source Layer 4 port numbers (TCP/UDP) to the public IP `203.0.113.2`. When return packets arrive at the router, the router inspects the destination port number, consults its NAT translation table, and rewrites the packet back to the specific internal private IP and original port number.

### Q4: Why is Telnet considered insecure compared to SSH?
**Answer**: Telnet transmits all data—including usernames, passwords, and commands—in plaintext over TCP port 23. Anyone running a packet sniffer (like Wireshark) on an intermediate link can intercept administrative credentials. SSH encrypts all communications using asymmetric key cryptography (RSA) and symmetric session ciphers.

### Q5: What is the role of 802.1Q tagging on trunk links?
**Answer**: Standard Ethernet frames do not have a VLAN tag. When a frame crosses a trunk link between switches or between a switch and router, the switch inserts a 4-byte 802.1Q header containing a 12-bit VLAN Identifier (VID). The receiving switch or router reads this tag to know which VLAN the frame belongs to, and strips it before delivering it to an access port.

---

## 12. Packet Tracer Simulator Notes & Observations

1. **Access Point SSID Configuration**: In Cisco Packet Tracer 9.0.1, the basic `AccessPoint-PT` models broadcast on the default SSID across Port 1. Laptops with wireless NICs automatically associate with the nearest AP on the shared wireless medium.
2. **Serial Interfaces**: The Cisco 2911 router does not include serial ports on its motherboard by default; they are provided by adding the `HWIC-2T` expansion card into slot `0/0`, creating interfaces `Serial0/0/0` and `Serial0/0/1`.
3. **Cabling Convention**: Switch-to-switch uplink interconnections use crossover cables (`cross`), while router-to-switch and PC-to-switch connections use straight-through cables (`straight`).
