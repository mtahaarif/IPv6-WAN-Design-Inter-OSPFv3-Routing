# IPv6 WAN Design & OSPFv3 Routing

A Cisco Packet Tracer simulation implementing a pure IPv6 wide-area network with dynamic routing via **OSPFv3**, built for the **Computer Networks (EC330)** Complex Engineering Problem (CEP), Semester 5, Fall 2024 — NUST CEME, Instructor: Dr. Salman, Lab Engineer: Engr. Usama Shoukat.

> Deliverables required by the CEP brief: a complete network topology, configuration commands, and a project report demonstrating full reachability between at least three end devices using IPv6 addressing only.

---

## 1. Problem Statement

IPv4's 32-bit address space (~4.3 billion addresses) is exhausted relative to global demand. IPv6 replaces it with a 128-bit address space (~3.4×10³⁸ addresses), and this project's goal was to design and implement a small WAN that:

- Uses **IPv6 addressing exclusively** (no IPv4 configured on any interface)
- Connects **at least 3 end devices** across **2 separate sites/subnets**
- Achieves **full reachability** — every device and every active port can ping every other device
- Uses a **dynamic routing protocol** (OSPFv3) rather than static routes, so the design demonstrates real WAN-scale route propagation instead of manually maintained routing tables

---

## 2. Network Topology

![Topology Diagram](/topology-diagram.png)

The topology is two LAN sites connected over a simulated WAN (serial) link:

```
   Site A (LAN)                 WAN Link                  Site C (LAN)
┌───────────────┐                                      ┌───────────────┐
│  PC0 ── Sw5 ── │  Fa0/0        Se1/0        Se1/0      │ ── Sw6 ── PC4 │
│  PC3 ──┘       ├──────── Router0 ═══════════ Router1 ──┤    Fa0/0     │
│  2001:A::/64   │        (RID 1.1.1.1)  (RID 2.2.2.2)   │  2001:C::/64  │
└───────────────┘         2001:B::/64 (P2P link)        └───────────────┘
```

| Layer | Role |
|---|---|
| **Router0 (Cisco 2811)** | Site A gateway; OSPFv3 router-id `1.1.1.1`; provides the DCE clocking on the serial WAN link |
| **Router1 (Cisco 2811)** | Site C gateway; OSPFv3 router-id `2.2.2.2` |
| **Switch5 (Cisco 2950-24)** | Layer-2 access switch for Site A, connects PC0 and PC3 to Router0's Fa0/0 |
| **Switch6 (Cisco 2950-24)** | Layer-2 access switch for Site C, connects PC4 to Router1's Fa0/0 |
| **PC0, PC3, PC4** | End hosts, one endpoint per required minimum of 3 devices |

### Device selection rationale
- **Cisco 2811 Integrated Services Routers** were used at the WAN edge because they support both IPv6 unicast routing and OSPFv3, and expose a serial interface (`Se1/0`) needed to simulate a leased-line/WAN link between the two sites — something an access switch cannot do.
- **Cisco 2950-24 switches** provide simple Layer-2 connectivity for each site's LAN segment. Since this design only required host connectivity within a single flat subnet per site (no VLAN segmentation was implemented), unmanaged/Layer-2-only switching was sufficient and kept the topology within the CEP's minimum scope.
- A **point-to-point serial link** between the routers (rather than a shared Ethernet backbone) was chosen to realistically model a WAN connection between two separate sites, as opposed to a single-site LAN.

---

## 3. IPv6 Addressing Plan

All addresses use `/64` prefixes, the standard IPv6 subnet size for a single link, taken from the documentation address block `2001:xxxx::/32` (reserved for examples in this simulation).

| Subnet | Purpose | Devices | Address |
|---|---|---|---|
| `2001:A::/64` | Site A LAN | Router0 Fa0/0 | `2001:A::1/64` |
| | | PC0 | `2001:A::2/64` |
| | | PC3 | `2001:A::3/64` |
| `2001:B::/64` | Inter-router WAN link | Router0 Se1/0 | `2001:B::1/64` |
| | | Router1 Se1/0 | `2001:B::2/64` |
| `2001:C::/64` | Site C LAN | Router1 Fa0/0 | `2001:C::1/64` |
| | | PC4 | `2001:C::2/64` |

Each router uses an IPv4-style dotted **OSPFv3 router-id**, required because OSPFv3 still needs a 32-bit router-id even in an IPv6-only network with no IPv4 addresses configured to auto-derive one from:
- Router0 → `1.1.1.1`
- Router1 → `2.2.2.2`

---

## 4. Protocols & Configuration

### 4.1 Global IPv6 routing
IPv6 forwarding is off by default on Cisco IOS, so the first command on every router enables it before anything else:

```
Router(config)# ipv6 unicast-routing
```

### 4.2 Router0 — full configuration

```
enable
configure terminal
ipv6 unicast-routing

interface fastEthernet 0/0
 ipv6 address 2001:A::1/64
 no shutdown
 exit

interface serial 1/0
 ipv6 address 2001:B::1/64
 no shutdown
 exit

ipv6 router ospf 1
 router-id 1.1.1.1
 exit

interface fastEthernet 0/0
 ipv6 ospf 1 area 0
 exit

interface serial 1/0
 ipv6 ospf 1 area 0
 exit
```

### 4.3 Router1 — full configuration

```
enable
configure terminal
ipv6 unicast-routing

interface serial 1/0
 ipv6 address 2001:B::2/64
 no shutdown
 exit

interface fastEthernet 0/0
 ipv6 address 2001:C::1/64
 no shutdown
 exit

ipv6 router ospf 1
 router-id 2.2.2.2
 exit

interface serial 1/0
 ipv6 ospf 1 area 0
 exit

interface fastEthernet 0/0
 ipv6 ospf 1 area 0
 exit
```

### 4.4 Why OSPFv3
OSPFv3 is the IPv6-native version of OSPF (RFC 5340) — it runs directly over IPv6 rather than being carried inside it, uses link-local addresses for neighbor adjacencies, and (like OSPFv2) is enabled per-interface with `ipv6 ospf <process-id> area <area-id>` instead of the `network` statements used in OSPFv2. A single area (`area 0`, the backbone) was used since the topology is small enough that no area hierarchy is needed. All interfaces on both routers were placed in area 0, and the adjacency between Router0 and Router1 formed over the serial WAN link.

### 4.5 Configuration notes / troubleshooting encountered
During implementation the following IOS syntax errors were hit and corrected (kept here for reference since they're a realistic part of the CLI learning curve):
- `ipv6 uni-cast routing` → invalid; correct syntax has no hyphen: `ipv6 unicast-routing`
- `ipv6 ospf 1` (global config mode) → invalid; OSPFv3 process is created with `ipv6 router ospf 1`, not `ipv6 ospf 1`
- `ipv6 routerospf 1` → unrecognized command (typo)
- On process creation, IOS raised `%OSPFv3-4-NORTRID: OSPFv3 process 1 could not pick a router-id, please configure manually` because no IPv4 address exists anywhere on the router for OSPFv3 to derive a router-id from — resolved by manually setting `router-id` under `ipv6 router ospf 1`

---

## 5. Verification & Testing

Full mesh reachability was confirmed by pinging across sites, routers, and the WAN link. Sample verification — pinging Site C's host from Site A:

![Ping verification](assets/ping-verification.png)

```
C:\>ping 2001:c::2

Pinging 2001:c::2 with 32 bytes of data:
Reply from 2001:C::2: bytes=32 time=1ms TTL=126
Reply from 2001:C::2: bytes=32 time=1ms TTL=126
Reply from 2001:C::2: bytes=32 time=6ms TTL=126
Reply from 2001:C::2: bytes=32 time=6ms TTL=126

Ping statistics for 2001:C::2:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
Approximate round trip times in milli-seconds:
    Minimum = 1ms, Maximum = 6ms, Average = 3ms
```

The OSPFv3 adjacency log also confirms the neighbor relationship reached `FULL` state across the WAN link:

```
%OSPFv3-5-ADJCHG: Process 1, Nbr 2.2.2.2 on Serial1/0 from LOADING to FULL, Loading Done   (seen on Router0)
%OSPFv3-5-ADJCHG: Process 1, Nbr 1.1.1.1 on Serial1/0 from LOADING to FULL, Loading Done   (seen on Router1)
```

**Result:** 0% packet loss, TTL=126 (2 hops — one router each way, consistent with the topology), confirming the route was learned dynamically via OSPFv3 rather than a directly connected path. All devices (PC0, PC3, PC4) and all active router/switch ports were able to reach one another, satisfying the CEP's minimum-3-device full-reachability requirement.

---

## 6. Repository Contents

| File | Description |
|---|---|
| `CN Project.pkt` | Cisco Packet Tracer simulation file — final working topology |
| `CNP_PROJ_IN_1.pkt` | Earlier/alternate save of the Packet Tracer topology |
| `CN Project.docx` | Project report containing topology screenshot, router CLI configuration logs, and ping verification output |
| `complex-engineering-problem.pdf` | Official CEP assignment brief issued by the course instructor |
| `assets/` | Diagram and CLI screenshots extracted from the project report, embedded above |
| `README.md` | This document |

> **Note:** `.pkt` files are Cisco Packet Tracer's proprietary encrypted format and can only be opened/edited in Packet Tracer itself. To reproduce or extend this topology, open `CN Project.pkt` in Cisco Packet Tracer.

---

## 7. Design Scope & Possible Extensions

The implemented topology satisfies the CEP requirements as a **flat-per-site IPv6 network** — each site (`2001:A::/64`, `2001:C::/64`) is a single broadcast domain with Layer-2-only switches, and both LANs are joined by an OSPFv3-routed WAN link (`2001:B::/64`). Natural next steps if the design were extended for a larger deployment:

- **VLAN segmentation** at each site (e.g. separating staff/guest/server traffic) using 802.1Q trunking between the access switches and a **router-on-a-stick** configuration on the site routers' Fa0/0 sub-interfaces, each with its own IPv6 prefix and OSPFv3 participation
- **Multiple OSPFv3 areas** if the topology grew beyond a small backbone
- **IPv6 access control lists** on the WAN-facing interfaces for basic perimeter filtering
- **DHCPv6 / SLAAC** for the LAN hosts instead of static IPv6 addressing, to demonstrate automatic address assignment

---

## 8. Key Learning Outcomes

- Enabling and configuring IPv6-only routing on Cisco IOS (`ipv6 unicast-routing`, per-interface `ipv6 address`)
- Configuring and verifying **OSPFv3** neighbor adjacencies and manual router-id assignment
- Designing an IPv6 addressing plan using `/64` subnets across multiple sites and a WAN transit link
- Diagnosing and correcting common Cisco IOS CLI syntax pitfalls specific to IPv6/OSPFv3 configuration
- Verifying end-to-end reachability and interpreting `ping` and IOS log output (TTL, adjacency state changes) as evidence of correct dynamic routing
