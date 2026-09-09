# Lab 11 – Pará State OSPF Backbone (10-Site Ring Topology + Internet Access via NAT)

## Overview

![Backbone topology diagram](./images/topologia-backbone.png)

This lab simulates a statewide corporate backbone interconnecting **10 sites** across the state of Pará (Belém, Castanhal, Paragominas, Marabá, Redenção, Altamira, Itaituba, Santarém, Óbidos and Oriximiná), built in Cisco Packet Tracer as part of my Network Administrator training (SENAC).

The mission was split into four stages:

1. **Physical cabling** — connect all 10 routers in a **ring topology**, each site also connected to its local access switch.
2. **Base configuration** — apply VLANs, trunking, router-on-a-stick subinterfaces and DHCP per site (provided as a base script, no routing yet).
3. **Dynamic routing** — bring up **OSPF (Area 0)** across all 10 routers so every site can reach every other site, with LAN interfaces set as passive.
4. **Internet edge (bonus)** — turn the Belém router (R1) into the Internet gateway for the entire state using **NAT/PAT**, connected to a simulated ISP router.

A "chaos test" validates OSPF's convergence: while a continuous ping runs from Belém to Santarém, the Castanhal link is shut down, and OSPF recalculates the path within seconds.

## Topology

- 10 sites, each with **1 router + 1 access switch + 1 PC**, in a physical ring:

```
R1(Belém) - R2(Castanhal) - R3(Paragominas) - R4(Marabá) - R5(Redenção) -
R6(Altamira) - R7(Itaituba) - R8(Santarém) - R9(Óbidos) - R10(Oriximiná) - back to R1
```

- Each router uses:
  - `Gig0/0` (with a dot1Q subinterface) → connected to the local access switch (LAN/VLAN)
  - `Gig0/1` → WAN link to the **next** site in the ring
  - `Gig0/2` → WAN link to the **previous** site in the ring
- Internet edge (bonus): `R1 (Belém)` gets a serial WIC module and connects via a serial cable to an `ISP_PROVEDOR` router, which has a server (`8.8.8.8`) attached.

## Cabling Guide

| Origin | Source Port | → | Destination Port | Destination |
|---|---|---|---|---|
| R1 (Belém) | Gig0/1 | → | Gig0/2 | R2 (Castanhal) |
| R2 (Castanhal) | Gig0/1 | → | Gig0/2 | R3 (Paragominas) |
| R3 (Paragominas) | Gig0/1 | → | Gig0/2 | R4 (Marabá) |
| R4 (Marabá) | Gig0/1 | → | Gig0/2 | R5 (Redenção) |
| R5 (Redenção) | Gig0/1 | → | Gig0/2 | R6 (Altamira) |
| R6 (Altamira) | Gig0/1 | → | Gig0/2 | R7 (Itaituba) |
| R7 (Itaituba) | Gig0/1 | → | Gig0/2 | R8 (Santarém) |
| R8 (Santarém) | Gig0/1 | → | Gig0/2 | R9 (Óbidos) |
| R9 (Óbidos) | Gig0/1 | → | Gig0/2 | R10 (Oriximiná) |
| R10 (Oriximiná) | Gig0/1 | → | Gig0/2 | R1 (Belém) |

Every router's `Gig0/0` connects to its own local switch.

## IP Addressing Plan (/24)

> ⚠️ **Correction applied:** the original task sheet had a typo on the R9↔R10 link, listing it as `172.16.910.9` / `172.16.910.10` (not a valid /24 network — "910" is not a valid octet). The correct network is **`172.16.90.0/24`**, used below. This is reflected in every script in this document.

| Router | LAN Subinterface (Gateway) | WAN 1 IP (Gig0/1) | WAN 2 IP (Gig0/2) |
|---|---|---|---|
| R1 (Belém) | Gig0/0.10 – 192.168.10.1 | 172.16.12.1 | 172.16.101.1 |
| R2 (Castanhal) | Gig0/0.20 – 192.168.20.1 | 172.16.23.2 | 172.16.12.2 |
| R3 (Paragominas) | Gig0/0.30 – 192.168.30.1 | 172.16.34.3 | 172.16.23.3 |
| R4 (Marabá) | Gig0/0.40 – 192.168.40.1 | 172.16.45.4 | 172.16.34.4 |
| R5 (Redenção) | Gig0/0.50 – 192.168.50.1 | 172.16.56.5 | 172.16.45.5 |
| R6 (Altamira) | Gig0/0.60 – 192.168.60.1 | 172.16.67.6 | 172.16.56.6 |
| R7 (Itaituba) | Gig0/0.70 – 192.168.70.1 | 172.16.78.7 | 172.16.67.7 |
| R8 (Santarém) | Gig0/0.80 – 192.168.80.1 | 172.16.89.8 | 172.16.78.8 |
| R9 (Óbidos) | Gig0/0.90 – 192.168.90.1 | **172.16.90.9** | 172.16.89.9 |
| R10 (Oriximiná) | Gig0/0.100 – 192.168.100.1 | 172.16.101.10 | **172.16.90.10** |

## Base Configuration Scripts (VLAN, Trunk, IP, DHCP)

These scripts were applied to each site first, before any routing protocol — they only bring up VLANs, trunking, interface IPs and DHCP pools.

### R1 (Belém) + Switch 1

```
! SWITCH 1
enable
configure terminal
vlan 10
 name BELEM
 exit
interface fa0/1
 switchport mode access
 switchport access vlan 10
 exit
interface gig0/1
 switchport mode trunk
 exit

! ROUTER 1
enable
configure terminal
interface gig0/1
 ip address 172.16.12.1 255.255.255.0
 no shutdown
 exit
interface gig0/2
 ip address 172.16.101.1 255.255.255.0
 no shutdown
 exit
interface gig0/0
 no shutdown
 exit
interface gig0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 exit
ip dhcp excluded-address 192.168.10.1
ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 192.168.10.254
end
```

### R2 (Castanhal) + Switch 2

```
! SWITCH 2
enable
configure terminal
vlan 20
 name CASTANHAL
 exit
interface fa0/1
 switchport mode access
 switchport access vlan 20
 exit
interface gig0/1
 switchport mode trunk
 exit

! ROUTER 2
enable
configure terminal
interface gig0/1
 ip address 172.16.23.2 255.255.255.0
 no shutdown
 exit
interface gig0/2
 ip address 172.16.12.2 255.255.255.0
 no shutdown
 exit
interface gig0/0
 no shutdown
 exit
interface gig0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 exit
ip dhcp excluded-address 192.168.20.1
ip dhcp pool VLAN20
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 192.168.10.254
end
```

### R3 (Paragominas) + Switch 3

```
! SWITCH 3
enable
configure terminal
vlan 30
 name PARAGOMINAS
 exit
interface fa0/1
 switchport mode access
 switchport access vlan 30
 exit
interface gig0/1
 switchport mode trunk
 exit

! ROUTER 3
enable
configure terminal
interface gig0/1
 ip address 172.16.34.3 255.255.255.0
 no shutdown
 exit
interface gig0/2
 ip address 172.16.23.3 255.255.255.0
 no shutdown
 exit
interface gig0/0
 no shutdown
 exit
interface gig0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 exit
ip dhcp excluded-address 192.168.30.1
ip dhcp pool VLAN30
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 192.168.10.254
end
```

### R4 (Marabá) + Switch 4

```
! SWITCH 4
enable
configure terminal
vlan 40
 name MARABA
 exit
interface fa0/1
 switchport mode access
 switchport access vlan 40
 exit
interface gig0/1
 switchport mode trunk
 exit

! ROUTER 4
enable
configure terminal
interface gig0/1
 ip address 172.16.45.4 255.255.255.0
 no shutdown
 exit
interface gig0/2
 ip address 172.16.34.4 255.255.255.0
 no shutdown
 exit
interface gig0/0
 no shutdown
 exit
interface gig0/0.40
 encapsulation dot1Q 40
 ip address 192.168.40.1 255.255.255.0
 exit
ip dhcp excluded-address 192.168.40.1
ip dhcp pool VLAN40
 network 192.168.40.0 255.255.255.0
 default-router 192.168.40.1
 dns-server 192.168.10.254
end
```

### R5 (Redenção) + Switch 5

```
! SWITCH 5
enable
configure terminal
vlan 50
 name REDENCAO
 exit
interface fa0/1
 switchport mode access
 switchport access vlan 50
 exit
interface gig0/1
 switchport mode trunk
 exit

! ROUTER 5
enable
configure terminal
interface gig0/1
 ip address 172.16.56.5 255.255.255.0
 no shutdown
 exit
interface gig0/2
 ip address 172.16.45.5 255.255.255.0
 no shutdown
 exit
interface gig0/0
 no shutdown
 exit
interface gig0/0.50
 encapsulation dot1Q 50
 ip address 192.168.50.1 255.255.255.0
 exit
ip dhcp excluded-address 192.168.50.1
ip dhcp pool VLAN50
 network 192.168.50.0 255.255.255.0
 default-router 192.168.50.1
 dns-server 192.168.10.254
end
```

### R6 (Altamira) + Switch 6

```
! SWITCH 6
enable
configure terminal
vlan 60
 name ALTAMIRA
 exit
interface fa0/1
 switchport mode access
 switchport access vlan 60
 exit
interface gig0/1
 switchport mode trunk
 exit

! ROUTER 6
enable
configure terminal
interface gig0/1
 ip address 172.16.67.6 255.255.255.0
 no shutdown
 exit
interface gig0/2
 ip address 172.16.56.6 255.255.255.0
 no shutdown
 exit
interface gig0/0
 no shutdown
 exit
interface gig0/0.60
 encapsulation dot1Q 60
 ip address 192.168.60.1 255.255.255.0
 exit
ip dhcp excluded-address 192.168.60.1
ip dhcp pool VLAN60
 network 192.168.60.0 255.255.255.0
 default-router 192.168.60.1
 dns-server 192.168.10.254
end
```

### R7 (Itaituba) + Switch 7

```
! SWITCH 7
enable
configure terminal
vlan 70
 name ITAITUBA
 exit
interface fa0/1
 switchport mode access
 switchport access vlan 70
 exit
interface gig0/1
 switchport mode trunk
 exit

! ROUTER 7
enable
configure terminal
interface gig0/1
 ip address 172.16.78.7 255.255.255.0
 no shutdown
 exit
interface gig0/2
 ip address 172.16.67.7 255.255.255.0
 no shutdown
 exit
interface gig0/0
 no shutdown
 exit
interface gig0/0.70
 encapsulation dot1Q 70
 ip address 192.168.70.1 255.255.255.0
 exit
ip dhcp excluded-address 192.168.70.1
ip dhcp pool VLAN70
 network 192.168.70.0 255.255.255.0
 default-router 192.168.70.1
 dns-server 192.168.10.254
end
```

### R8 (Santarém) + Switch 8

```
! SWITCH 8
enable
configure terminal
vlan 80
 name SANTAREM
 exit
interface fa0/1
 switchport mode access
 switchport access vlan 80
 exit
interface gig0/1
 switchport mode trunk
 exit

! ROUTER 8
enable
configure terminal
interface gig0/1
 ip address 172.16.89.8 255.255.255.0
 no shutdown
 exit
interface gig0/2
 ip address 172.16.78.8 255.255.255.0
 no shutdown
 exit
interface gig0/0
 no shutdown
 exit
interface gig0/0.80
 encapsulation dot1Q 80
 ip address 192.168.80.1 255.255.255.0
 exit
ip dhcp excluded-address 192.168.80.1
ip dhcp pool VLAN80
 network 192.168.80.0 255.255.255.0
 default-router 192.168.80.1
 dns-server 192.168.10.254
end
```

### R9 (Óbidos) + Switch 9

> Corrected: `Gig0/1` uses `172.16.90.9/24` (original sheet had the invalid `172.16.910.9`).

```
! SWITCH 9
enable
configure terminal
vlan 90
 name OBIDOS
 exit
interface fa0/1
 switchport mode access
 switchport access vlan 90
 exit
interface gig0/1
 switchport mode trunk
 exit

! ROUTER 9
enable
configure terminal
interface gig0/1
 ip address 172.16.90.9 255.255.255.0
 no shutdown
 exit
interface gig0/2
 ip address 172.16.89.9 255.255.255.0
 no shutdown
 exit
interface gig0/0
 no shutdown
 exit
interface gig0/0.90
 encapsulation dot1Q 90
 ip address 192.168.90.1 255.255.255.0
 exit
ip dhcp excluded-address 192.168.90.1
ip dhcp pool VLAN90
 network 192.168.90.0 255.255.255.0
 default-router 192.168.90.1
 dns-server 192.168.10.254
end
```

### R10 (Oriximiná) + Switch 10

> Corrected: `Gig0/2` uses `172.16.90.10/24` (original sheet had the invalid `172.16.910.10`).

```
! SWITCH 10
enable
configure terminal
vlan 100
 name ORIXIMINA
 exit
interface fa0/1
 switchport mode access
 switchport access vlan 100
 exit
interface gig0/1
 switchport mode trunk
 exit

! ROUTER 10
enable
configure terminal
interface gig0/1
 ip address 172.16.101.10 255.255.255.0
 no shutdown
 exit
interface gig0/2
 ip address 172.16.90.10 255.255.255.0
 no shutdown
 exit
interface gig0/0
 no shutdown
 exit
interface gig0/0.100
 encapsulation dot1Q 100
 ip address 192.168.100.1 255.255.255.0
 exit
ip dhcp excluded-address 192.168.100.1
ip dhcp pool VLAN100
 network 192.168.100.0 255.255.255.0
 default-router 192.168.100.1
 dns-server 192.168.10.254
end
```

## OSPF Configuration (Area 0, All 10 Routers)

With the base scripts applied everywhere, the ring is up but each site is isolated — routing was not configured yet. The mission was to enable **OSPF Area 0** on every router, advertising the LAN and both WAN networks, and to keep the LAN subinterface **passive** (no OSPF hellos sent toward the end-user VLAN).

General pattern used on every router:

```
router ospf 1
 network [LAN_NETWORK] 0.0.0.255 area 0
 network [WAN1_NETWORK] 0.0.0.255 area 0
 network [WAN2_NETWORK] 0.0.0.255 area 0
 passive-interface gig0/0.XX
```

Applied per router:

```
! R1 (Belém)
router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 172.16.12.0 0.0.0.255 area 0
 network 172.16.101.0 0.0.0.255 area 0
 passive-interface gig0/0.10

! R2 (Castanhal)
router ospf 1
 network 192.168.20.0 0.0.0.255 area 0
 network 172.16.23.0 0.0.0.255 area 0
 network 172.16.12.0 0.0.0.255 area 0
 passive-interface gig0/0.20

! R3 (Paragominas)
router ospf 1
 network 192.168.30.0 0.0.0.255 area 0
 network 172.16.34.0 0.0.0.255 area 0
 network 172.16.23.0 0.0.0.255 area 0
 passive-interface gig0/0.30

! R4 (Marabá)
router ospf 1
 network 192.168.40.0 0.0.0.255 area 0
 network 172.16.45.0 0.0.0.255 area 0
 network 172.16.34.0 0.0.0.255 area 0
 passive-interface gig0/0.40

! R5 (Redenção)
router ospf 1
 network 192.168.50.0 0.0.0.255 area 0
 network 172.16.56.0 0.0.0.255 area 0
 network 172.16.45.0 0.0.0.255 area 0
 passive-interface gig0/0.50

! R6 (Altamira)
router ospf 1
 network 192.168.60.0 0.0.0.255 area 0
 network 172.16.67.0 0.0.0.255 area 0
 network 172.16.56.0 0.0.0.255 area 0
 passive-interface gig0/0.60

! R7 (Itaituba)
router ospf 1
 network 192.168.70.0 0.0.0.255 area 0
 network 172.16.78.0 0.0.0.255 area 0
 network 172.16.67.0 0.0.0.255 area 0
 passive-interface gig0/0.70

! R8 (Santarém)
router ospf 1
 network 192.168.80.0 0.0.0.255 area 0
 network 172.16.89.0 0.0.0.255 area 0
 network 172.16.78.0 0.0.0.255 area 0
 passive-interface gig0/0.80

! R9 (Óbidos)
router ospf 1
 network 192.168.90.0 0.0.0.255 area 0
 network 172.16.90.0 0.0.0.255 area 0
 network 172.16.89.0 0.0.0.255 area 0
 passive-interface gig0/0.90

! R10 (Oriximiná)
router ospf 1
 network 192.168.100.0 0.0.0.255 area 0
 network 172.16.101.0 0.0.0.255 area 0
 network 172.16.90.0 0.0.0.255 area 0
 passive-interface gig0/0.100
```

### Chaos Test (High Availability Validation)

1. On the Belém PC, start a continuous ping to the Santarém PC: `ping -t [Santarém PC IP]`.
2. On R2 (Castanhal), shut down the link toward Belém: `interface gig0/2` → `shutdown`.
3. Observe a few dropped pings while OSPF recalculates, then traffic resumes automatically over the alternate path around the ring (via R10 → R9 → R8) — no manual intervention needed.

## Bonus: Internet Access via NAT (PAT)

To provide Internet access for the whole state, an `HWIC-2T` serial module was added to R1 (Belém) and to a new `ISP_PROVEDOR` router, connected via a serial cable, with a server (`8.8.8.8`) attached to the ISP side.

### ISP Router Script

```
enable
configure terminal
hostname ISP_PROVEDOR
interface serial 0/0/0
 ip address 200.100.50.2 255.255.255.252
 clock rate 64000
 no shutdown
 exit
interface gigabitEthernet 0/0
 ip address 8.8.8.1 255.255.255.0
 no shutdown
 exit
end
```

### R1 (Belém) — Internet Edge Configuration

```
enable
configure terminal
interface serial 0/0/0
 ip address 200.100.50.1 255.255.255.252
 no shutdown
 exit

! Default route toward the ISP
ip route 0.0.0.0 0.0.0.0 200.100.50.2

! Access-list permitting the internal networks (state backbone) to be NATed
access-list 1 permit 192.168.0.0 0.0.255.255
access-list 1 permit 172.16.0.0 0.0.255.255

! NAT overload (PAT) on the outside (Internet-facing) interface
ip nat inside source list 1 interface serial 0/0/0 overload

! Mark inside vs outside interfaces
interface gig0/1
 ip nat inside
 exit
interface gig0/2
 ip nat inside
 exit
interface gig0/0
 ip nat inside
 exit
interface serial 0/0/0
 ip nat outside
 exit

! Inject the default route into OSPF so every site learns the Internet path
router ospf 1
 default-information originate
end
```

### Internet Test

From the Santarém PC, ping `8.8.8.8` — the packet travels across the OSPF backbone to Belém, gets NATed (PAT) on R1's serial interface, and reaches the simulated ISP server.

## Verification Commands Used

| Command | Purpose |
|---|---|
| `show ip ospf neighbor` | Confirms OSPF adjacency (FULL state) with the neighboring routers on the ring |
| `show ip route` | Confirms every VLAN/site subnet is learned via OSPF (`O`) plus the default route (`O*E2`) after NAT bonus |
| `show ip interface brief` | Confirms all physical/logical interfaces are up/up with the correct IPs |
| `show ip nat translations` | Confirms PAT translations on R1 while testing Internet access |
| `ping` (from each site's PC) | End-to-end reachability test between VLANs across the state |

## Errors and Corrections

| Issue | Original (incorrect) | Corrected | Where |
|---|---|---|---|
| Invalid /24 network on the R9↔R10 link | `172.16.910.9` / `172.16.910.10` | `172.16.90.9` / `172.16.90.10` | R9 Gig0/1, R10 Gig0/2, base scripts and OSPF `network` statements |

## Skills Practiced

- VLAN creation and access-port assignment on Cisco switches
- 802.1Q trunking between switch and router
- Router-on-a-stick (subinterfaces) for inter-VLAN routing
- DHCP pool configuration on Cisco IOS
- OSPF single-area (Area 0) design and configuration across a 10-router ring topology
- OSPF passive interfaces to avoid unnecessary hellos toward end-user VLANs
- Static default routing + `default-information originate` to distribute an Internet route via OSPF
- NAT overload (PAT) with standard ACLs to give an entire routed domain shared Internet access
- Fault-tolerance validation on a ring topology (link failure + automatic OSPF reconvergence)
