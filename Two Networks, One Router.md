# Lab 01 — Two Networks, One Router

## Objective
Connect two PCs on different networks (192.168.1.0/24 and 192.168.2.0/24)
through a single router and get them to communicate with each other.

## Topology
![topology](topologia.png)

```
PC0 -- Switch0 -- G0/0 [Router] G0/1 -- Switch1 -- PC1
```

## Rules
- PC0 and PC1 must NOT be on the same network.
- Each side of the router must belong to a different network.
- IP addressing configured via DHCP, with two separate pools on the router
  (one per network).

## Configuration
- Network A: `192.168.1.0/24` — gateway `192.168.1.1`
- Network B: `192.168.2.0/24` — gateway `192.168.2.1`
- IP addresses assigned via DHCP configured directly on the router.

### Commands used
```
ip dhcp pool MY_NETWORK1
network 192.168.1.0 255.255.255.0
default-router 192.168.1.1

ip dhcp pool MY_NETWORK2
network 192.168.2.0 255.255.255.0
default-router 192.168.2.1
```

## Problem found and fix
PC1 could not ping PC0. The cause was an incomplete DHCP pool configuration,
the default gateway wasn't being correctly assigned to one of the PCs.
After reviewing and completing the pool configuration, the PC picked up the
correct gateway and the ping succeeded, with a TTL of 127 confirming the
packet had passed through the router.

## Result
```
C:\>ping 192.168.1.2

Reply from 192.168.1.2: bytes=32 time<1ms TTL=127
Reply from 192.168.1.2: bytes=32 time<1ms TTL=127
Reply from 192.168.1.2: bytes=32 time<1ms TTL=127
Reply from 192.168.1.2: bytes=32 time<1ms TTL=127

Ping statistics for 192.168.1.2:
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

## What I learned
- A host only talks directly to other hosts on its own network.
- For any destination outside its own network, a host forwards the packet to
  its default gateway.
- The router decides, based on its routing table, which interface to forward
  the packet out of.
- A lower TTL on a ping response is a sign that the packet passed through at
  least one router (each hop decrements TTL by 1).

## Next step
Redo this same lab using static IP addressing instead of DHCP.
