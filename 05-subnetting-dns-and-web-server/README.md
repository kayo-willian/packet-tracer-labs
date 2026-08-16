# Lab 05, Subnetting, DNS and Web Server

## Objective
Practice subnetting for the first time using static addressing, dividing
a single `/24` network into three smaller `/28` subnets, and add DNS and
HTTP/Web services running on the same server, reachable by name instead
of raw IP.

The main goal was understanding, in practice, concepts like network
address, host range, broadcast address, and how a router connects
separate subnets through different interfaces.

## Topology
![topology](images/01-topology.png)

```
IT SECTOR                HR SECTOR                SERVERS SECTOR

IT-0, IT-1, IT-2         HR-0, HR-1, HR-2          Server-PT (DNS + Web)
     |                        |                          |
  Switch-IT               Switch-HR                  Switch-SRV
     |                        |                          |
     +----------------------- Router ----------------------+
       (Gi0/0)              (Gi0/1)                  (Gi0/2)
```

- 1 router, 3 switches, 6 PCs, 1 server
- 3 subnets, all carved out of the same private range `192.168.1.0/24`,
  divided using `/28`
- Devices connected with Copper Straight Through cables
- The server runs both DNS and HTTP/Web services on the same machine

## Addressing

| Network | CIDR | Mask | Gateway | Usable host range | Broadcast | Devices |
|---------|------|------|---------|--------------------|-----------|---------|
| IT | `192.168.1.0/28` | `255.255.255.240` | `192.168.1.1` | `.2` to `.14` | `.15` | IT-0 (`.2`), IT-1 (`.3`), IT-2 (`.4`) |
| HR | `192.168.1.16/28` | `255.255.255.240` | `192.168.1.17` | `.18` to `.30` | `.31` | HR-0 (`.18`), HR-1 (`.19`), HR-2 (`.20`) |
| Servers | `192.168.1.32/28` | `255.255.255.240` | `192.168.1.33` | `.34` to `.46` | `.47` | Server-PT (`.34`), running DNS and HTTP/Web |

All PCs and the server were addressed statically. Every PC's DNS Server
field points to `192.168.1.34`, the same server running the DNS service.

## Configuration summary
- Router interfaces:
  - `GigabitEthernet0/0`: `192.168.1.1/28` (IT)
  - `GigabitEthernet0/1`: `192.168.1.17/28` (HR)
  - `GigabitEthernet0/2`: `192.168.1.33/28` (Servers)
- No static routes were needed, all three subnets are directly connected
  to the same router, one per interface.
- Static IP addressing on every PC and on the server, no DHCP in this lab.

Router interfaces confirmed active with `show ip interface brief`:
![router interfaces](images/02-router-interfaces.png)

## DNS configuration
DNS service enabled on Server-PT, with a single A record:

![dns record](images/03-dns-record.png)

| Name | Type | Address |
|------|------|---------|
| `www.company.com` | A Record | `192.168.1.34` |

## Web server
A simple test page was created for the fictional "Company," confirming
the server is online and identifying it as a lab environment.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Company Test Server</title>
</head>
<body>
    <h1>Welcome to Company</h1>
    <p>This is a test website hosted on the company's web server.</p>
    <hr>
    <h2>Server Status</h2>
    <p>Web server: <strong>Online</strong></p>
    <p>Domain: <strong>www.company.com</strong></p>
    <p>Network laboratory test environment.</p>
</body>
</html>
```

Accessed by domain name (not by raw IP) from a PC on the IT subnet:
![web page accessed by domain](images/04-web-page.png)

## Connectivity tests

Tested from IT-0:

```
C:\>ping 192.168.1.18

Pinging 192.168.1.18 with 32 bytes of data:

Request timed out.
Reply from 192.168.1.18: bytes=32 time<1ms TTL=127
Reply from 192.168.1.18: bytes=32 time<1ms TTL=127
Reply from 192.168.1.18: bytes=32 time<1ms TTL=127

Ping statistics for 192.168.1.18:
Packets: Sent = 4, Received = 3, Lost = 1 (25% loss)

C:\>ping 192.168.1.34

Pinging 192.168.1.34 with 32 bytes of data:

Reply from 192.168.1.34: bytes=32 time<1ms TTL=127
Reply from 192.168.1.34: bytes=32 time<1ms TTL=127
Reply from 192.168.1.34: bytes=32 time<1ms TTL=127
Reply from 192.168.1.34: bytes=32 time=1ms TTL=127

Ping statistics for 192.168.1.34:
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```
![connectivity test](images/05-connectivity-test.png)

This confirms IT-0 (IT subnet) successfully reaching HR-0 (HR subnet)
through the router, and reaching the server (Servers subnet) directly by
IP. The first timeout on the HR ping is consistent with ARP resolution
delay on the first packet, same pattern seen in earlier labs.

Reaching `www.company.com` by domain name from IT-0, and accessing the
web page through HTTP using that same domain, were also tested and
confirmed successful, shown in the DNS and web server sections above.
Ping between IT-0 and IT-1 (same subnet) and HR-0 reaching the server
were also tested successfully, though no separate screenshot was kept for
those specific runs.

## Problems and troubleshooting
Subnetting math was the main difficulty in this lab. Calculating the
network address, host range, and broadcast address by hand for each `/28`
block did not go smoothly at first. An online subnetting calculator was
used to check the numbers and confirm they were correct before applying
them in Packet Tracer. The goal was not to just copy the calculator's
output, but to use it as a way to verify manual reasoning while still
learning the concept, not to skip understanding it.

One small typo happened during testing, a ping was mistakenly sent to
`192.168.1.334` (an extra digit), which correctly failed with a host not
found error since that is not a valid address. Not a configuration
problem, just a typing mistake, but it is a good reminder that Packet
Tracer will validate the address format itself.

## Subnetting reasoning
Starting from `192.168.1.0/24` and using a `/28` mask, each subnet block
is exactly 16 addresses wide, since 4 bits are left for host addressing
(2^4 = 16). That is why each subnet starts at a multiple of 16: `.0`,
`.16`, `.32`, and so on. Out of each 16 address block, the first address
is reserved as the network address, the last is reserved as the broadcast
address, leaving 14 usable addresses for hosts and the gateway in between.

That is how the three subnets ended up as `192.168.1.0/28` (IT),
`192.168.1.16/28` (HR), and `192.168.1.32/28` (Servers), each one a
separate slice of the same original `/24` block.

## Result
- The three subnets are correctly separated and each is reachable through
  its own router interface.
- Hosts in different subnets (IT and HR) can communicate with each other
  through the router.
- DNS resolves `www.company.com` to the server's IP.
- The web page is reachable using the domain name, not just the raw IP,
  confirming DNS and HTTP are working together.

## What I learned
- A `/28` mask creates blocks of 16 addresses each.
- Every subnet has a network address, a broadcast address, and a usable
  host range in between, none of which can be assigned to a device.
- `255.255.255.240` is the same thing as `/28`, just written differently.
- `192.168.1.0/28` and `192.168.1.16/28` are two completely separate
  subnets, even though they come from the same `/24` block.
- A router can connect multiple subnets by giving each one its own
  interface, and hosts on different subnets need to go through the router
  to reach each other.
- I also got a clearer sense of the difference between a network address,
  a host address, and a broadcast address, three things that are easy to
  confuse at first.
- Using a subnetting calculator to check manual work is a reasonable way
  to learn, as long as the goal stays understanding the math, not just
  getting the right numbers.
