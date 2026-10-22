# Lab 01: Enterprise Core & Layer 2 Fundamentals

## Executive Summary
This lab demonstrates the architectural design and deployment of an enterprise campus network core and access layer. Built on Cisco IOS devices, the topology implements **Variable Length Subnet Masking (VLSM)**, **VLAN segmentation**, **802.1Q trunking**, **Switched Virtual Interface (SVI) Inter-VLAN routing**, and **Spanning Tree Protocol (STP) Root Bridge optimization**. 

By terminating broadcast domains at the Layer 3 Multilayer Core Switch, this architecture enforces security isolation between departments while enabling high-bandwidth inter-subnet communication.

---

## Network Architecture & Topology

### Physical Topology Diagrams
![Topology Diagram](image_731a3d.png)

### Network Layout & Interconnection Matrix
The campus topology consists of a central Multilayer Switch (`Core-SW`) acting as the Layer 3 core routing engine, connected via 802.1Q trunk links to two access switches (`Switch1` and `Switch2`).

| Source Device | Source Port | Destination Device | Destination Port | Link / Port Mode | VLAN Membership |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PC1** | Fa0/0 | **Switch1** | FastEthernet 0/2 | Access Port | VLAN 10 (HR) |
| **PC2** | Fa0/0 | **Switch1** | FastEthernet 0/3 | Access Port | VLAN 99 (Management) |
| **Switch1** | Gig0/1 | **Core-Sw** | GigabitEthernet 0/1 | 802.1Q Trunk | VLANs 1, 10, 20, 99 (Native: 99) |
| **Core-Sw** | Gig0/2 | **Switch2** | GigabitEthernet 0/3 | 802.1Q Trunk | VLANs 1, 10, 20, 99 (Native: 99) |
| **Switch2** | Fa0/2 | **PC3** | FastEthernet 0/0 | Access Port | VLAN 20 (Engineering) |
| **Switch2** | Fa0/3 | **PC4** | FastEthernet 0/0 | Access Port | VLAN 20 (Engineering) |

---

## Addressing & Subnetting Table

The enterprise core utilizes a base block of `192.168.10.0/24`, subnetted using VLSM to optimize address consumption across departments and management interfaces:

| VLAN ID | Subnet Name | Subnet Network | Subnet Mask | Usable Range | Core-SW SVI Gateway | Host IP Address |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- |
| **10** | HR | `192.168.10.0/26` | `255.255.255.192` | `.1` – `.62` | `192.168.10.1` | **PC1:** `192.168.10.6` |
| **20** | Engineering | `192.168.10.64/26` | `255.255.255.192` | `.65` – `.126` | `192.168.10.65` | **PC3:** `192.168.10.66`<br>**PC4:** `192.168.10.67` |
| **99** | Management | `192.168.10.128/28` | `255.255.255.240` | `.129` – `.142` | `192.168.10.129` | **PC2:** `192.168.10.130` |

---

## Verification & Operational Proof

### 1. Inter-VLAN Routing Table (`Core-Sw`)
Executing `show ip route` confirms active SVIs on `Core-Sw`, populating the routing table with directly connected (`C`) subnets:

```text
Core-Sw#show ip route
Codes: C - connected, S - static, I - IGRP, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

     192.168.10.0/24 is variably subnetted, 3 subnets, 2 masks
C       192.168.10.0/26 is directly connected, Vlan10
C       192.168.10.64/26 is directly connected, Vlan20
C       192.168.10.128/28 is directly connected, Vlan99
```

### 2. 802.1Q Trunking & Native VLAN Operational Status
Output of `show interfaces trunk` on `Core-Sw` confirms dot1q encapsulation, active forwarding state, and alignment of Native VLAN 99 across both trunk links (`Gig0/1` and `Gig0/2`):

```text
Core-Sw#show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Gig0/1      on           802.1q         trunking      99
Gig0/2      on           802.1q         trunking      99

Port        Vlans allowed on trunk
Gig0/1      1-1005
Gig0/2      1-1005

Port        Vlans allowed and active in management domain
Gig0/1      1,10,20,99
Gig0/2      1,10,20,99

Port        Vlans in spanning tree forwarding state and not pruned
Gig0/1      1,10,20,99
Gig0/2      1,10,20,99
```

### 3. Spanning Tree Protocol (STP) Primary Root Bridge State
Running `show spanning-tree` verifies that `Core-Sw` is successfully tuned to be the **Root Bridge** for departmental VLANs (e.g., VLAN 10 with Priority `24586`), ensuring deterministic path selection across all access uplinks:

```text
Core-Sw#show spanning-tree
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.9770.920E
             Cost        4
             Port        25 (GigabitEthernet0/1)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     0009.7C12.D39B
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi0/1            Root FWD 4         128.25   P2p
Gi0/2            Desg FWD 4         128.26   P2p

VLAN0010
  Spanning tree enabled protocol ieee
  Root ID    Priority    24586
             Address     0009.7C12.D39B
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24586  (priority 24576 sys-id-ext 10)
             Address     0009.7C12.D39B
```

---

## Key CLI Configuration Files

### 1. Core Multilayer Switch (`Core-Sw`) Configuration
```text
enable
configure terminal
hostname Core-Sw

! Enable Hardware Layer 3 Routing
ip routing

! Create Active VLAN Database
vlan 10
 name HR
vlan 20
 name Engineering
vlan 99
 name Management
exit

! Configure Trunk Uplinks
interface range GigabitEthernet 0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 no shutdown
 exit

! Configure Layer 3 Switched Virtual Interfaces (SVIs)
interface Vlan 10
 ip address 192.168.10.1 255.255.255.192
 no shutdown
 exit

interface Vlan 20
 ip address 192.168.10.65 255.255.255.192
 no shutdown
 exit

interface Vlan 99
 ip address 192.168.10.129 255.255.255.240
 no shutdown
 exit

! STP Optimization: Enforce Core Switch as Primary Root
spanning-tree vlan 10,20,99 root primary
end
write memory
```

### 2. Access Switch (`Switch1`) Configuration
```text
enable
configure terminal
hostname Switch1

! Create VLANs
vlan 10
 name HR
vlan 20
 name Engineering
vlan 99
 name Management
exit

! Access Port Assignments
interface FastEthernet 0/2
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit

interface FastEthernet 0/3
 switchport mode access
 switchport access vlan 99
 no shutdown
 exit

! Trunk Uplink to Core Switch
interface GigabitEthernet 0/1
 switchport mode trunk
 switchport trunk native vlan 99
 no shutdown
 exit
end
write memory
```

### 3. Access Switch (`Switch2`) Configuration
```text
enable
configure terminal
hostname Switch2

! Create VLANs
vlan 10
 name HR
vlan 20
 name Engineering
vlan 99
 name Management
exit

! Access Port Assignments for VLAN 20
interface range FastEthernet 0/2 - 3
 switchport mode access
 switchport access vlan 20
 no shutdown
 exit

! Trunk Uplink to Core Switch
interface GigabitEthernet 0/3
 switchport mode trunk
 switchport trunk native vlan 99
 no shutdown
 exit
end
write memory
```

---

## Architectural & Security Insights

* **Native VLAN Hardening:** Changing the native VLAN from the default `VLAN 1` to a dedicated `VLAN 99` across all trunks mitigates **VLAN Hopping attacks** (e.g., double-tagging exploits).
* **Broadcast Domain Isolation:** By isolating HR (VLAN 10) and Engineering (VLAN 20) into distinct Layer 2 domains, broadcast storms are localized, protecting switch processing cycles and link bandwidth.
* **Predictable Root Election:** Manually tuning STP priorities ensures that traffic paths are deterministically built around the high-speed Core Switch, preventing lower-tier access switches from becoming rogue root bridges during topology changes.