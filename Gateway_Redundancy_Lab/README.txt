# Lab 3 — Gateway Redundancy & Centralised Services: HSRP, Inter-VLAN Routing, OSPF & DHCP Relay

| | |
|---|---|
| **Type** | Layer 3 Redundancy — Dual-Router Gateway, Centralised DHCP, Inter-VLAN Routing |
| **Platform** | Cisco Packet Tracer |
| **IOS Version** | Cisco IOS (2911 Series Routers, 2960 Series Switches) |
| **Skills Demonstrated** | HSRP per-VLAN load balancing, router-on-a-stick inter-VLAN routing, OSPF with default route injection, centralised DHCP with ip helper-address relay, null route loop prevention, gateway failover |
| **Devices** | 3× Routers (R1, R2, R3), 1× Distribution Switch, 3× Access Switches, 1× DHCP Server, 6× PCs |
| **Time to Review** | ~4 min |

---

## Topology

![Topology](Topology.png)

*Post-failover topology with R1 offline:*

![Topology Post Failover](Topology_Post_Failover.png)

---

## What Was Built & Why

This lab simulates a small enterprise network with three distinct user segments — Guest (VLAN 10), Staff (VLAN 20), and IT Department — served by a redundant dual-router architecture. Every design decision was made with a specific operational or security rationale, documented below.

### Architecture Overview

Two routers, R1 and R2, act as the default gateways for all user segments via **router-on-a-stick** subinterfaces — each physical uplink to the distribution switch carries both VLAN 10 and VLAN 20 traffic simultaneously using 802.1Q tagging. Neither router connects directly to the other; they communicate exclusively through R3 (simulating an upstream ISP boundary), which means every inter-router path traverses R3. OSPF runs across all three routers, advertising internal subnets and injecting a default route toward R3's simulated internet destination.

The distribution switch (DSW1) operates as a pure Layer 2 switching fabric — no routing, no SVIs for end hosts. All routing intelligence lives on R1 and R2. Below DSW1, three access switches serve the user segments: ASW2 and ASW3 carry VLAN 10 and VLAN 20 end hosts respectively, while ASW1 connects the IT department hosts directly to R1's dedicated routed interface.

A centralised DHCP server sits behind R2 on a dedicated /28 server subnet, serving all three user segments via DHCP relay. This is a deliberate architectural choice — centralised address management means a single point of visibility for all lease assignments, exclusions, and troubleshooting, regardless of which physical segment the client lives on.

### Why Router-on-a-Stick for Inter-VLAN Routing

The distribution switch is a Layer 2 device. Without a Layer 3 device handling inter-VLAN routing, a Guest PC and a Staff PC can never communicate — they exist in completely separate broadcast domains with no path between them at Layer 2. Router-on-a-stick places that routing function on R1 and R2 via subinterfaces: each router has a logical interface in both VLAN 10 and VLAN 20, enabling it to receive a frame on one VLAN subinterface and route it out another. The distribution switch simply forwards the tagged frames — it has no awareness of the routing happening above it.

### Why HSRP for Gateway Redundancy

Every end host needs a default gateway — a single IP address it sends all non-local traffic to. In a single-router design, that router going down means every device on those VLANs loses connectivity instantly, with no automated recovery. HSRP solves this by presenting a **virtual IP address** shared between R1 and R2. End hosts point their default gateway at the virtual IP — never at a physical router interface. When the active router fails, the standby router claims the virtual IP transparently and begins responding to traffic. End hosts never reconfigure anything.

Per-VLAN load balancing was implemented — R1 is the HSRP active router for VLAN 10 (priority 150) and standby for VLAN 20 (priority 50). R2 is active for VLAN 20 (priority 200) and standby for VLAN 10 (priority 50). Both routers are actively serving traffic under normal conditions — neither sits idle.

**A note on VRRP:** In production multi-vendor environments, VRRP (RFC 5798) would be the preferred choice over HSRP — it is an open standard supported by Cisco, Juniper, Huawei, and Fortinet alike, making it vendor-agnostic. HSRP is Cisco proprietary. HSRP was used here due to Packet Tracer's simulation constraints with the 2911 platform, but the operational concept is identical. This distinction is worth knowing before any networking interview in the UAE market where mixed-vendor infrastructure is common.

### Why Centralised DHCP with Relay

The DHCP server sits on `172.16.1.0/28` behind R2 — a dedicated server subnet with 14 usable addresses, sized deliberately to accommodate future server additions (monitoring, backup, DNS) without requiring subnet reconfiguration. The first 10 addresses of every subnet across this topology are reserved for infrastructure — router interfaces, virtual IPs, management addresses — regardless of current utilisation. This prevents address conflicts as the network scales without requiring DHCP reconfiguration later.

DHCP broadcasts do not cross Layer 3 boundaries by default. The `ip helper-address` command on R1 and R2's subinterfaces converts the client's broadcast DHCP Discover into a unicast packet addressed to the server at `172.16.1.2`, which the server receives, matches to the correct pool based on the originating subnet, and responds with a full address assignment — IP, subnet mask, default gateway (the HSRP virtual IP), and DNS server.

The IT department segment (`172.16.2.0/24`) behind R1 also pulls addresses from the same centralised server via relay on R1's IT-facing interface. This is the correct enterprise approach — a dedicated DHCP server provides unified visibility across all segments. Local router DHCP (configuring pools directly on R1) would fragment that visibility and is more appropriate for small branch offices without dedicated server infrastructure.

### Why a /28 for the Server Subnet

A /30 provides only 2 usable addresses — sufficient for a point-to-point link but not for a server segment that will grow. A /24 is wasteful for a handful of servers. A /28 (14 usable addresses) strikes the right balance — enough room for a DHCP server, a future DNS server, a monitoring host, and a backup server, without burning a large address block on a segment that will never have hundreds of devices.

### OSPF and Default Route Injection

OSPF Area 0 runs across R1, R2, and R3. Each router advertises its directly connected networks, allowing full route propagation across the topology. R3 hosts a loopback (`8.8.8.8/32`) simulating a Google DNS destination — reachable from any PC in the topology via OSPF learned routes.

Both R1 and R2 have static default routes pointing to R3, injected into OSPF via `default-information originate`. This means the distribution switch and all downstream devices learn the default route dynamically — no manual static configuration needed on the switches or end hosts.

**A production note worth documenting:** In a real enterprise, the boundary between the organisation's network and the ISP would run BGP (Border Gateway Protocol), not OSPF. OSPF is an Interior Gateway Protocol designed for use within a single administrative domain. BGP is the protocol that routes traffic across the internet between organisations. OSPF was used here to maintain lab simplicity while still demonstrating default route injection and dynamic routing — but this distinction is fundamental to understanding where enterprise networks end and the internet begins.

R1 and R2 do not connect directly — they communicate via R3. This means R1 reaches R2's server subnet (`172.16.1.0/28`) via the path R1→R3→R2, reflected in R1's OSPF routing table as a cost-3 route. This is not a misconfiguration — it accurately reflects the topology where R3 is the common upstream point. In a production design, a direct R1-R2 link would be added for efficiency, and this is addressed in Lab 4.

### Null Route on R3

R3 drops all traffic destined for unknown addresses via a `ip route 0.0.0.0 0.0.0.0 null0` — a black hole route. Without this, unknown destination traffic would loop between R1/R2 and R3 indefinitely until TTL expiry, consuming bandwidth and generating misleading error messages. The null route causes R3 to immediately return an ICMP Destination Unreachable for any address it has no specific route to, providing instant, clean feedback to the sender. This is standard practice on any router acting as an internet boundary.

### Security Observation — IT Department Reachability

During testing, bidirectional reachability between VLAN 10/20 end hosts and the IT department segment was confirmed working. From a network functionality standpoint this is correct — routing is working as designed.

However, from a security standpoint, regular users should not be able to initiate connections to IT infrastructure. A compromised Guest PC reaching IT management interfaces is a real attack vector. In production, an Access Control List (ACL) on R1's IT-facing interface would permit IT-initiated traffic to user segments while blocking unsolicited inbound connections from VLAN 10 and VLAN 20 hosts. This was identified during the lab, researched, and deliberately deferred — ACL implementation will be explored in Lab 4 where the full enterprise topology provides the right context for a comprehensive access control policy.

---

## Verification

### DHCP Server Pools

Three pools configured on the centralised server — VLAN10_GUESTS, VLAN20_STAFF, and IT_DEPT. Each pool specifies the correct HSRP virtual IP as the default gateway, ensuring end hosts always point to the redundant virtual address rather than a physical router interface.

![DHCP Server Pools](DHCP_Server_Pools.png)

### PC IP Configuration — DHCP Assignment Confirmed

All PCs received addresses dynamically via DHCP. The default gateway on VLAN 10 and VLAN 20 PCs shows `192.168.10.254` and `192.168.20.254` respectively — the HSRP virtual IPs, not the physical router interfaces. This confirms the relay is working correctly and end hosts are pointing at the redundant virtual gateway.

![VLAN10 PC IP Config](VLAN10_PC_IPConfig.png)
![VLAN20 PC IP Config](VLAN20_PC_IPConfig.png)
![IT PC IP Config](IT_PC_IPConfig.png)

### Trunk Verification

`show interfaces trunk` on DSW1 confirms all four trunk interfaces are active, native VLAN 99, carrying only VLANs 10 and 20. Trunk pruning eliminates unnecessary broadcast flooding from irrelevant VLANs across the switching fabric.

![DSW1 Trunk Configurations](DSW1_Trunk_Configurations.png)

### HSRP Verification — Baseline

`show standby brief` on R1 confirms Active state for VLAN 10 (priority 150, preempt enabled) and Standby for VLAN 20 (priority 50). R2 confirms Active for VLAN 20 (priority 200, preempt enabled) and Standby for VLAN 10. Both routers are actively serving traffic — per-VLAN load balancing confirmed.

![R1 HSRP Brief](R1_HSRP_Brief.png)
![R2 HSRP Brief](R2_HSRP_Brief.png)

### OSPF Routing Table — R1

R1's OSPF routing table confirms 8.8.8.8 learned via R3, R2's server subnet (172.16.1.0) learned via R3 at cost 3 (reflecting the R1→R3→R2 path in the absence of a direct R1-R2 link), and all relevant subnets propagated correctly across the topology.

![R1 OSPF Routes](R1_OSPF_Routes.png)

---

## Baseline Connectivity

Three cross-segment ping tests were run to establish baseline reachability before any failure simulation:

**Cross-VLAN and IT reachability:** PC3 (VLAN 10) successfully pinged PC4 (VLAN 20) and PC1 (IT department), and PC4 successfully pinged PC1 — confirming inter-VLAN routing is functioning correctly across all three segments via R1 and R2's subinterfaces. This is the first lab in this series where cross-VLAN communication works end to end — Lab 2 intentionally left VLANs isolated at Layer 2. The routing capability introduced here is what makes full network communication possible.

![PING All Networks](PING_All_Networks.png)

**Internet reachability:** PC6 (VLAN 20) and PC3 (VLAN 10) successfully pinged 8.8.8.8 — the simulated internet destination on R3's loopback — confirming OSPF route propagation, default route injection via `default-information originate`, and end-to-end path through the dual-router architecture to R3.

![PING to 8.8.8.8](PING_to_8.8.8.8.png)

**Unknown destination behaviour:** A ping to `192.168.99.1` and `192.168.67.42` — addresses that exist nowhere in the topology — returned immediate `Destination host unreachable` responses from R3's interfaces (10.1.13.2 and 10.1.23.2). This confirms the null route on R3 is working correctly — traffic reaches R3, finds no specific route, hits null0, and R3 returns an ICMP unreachable instantly rather than allowing the packet to loop until TTL expiry.

![PING Unknown Destination](PING_Unknown_Destination.png)

---

## HSRP Failover — R1 Shutdown

R1's g0/0 interface was administratively shut down to simulate a complete router failure. The topology immediately reflected the link going down.

![R1 Interface Shutdown](R1_Interface_Shutdown.png)

### HSRP Re-election — R2 Takes Both VLANs

With R1 offline, R2 autonomously claimed the Active role for both VLAN 10 and VLAN 20. `show standby brief` on R2 confirms Active/local for both groups — VLAN 10 showing `unknown` standby (no remaining standby router) and VLAN 20 maintaining its existing active role. No manual intervention was required.

![R2 HSRP Active Both VLANs](R2_HSRP_Active_Both_VLANs.png)

---

## Post-Failover Connectivity — Network Self-Healed

All three ping tests were repeated immediately after R1's failure:

**Cross-segment reachability:** All inter-VLAN and IT department pings succeeded through R2 — confirming that R2's subinterfaces for both VLAN 10 and VLAN 20 are handling routing for all segments simultaneously.

![PING All Networks Post Failover](PING_All_Networks_Post_Failover.png)

**Internet reachability:** 8.8.8.8 remained reachable post-failover — traffic now exits via R2's direct link to R3 rather than R1's. OSPF reconverged automatically.

![PING to 8.8.8.8 Post Failover](PING_to_8.8.8.8_Post_Failover.png)

**Unknown destination behaviour:** Unknown destination pings continued returning immediate unreachable responses — now exclusively from `10.1.23.2` (R3's R2-facing interface), confirming all traffic is now flowing through R2 and the null route on R3 remains effective.

![PING Unknown Destination Post Failover](PING_Unknown_Destination_Post_Failover.png)

---

## What I Learned

Three insights from this lab go beyond individual protocol knowledge.

**The default gateway is the single point of failure in any routed network.** Every device on a subnet sends all non-local traffic to one IP address. If that IP address stops responding, the entire subnet loses external connectivity — regardless of how redundant the rest of the infrastructure is. HSRP exists to protect exactly this point. Understanding *why* the virtual IP matters more than *how* to configure it is what drives correct FHRP design.

**Centralised services require relay — and relay requires routing.** The DHCP server sits on a different subnet from every client it serves. DHCP broadcasts are Layer 2 — they don't cross routers. `ip helper-address` bridges that gap by converting broadcasts to unicast at the router, but only works if the router has a route to the server. This created an interesting dependency in this topology: R1 reaches the DHCP server via OSPF through R3, because R1 and R2 have no direct link. The relay worked correctly — but the path it took revealed the importance of verifying routing table completeness before assuming services will function.

**Security awareness belongs in network design, not as an afterthought.** Identifying that bidirectional reachability between user VLANs and the IT segment is a security exposure — and knowing that ACLs are the correct remediation — is more valuable than blindly confirming the pings work. A network that routes correctly but exposes management infrastructure to untrusted users is not a well-designed network. This will be addressed with ACL implementation in Lab 4.

---

## Design Notes

**On VRRP vs HSRP:** VRRP (RFC 5798) is the open standard equivalent of HSRP and would be the production choice in any multi-vendor environment. Packet Tracer's 2911 platform does not support VRRP in simulation — HSRP was used as a functionally equivalent alternative. The configuration concepts transfer directly between protocols.

**On OSPF at the ISP boundary:** Real enterprise-to-ISP connections run BGP, not OSPF. OSPF was used here to keep the lab self-contained while still demonstrating default route injection and dynamic routing. The null route on R3 partially simulates ISP behaviour — dropping unknown destinations immediately rather than propagating them further.

**On the R1-R2 path via R3:** The absence of a direct R1-R2 link means inter-router traffic traverses R3. In production this would be a design concern — R3 becomes a single point of failure for R1-R2 communication. A direct link between R1 and R2 would be added in a production design. This is addressed in Lab 4 where the full enterprise topology introduces direct inter-router connectivity.

---

## File Index

| File | Description |
|---|---|
| `lab# Lab 3 — Gateway Redundancy & Centralised Services: HSRP, Inter-VLAN Routing, OSPF & DHCP Relay

| | |
|---|---|
| **Type** | Layer 3 Redundancy — Dual-Router Gateway, Centralised DHCP, Inter-VLAN Routing |
| **Platform** | Cisco Packet Tracer |
| **IOS Version** | Cisco IOS (2911 Series Routers, 2960 Series Switches) |
| **Skills Demonstrated** | HSRP per-VLAN load balancing, router-on-a-stick inter-VLAN routing, OSPF with default route injection, centralised DHCP with ip helper-address relay, null route loop prevention, gateway failover |
| **Devices** | 3× Routers (R1, R2, R3), 1× Distribution Switch, 3× Access Switches, 1× DHCP Server, 6× PCs |
| **Time to Review** | ~4 min |

---

## Topology

![Topology](Topology.png)

*Post-failover topology with R1 offline:*

![Topology Post Failover](Topology_Post_Failover.png)

---

## What Was Built & Why

This lab simulates a small enterprise network with three distinct user segments — Guest (VLAN 10), Staff (VLAN 20), and IT Department — served by a redundant dual-router architecture. Every design decision was made with a specific operational or security rationale, documented below.

### Architecture Overview

Two routers, R1 and R2, act as the default gateways for all user segments via **router-on-a-stick** subinterfaces — each physical uplink to the distribution switch carries both VLAN 10 and VLAN 20 traffic simultaneously using 802.1Q tagging. Neither router connects directly to the other; they communicate exclusively through R3 (simulating an upstream ISP boundary), which means every inter-router path traverses R3. OSPF runs across all three routers, advertising internal subnets and injecting a default route toward R3's simulated internet destination.

The distribution switch (DSW1) operates as a pure Layer 2 switching fabric — no routing, no SVIs for end hosts. All routing intelligence lives on R1 and R2. Below DSW1, three access switches serve the user segments: ASW2 and ASW3 carry VLAN 10 and VLAN 20 end hosts respectively, while ASW1 connects the IT department hosts directly to R1's dedicated routed interface.

A centralised DHCP server sits behind R2 on a dedicated /28 server subnet, serving all three user segments via DHCP relay. This is a deliberate architectural choice — centralised address management means a single point of visibility for all lease assignments, exclusions, and troubleshooting, regardless of which physical segment the client lives on.

### Why Router-on-a-Stick for Inter-VLAN Routing

The distribution switch is a Layer 2 device. Without a Layer 3 device handling inter-VLAN routing, a Guest PC and a Staff PC can never communicate — they exist in completely separate broadcast domains with no path between them at Layer 2. Router-on-a-stick places that routing function on R1 and R2 via subinterfaces: each router has a logical interface in both VLAN 10 and VLAN 20, enabling it to receive a frame on one VLAN subinterface and route it out another. The distribution switch simply forwards the tagged frames — it has no awareness of the routing happening above it.

### Why HSRP for Gateway Redundancy

Every end host needs a default gateway — a single IP address it sends all non-local traffic to. In a single-router design, that router going down means every device on those VLANs loses connectivity instantly, with no automated recovery. HSRP solves this by presenting a **virtual IP address** shared between R1 and R2. End hosts point their default gateway at the virtual IP — never at a physical router interface. When the active router fails, the standby router claims the virtual IP transparently and begins responding to traffic. End hosts never reconfigure anything.

Per-VLAN load balancing was implemented — R1 is the HSRP active router for VLAN 10 (priority 150) and standby for VLAN 20 (priority 50). R2 is active for VLAN 20 (priority 200) and standby for VLAN 10 (priority 50). Both routers are actively serving traffic under normal conditions — neither sits idle.

**A note on VRRP:** In production multi-vendor environments, VRRP (RFC 5798) would be the preferred choice over HSRP — it is an open standard supported by Cisco, Juniper, Huawei, and Fortinet alike, making it vendor-agnostic. HSRP is Cisco proprietary. HSRP was used here due to Packet Tracer's simulation constraints with the 2911 platform, but the operational concept is identical. This distinction is worth knowing before any networking interview in the UAE market where mixed-vendor infrastructure is common.

### Why Centralised DHCP with Relay

The DHCP server sits on `172.16.1.0/28` behind R2 — a dedicated server subnet with 14 usable addresses, sized deliberately to accommodate future server additions (monitoring, backup, DNS) without requiring subnet reconfiguration. The first 10 addresses of every subnet across this topology are reserved for infrastructure — router interfaces, virtual IPs, management addresses — regardless of current utilisation. This prevents address conflicts as the network scales without requiring DHCP reconfiguration later.

DHCP broadcasts do not cross Layer 3 boundaries by default. The `ip helper-address` command on R1 and R2's subinterfaces converts the client's broadcast DHCP Discover into a unicast packet addressed to the server at `172.16.1.2`, which the server receives, matches to the correct pool based on the originating subnet, and responds with a full address assignment — IP, subnet mask, default gateway (the HSRP virtual IP), and DNS server.

The IT department segment (`172.16.2.0/24`) behind R1 also pulls addresses from the same centralised server via relay on R1's IT-facing interface. This is the correct enterprise approach — a dedicated DHCP server provides unified visibility across all segments. Local router DHCP (configuring pools directly on R1) would fragment that visibility and is more appropriate for small branch offices without dedicated server infrastructure.

### Why a /28 for the Server Subnet

A /30 provides only 2 usable addresses — sufficient for a point-to-point link but not for a server segment that will grow. A /24 is wasteful for a handful of servers. A /28 (14 usable addresses) strikes the right balance — enough room for a DHCP server, a future DNS server, a monitoring host, and a backup server, without burning a large address block on a segment that will never have hundreds of devices.

### OSPF and Default Route Injection

OSPF Area 0 runs across R1, R2, and R3. Each router advertises its directly connected networks, allowing full route propagation across the topology. R3 hosts a loopback (`8.8.8.8/32`) simulating a Google DNS destination — reachable from any PC in the topology via OSPF learned routes.

Both R1 and R2 have static default routes pointing to R3, injected into OSPF via `default-information originate`. This means the distribution switch and all downstream devices learn the default route dynamically — no manual static configuration needed on the switches or end hosts.

**A production note worth documenting:** In a real enterprise, the boundary between the organisation's network and the ISP would run BGP (Border Gateway Protocol), not OSPF. OSPF is an Interior Gateway Protocol designed for use within a single administrative domain. BGP is the protocol that routes traffic across the internet between organisations. OSPF was used here to maintain lab simplicity while still demonstrating default route injection and dynamic routing — but this distinction is fundamental to understanding where enterprise networks end and the internet begins.

R1 and R2 do not connect directly — they communicate via R3. This means R1 reaches R2's server subnet (`172.16.1.0/28`) via the path R1→R3→R2, reflected in R1's OSPF routing table as a cost-3 route. This is not a misconfiguration — it accurately reflects the topology where R3 is the common upstream point. In a production design, a direct R1-R2 link would be added for efficiency, and this is addressed in Lab 4.

### Null Route on R3

R3 drops all traffic destined for unknown addresses via a `ip route 0.0.0.0 0.0.0.0 null0` — a black hole route. Without this, unknown destination traffic would loop between R1/R2 and R3 indefinitely until TTL expiry, consuming bandwidth and generating misleading error messages. The null route causes R3 to immediately return an ICMP Destination Unreachable for any address it has no specific route to, providing instant, clean feedback to the sender. This is standard practice on any router acting as an internet boundary.

### Security Observation — IT Department Reachability

During testing, bidirectional reachability between VLAN 10/20 end hosts and the IT department segment was confirmed working. From a network functionality standpoint this is correct — routing is working as designed.

However, from a security standpoint, regular users should not be able to initiate connections to IT infrastructure. A compromised Guest PC reaching IT management interfaces is a real attack vector. In production, an Access Control List (ACL) on R1's IT-facing interface would permit IT-initiated traffic to user segments while blocking unsolicited inbound connections from VLAN 10 and VLAN 20 hosts. This was identified during the lab, researched, and deliberately deferred — ACL implementation will be explored in Lab 4 where the full enterprise topology provides the right context for a comprehensive access control policy.

---

## Verification

### DHCP Server Pools

Three pools configured on the centralised server — VLAN10_GUESTS, VLAN20_STAFF, and IT_DEPT. Each pool specifies the correct HSRP virtual IP as the default gateway, ensuring end hosts always point to the redundant virtual address rather than a physical router interface.

![DHCP Server Pools](DHCP_Server_Pools.png)

### PC IP Configuration — DHCP Assignment Confirmed

All PCs received addresses dynamically via DHCP. The default gateway on VLAN 10 and VLAN 20 PCs shows `192.168.10.254` and `192.168.20.254` respectively — the HSRP virtual IPs, not the physical router interfaces. This confirms the relay is working correctly and end hosts are pointing at the redundant virtual gateway.

![VLAN10 PC IP Config](VLAN10_PC_IPConfig.png)
![VLAN20 PC IP Config](VLAN20_PC_IPConfig.png)
![IT PC IP Config](IT_PC_IPConfig.png)

### Trunk Verification

`show interfaces trunk` on DSW1 confirms all four trunk interfaces are active, native VLAN 99, carrying only VLANs 10 and 20. Trunk pruning eliminates unnecessary broadcast flooding from irrelevant VLANs across the switching fabric.

![DSW1 Trunk Configurations](DSW1_Trunk_Configurations.png)

### HSRP Verification — Baseline

`show standby brief` on R1 confirms Active state for VLAN 10 (priority 150, preempt enabled) and Standby for VLAN 20 (priority 50). R2 confirms Active for VLAN 20 (priority 200, preempt enabled) and Standby for VLAN 10. Both routers are actively serving traffic — per-VLAN load balancing confirmed.

![R1 HSRP Brief](R1_HSRP_Brief.png)
![R2 HSRP Brief](R2_HSRP_Brief.png)

### OSPF Routing Table — R1

R1's OSPF routing table confirms 8.8.8.8 learned via R3, R2's server subnet (172.16.1.0) learned via R3 at cost 3 (reflecting the R1→R3→R2 path in the absence of a direct R1-R2 link), and all relevant subnets propagated correctly across the topology.

![R1 OSPF Routes](R1_OSPF_Routes.png)

---

## Baseline Connectivity

Three cross-segment ping tests were run to establish baseline reachability before any failure simulation:

**Cross-VLAN and IT reachability:** PC3 (VLAN 10) successfully pinged PC4 (VLAN 20) and PC1 (IT department), and PC4 successfully pinged PC1 — confirming inter-VLAN routing is functioning correctly across all three segments via R1 and R2's subinterfaces. This is the first lab in this series where cross-VLAN communication works end to end — Lab 2 intentionally left VLANs isolated at Layer 2. The routing capability introduced here is what makes full network communication possible.

![PING All Networks](PING_All_Networks.png)

**Internet reachability:** PC6 (VLAN 20) and PC3 (VLAN 10) successfully pinged 8.8.8.8 — the simulated internet destination on R3's loopback — confirming OSPF route propagation, default route injection via `default-information originate`, and end-to-end path through the dual-router architecture to R3.

![PING to 8.8.8.8](PING_to_8.8.8.8.png)

**Unknown destination behaviour:** A ping to `192.168.99.1` and `192.168.67.42` — addresses that exist nowhere in the topology — returned immediate `Destination host unreachable` responses from R3's interfaces (10.1.13.2 and 10.1.23.2). This confirms the null route on R3 is working correctly — traffic reaches R3, finds no specific route, hits null0, and R3 returns an ICMP unreachable instantly rather than allowing the packet to loop until TTL expiry.

![PING Unknown Destination](PING_Unknown_Destination.png)

---

## HSRP Failover — R1 Shutdown

R1's g0/0 interface was administratively shut down to simulate a complete router failure. The topology immediately reflected the link going down.

![R1 Interface Shutdown](R1_Interface_Shutdown.png)

### HSRP Re-election — R2 Takes Both VLANs

With R1 offline, R2 autonomously claimed the Active role for both VLAN 10 and VLAN 20. `show standby brief` on R2 confirms Active/local for both groups — VLAN 10 showing `unknown` standby (no remaining standby router) and VLAN 20 maintaining its existing active role. No manual intervention was required.

![R2 HSRP Active Both VLANs](R2_HSRP_Active_Both_VLANs.png)

---

## Post-Failover Connectivity — Network Self-Healed

All three ping tests were repeated immediately after R1's failure:

**Cross-segment reachability:** All inter-VLAN and IT department pings succeeded through R2 — confirming that R2's subinterfaces for both VLAN 10 and VLAN 20 are handling routing for all segments simultaneously.

![PING All Networks Post Failover](PING_All_Networks_Post_Failover.png)

**Internet reachability:** 8.8.8.8 remained reachable post-failover — traffic now exits via R2's direct link to R3 rather than R1's. OSPF reconverged automatically.

![PING to 8.8.8.8 Post Failover](PING_to_8.8.8.8_Post_Failover.png)

**Unknown destination behaviour:** Unknown destination pings continued returning immediate unreachable responses — now exclusively from `10.1.23.2` (R3's R2-facing interface), confirming all traffic is now flowing through R2 and the null route on R3 remains effective.

![PING Unknown Destination Post Failover](PING_Unknown_Destination_Post_Failover.png)

---

## What I Learned

Three insights from this lab go beyond individual protocol knowledge.

**The default gateway is the single point of failure in any routed network.** Every device on a subnet sends all non-local traffic to one IP address. If that IP address stops responding, the entire subnet loses external connectivity — regardless of how redundant the rest of the infrastructure is. HSRP exists to protect exactly this point. Understanding *why* the virtual IP matters more than *how* to configure it is what drives correct FHRP design.

**Centralised services require relay — and relay requires routing.** The DHCP server sits on a different subnet from every client it serves. DHCP broadcasts are Layer 2 — they don't cross routers. `ip helper-address` bridges that gap by converting broadcasts to unicast at the router, but only works if the router has a route to the server. This created an interesting dependency in this topology: R1 reaches the DHCP server via OSPF through R3, because R1 and R2 have no direct link. The relay worked correctly — but the path it took revealed the importance of verifying routing table completeness before assuming services will function.

**Security awareness belongs in network design, not as an afterthought.** Identifying that bidirectional reachability between user VLANs and the IT segment is a security exposure — and knowing that ACLs are the correct remediation — is more valuable than blindly confirming the pings work. A network that routes correctly but exposes management infrastructure to untrusted users is not a well-designed network. This will be addressed with ACL implementation in Lab 4.

---

## Design Notes

**On VRRP vs HSRP:** VRRP (RFC 5798) is the open standard equivalent of HSRP and would be the production choice in any multi-vendor environment. Packet Tracer's 2911 platform does not support VRRP in simulation — HSRP was used as a functionally equivalent alternative. The configuration concepts transfer directly between protocols.

**On OSPF at the ISP boundary:** Real enterprise-to-ISP connections run BGP, not OSPF. OSPF was used here to keep the lab self-contained while still demonstrating default route injection and dynamic routing. The null route on R3 partially simulates ISP behaviour — dropping unknown destinations immediately rather than propagating them further.

**On the R1-R2 path via R3:** The absence of a direct R1-R2 link means inter-router traffic traverses R3. In production this would be a design concern — R3 becomes a single point of failure for R1-R2 communication. A direct link between R1 and R2 would be added in a production design. This is addressed in Lab 4 where the full enterprise topology introduces direct inter-router connectivity.

---

## File Index

| File | Description |
|---|---|
| `Small_Office_Gateway_Redundancy_HSRP_OSPF_DHCP.pkt` | Packet Tracer source file — open to inspect full device configurations |
| `Topology.png` | Full topology — normal operation, all links active |
| `Topology_Post_Failover.png` | Topology — R1 offline, R2 handling all gateway duties |
| `DHCP_Server_Pools.png` | Centralised server pools — VLAN10_GUESTS, VLAN20_STAFF, IT_DEPT |
| `VLAN10_PC_IPConfig.png` | PC3 ipconfig — 192.168.10.x, gateway 192.168.10.254 (HSRP VIP) |
| `VLAN20_PC_IPConfig.png` | PC4 ipconfig — 192.168.20.x, gateway 192.168.20.254 (HSRP VIP) |
| `IT_PC_IPConfig.png` | PC1 ipconfig — 172.16.2.x, gateway 172.16.2.254 |
| `DSW1_Trunk_Configurations.png` | show interfaces trunk — native VLAN 99, pruned to VLAN 10 and 20 |
| `R1_HSRP_Brief.png` | R1 standby brief — Active VLAN 10 (pri 150), Standby VLAN 20 (pri 50) |
| `R2_HSRP_Brief.png` | R2 standby brief — Active VLAN 20 (pri 200), Standby VLAN 10 (pri 50) |
| `R1_OSPF_Routes.png` | R1 OSPF routing table — 8.8.8.8, server subnet, all internal routes |
| `PING_All_Networks.png` | Baseline — cross-VLAN and IT department reachability confirmed |
| `PING_to_8.8.8.8.png` | Baseline — internet reachability via OSPF default route |
| `PING_Unknown_Destination.png` | Baseline — unknown destination dropped at R3 via null route |
| `R1_Interface_Shutdown.png` | R1 g0/0 shutdown — failover trigger |
| `R2_HSRP_Active_Both_VLANs.png` | R2 post-failover — Active for both VLAN 10 and VLAN 20 |
| `PING_All_Networks_Post_Failover.png` | Post-failover — all cross-segment pings successful via R2 |
| `PING_to_8.8.8.8_Post_Failover.png` | Post-failover — internet still reachable via R2 |
| `PING_Unknown_Destination_Post_Failover.png` | Post-failover — null route still effective, unreachable from R3's R2-facing interface |.pkt` | Packet Tracer source file — open to inspect full device configurations |
| `Topology.png` | Full topology — normal operation, all links active |
| `Topology_Post_Failover.png` | Topology — R1 offline, R2 handling all gateway duties |
| `DHCP_Server_Pools.png` | Centralised server pools — VLAN10_GUESTS, VLAN20_STAFF, IT_DEPT |
| `VLAN10_PC_IPConfig.png` | PC3 ipconfig — 192.168.10.x, gateway 192.168.10.254 (HSRP VIP) |
| `VLAN20_PC_IPConfig.png` | PC4 ipconfig — 192.168.20.x, gateway 192.168.20.254 (HSRP VIP) |
| `IT_PC_IPConfig.png` | PC1 ipconfig — 172.16.2.x, gateway 172.16.2.254 |
| `DSW1_Trunk_Configurations.png` | show interfaces trunk — native VLAN 99, pruned to VLAN 10 and 20 |
| `R1_HSRP_Brief.png` | R1 standby brief — Active VLAN 10 (pri 150), Standby VLAN 20 (pri 50) |
| `R2_HSRP_Brief.png` | R2 standby brief — Active VLAN 20 (pri 200), Standby VLAN 10 (pri 50) |
| `R1_OSPF_Routes.png` | R1 OSPF routing table — 8.8.8.8, server subnet, all internal routes |
| `PING_All_Networks.png` | Baseline — cross-VLAN and IT department reachability confirmed |
| `PING_to_8.8.8.8.png` | Baseline — internet reachability via OSPF default route |
| `PING_Unknown_Destination.png` | Baseline — unknown destination dropped at R3 via null route |
| `R1_Interface_Shutdown.png` | R1 g0/0 shutdown — failover trigger |
| `R2_HSRP_Active_Both_VLANs.png` | R2 post-failover — Active for both VLAN 10 and VLAN 20 |
| `PING_All_Networks_Post_Failover.png` | Post-failover — all cross-segment pings successful via R2 |
| `PING_to_8.8.8.8_Post_Failover.png` | Post-failover — internet still reachable via R2 |
| `PING_Unknown_Destination_Post_Failover.png` | Post-failover — null route still effective, unreachable from R3's R2-facing interface |