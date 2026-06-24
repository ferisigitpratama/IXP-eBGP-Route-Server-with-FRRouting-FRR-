# IXP eBGP Route Server with FRR

A multi-vendor **Internet Exchange Point (IXP)** lab built in **GNS3** using **FRRouting (FRR)** as an **eBGP Route Server**.  
This project simulates how multiple autonomous systems exchange routes over a shared peering LAN through a centralized route server, without requiring full-mesh BGP sessions between every participant.

---

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [Topology](#topology)
- [IP Addressing & ASN Plan](#ip-addressing--asn-plan)
- [Technologies Used](#technologies-used)
- [How It Works](#how-it-works)
- [FRR Route Server Configuration](#frr-route-server-configuration)
- [Participant Configuration Summary](#participant-configuration-summary)
  - [Juniper vMX](#juniper-vmx)
  - [MikroTik RouterOS v7](#mikrotik-routeros-v7)
  - [Huawei NE40E](#huawei-ne40e)
  - [Simulated Google / Cloudflare](#simulated-google--cloudflare)
- [Verification](#verification)
- [Troubleshooting Notes](#troubleshooting-notes)
- [Project Structure](#project-structure)
- [Future Improvements](#future-improvements)
- [Author](#author)

---

## Overview

This project demonstrates the implementation of an **Internet Exchange Point (IXP)** environment using **FRRouting (FRR)** as a **Route Server (RS)**.  
Each participant peers only with the route server using **eBGP**, while FRR redistributes prefixes between all members using a **multilateral peering model**.

The lab includes a **multi-vendor environment** consisting of:

- **Juniper vMX**
- **MikroTik RouterOS v7**
- **Huawei NE40E**
- Simulated **Google** and **Cloudflare** route sources

The route server receives and distributes routes such as:

- **1.1.1.1/32** from **Cloudflare (AS13335)**
- **8.8.8.8/32** from **Google (AS15169)**

This lab is intended for learning and validating:

- **IXP route server architecture**
- **eBGP multilateral peering**
- **multi-vendor BGP interoperability**
- **next-hop preservation**
- **route advertisement and propagation**
- **BGP troubleshooting in a shared IX segment**

---

## Objectives

The main goals of this project are:

- Build a virtual **Internet Exchange Point (IXP)** in **GNS3**
- Deploy **FRRouting (FRR)** as a central **eBGP Route Server**
- Establish **BGP peering** between multiple participants and the route server
- Simulate route exchange without direct full-mesh peering
- Validate **prefix propagation**, **AS_PATH**, and **next-hop** behavior
- Practice troubleshooting **BGP session**, **route advertisement**, and **vendor interoperability** issues

---

## Topology

The topology uses a shared IX peering LAN `10.10.10.0/24`, where all participants establish eBGP sessions toward the FRR Route Server.

### High-Level Design

- **FRR-1** acts as the **Route Server**
- All routers peer to **FRR-1**
- **Google** advertises `8.8.8.8/32`
- **Cloudflare** advertises `1.1.1.1/32`
- Route Server distributes these routes to all IXP participants

### Topology Diagram

> Add your topology screenshot here, for example:

```md
![IXP Topology](./screenshots/topology.png)
```

---

## IP Addressing & ASN Plan

### Route Server

| Device | Role | ASN | Router ID | Peering IP |
|--------|------|-----|-----------|------------|
| FRR-1 | Route Server | 64620 | 10.0.0.100 | 10.10.10.100/24 |

### IXP Participants

| Device | Vendor | ASN | Router ID | Peering IP |
|--------|--------|-----|-----------|------------|
| vMX-1 | Juniper vMX | 64621 | 10.0.0.1 | 10.10.10.1/24 |
| R1 | Huawei NE40E | 64622 | 10.0.0.2 | 10.10.10.2/24 |
| MikroTik-1 | MikroTik ROS7 | 64623 | 10.0.0.3 | 10.10.10.3/24 |
| vMX-2 | Juniper vMX | 64624 | 10.0.0.4 | 10.10.10.4/24 |
| R2 | Huawei NE40E | 64625 | 10.0.0.5 | 10.10.10.5/24 |
| MikroTik-2 | MikroTik ROS7 | 64626 | 10.0.0.6 | 10.10.10.6/24 |

### Simulated External Networks

| Device | Role | ASN | Prefix | Peering IP |
|--------|------|-----|--------|------------|
| Google | Route Originator | 15169 | 8.8.8.8/32 | 10.10.10.11/24 |
| Cloudflare | Route Originator | 13335 | 1.1.1.1/32 | 10.10.10.12/24 |

---

## Technologies Used

- **GNS3** – lab orchestration and network simulation
- **FRRouting (FRR)** – BGP Route Server
- **Juniper vMX** – IXP participant router
- **MikroTik RouterOS v7** – IXP participant router
- **Huawei NE40E** – IXP participant router
- **eBGP** – routing protocol for peering
- **Route Server Client** – FRR multilateral peering model

---

## How It Works

In a traditional IXP, every participant would need to establish a **full-mesh BGP session** with all other members.  
That approach does not scale well as the number of participants grows.

To solve this, an **IXP Route Server** is used:

1. Every participant establishes **one BGP session** to the Route Server.
2. The Route Server receives routes from all participants.
3. The Route Server re-advertises those routes to the other members.
4. The original **AS_PATH** and **NEXT_HOP** are preserved so that traffic still flows directly between participants.

In this lab:

- **Google** advertises `8.8.8.8/32`
- **Cloudflare** advertises `1.1.1.1/32`
- **FRR** learns both routes
- FRR distributes them to all participants
- Routers such as **vMX**, **MikroTik**, and **Huawei** install those prefixes with the original next-hop

---

## FRR Route Server Configuration

Below is the FRR BGP configuration used for the route server.

```frr
router bgp 64620
 no bgp ebgp-requires-policy
 neighbor 10.10.10.1 remote-as 64621
 neighbor 10.10.10.2 remote-as 64622
 neighbor 10.10.10.3 remote-as 64623
 neighbor 10.10.10.4 remote-as 64624
 neighbor 10.10.10.5 remote-as 64625
 neighbor 10.10.10.6 remote-as 64626
 neighbor 10.10.10.11 remote-as 15169
 neighbor 10.10.10.12 remote-as 13335
 !
 address-family ipv4 unicast
  neighbor 10.10.10.1 route-server-client
  neighbor 10.10.10.1 attribute-unchanged next-hop
  neighbor 10.10.10.2 route-server-client
  neighbor 10.10.10.2 attribute-unchanged next-hop
  neighbor 10.10.10.3 route-server-client
  neighbor 10.10.10.3 attribute-unchanged next-hop
  neighbor 10.10.10.4 route-server-client
  neighbor 10.10.10.4 attribute-unchanged next-hop
  neighbor 10.10.10.5 route-server-client
  neighbor 10.10.10.5 attribute-unchanged next-hop
  neighbor 10.10.10.6 route-server-client
  neighbor 10.10.10.6 attribute-unchanged next-hop
  neighbor 10.10.10.11 route-server-client
  neighbor 10.10.10.11 attribute-unchanged next-hop
  neighbor 10.10.10.12 route-server-client
  neighbor 10.10.10.12 attribute-unchanged next-hop
 exit-address-family
```

### Important FRR Behavior

- `route-server-client` enables multilateral route distribution
- `attribute-unchanged next-hop` preserves the original next-hop
- `no bgp ebgp-requires-policy` allows route exchange without mandatory route-map policy during lab testing

---

## Participant Configuration Summary

## Juniper vMX

Example Juniper vMX participant configuration:

```junos
set interfaces em2 unit 0 family inet address 10.10.10.1/24
set interfaces lo0 unit 0 family inet address 10.0.0.1/32

set routing-options router-id 10.0.0.1
set routing-options autonomous-system 64621

set protocols bgp group RS type external
set protocols bgp group RS peer-as 64620
set protocols bgp group RS neighbor 10.10.10.100
set policy-options policy-statement EXPORT-LOOPBACK term 1 from route-filter 10.0.0.1/32 exact
set policy-options policy-statement EXPORT-LOOPBACK term 1 then accept
set protocols bgp group RS export EXPORT-LOOPBACK
```

### Expected Result

Juniper should receive:

- `1.1.1.1/32` via `10.10.10.12`
- `8.8.8.8/32` via `10.10.10.11`

---

## MikroTik RouterOS v7

Example MikroTik participant configuration:

```routeros
/interface bridge
add name=lo

/ip address
add address=10.10.10.3/24 interface=ether1
add address=10.0.0.3/32 interface=lo

/routing bgp template
add name=AS64623 as=64623 router-id=10.0.0.3

/ip firewall address-list
add list=BGP-NETWORK address=10.0.0.3/32

/routing bgp connection
add name=RS \
    remote.address=10.10.10.100 \
    remote.as=64620 \
    local.role=ebgp \
    templates=AS64623 \
    output.network=BGP-NETWORK
```

### Notes for RouterOS v7

- It is recommended to use a **custom BGP template name** instead of `default`
- Ensure the **correct ASN** is applied in the template
- `output.network` is used to advertise the local loopback/prefix

---

## Huawei NE40E

Example Huawei participant configuration:

```huawei
system-view

interface GigabitEthernet0/0/0
 ip address 10.10.10.2 255.255.255.0
 quit

interface LoopBack0
 ip address 10.0.0.2 255.255.255.255
 quit

bgp 64622
 router-id 10.0.0.2
 peer 10.10.10.100 as-number 64620

 ipv4-family unicast
  network 10.0.0.2 255.255.255.255
  peer 10.10.10.100 enable
 quit
```

### Expected Result

Huawei should learn:

- `1.1.1.1/32`
- `8.8.8.8/32`

from the route server.

---

## Simulated Google / Cloudflare

Google and Cloudflare were simulated as external route originators that inject public-looking test prefixes into the IXP.

### Google
- ASN: **15169**
- Advertised Prefix: **8.8.8.8/32**
- Peering IP: **10.10.10.11**

### Cloudflare
- ASN: **13335**
- Advertised Prefix: **1.1.1.1/32**
- Peering IP: **10.10.10.12**

These routers establish eBGP sessions to FRR and announce their respective loopback prefixes.

---

## Verification

This section contains sample verification commands and expected outputs.

---

### 1) Verify BGP on FRR

```bash
show bgp summary
show bgp ipv4 unicast
```

Expected route table:

```text
*> 1.1.1.1/32       10.10.10.12                            0 13335 i
*> 8.8.8.8/32       10.10.10.11                            0 15169 i
```

---

### 2) Verify BGP on Juniper vMX

```bash
show bgp summary
show route protocol bgp
```

Expected route example:

```text
1.1.1.1/32  [BGP/170] via 10.10.10.12
8.8.8.8/32  [BGP/170] via 10.10.10.11
```

---

### 3) Verify BGP on MikroTik

```routeros
/routing bgp session print detail
/routing route print where dst-address=1.1.1.1/32
/routing route print where dst-address=8.8.8.8/32
```

Expected:

- `1.1.1.1/32` via `10.10.10.12`
- `8.8.8.8/32` via `10.10.10.11`

---

### 4) Verify BGP on Huawei

```huawei
display bgp peer
display bgp routing-table
```

Expected:

- Peer to `10.10.10.100` in **Established** state
- BGP table contains `1.1.1.1/32` and `8.8.8.8/32`

---

## Troubleshooting Notes

During the lab build and testing process, several issues were encountered.  
These were useful for understanding how different platforms behave in a multi-vendor BGP route server environment.

### 1. MikroTik RouterOS v7 Template / ASN Issues
Some BGP sessions remained **Idle** or **Inactive** because the BGP connection inherited the wrong ASN/template configuration.

**Key lessons:**
- Use a dedicated template name such as `AS64623`, `AS64626`, etc.
- Verify that the correct `router-id` and `as` values are applied
- Recreate the BGP connection if RouterOS still references an old template state

---

### 2. Incorrect Peer ASN / Bad Peer AS
If the participant ASN configured on FRR does not match the ASN configured on the router, the BGP session will fail with **Bad Peer AS** or remain stuck in **Active / Idle**.

**Checks to perform:**
- Verify `remote-as` on FRR
- Verify `local AS` on each participant
- Confirm the correct peering IP and router-id

---

### 3. Next-Hop Resolution
Because FRR is configured as a **Route Server**, it preserves the original next-hop of the advertised routes.

That means:
- `1.1.1.1/32` should point to **10.10.10.12**
- `8.8.8.8/32` should point to **10.10.10.11**

If the next-hop cannot be resolved, the route may not become active.

---

### 4. Shared IX Segment Validation
Since all participants are connected to the same IX subnet (`10.10.10.0/24`), ARP resolution and peering connectivity must be verified.

Useful checks:
- `show arp` on Juniper
- `display arp all` on Huawei
- `/ip arp print` on MikroTik

This helps detect:
- duplicate IPs
- duplicate MACs
- unresolved next-hop issues
- broken shared segment connectivity

---

### 5. Legacy Cisco IOS Interoperability
During testing, legacy Cisco IOS platforms showed compatibility issues when peering with FRR as a route server, especially around **AS_PATH attribute handling** and malformed update interpretation.

Because of that, the final lab focuses on the more stable combination of:

- **FRR**
- **Juniper vMX**
- **MikroTik RouterOS v7**
- **Huawei NE40E**

---

## Project Structure

A recommended repository structure for this project:

```text
ixp-ebgp-rs-frr/
├── README.md
├── configs/
│   ├── frr/
│   │   └── frr.conf
│   ├── juniper/
│   │   ├── vmx-1.conf
│   │   └── vmx-2.conf
│   ├── mikrotik/
│   │   ├── mikrotik-1.rsc
│   │   ├── mikrotik-2.rsc
│   │   ├── google.rsc
│   │   └── cloudflare.rsc
│   └── huawei/
│       ├── huawei-1.txt
│       └── huawei-2.txt
├── screenshots/
│   ├── topology.png
│   ├── frr-bgp-table.png
│   ├── vmx-route-table.png
│   ├── mikrotik-route.png
│   └── huawei-bgp.png
└── docs/
    ├── addressing-plan.md
    ├── troubleshooting.md
    └── verification.md
```

---

## Future Improvements

Possible future enhancements for this lab:

- Add **IPv6 peering**
- Implement **BGP Communities**
- Simulate **blackhole community** behavior
- Add **RPKI validation**
- Introduce **more IXP participants**
- Build **route filtering / policy control**
- Add **traffic engineering** examples
- Export the full GNS3 project and startup configs

---

## Author

**Feri Pratama**  
Network Engineer / NOC / ISP Operations Enthusiast

If you find this project useful, feel free to fork it, improve it, or use it as a reference for your own **BGP Route Server / IXP lab**.

---
