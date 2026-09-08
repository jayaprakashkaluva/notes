# Computer Networking: From Bits on a Wire to Planet-Scale Systems

A staff-engineer's treatment of computer networking. The goal is not to catalogue protocols but to leave you with the **mechanisms**, the **numbers**, the **failure modes**, and the **design judgement** that separate someone who has read about TCP from someone who can be paged at 3 a.m. for "the service is slow", find the retransmissions in a packet capture, explain *why* the congestion window collapsed, and then argue in the design review about whether the fix is a kernel sysctl, a connection pool, a different load-balancer tier, or a different protocol.

Every section is built the same way: **the problem** → **the mechanism that solves it** → **the numbers you should have in your head** → **how it fails in production** → **what a senior engineer decides**.

Companion notes in this repo:

- [OSI and TCP/IP Models](osi-and-tcp-ip-models.md) — why networks are layered, encapsulation, and where the layer model leaks. This note assumes you have that vocabulary and goes deep on each layer's actual protocols.
- [What Happens When You Type a URL](what-happens-when-you-type-a-url.md) — the end-to-end narrative (keystroke → DNS → TCP/TLS → HTTP → CDN/origin → render) that walks through the mechanisms in this note in the order a real page load hits them.
- [Computer Architecture](../computer-architecture/computer-architecture.md) — §9 covers the NIC, DMA rings, and interrupts from the hardware side.
- [REST API Best Practices](../software-engineering/rest-api-best-practices.md) — application-layer design that sits on top of everything here.
- [Computer Networks (survey)](../computer-science/05-computer-networks.md) — the short version of this note.

---

## Table of Contents

1. [First Principles: What a Network Has to Solve](#1--first-principles-what-a-network-has-to-solve)
2. [The Physical Layer: Signals, Media, and the Speed of Light](#2--the-physical-layer-signals-media-and-the-speed-of-light)
3. [The Link Layer: Ethernet, Switching, ARP, and Wi-Fi](#3--the-link-layer-ethernet-switching-arp-and-wi-fi)
4. [The Network Layer: IP, Addressing, NAT, ICMP, and IPv6](#4--the-network-layer-ip-addressing-nat-icmp-and-ipv6)
5. [Routing: How the Internet Finds a Path](#5--routing-how-the-internet-finds-a-path)
6. [The Transport Layer: UDP and TCP in Depth](#6--the-transport-layer-udp-and-tcp-in-depth)
7. [QUIC: Rebuilding Transport in User Space](#7--quic-rebuilding-transport-in-user-space)
8. [Naming: DNS](#8--naming-dns)
9. [The Application Layer: HTTP/1.1, HTTP/2, HTTP/3, WebSockets, gRPC](#9--the-application-layer-http11-http2-http3-websockets-grpc)
10. [Security: TLS, PKI, Firewalls, VPNs, and Attacks](#10--security-tls-pki-firewalls-vpns-and-attacks)
11. [Performance Engineering: Latency, Bandwidth, and the Numbers That Matter](#11--performance-engineering-latency-bandwidth-and-the-numbers-that-matter)
12. [The Host Stack: Sockets, Event Loops, and the Kernel Path](#12--the-host-stack-sockets-event-loops-and-the-kernel-path)
13. [Datacenter and Cloud Networking](#13--datacenter-and-cloud-networking)
14. [Load Balancers, Proxies, and CDNs](#14--load-balancers-proxies-and-cdns)
15. [Wireless and Mobile: Why the Last Hop Is Different](#15--wireless-and-mobile-why-the-last-hop-is-different)
16. [Observability and Debugging: Method, Tools, and Failure Signatures](#16--observability-and-debugging-method-tools-and-failure-signatures)
17. [Design Principles and War Stories](#17--design-principles-and-war-stories)
18. [The Staff Engineer's Decision Checklist](#18--the-staff-engineers-decision-checklist)
19. [Mental Models Worth Keeping](#19--mental-models-worth-keeping)
20. [A Reading and Practice Path](#20--a-reading-and-practice-path)

---

# 1 — First Principles: What a Network Has to Solve

Strip away every protocol and a network has to solve exactly four problems. Every RFC you will ever read is an answer to one of them.

| Problem | Question | Answered by |
|---|---|---|
| **Naming and addressing** | How do I refer to the machine I want, and how does the network know where it is? | MAC addresses, IP addresses, DNS names, ports |
| **Routing / forwarding** | Given an address, which way do I send the packet at each hop? | Switch MAC tables, IP forwarding tables, OSPF, BGP |
| **Reliability over an unreliable substrate** | Links drop, corrupt, reorder, and duplicate. How do I get a correct byte stream anyway? | Checksums, sequence numbers, ACKs, retransmission, TCP |
| **Sharing a finite resource** | Many senders, one wire. Who gets to send, how much, and what happens when demand exceeds capacity? | CSMA, queues, congestion control, QoS, fair queueing |

Everything else — security, performance, discovery, load balancing — is a refinement layered on those four.

## 1.1 Packet switching versus circuit switching

The telephone network reserved a dedicated path (a *circuit*) for the duration of a call. Capacity was guaranteed and wasted: a silent caller still owned the wire. The internet's founding bet was **packet switching**: chop data into independently-addressed packets, let every packet fight for the wire, and accept that some will be delayed or dropped.

The payoff is **statistical multiplexing**. If 100 users each need the link 1% of the time, a circuit design needs 100 circuits; a packet design needs roughly one link plus a queue. The cost is that the network now needs queues, and queues mean variable delay and loss. The whole of congestion control (§6.7) exists to manage that trade.

## 1.2 The end-to-end argument

Saltzer, Reed, and Clark (1984) made the argument that shapes every layer boundary: **a function should be implemented in the layer that can do it completely and correctly, and that is usually the endpoints, not the middle.** Reliable delivery is the canonical example. Even if every link retransmitted lost frames, a router could still crash and lose a packet, so the endpoints must check anyway. Once the endpoints check, link-level reliability is only an optimisation, not a correctness requirement.

This is why IP is deliberately dumb ("best effort"), why TCP lives on hosts, why the internet could add TLS, HTTP/2, and QUIC without touching a single router, and why the middleboxes that violate the argument (NAT, deep-packet-inspection firewalls, TCP-optimising proxies) are the main source of protocol ossification (§7.1, §17.3).

## 1.3 Postel's robustness principle and its critics

"Be conservative in what you send, liberal in what you accept." It got the internet interoperating in the 1980s. Forty years on, it is widely blamed for security bugs (parsers that accept garbage become attack surfaces) and for ossification (senders can never fix a quirk because receivers depend on it). Modern protocol design (TLS 1.3, QUIC, HTTP/2's strict framing) leans toward **fail loudly on malformed input**. Know both positions; the right answer depends on whether you are building a 1985 mail relay or a 2026 TLS stack.

## 1.4 The physical limits nobody can engineer away

- **Propagation delay** is bounded by the speed of light in the medium. Nothing you do in software changes the 5 µs per kilometre cost of fibre.
- **Bandwidth** is bounded by Shannon's theorem: capacity = B · log₂(1 + S/N). More signal, less noise, or more spectrum; there is no fourth option.
- **Queueing delay** grows without bound as utilisation approaches 100%. A link at 95% load has, by the M/M/1 model, twenty times the average queueing delay of a link at 50%. This is why "we still have headroom" at 90% utilisation is wrong.

Those three lines explain most latency problems you will ever meet.

---

# 2 — The Physical Layer: Signals, Media, and the Speed of Light

## 2.1 Bits become signals

A bit is an abstraction; a wire carries a voltage, a fibre carries light, and air carries a modulated radio wave. **Line coding** maps bits to signal changes so the receiver can recover the clock (long runs of identical bits would otherwise drift). Ethernet uses codes like 4B/5B, 8b/10b, and 64b/66b: the "66" in 64b/66b is why 10 Gigabit Ethernet actually signals at 10.3125 Gbaud.

**Modulation** packs more bits per symbol by varying amplitude, phase, or frequency. 256-QAM carries 8 bits per symbol; it needs a clean channel. Wi-Fi and cellular radios continuously renegotiate modulation as signal quality changes, which is one reason wireless throughput is so variable.

## 2.2 Media

| Medium | Typical use | Reach | Notes |
|---|---|---|---|
| Twisted-pair copper (Cat 5e/6/6a) | Office, home, top-of-rack | 100 m | 1–10 Gbps; cheap; electrical noise sensitive |
| Direct-attach copper (DAC) / twinax | Inside a rack | 1–7 m | 10–100+ Gbps; cheapest short-reach option |
| Multi-mode fibre (MMF) | Inside a building / datacenter | up to ~400 m | 850 nm lasers; cheap optics |
| Single-mode fibre (SMF) | Between buildings, metro, subsea | 10 km – thousands of km with amplifiers | 1310/1550 nm; one strand can carry many wavelengths (DWDM) |
| Wi-Fi (802.11) | Last 30 m | tens of metres | Shared medium, half-duplex per channel, lossy |
| Cellular (4G/5G) | Mobile | km | Licensed spectrum; radio state machine adds latency (§15) |
| Satellite (GEO / LEO) | Remote | global | GEO: ~600 ms RTT; LEO (Starlink): ~25–60 ms |

A modern hyperscale datacenter is almost entirely fibre above the top-of-rack switch, with DAC inside the rack. Long-haul is single-mode fibre with dense wavelength-division multiplexing: dozens of colours per strand, each at 100–800 Gbps.

## 2.3 The numbers to memorise

| Quantity | Value |
|---|---|
| Speed of light in vacuum | ~300,000 km/s |
| Speed of light in fibre | ~200,000 km/s (refractive index ≈ 1.5) → **5 µs per km, 1 ms per 200 km** |
| New York ↔ London, straight fibre, one way | ~28 ms (5,570 km). Real RTT ≈ 65–75 ms because cables are not straight |
| New York ↔ Sydney, one way | ~80 ms theoretical; real RTT ≈ 200–250 ms |
| Within one datacenter region, RTT | 100–500 µs |
| Across availability zones in the same region | ~1–2 ms |
| Serialisation of a 1500-byte frame at 1 Gbps / 10 Gbps / 100 Gbps | 12 µs / 1.2 µs / 0.12 µs |

**Propagation dominates everything at distance.** A 100 Gbps transatlantic link and a 1 Gbps one have the same 70 ms RTT. Bandwidth only helps if you can keep the pipe full, which is the bandwidth-delay product problem in §11.2.

## 2.4 Failures at this layer

Bad optics, dirty fibre connectors, bent cables, and duplex mismatches produce **CRC errors and frame drops** that show up as unexplained TCP retransmissions on one path. `ethtool -S eth0` and switch interface counters (`rx_crc_errors`, `input errors`) are the diagnostic. If one host out of a hundred is slow to one destination and the packets are being *corrupted* rather than *dropped by a queue*, suspect the physical path.

---

# 3 — The Link Layer: Ethernet, Switching, ARP, and Wi-Fi

The link layer moves **frames** between directly connected machines. Its address is the **MAC address** (48 bits, burned into the NIC, first 24 bits identify the vendor), and its scope is a single **broadcast domain**: the set of machines that receive a frame sent to `ff:ff:ff:ff:ff:ff`.

## 3.1 Ethernet frame

```
| Preamble/SFD (8) | Dst MAC (6) | Src MAC (6) | [802.1Q tag (4)] | EtherType (2) | Payload (46–1500) | FCS/CRC (4) |
```

- **EtherType** says what is inside: `0x0800` IPv4, `0x86DD` IPv6, `0x0806` ARP, `0x8100` VLAN-tagged.
- **MTU (Maximum Transmission Unit)** = 1500 bytes of payload by default. This number, fixed in 1980 for 10 Mbps coax, still constrains every packet on the internet. Jumbo frames (9000 bytes) are common inside datacenters and never cross the public internet.
- **FCS** is a CRC-32. A corrupted frame is silently dropped; the link layer does not retransmit (end-to-end argument). TCP will notice the gap.
- Minimum frame is 64 bytes, a relic of collision detection on shared coax.

## 3.2 From a shared wire to a switched network

Original Ethernet was a shared coaxial cable using **CSMA/CD** (Carrier Sense Multiple Access with Collision Detection): listen, transmit, and if you hear a collision, back off for a random time. Throughput collapsed under load.

**Switches** ended that. A switch is a multi-port bridge with a **MAC learning table**:

1. A frame arrives on port 3 from MAC `A`. The switch records "`A` is on port 3".
2. It looks up the destination MAC. Known → forward out that one port. Unknown or broadcast → **flood** out every other port.
3. Entries age out (typically 300 s).

Every port is a separate collision domain, links are full duplex, and collisions no longer exist on switched Ethernet. What remains is the **broadcast domain**: broadcasts and unknown-unicast flooding still reach every port, which is why a flat network with 10,000 hosts melts down from ARP traffic alone. That is the problem VLANs and IP subnets solve.

## 3.3 VLANs, trunks, and spanning tree

- **VLAN (802.1Q)** inserts a 4-byte tag carrying a 12-bit VLAN ID (4094 usable). A switch keeps a separate MAC table per VLAN, turning one physical switch into many logical broadcast domains. A **trunk** port carries tagged frames for many VLANs between switches; an **access** port strips tags and belongs to one VLAN.
- **Spanning Tree Protocol (STP / RSTP)** exists because Ethernet frames have no TTL. A physical loop between switches would circulate a broadcast forever, and the flood amplifies it: a **broadcast storm**. STP elects a root bridge and disables redundant links so the topology is a tree. The costs are that half your links sit idle and that a reconvergence can black-hole traffic for seconds. Modern datacenters avoid STP entirely by routing at layer 3 down to the top-of-rack switch (§13.1).
- **Link aggregation (LACP, 802.3ad)** bonds several physical links into one logical link. Traffic is hashed per flow across members, so one TCP flow never exceeds the speed of one member link.

## 3.4 ARP: gluing layer 3 to layer 2

IP knows the next hop's IP address; Ethernet needs its MAC. **ARP (Address Resolution Protocol)** broadcasts "who has 10.0.0.1?" and the owner replies unicast "10.0.0.1 is at `aa:bb:cc:...`". Hosts cache the answer (Linux: `ip neigh`), typically for a minute or so, with reachability confirmation.

Facts worth knowing:

- A host only ARPs for addresses **on its own subnet**. For anything else it ARPs for the **default gateway** and sends the frame there. Get the subnet mask wrong and the host will ARP for an address that will never answer: the classic "can ping the gateway but nothing beyond it" symptom in reverse.
- **Gratuitous ARP** announces "I am 10.0.0.1" unsolicited. Used for failover (VRRP/keepalived moves a virtual IP by sending gratuitous ARP so switches relearn the port) and, maliciously, for **ARP spoofing** on untrusted LANs: any host can claim to be the gateway and become a man-in-the-middle. This is why hotel Wi-Fi without TLS is dangerous, and why cloud VPCs do not run real ARP at all: the hypervisor answers.
- IPv6 replaces ARP with **Neighbour Discovery (NDP)** over ICMPv6 multicast (§4.7).

## 3.5 Wi-Fi (802.11)

Wi-Fi is Ethernet's shared-medium past, in the air. Radios cannot detect collisions while transmitting (their own signal drowns out everything), so 802.11 uses **CSMA/CA (Collision Avoidance)**: listen, wait a random back-off, transmit, and require a **link-layer ACK** for every unicast frame. A frame without an ACK is retransmitted by the radio, up to a limit, before the loss is exposed to IP.

Consequences:

- **Wi-Fi is half duplex per channel and shared** among every client on the access point. Twenty laptops on one AP share airtime, and a client with a weak signal negotiates a slower modulation and consumes more airtime per byte, slowing everyone.
- **Latency is jittery** (1–100+ ms) because of back-off, retries, and power-saving sleep cycles. TCP's RTT estimator copes, but real-time applications feel it.
- **Link-layer retransmission hides loss from TCP** but adds delay spikes. It is an instance of the end-to-end argument's "optimisation, not correctness" role.
- Generations: 802.11n (Wi-Fi 4), ac (Wi-Fi 5), ax (Wi-Fi 6/6E, adds OFDMA so the AP can schedule many clients in one transmission), be (Wi-Fi 7, 320 MHz channels, multi-link).
- Security: WPA2 (AES-CCMP) is the floor; WPA3 adds a Dragonfly handshake resistant to offline dictionary attacks. WEP and WPA-TKIP are broken; treat them as open.

---

# 4 — The Network Layer: IP, Addressing, NAT, ICMP, and IPv6

The network layer's job is to deliver a **packet** from any host to any other host across many links, with no guarantee of delivery, order, or integrity of the payload. That deliberate weakness is the whole point: a dumb network scales.

## 4.1 The IPv4 header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------+-------+---------------+-------------------------------+
|Ver=4  |  IHL  |  DSCP   | ECN |         Total Length          |
+-------+-------+---------------+-----+-------------------------+
|         Identification        |Flags|    Fragment Offset      |
+---------------+---------------+-----+-------------------------+
|      TTL      |   Protocol    |       Header Checksum         |
+---------------+---------------+-------------------------------+
|                       Source Address                          |
+---------------------------------------------------------------+
|                    Destination Address                        |
+---------------------------------------------------------------+
|                    Options (rare)  ...                        |
```

Fields that matter operationally:

- **TTL (Time To Live)** is a hop count, decremented by every router; at zero the packet is dropped and an ICMP Time Exceeded goes back. It prevents routing loops from circulating packets forever and is the mechanism behind `traceroute`. Linux starts at 64, Windows at 128.
- **Protocol** identifies the payload: 1 ICMP, 6 TCP, 17 UDP, 47 GRE, 50 ESP (IPsec), 89 OSPF.
- **DSCP** (Differentiated Services) marks QoS class; honoured inside enterprises and datacenters, mostly stripped on the public internet. **ECN** (Explicit Congestion Notification) lets routers mark instead of drop (§6.7.5).
- **Flags / Fragment Offset** support fragmentation: if a packet exceeds the next link's MTU and the **DF (Don't Fragment)** bit is clear, the router splits it. Fragmentation is a disaster in practice (any lost fragment loses the whole packet; middleboxes drop fragments; reassembly is an attack surface), so modern stacks set DF and rely on Path MTU Discovery (§4.4).
- **Header checksum** covers only the header. Payload integrity is TCP/UDP's job.

## 4.2 Addresses, CIDR, and subnetting

An IPv4 address is 32 bits, written as four decimal octets. **CIDR (Classless Inter-Domain Routing)** notation `10.20.0.0/22` says "the first 22 bits are the network; the remaining 10 bits identify hosts within it."

Worked example, `10.20.0.0/22`:

| | |
|---|---|
| Mask | `255.255.252.0` (22 ones) |
| Block size | 2^(32−22) = 1024 addresses |
| Range | `10.20.0.0` – `10.20.3.255` |
| Network address | `10.20.0.0` (all host bits zero) |
| Broadcast | `10.20.3.255` (all host bits one) |
| Usable hosts | 1022 on a classic LAN; 1019 in AWS, which reserves the first four and the last |
| Next /22 | `10.20.4.0/22` |

Splitting a /22 into four /24s gives `10.20.0.0/24`, `10.20.1.0/24`, `10.20.2.0/24`, `10.20.3.0/24`. Aggregating the other way ("supernetting") is how the global routing table stays at ~1 million prefixes instead of one per organisation.

Reserved ranges you will meet constantly:

| Range | Purpose |
|---|---|
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Private (RFC 1918). Never routed on the internet; behind NAT |
| `100.64.0.0/10` | Carrier-grade NAT shared space (RFC 6598). Also used by Tailscale |
| `127.0.0.0/8` | Loopback. `127.0.0.1` is the usual one; the whole /8 works |
| `169.254.0.0/16` | Link-local (APIPA). `169.254.169.254` is the cloud instance metadata service |
| `0.0.0.0` | "Any address" when binding; "default route" as `0.0.0.0/0` |
| `224.0.0.0/4` | Multicast |

**The default route** `0.0.0.0/0` matches everything; longest-prefix match (§5.1) ensures more specific routes win. A host's routing table is normally just "my subnet → direct" plus "everything → gateway".

## 4.3 NAT: the hack that ate the internet

IPv4 has 4.3 billion addresses and ran out in 2011. **Network Address Translation** let one public address front an entire private network:

1. Host `192.168.1.10:51000` sends to `93.184.216.34:443`.
2. The NAT router rewrites the source to its public `203.0.113.5:40001` and records the mapping `(192.168.1.10, 51000) ↔ (203.0.113.5, 40001)` in a **connection tracking table**.
3. The reply to `203.0.113.5:40001` is rewritten back to `192.168.1.10:51000`.

That is NAPT (port translation), the form everyone means by "NAT". It has enormous consequences:

- **Inbound connections are impossible** unless a mapping exists. That killed the peer-to-peer internet and created the industry of **hole punching** (STUN, TURN, ICE) that WebRTC and every video-call app depend on. A "symmetric NAT" allocates a different public port per destination and defeats most hole punching; you then relay through TURN.
- **State lives in the middle**, violating the end-to-end argument. NAT tables have idle timeouts (often 30–300 s for UDP, minutes to hours for TCP). A long-idle TCP connection gets silently dropped from the table; the next packet is discarded, and the application sees a timeout or a reset. This is why TCP keepalives and application heartbeats exist (§6.9).
- **CGNAT (Carrier-Grade NAT)** adds a second layer at the ISP. Thousands of subscribers share one public IP, so IP-based rate limiting and abuse blocking punish innocents, and geolocation is fuzzy.
- Cloud VPCs implement NAT as "NAT Gateway" for outbound-only private subnets. It bills per byte and has its own port-exhaustion limit (~55,000 concurrent connections per destination per NAT GW IP on AWS), which teams discover when a fleet of workers all hit one external API.

## 4.4 ICMP: the network's control channel

**ICMP (Internet Control Message Protocol)** carries errors and diagnostics inside IP. It is not optional plumbing; blocking it breaks the internet in subtle ways.

| Type/Code | Meaning | Why you care |
|---|---|---|
| 8 / 0 | Echo request / reply | `ping`. Measures RTT and reachability, not whether your service works |
| 11 | Time Exceeded (TTL hit zero) | `traceroute` sends packets with TTL 1, 2, 3… and reads who replies |
| 3 / 3 | Destination unreachable: port unreachable | How UDP tells you nobody is listening; how Linux `traceroute` knows it reached the end |
| 3 / 4 | Fragmentation needed and DF set | **Path MTU Discovery**. Carries the next-hop MTU. Blocking it causes the MTU black hole below |
| 5 | Redirect | "Use this other router". Mostly disabled for security |

**The MTU black hole**, the single most frequent "TLS handshake works but large responses hang" bug: a VPN or tunnel somewhere lowers the path MTU to, say, 1400. The sender emits 1500-byte packets with DF set. A router must drop them and send ICMP type 3 code 4. If a firewall blocks ICMP, the sender never learns, keeps retransmitting the same full-size segment, and the connection stalls forever. Small packets (handshakes, small requests) go through; anything large dies. Fixes: allow ICMP, **clamp TCP MSS** on the tunnel (`iptables -t mangle ... --clamp-mss-to-pmtu`), or rely on Packetisation-Layer PMTUD (RFC 4821) which probes with TCP itself. QUIC does its own PMTUD for exactly this reason.

## 4.5 IPv6

IPv6 is not "IPv4 with longer addresses" though that is the headline. 128-bit addresses, written as eight 16-bit hex groups with `::` compressing one run of zeros: `2001:db8::1`.

| Prefix | Meaning |
|---|---|
| `2000::/3` | Global unicast (the public internet) |
| `fe80::/10` | Link-local. Every interface auto-assigns one; used for NDP and routing protocols |
| `fc00::/7` (in practice `fd00::/8`) | Unique local (private, like RFC 1918) |
| `::1` | Loopback |
| `ff00::/8` | Multicast. There is **no broadcast** in IPv6 |
| `::ffff:a.b.c.d` | IPv4-mapped, used by dual-stack sockets |

What actually changed:

- **Fixed 40-byte header, no checksum, no router fragmentation.** Routers never fragment; the minimum link MTU is 1280 and senders must do PMTUD. Options moved into chained extension headers.
- **No ARP, no DHCP required.** Neighbour Discovery (ICMPv6) resolves addresses; **SLAAC** (Stateless Address Autoconfiguration) lets a host build its own address from the router-advertised /64 prefix plus an interface identifier. Privacy extensions randomise that identifier so you are not tracked by MAC.
- **/64 per subnet, always.** A single home gets a /56 or /48. Address scarcity is over, which means **no NAT** and a return to end-to-end reachability (with firewalls doing the job NAT accidentally did).
- **Every modern OS prefers IPv6 when a AAAA record exists** and uses **Happy Eyeballs (RFC 8305)**: race an IPv6 and an IPv4 connection with a ~250 ms head start for v6, use whichever connects first. That algorithm is why broken IPv6 usually manifests as "everything is 250 ms slower" rather than "nothing works".

Adoption sits around 45–50% of Google traffic globally as of the mid-2020s. Mobile carriers are heavily v6 (T-Mobile US is v6-only internally with **464XLAT** translating for legacy apps); enterprise networks and many cloud deployments still lag. As an engineer: bind to `::` with dual-stack sockets, never assume an IP fits in 32 bits, never parse addresses with a regex, and test with a v6-only client at least once.

## 4.6 Multicast and anycast

- **Multicast**: one packet, many receivers, replicated by the network. Used inside enterprises (IPTV, market data feeds where microseconds matter), by OSPF and NDP, and almost never across the public internet.
- **Anycast**: the *same* IP prefix is announced from many locations; BGP routes each client to the topologically nearest. DNS root servers, CDNs, and `1.1.1.1`/`8.8.8.8` are anycast. Works beautifully for stateless UDP and short TCP; long TCP connections can break if routing shifts mid-connection, which is one motivation for QUIC connection IDs (§7.3).

---

# 5 — Routing: How the Internet Finds a Path

Forwarding is the per-packet data-plane act of looking up a destination and sending the packet out one interface. Routing is the control-plane act of building that table. They run at wildly different speeds: forwarding at hundreds of millions of packets per second in ASIC hardware, routing at "converges in seconds to minutes" in software.

## 5.1 Longest-prefix match

A forwarding table is a set of `(prefix, next-hop, interface)` entries. For each packet, the router picks the entry with the **longest prefix that matches** the destination. `10.20.1.0/24` beats `10.20.0.0/22` beats `0.0.0.0/0`. Hardware implements this with TCAM (ternary content-addressable memory) or specialised tries; the size and cost of TCAM is why the ~1M-entry global routing table is a real constraint for ISP routers.

Show your own: `ip route` (Linux), `route print` (Windows), `netstat -rn` (macOS). Trace a decision: `ip route get 8.8.8.8`.

## 5.2 The two-tier architecture: IGPs and BGP

The internet is roughly 75,000 **Autonomous Systems (ASes)**, each an organisation running its own network under one routing policy: an ISP, a cloud provider, a university, a large enterprise. Routing is split accordingly:

- **Inside an AS**: an **Interior Gateway Protocol** finds *shortest* paths. Correctness and fast convergence matter; policy does not.
- **Between ASes**: **BGP** chooses paths by *policy* (money, contracts, politics). Shortest path is a tie-breaker at best.

## 5.3 Interior routing: OSPF and IS-IS

**Link-state** protocols (OSPF, IS-IS) have every router flood a description of its own links to every other router. Each router then holds an identical map of the whole area and runs **Dijkstra** locally to compute shortest paths. Convergence after a link failure is fast (sub-second with tuning) because everyone recomputes from the same map. Areas limit flooding scope in large networks.

**Distance-vector** (RIP, and EIGRP in its hybrid way) instead has each router tell neighbours "I can reach X at cost N". Simpler, slower to converge, prone to count-to-infinity loops. RIP is dead outside classrooms; the datacenter runs OSPF or, more commonly now, BGP itself as an IGP (§13.1).

## 5.4 BGP: the protocol that holds the internet together

**BGP-4** (RFC 4271) is a **path-vector** protocol running over TCP port 179 between pairs of routers ("peers" or "neighbours"). Each router announces prefixes with an **AS_PATH**, the list of ASes the announcement has traversed. A router that sees its own AS number in the path rejects the route: that is loop prevention.

### Route selection

When several paths to a prefix exist, a BGP router chooses, in order:

1. Highest **LOCAL_PREF** (set by your own policy: "prefer the cheap transit provider").
2. Shortest **AS_PATH**.
3. Lowest ORIGIN.
4. Lowest **MED** (a hint from the neighbouring AS about which of its entry points it prefers).
5. eBGP-learned over iBGP-learned.
6. Lowest IGP cost to the next hop ("hot-potato": get it off my network as cheaply as possible).
7. Oldest route, then lowest router ID.

Notice that step 1 is pure policy and beats everything topological. The internet routes on money first.

### Business relationships

- **Transit**: you pay a larger ISP to carry your traffic to the whole internet. They announce everything to you and announce your prefixes to everyone.
- **Peering**: two networks exchange traffic between their own customers for free (settlement-free) because it is cheaper than paying transit. Happens at **Internet Exchange Points (IXPs)** like DE-CIX, AMS-IX, and LINX, or over private cross-connects.
- The **Gao–Rexford rule** (valley-free routing) follows: you export customer routes to everyone, but peer and provider routes only to customers. Get this wrong and you become free transit for two giant ISPs, which is exactly what a **route leak** is.

### How BGP fails

- **Hijacks.** BGP has no built-in authentication of who owns a prefix. In February 2008 Pakistan Telecom, trying to block YouTube domestically, announced a more-specific `208.65.153.0/24` covering YouTube's space. Its upstream PCCW propagated it, and for two hours most of the world's YouTube traffic went to Pakistan and was dropped. Longest-prefix match made the hijack win globally. Deliberate hijacks for cryptocurrency theft and traffic interception have followed.
- **Route leaks.** In June 2019 a small Pennsylvania ISP leaked routes learned from one provider to another via a misconfigured BGP optimiser; Verizon accepted them, and Cloudflare, Amazon, and others lost a chunk of traffic for hours.
- **Withdrawal cascades.** On 4 October 2021 a Facebook maintenance command accidentally withdrew the backbone routes connecting its datacenters. Its DNS servers were designed to withdraw their own BGP announcements if they lost contact with the datacenters (so a broken DNS site would stop attracting traffic). Every DNS server did so simultaneously. Facebook, Instagram, and WhatsApp vanished from the internet for six hours, and the internal tools needed to fix it were behind the same DNS. Lesson: a well-intentioned health check with a correlated failure mode is an outage amplifier.

### Defences

- **RPKI (Resource Public Key Infrastructure)**: cryptographically signed "AS X may originate prefix P" statements (ROAs). Routers doing **Route Origin Validation** drop announcements from the wrong origin AS. Now covering a majority of routed prefixes; stops accidental origin hijacks, not path manipulation.
- **Prefix filtering** at peering edges, **max-prefix limits**, and **IRR** databases. Mundane, and they prevent most leaks.
- **BGPsec** signs the whole path; barely deployed because of router CPU cost.
- **ASPA** (Autonomous System Provider Authorization) is the emerging answer to route leaks.

## 5.5 Anycast, ECMP, and traffic engineering

- **ECMP (Equal-Cost Multi-Path)**: when several next hops tie, hash the 5-tuple (src IP, dst IP, protocol, src port, dst port) and spread flows across them. It keeps a flow on one path (so TCP does not see reordering) at the price of **elephant flows**: one huge transfer occupies one path while the others idle. Datacenter fabrics live and die by ECMP.
- **MPLS (Multiprotocol Label Switching)**: routers push a short label on the packet and forward by label instead of IP lookup. ISPs use it for traffic engineering (steer a class of traffic down a chosen path), VPNs (L3VPN separates customer routing tables), and fast reroute. Segment Routing is its modern successor.
- **Traffic engineering** at hyperscalers (Google's B4, Meta's Express Backbone) treats the WAN as a centrally optimised resource, computing paths in software (SDN) rather than trusting distributed shortest-path.

---

# 6 — The Transport Layer: UDP and TCP in Depth

IP delivers packets to a *host*. The transport layer delivers them to a *process*, identified by a **port**, and optionally adds reliability. A connection is identified by the **5-tuple** `(protocol, src IP, src port, dst IP, dst port)`; this is what the kernel hashes to find the socket, what NAT tracks, what ECMP hashes, and what a firewall's state table stores.

Ports 0–1023 are "well known" and need root to bind on Unix; 1024–49151 are registered; **ephemeral** ports (Linux default 32768–60999, ~28,000 of them) are what a client borrows for the source side. That count matters (§6.8).

## 6.1 UDP: IP plus ports

```
| Src Port (2) | Dst Port (2) | Length (2) | Checksum (2) | Payload |
```

Eight bytes. No connection, no ordering, no retransmission, no congestion control, message boundaries preserved. You use UDP when:

- Retransmission is worse than loss (voice, video, games: a late frame is useless).
- You are building your own transport (QUIC, WireGuard, RTP, custom RPC in HFT).
- One request, one response, tiny (DNS, NTP, SNMP, DHCP).
- You need multicast or broadcast.

The rule that comes with it: **if you send UDP at scale, you own congestion control.** An application that blasts UDP without backing off can collapse a network, and the IETF now requires new UDP protocols to be "TCP-friendly". A UDP server also must handle **amplification abuse**: if a 60-byte request yields a 3,000-byte reply, attackers will spoof victims' source addresses at you (§10.7).

## 6.2 The TCP header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-------------------------------+
|          Source Port          |       Destination Port        |
+-------------------------------+-------------------------------+
|                        Sequence Number                        |
+---------------------------------------------------------------+
|                    Acknowledgment Number                      |
+-------+-----+-+-+-+-+-+-+-+-+-+-------------------------------+
| Data  |Rsvd |C|E|U|A|P|R|S|F|                               |
| Offset|     |W|C|R|C|S|S|Y|I|            Window             |
|       |     |R|E|G|K|H|T|N|N|                               |
+-------+-----+-+-+-+-+-+-+-+-+-+-------------------------------+
|           Checksum            |         Urgent Pointer        |
+-------------------------------+-------------------------------+
|   Options (MSS, SACK, Timestamps, Window Scale) ... padding   |
```

- **Sequence number**: the byte offset of the first payload byte in this segment, relative to a random Initial Sequence Number (random to defeat off-path injection).
- **Acknowledgment number**: "I have received every byte up to but not including this one." ACKs are **cumulative**.
- **Window**: how many more bytes the sender of this segment is willing to receive (flow control). Sixteen bits, so 64 KB max unless the **window scale** option multiplies it, which every modern stack negotiates.
- Flags: **SYN** open, **FIN** close, **RST** abort, **ACK**, **PSH** deliver now, **URG** (dead), **ECE/CWR** ECN signalling.
- Options: **MSS** (max segment size the sender can receive, typically 1460 = 1500 − 20 − 20), **SACK permitted**, **Timestamps** (for RTT measurement and PAWS wrap protection), **Window Scale**.

Overhead: 20 bytes IP + 20 bytes TCP + 14 Ethernet + 4 FCS = 58 bytes per 1500-byte frame, so the best case is ~94.9% payload efficiency on Ethernet, lower with options and VLAN/VXLAN.

## 6.3 Connection establishment: the three-way handshake

```
Client                                   Server
  |  SYN  seq=x  [MSS, WS, SACK, TS]        |
  |---------------------------------------->|   server: SYN_RECV, entry in SYN queue
  |  SYN+ACK seq=y ack=x+1 [options]        |
  |<----------------------------------------|
  |  ACK  seq=x+1 ack=y+1                   |
  |---------------------------------------->|   server: ESTABLISHED, moved to accept queue
  |  (data may ride on this ACK)            |
```

One full RTT before the client can send data, and the server cannot send until it has the third packet. Both sides agree on options and ISNs here; nothing about them can change later, which is why a connection cannot "upgrade" to window scaling after the fact.

**Two queues on the server** (Linux):

- The **SYN queue** holds half-open connections awaiting the third ACK. Sized by `net.ipv4.tcp_max_syn_backlog`.
- The **accept queue** holds fully established connections waiting for the application to call `accept()`. Sized by `min(listen(backlog), net.core.somaxconn)` (`somaxconn` default 4096 since Linux 5.4, 128 before, which bit many people).

If the accept queue is full, Linux by default **drops the third ACK** and lets the client retransmit; the client thinks it is connected and sends data into the void until it times out. `ss -ltn` shows `Recv-Q` (current accept-queue depth) vs `Send-Q` (limit); `nstat -az TcpExtListenOverflows` counts overflows. A slow event loop that cannot `accept()` fast enough manifests as **client connection timeouts, not errors**, which sends people hunting in the wrong place.

**SYN flood** attacks fill the SYN queue with spoofed SYNs. **SYN cookies** defend by encoding the connection state cryptographically in the server's ISN and keeping no state until the third ACK arrives with a valid cookie. Linux enables them automatically when the SYN queue overflows.

**TCP Fast Open (TFO)** lets a returning client send data in the SYN using a cookie from a previous connection, saving an RTT. Middleboxes break it often enough that it never went mainstream; QUIC's 0-RTT is the design that won.

## 6.4 Reliable delivery: sequence numbers, ACKs, and retransmission

The sender keeps every unacknowledged byte in a buffer and a timer. The receiver ACKs the highest contiguous byte received. Two ways to detect loss:

1. **Retransmission timeout (RTO).** Computed from smoothed RTT and its variance: `RTO = SRTT + 4·RTTVAR` (Jacobson/Karels), clamped to a minimum (Linux: 200 ms) and starting at 1 s before any RTT sample. On timeout, retransmit and **double the RTO** (exponential backoff). A timeout is treated as severe congestion: cwnd resets to one segment. On a 1 ms datacenter RTT, the 200 ms minimum RTO is 200× the RTT, which is why datacenter tail latency is so sensitive to any loss, and why Google and others patch the minimum down.
2. **Fast retransmit.** Three duplicate ACKs for the same byte mean "a later segment arrived, this one did not". Retransmit immediately without waiting for the timer, and only halve cwnd (fast recovery) instead of resetting it.

**SACK (Selective Acknowledgment)** lets the receiver say "I have 1000–2000 and 3000–4000", so the sender retransmits exactly the 2000–3000 hole instead of everything after 2000. Universal today. **RACK** (Recent ACK, Linux default since 4.x) uses per-segment send timestamps to detect loss more robustly than counting duplicate ACKs, and **Tail Loss Probe** sends a probe after a short idle so the loss of the *last* segment of a request (which can never trigger dup-ACKs) is caught in ~2 RTTs rather than a full RTO. Most short web requests that lose a packet lose the tail; TLP was a measurable win for page-load latency.

## 6.5 Flow control: the receive window

The receiver advertises how much buffer it has free (`rwnd`). The sender never has more than `rwnd` bytes in flight. If the application stops reading, the buffer fills, `rwnd` shrinks to zero, the sender stops, and periodically sends **zero-window probes**. In a capture, a stream of "TCP ZeroWindow" followed by "Window Update" means **the receiving application is the bottleneck**, not the network. This is back-pressure propagating across the wire, and it is why a slow consumer eventually stalls a fast producer through every proxy in between.

Linux auto-tunes receive buffers (`net.ipv4.tcp_rmem`) up to a maximum; if the maximum is too small for the bandwidth-delay product, throughput caps (§11.2).

## 6.6 Connection teardown and TIME_WAIT

```
Active closer                          Passive closer
  |  FIN                                   |
  |--------------------------------------->|  CLOSE_WAIT (app must still call close())
  |  ACK                                   |
  |<---------------------------------------|  FIN_WAIT_2
  |  FIN                                   |
  |<---------------------------------------|  LAST_ACK
  |  ACK                                   |
  |--------------------------------------->|  closed
 TIME_WAIT for 2·MSL (60 s on Linux)
```

Each direction closes independently (a "half-close": you can send FIN and still receive). The side that closes first enters **TIME_WAIT** for two Maximum Segment Lifetimes so that (a) a lost final ACK can be re-sent when the peer retransmits its FIN, and (b) old delayed segments die before a new connection reuses the same 5-tuple and mistakes them for its own data.

Operational consequences:

- A busy client that opens and closes thousands of short connections per second to one server accumulates tens of thousands of TIME_WAIT sockets, each holding an ephemeral port. With ~28,000 ephemeral ports and a 60 s wait, that caps you around **470 new connections per second per (src IP, dst IP, dst port)** before `EADDRNOTAVAIL`. Fixes, in order of correctness: **use keep-alive / connection pooling**, widen `ip_local_port_range`, enable `net.ipv4.tcp_tw_reuse` (safe for outbound with timestamps), add source IPs. Never `tcp_tw_recycle` (removed in Linux 4.12 because it broke NAT'd clients).
- **CLOSE_WAIT piling up** means *your application received a FIN and never called close()*: a file-descriptor leak. It is always a bug in the process, never the network.
- A **RST** aborts immediately with no TIME_WAIT. Sent when a segment arrives for a closed port, when a process exits with unread data in its receive buffer, or with `SO_LINGER` timeout 0. `ECONNRESET` in application logs means the peer, or a middlebox impersonating it, sent RST.

## 6.7 Congestion control: the part that makes the internet work

Flow control protects the receiver. **Congestion control protects the network**, and it is entirely inferred: routers do not tell senders how much capacity exists (except through ECN marks). The sender maintains a **congestion window (cwnd)** and sends at most `min(cwnd, rwnd)` bytes per RTT.

### 6.7.1 Slow start and AIMD

- **Slow start**: begin with an initial window (10 segments ≈ 14.6 KB since RFC 6928; it was 1–4 for decades) and double cwnd every RTT. It is "slow" only compared with blasting at line rate.
- On the first loss, or when cwnd passes `ssthresh`, switch to **congestion avoidance**: grow by one segment per RTT (additive increase), and on loss cut cwnd by half (multiplicative decrease). **AIMD** converges to a fair share among competing flows; anything more aggressive does not.

The sawtooth this produces is the defining picture of classic TCP: ramp, drop, halve, ramp.

### 6.7.2 The throughput formula

For loss-based congestion control, the Mathis equation gives the steady-state ceiling:

```
throughput ≤ (MSS / RTT) × (C / √p)      where C ≈ 1.22 and p = packet loss rate
```

Example: MSS 1460 bytes, RTT 100 ms, loss rate 0.01% (p = 10⁻⁴):

| Term | Value |
|---|---|
| MSS / RTT | 14,600 bytes/s ≈ 117 kbit/s |
| 1 / √p | 100 |
| × 1.22 | ≈ **14 Mbit/s** |

One lost packet in ten thousand caps a 100 ms path at 14 Mbps, no matter how fat the pipe. Cut RTT to 10 ms and you get 140 Mbps. This is why CDNs and edge PoPs exist: **shorter RTT is the lever**. It is also why high-speed long-distance transfers use parallel streams (Aspera, GridFTP, `bbcp`), which is the same thing as being N times as aggressive.

### 6.7.3 Reno, CUBIC, and BBR

| Algorithm | Signal | Behaviour | Where |
|---|---|---|---|
| **Reno / NewReno** | Loss | Classic AIMD | Historical baseline; Windows until 2010s |
| **CUBIC** | Loss | Grows cwnd along a cubic curve: fast away from the last loss point, plateau near it, then probe beyond. Much better on high-BDP paths | **Linux default since 2.6.19 (2006)**; macOS; Windows 10+ |
| **BBR (Bottleneck Bandwidth and RTT)** | Measured delivery rate and min RTT | Models the pipe instead of waiting for loss. Aims for the knee where the queue starts to build. Ignores random loss, so it thrives on lossy paths where CUBIC starves | Google (YouTube, google.com), Linux option, Spotify, Dropbox. BBRv1 was unfair to CUBIC on shared bottlenecks; v2/v3 fixed much of that |
| **DCTCP** | ECN marks | Uses the *fraction* of marked packets as a fine-grained signal. Keeps queues tiny | Datacenters only (needs ECN on every switch) |
| **Vegas / Compound / Illinois** | Delay | Historical delay-based schemes | Mostly academic |

Choosing: on Linux, `sysctl net.ipv4.tcp_congestion_control`. CUBIC unless you have measured otherwise; BBR when serving over the lossy public internet at scale; DCTCP inside a fabric you control end to end.

### 6.7.4 Bufferbloat

Big router buffers seemed harmless: no drops. But loss-based TCP only slows down when it sees a drop, so it fills the buffer completely first. A 1 MB buffer on a 10 Mbps link adds **800 ms of queueing delay** before a single packet is lost. Your video call dies not from lack of bandwidth but from the queue your download built. The fixes are **Active Queue Management**: CoDel and FQ-CoDel (`tc qdisc add dev eth0 root fq_codel`) keep queues short by dropping or marking early, and CAKE is the home-router version. On Linux servers, `fq` with pacing (which BBR requires) is the right default.

### 6.7.5 ECN

Explicit Congestion Notification lets a router **mark** the IP header instead of dropping; the receiver echoes the mark (ECE flag), the sender reduces cwnd and confirms (CWR). Same signal as a loss without the retransmission. Datacenters (DCTCP) depend on it; the public internet has it enabled on most servers now, with **L4S** (Low Latency, Low Loss, Scalable throughput) rolling out as a finer-grained successor.

## 6.8 Small-packet pathologies: Nagle and delayed ACK

**Nagle's algorithm** (on by default): if there is unacknowledged data in flight, buffer small writes until an ACK returns or a full segment accumulates. Prevents 41-byte packets carrying one keystroke.

**Delayed ACK** (on by default): the receiver waits up to 40 ms (Linux) / 200 ms (Windows) hoping to piggyback the ACK on response data.

Together they deadlock: sender writes a header (sent), then a body (Nagle holds it, waiting for the ACK of the header); receiver holds the ACK waiting for more data. **Forty-millisecond stalls** on every request. Every RPC framework and HTTP client sets `TCP_NODELAY` to disable Nagle for this reason; Go, Node, and Java's Netty do it by default. If you see suspicious 40 ms or 200 ms latency quanta, this is the first suspect. `TCP_QUICKACK` disables delayed ACK per call on Linux; `TCP_CORK` is the opposite of NODELAY for assembling a response from several writes.

## 6.9 Keepalives and half-open connections

If a peer crashes or a NAT drops its mapping, a TCP connection with nothing in flight sits **ESTABLISHED forever** on the surviving side: nothing on the wire tells it. TCP keepalives (`SO_KEEPALIVE`, with Linux defaults of 7200 s idle before the first probe, which is useless; set `TCP_KEEPIDLE` to 30–60 s for anything behind NAT or a cloud load balancer) send empty probes and tear down after N failures. Application-level heartbeats (HTTP/2 PING, WebSocket ping, gRPC keepalive) do the same above TLS and are more portable. Cloud load balancers have idle timeouts (AWS NLB 350 s, ALB 60 s default) that silently drop connections; a pool that hands out a connection idle for longer than that produces sporadic `ECONNRESET` on the first request. **Set the client idle timeout below the LB's, and the server's below the client's.**

## 6.10 Head-of-line blocking

TCP delivers bytes in order. If segment 5 is lost, segments 6–100 sit in the receiver's buffer, undeliverable to the application until 5 is retransmitted, even if they belong to logically independent requests. HTTP/2 multiplexes many streams on one TCP connection and therefore suffers this: one lost packet stalls every stream. QUIC (§7) was built largely to fix it.

## 6.11 SCTP and the transports that did not win

SCTP (Stream Control Transmission Protocol) offered multi-streaming, multi-homing, and message boundaries in 2000. It lost because NATs and firewalls only understand TCP and UDP. It survives in telecom signalling and, ironically, inside WebRTC data channels, tunnelled over DTLS over UDP. The lesson became QUIC's design rule: **anything new must look like UDP to the middleboxes.**

---

# 7 — QUIC: Rebuilding Transport in User Space

QUIC (RFC 9000, 2021) is a transport protocol carried inside UDP datagrams, always encrypted with TLS 1.3, originally built by Google and now the basis of HTTP/3. It moves the reliability, congestion control, and stream logic out of the kernel into a user-space library, which is both its genius and its cost.

## 7.1 Why UDP: the ossification problem

By 2010 the internet could not deploy a new transport protocol or even a new TCP option. Middleboxes (NATs, firewalls, "TCP accelerators", load balancers) parsed TCP headers, rewrote them, and dropped anything unfamiliar. Any protocol number other than 6 or 17 was dropped outright. The only thing that reliably passed through everything was an encrypted blob inside UDP. So QUIC encrypts *everything* except a few header bits, including the packet numbers and ACK frames, precisely so that no middlebox can develop an opinion about it. The one unencrypted bit that could be observed, the "spin bit", is optional and deliberately negotiated.

## 7.2 What it fixes

| TCP + TLS problem | QUIC answer |
|---|---|
| 1 RTT for TCP handshake + 1 RTT for TLS 1.3 = 2 RTTs before first byte | Combined crypto/transport handshake: **1 RTT**, and **0 RTT** for resumed connections (client sends the request in its first packet) |
| Head-of-line blocking across multiplexed streams | **Independent streams**: a loss on stream 3 stalls only stream 3 |
| Connection identified by 5-tuple; dies when IP changes (Wi-Fi → cellular) or under anycast route shifts | **Connection ID**: the connection survives address changes; **connection migration** |
| Congestion control locked in the kernel; years to deploy a change | User-space; Chrome or a server can ship a new algorithm in a browser release |
| Ambiguous RTT samples from retransmissions (same seq number) | Every packet, including retransmissions, has a **new packet number**; RTT is always unambiguous |
| Middleboxes can inspect, modify, RST | Encrypted; nothing to inspect. Only endpoints can close |
| PMTUD depends on ICMP getting through | Built-in DPLPMTUD probing |

## 7.3 The costs

- **CPU.** User-space UDP with per-packet crypto costs 2–3× the CPU of kernel TCP with hardware TSO/GRO offload. Segmentation offload for UDP (UDP GSO/GRO) and kernel-TLS-style offloads narrowed the gap but did not close it. Large CDNs eat this; a small service may not want to.
- **UDP hostility.** Some corporate networks block or rate-limit UDP 443. Every HTTP/3 client falls back to HTTP/2 over TCP; the fallback logic (racing, `Alt-Svc` caching) is itself complex.
- **0-RTT replay.** Data sent in 0-RTT can be replayed by an attacker who captured it. Servers must only accept idempotent requests in 0-RTT (HTTP GET, not POST), and the application must actually enforce that.
- **Load balancer awareness.** An L4 balancer that hashes the 5-tuple breaks connection migration; it must hash on the QUIC connection ID instead (Facebook's Katran and Google's Maglev do).
- **Observability.** `tcpdump` shows opaque UDP. Debugging needs `qlog` from the endpoints and SSLKEYLOGFILE-style key export.

Where it stands: roughly a third of web traffic by volume as of the mid-2020s (Google, Meta, Cloudflare, Akamai serving it), the transport for HTTP/3 and increasingly for MASQUE proxies, DNS-over-QUIC, and media (RTP-over-QUIC / MoQ). Inside datacenters, TCP still dominates for its offload economics.

---

# 8 — Naming: DNS

DNS maps names to records. It is a globally distributed, hierarchical, eventually consistent, heavily cached key-value store that predates every database you have used, and it is on the critical path of essentially every request. "It's always DNS" is a joke because it is so often true.

## 8.1 The hierarchy and the resolution walk

The namespace is a tree: root `.` → TLD `com.` → `example.com.` → `www.example.com.`. Each zone is served by **authoritative** servers. A client's **stub resolver** (the OS) asks a **recursive resolver** (your ISP's, or `1.1.1.1`/`8.8.8.8`, or a corporate one) which walks the tree:

```
stub ──▶ recursive resolver
             │  "www.example.com A?"  ──▶ root server (13 names a–m, hundreds of anycast instances)
             │  ◀── "ask the .com servers: a.gtld-servers.net ... (+ glue A records)"
             │  "www.example.com A?"  ──▶ .com TLD server
             │  ◀── "ask ns1.example.com (+ glue)"
             │  "www.example.com A?"  ──▶ ns1.example.com (authoritative)
             │  ◀── "www.example.com A 93.184.216.34, TTL 300"
             └──▶ cache it, return to stub
```

Every answer is cached according to its **TTL**, at the recursive resolver, often at the OS, and sometimes in the application runtime. Root and TLD referrals are cached for days, so a warm resolver usually needs one round trip. A cold, uncached lookup from a distant resolver can cost 100–300 ms; that is the difference between a fast and a slow page load.

**Glue records** are A/AAAA records for name servers *inside* the zone they serve (`ns1.example.com` for `example.com`), handed out by the parent to break the circularity. Missing or stale glue is a classic "domain intermittently resolves" bug.

## 8.2 Record types

| Type | Purpose | Notes |
|---|---|---|
| A / AAAA | Name → IPv4 / IPv6 | Multiple records = crude round-robin; clients pick arbitrarily |
| CNAME | Alias to another name | Cannot coexist with other records at the same name, so **cannot be used at a zone apex** (`example.com`). Providers invented ALIAS/ANAME/flattening to work around it |
| NS | Delegation to name servers | |
| SOA | Zone metadata: serial, refresh, **negative-caching TTL** | |
| MX | Mail exchangers with priority | |
| TXT | Free text: SPF, DKIM, DMARC, domain verification, ACME challenges | |
| SRV | Service discovery: `_service._proto.name → priority, weight, port, target` | Used by Consul, Kubernetes headless services, SIP, XMPP; ignored by browsers |
| PTR | Reverse: IP → name (`34.216.184.93.in-addr.arpa`) | Mail servers check it |
| CAA | Which CAs may issue certificates for this name | |
| HTTPS / SVCB | Modern: advertise HTTP/3 support, ECH keys, and IPs in one lookup | Enables an apex "CNAME" that works |

## 8.3 Caching subtleties that cause outages

- **TTL is a maximum.** Resolvers may serve stale under load (RFC 8767 "serve-stale"), and some clamp very low TTLs upward. Plan a migration assuming the old address will be hit for hours after TTL expiry.
- **Negative caching.** An NXDOMAIN is cached too, for the SOA's minimum TTL. Create a record, query it before it propagates, and you have cached the *absence* for the TTL.
- **Java** historically cached successful lookups **forever** when a SecurityManager was present (`networkaddress.cache.ttl`); modern JVMs default to 30 s. A JVM that resolved the database once and never again is a real failover killer. **Go** and most others honour TTLs via the OS or their own resolver, but check.
- **Kubernetes `ndots:5`**: a pod's `resolv.conf` has `search ns.svc.cluster.local svc.cluster.local cluster.local` and `ndots:5`. Any name with fewer than five dots is tried with each search suffix *first*. Resolving `api.stripe.com` from a pod therefore issues **eight** queries (four suffixes × A and AAAA) that all fail before the real one succeeds. Use a trailing dot (`api.stripe.com.`) or lower `ndots` for external-heavy workloads; watch CoreDNS QPS.
- **The 5-second timeout.** glibc's resolver defaults to a 5 s timeout and two attempts, and older kernels had a conntrack race that dropped one of the parallel A/AAAA UDP queries. Sporadic 5 s latency spikes in a service that is otherwise fast are DNS until proven otherwise (`options single-request-reopen` was the workaround; NodeLocal DNSCache is the modern fix).

## 8.4 DNS as a load-balancing and failover tool

Returning different answers per client is how **GSLB (global server load balancing)** works: Route 53, NS1, Cloudflare, and Akamai answer with the healthiest or closest endpoint. **EDNS Client Subnet** lets the recursive resolver forward the client's /24 so geo-steering is not fooled by a resolver in another country. The weaknesses:

- Health-check-driven failover is bounded by TTL and by clients that ignore TTL.
- Browsers and OSes do not fail over across multiple A records reliably.
- Low TTLs (30–60 s) multiply query load and put the DNS provider on your availability critical path (Dyn, §17.4).

DNS-based steering picks the region; a real load balancer (§14) picks the server.

## 8.5 Security and privacy: DNSSEC, DoT, DoH

Classic DNS is plaintext UDP 53 with 16-bit query IDs; **cache poisoning** (Kaminsky, 2008) injected forged answers into resolvers by guessing the ID. Source-port randomisation made it hard; **DNSSEC** makes it cryptographically impossible by signing records from the root down (RRSIG, DNSKEY, DS records). Deployment is uneven: most TLDs are signed, only a minority of domains are, and validation failures produce hard-to-debug SERVFAILs that have taken down large sites during key rollovers. Enable it if you can operate it; automate the rollover.

DNSSEC authenticates but does not encrypt. **DNS over TLS (DoT, port 853)** and **DNS over HTTPS (DoH, port 443)** encrypt the stub-to-resolver hop so the coffee-shop network cannot see or alter your lookups. Browsers ship DoH on by default in many regions; enterprises fight it because it bypasses their filtering resolvers. **Encrypted Client Hello (ECH)** closes the remaining leak, the TLS SNI, using a key published in the HTTPS record.

---

# 9 — The Application Layer: HTTP/1.1, HTTP/2, HTTP/3, WebSockets, gRPC

## 9.1 HTTP/1.1: text, one request at a time

```
GET /api/users/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer ...

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 187
Cache-Control: private, max-age=60
ETag: "a1b2c3"

{...}
```

The semantics (methods, status codes, headers, caching) are now defined in RFC 9110 independently of the wire format, which is the right way to think: HTTP/1.1, /2, and /3 are three encodings of one protocol.

Mechanics that matter:

- **Persistent connections** are default in 1.1 (`Connection: close` to opt out). Without them each request pays TCP + TLS setup, and each new connection starts in slow start.
- **Message framing** is `Content-Length` or `Transfer-Encoding: chunked` (stream a response of unknown length; also enables **trailers**). Ambiguity between the two is the root of **request-smuggling** attacks against proxy chains; strict parsers reject messages that carry both.
- **Pipelining** (send several requests before the responses come back) exists in the spec and is disabled everywhere: responses must return in order, so one slow response blocks the rest, and proxies mishandled it. Browsers instead open **6 parallel connections per host**, which is why the 2010s web used domain sharding, sprites, and bundling.
- **Caching** is HTTP's superpower: `Cache-Control` (`max-age`, `s-maxage`, `no-store`, `private`, `stale-while-revalidate`, `immutable`), validators (`ETag`, `Last-Modified`) with conditional requests (`If-None-Match` → `304 Not Modified`), and `Vary` to key the cache on request headers. A CDN is an HTTP cache with a global footprint; get the headers right and it does the work for you.
- **Status classes**: 2xx success, 3xx redirect (301 permanent cached, 302/307 temporary, 308 permanent preserving method), 4xx client error (400, 401 unauthenticated, 403 forbidden, 404, 409 conflict, 429 rate limited), 5xx server error. Three that are *network* signals when they come from a proxy: **502** the upstream replied with garbage or reset the connection, **503** no healthy upstream / overloaded, **504** the upstream did not answer within the proxy's timeout. Nginx's non-standard **499** means the *client* gave up first.

## 9.2 HTTP/2: binary framing and multiplexing

HTTP/2 (RFC 9113) keeps the semantics and replaces the text encoding with **binary frames** on a single TCP connection:

- **Streams**: many concurrent requests/responses, each a stream ID, interleaved frame by frame. No more six connections; no more head-of-line blocking *at the HTTP layer*.
- **HPACK** header compression: a static table of common headers plus a per-connection dynamic table, so a repeated `Authorization` header costs one byte after the first time. (Designed to be immune to the CRIME compression-oracle attack, hence its odd shape.)
- **Flow control per stream and per connection** at the HTTP layer, independent of TCP's; a small default window (65 KB) misconfigured on either side is a classic HTTP/2 throughput cap.
- **Server push** was meant to preload sub-resources; it was removed from Chrome in 2022 because it wasted bandwidth. `103 Early Hints` replaced it.
- **Prioritisation** via a dependency tree was so poorly implemented that RFC 9218 replaced it with a simpler urgency scheme.
- Runs over TLS in practice (`h2` negotiated by **ALPN**); cleartext `h2c` exists for gRPC inside datacenters.

Its Achilles heel is §6.10: TCP head-of-line blocking now stalls *every* stream on a loss, so on lossy mobile networks HTTP/2 can underperform six HTTP/1.1 connections. That is HTTP/3's reason to exist.

Attacks to know: **Rapid Reset** (CVE-2023-44487) opened and immediately cancelled streams faster than servers could account for them, producing record DDoS volumes in 2023; mitigations cap the rate of stream creation and cancellation.

## 9.3 HTTP/3: the same frames over QUIC

HTTP/3 (RFC 9114) maps HTTP/2's stream model onto QUIC streams so a lost packet stalls only its own stream. **QPACK** replaces HPACK because the dynamic table must tolerate out-of-order stream delivery. Discovery is via the `Alt-Svc` header or the HTTPS DNS record, with clients racing h3 and h2 and remembering what worked. All the trade-offs of §7.3 apply.

## 9.4 WebSockets, Server-Sent Events, and long polling

- **WebSocket** (RFC 6455): an HTTP/1.1 `Upgrade` handshake, then a bidirectional message framing protocol on the same TCP connection. Proxies must understand the upgrade (most do), and load balancers must support long-lived connections (idle timeouts again). Over HTTP/2 and /3 it uses the `CONNECT` extended method (RFC 8441 / 9220).
- **Server-Sent Events**: a plain long-lived HTTP response with `text/event-stream`. Server → client only, auto-reconnects, works through any HTTP infrastructure, and is the right choice more often than WebSocket for feeds, notifications, and LLM token streaming.
- **Long polling** is the fallback that always works.

## 9.5 gRPC and RPC over HTTP/2

gRPC is protobuf-encoded messages carried as HTTP/2 streams, using trailers for status, which is exactly why it does not work through many HTTP/1.1 proxies and why browsers need gRPC-Web. The networking facts that bite: one HTTP/2 connection per channel means **all RPCs hash to one backend behind an L4 balancer**, so you need an L7 (gRPC-aware) balancer or client-side load balancing with a resolver (xDS, or DNS with `round_robin`); keepalive settings must agree with the server's minimum or it sends `GOAWAY ENHANCE_YOUR_CALM`; and stream-level deadlines are propagated as a header, which is the correct model for timeouts (§17.2). For design-level comparison with REST see the companion note.

## 9.6 Everything else, briefly

| Protocol | Port | Transport | Note |
|---|---|---|---|
| SSH | 22 | TCP | Encrypted shell and tunnels; the `ProxyJump` and `-L/-R/-D` flags are a poor engineer's VPN |
| SMTP / IMAP | 25, 587 / 993 | TCP | Mail. SPF/DKIM/DMARC live in DNS TXT |
| NTP | 123 | UDP | Time. Distributed systems that assume synchronised clocks are assuming this works |
| DHCP | 67/68 | UDP broadcast | Address, mask, gateway, DNS handed to a booting host |
| SNMP | 161 | UDP | Network device monitoring; being replaced by streaming telemetry (gNMI) |
| RTP / RTCP / SRTP | dynamic | UDP | Real-time media; WebRTC's data plane |
| FTP | 21 (+ data) | TCP | Do not. Use SFTP (SSH) or HTTPS |

---

# 10 — Security: TLS, PKI, Firewalls, VPNs, and Attacks

## 10.1 The threat model

Assume an **on-path attacker** who can read, modify, drop, replay, and inject packets anywhere between two endpoints: the coffee-shop Wi-Fi, a compromised ISP router, a hijacked BGP route, a nation-state tap on a subsea cable. Every network protocol older than 1995 was designed as if that attacker did not exist. Everything since is an attempt to make the network untrusted by construction ("zero trust" is this idea applied to the corporate LAN).

TLS gives you three properties on one connection: **confidentiality**, **integrity**, and **server authentication** (client authentication optional). It does *not* hide **who is talking to whom** (IPs), **how much** (traffic analysis), or by default **which hostname** (SNI, until ECH).

## 10.2 The TLS 1.3 handshake

```
Client                                              Server
ClientHello: versions, cipher suites, SNI, ALPN,
             key_share (client's ECDHE public key)   ──▶
                                                    ◀── ServerHello: chosen suite, key_share
                                                        {EncryptedExtensions, Certificate,
                                                         CertificateVerify, Finished}   (encrypted)
{Finished}                                           ──▶
{Application data}  ◀──▶  {Application data}
```

- **One round trip** (TLS 1.2 needed two). Both sides derive the shared secret from the ECDHE key shares (X25519 or P-256) and can encrypt from the second flight onward.
- **Forward secrecy is mandatory**: the ephemeral key is discarded, so a stolen server private key does not decrypt recorded past sessions. TLS 1.2's RSA key exchange did not have this and is gone.
- The server proves identity by signing the handshake transcript with the private key matching its **certificate**.
- Only five AEAD cipher suites survive (`TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256`, and two CCM variants). The negotiation surface that produced a decade of downgrade attacks (POODLE, FREAK, Logjam) is gone.
- **Session resumption** uses a server-issued ticket (PSK). A resumed handshake can carry **0-RTT** data, with the replay caveat from §7.3.
- **SNI (Server Name Indication)** tells the server which certificate to present when one IP hosts many names. It is sent in plaintext in the ClientHello and is the last metadata leak; **ECH** encrypts it under a key from DNS.
- **ALPN** negotiates the application protocol (`h2`, `http/1.1`, `h3`) inside the handshake, avoiding an extra round trip.

Debug a handshake: `openssl s_client -connect host:443 -servername host -alpn h2 -tls1_3`, `curl -v --tlsv1.3`, or Wireshark with `SSLKEYLOGFILE` exported from the browser or client.

## 10.3 Certificates and the PKI

A certificate binds a public key to a name, signed by a **Certificate Authority**. The client walks the **chain**: leaf → intermediate(s) → a root it already trusts (the OS or browser root store, ~150 organisations). Things that go wrong, in decreasing frequency:

1. **Expired certificate.** Still the #1 cause of TLS outages at large companies. Automate renewal (ACME / Let's Encrypt, 90-day certificates forcing automation; the industry is moving to 47-day maximum lifetimes by 2029).
2. **Missing intermediate.** Works in Chrome (which caches and fetches intermediates) and fails in `curl`, Java, Python. Always serve the full chain minus the root.
3. **Hostname mismatch.** Certificate for `example.com` served for `www.example.com`; SANs (Subject Alternative Names) must list every name; wildcards cover one level only.
4. **Clock skew** on the client making a valid certificate "not yet valid".
5. **Private CA not in the trust store** of the container image / JVM `cacerts` / Python `certifi`. Each runtime has its own store.

Revocation is the PKI's weak spot: **CRLs** are large, **OCSP** leaks browsing to the CA and fails open, **OCSP stapling** lets the server attach a fresh OCSP response. Browsers now mostly use proprietary summaries (CRLSets, CRLite). **Certificate Transparency** logs every issued certificate publicly; Chrome rejects certificates without SCTs, and monitoring CT logs for your domains catches mis-issuance. **CAA** DNS records restrict which CAs may issue for you.

**Mutual TLS (mTLS)** has the client present a certificate too. It is the standard identity mechanism between services in a mesh (SPIFFE/SPIRE issue short-lived SVIDs), and for device authentication. Certificate **pinning** (accept only a specific key) hardens mobile apps against rogue CAs and has broken countless apps on rotation; prefer CT monitoring and short lifetimes.

## 10.4 Where to terminate TLS

| Termination point | Pros | Cons |
|---|---|---|
| Edge load balancer / CDN | Offloads CPU, central cert management, L7 routing possible | Plaintext from LB to backend unless re-encrypted; LB sees everything |
| Sidecar / mesh (mTLS everywhere) | Zero-trust inside the cluster, per-service identity | CPU and latency per hop, operational complexity |
| The application itself | End-to-end | Every service manages certificates; L4 LB only |

The common shape today: public TLS terminated at the edge, **re-encrypted** (or mTLS) to the backend, with the mesh handling service-to-service identity.

## 10.5 Firewalls, security groups, and NACLs

A **stateful firewall** tracks connections (conntrack) and allows return traffic for anything it saw go out. A **stateless** filter evaluates each packet alone and must have explicit rules for both directions, including ephemeral-port ranges. Cloud translation: AWS **security groups** are stateful and attached to instances; **NACLs** are stateless and attached to subnets, and the "return traffic blocked because the NACL has no ephemeral-port rule" bug is a rite of passage. Linux hosts have `nftables`/`iptables` (and Kubernetes uses them for Services and NetworkPolicy). Web Application Firewalls inspect L7 and are as good as their rule sets.

Firewalls that drop silently produce **timeouts**; firewalls that reject produce **RST / ICMP unreachable** and fast failure. From a debugging standpoint, silent drops are far more expensive. Internally, prefer reject.

## 10.6 VPNs and tunnels

- **IPsec** (ESP for encryption, IKEv2 for key exchange) is the standard for site-to-site links: cloud VPN gateways, branch offices. Complex, mature, hardware-accelerated.
- **WireGuard** is ~4,000 lines, UDP-based, Noise-protocol keys, no negotiation, in the Linux kernel since 5.6. It is what modern overlay networks (Tailscale, Cloudflare WARP) are built on.
- **TLS VPNs** (OpenVPN, AnyConnect) exist to get through hostile networks that only pass 443.
- Overlays cost MTU: WireGuard defaults to 1420, IPsec ~1400, and every tunnel is a PMTU black hole waiting to happen (§4.4).

## 10.7 Attacks and how the defences work

| Attack | Layer | Mechanism | Defence |
|---|---|---|---|
| ARP spoofing | 2 | Claim the gateway's IP on a LAN | Dynamic ARP inspection on switches, TLS everywhere, no trust in the LAN |
| DNS cache poisoning | 8 | Forge answers to a resolver | Port randomisation, DNSSEC, DoH/DoT |
| BGP hijack / leak | 5 | Announce someone else's prefix | RPKI ROV, filtering, monitoring |
| TCP RST injection | 4 | Guess seq number, send RST (the Great Firewall's technique) | Large windows make guessing hard; TLS/QUIC endpoints can ignore; QUIC is immune |
| TLS stripping | 7 | Downgrade `https://` to `http://` on first visit | HSTS, HSTS preload lists, HTTPS-only mode |
| **SYN flood** | 4 | Fill the SYN queue | SYN cookies, upstream scrubbing |
| **Amplification DDoS** | 4/7 | Spoof victim's IP to open DNS/NTP/memcached servers with small requests and huge replies (memcached: up to 51,000×) | BCP 38 source-address validation at ISPs, rate limiting responses, closing open resolvers, anycast absorption |
| **Volumetric DDoS** | 3 | Just more bits than your pipe (record: multi-Tbps from Mirai-style botnets) | Anycast networks with more capacity than any attacker (Cloudflare, Akamai, AWS Shield) |
| L7 DDoS (HTTP floods, Rapid Reset) | 7 | Exhaust application resources | Rate limits, challenges, connection caps, autoscaling with care |
| Request smuggling | 7 | Framing disagreement between proxy and backend | Strict, consistent parsing; HTTP/2 to the backend |

The strategic point: **you cannot defend a volumetric attack at your own edge.** Capacity is the defence, which is why DDoS mitigation is a service you buy from someone with more bandwidth than the botnet.

---

# 11 — Performance Engineering: Latency, Bandwidth, and the Numbers That Matter

## 11.1 Latency versus bandwidth, and which one you are fighting

Bandwidth is how many bytes per second the pipe carries. Latency is how long one byte takes to cross it. Adding bandwidth does not reduce latency, and most application slowness is latency, in the form of **round trips**.

Count round trips for a cold HTTPS request from a browser to a new origin:

| Step | RTTs |
|---|---|
| DNS lookup (uncached) | 1 (to resolver; more behind it) |
| TCP handshake | 1 |
| TLS 1.3 handshake | 1 |
| HTTP request/response | 1 |
| **Total before first byte of the response** | **4** |

At 100 ms RTT that is 400 ms before any content, plus slow start meaning the first RTT of data delivers only ~14 KB. With HTTP/3 and resumption: DNS + 0-RTT QUIC = ~2 RTTs. With a warm connection and a CDN edge 10 ms away: one 10 ms RTT. **The entire CDN/edge industry is round-trip arithmetic.**

## 11.2 The bandwidth-delay product

To keep a pipe full, the sender must have `bandwidth × RTT` bytes in flight (unacknowledged). That is the **BDP**, and both the congestion window and the receiver's window must reach it.

| Path | Bandwidth | RTT | BDP |
|---|---|---|---|
| Datacenter east–west | 25 Gbps | 0.2 ms | 625 KB |
| Cross-region cloud | 10 Gbps | 70 ms | **87.5 MB** |
| Home fibre to a CDN | 1 Gbps | 15 ms | 1.9 MB |
| Mobile 4G to a distant origin | 50 Mbps | 120 ms | 750 KB |

The default 64 KB TCP window without scaling caps a 70 ms path at 64 KB / 0.07 s ≈ 7.5 Mbps regardless of link speed. Linux's default `tcp_rmem` max of 6 MB caps that same 10 Gbps cross-region path at ~700 Mbps. Anyone moving bulk data between regions must raise `net.core.rmem_max`, `net.ipv4.tcp_rmem`/`tcp_wmem`, or use parallel streams. And slow start still needs `log₂(BDP / initial window)` RTTs to get there: about 13 RTTs (~1 s) to fill 87 MB from a 14 KB start, which is why short transfers never see the link's bandwidth.

## 11.3 Tail latency

Averages hide everything. A request that fans out to 100 backends is as slow as the slowest, so if each backend is at p99 = 100 ms, the fan-out's *median* is near 100 ms (Dean & Barroso, "The Tail at Scale"). Network causes of tail latency, in rough order of frequency in a datacenter:

1. Packet loss → 200 ms minimum RTO on a 0.2 ms RTT path (§6.4).
2. Incast: many servers reply to one at once, overflow a shallow switch buffer, synchronised drops.
3. Delayed ACK / Nagle 40 ms quanta (§6.8).
4. DNS 5 s timeouts (§8.3).
5. Connection setup on a cold pool (§11.4).
6. GC or CPU stalls that look like network delay (check `ss -ti` for `rcv_space`/`rwnd` collapse and the socket `Recv-Q`).

Tools for the tail: **hedged requests** (send a second copy after the p95 and take the first reply), **tied requests**, **backup requests with cancellation**, and deadlines that shrink as they propagate.

## 11.4 Connection management

- **Pool connections.** Never open a TCP+TLS connection per request. Size the pool for concurrency, not QPS: `pool ≈ QPS × mean latency` (Little's law), plus headroom.
- Go's default `http.Transport` keeps only **2 idle connections per host**; under load it opens and closes a connection per request and you drown in TIME_WAIT. Raise `MaxIdleConnsPerHost`. Every language has an equivalent footgun.
- Warm pools on startup; drain them on shutdown (send `GOAWAY` / `Connection: close`, stop accepting, finish in-flight, then exit; on Kubernetes, honour `preStop` and the readiness probe so the LB stops routing before the pod dies).
- Match idle timeouts: client < LB < server, so the client always closes first and never picks up a dead connection.
- Consider **HTTP/2 to backends** for a single multiplexed connection per host, remembering the L4-balancer pinning problem (§9.5).

## 11.5 The application-level checklist

| Technique | Why it works |
|---|---|
| Put an edge/CDN in front | Cuts RTT, absorbs slow start on warm connections, caches |
| TLS 1.3, session resumption, OCSP stapling | Fewer handshake round trips |
| HTTP/2 or /3 | One connection, header compression, no HoL at the HTTP layer |
| Compression (Brotli for text, zstd for APIs) | Fewer bytes → fewer RTTs in slow start |
| Cache headers done right | The fastest request is the one that never leaves the client |
| `TCP_NODELAY`, no small synchronous writes | Avoid the 40 ms trap |
| Preconnect / prefetch / `103 Early Hints` | Overlap round trips |
| Keep payloads under ~14 KB for the first response | Delivered in the first slow-start window |
| Pacing (`fq` qdisc) | Smooths bursts that cause drops at shallow buffers |
| Measure with `curl -w` timing fields and RUM | Know which step you are optimising |

```
curl -o /dev/null -s -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://example.com/
```

---

# 12 — The Host Stack: Sockets, Event Loops, and the Kernel Path

## 12.1 The socket API

```
Server                                  Client
socket(AF_INET6, SOCK_STREAM, 0)        socket(...)
setsockopt(SO_REUSEADDR)
bind(addr, port)
listen(backlog)
accept()  ──▶ new fd per connection   ◀──  connect(server_addr)
read()/write() / recv()/send()          read()/write()
close()                                 close()
```

A socket is a file descriptor with two kernel buffers (send and receive) and a state machine. `write()` returns when data is copied into the send buffer, **not** when it is delivered; `read()` returns whatever is in the receive buffer, which may be a partial message. TCP is a byte stream: **message boundaries are your problem**, so every protocol on top defines framing (length prefix, delimiter, chunked encoding).

Options every backend engineer should recognise:

| Option | Effect |
|---|---|
| `SO_REUSEADDR` | Bind while old connections sit in TIME_WAIT. Every server needs it |
| `SO_REUSEPORT` | Several processes bind the same port; kernel load-balances new connections across them (nginx, Envoy, Go with care) |
| `TCP_NODELAY` | Disable Nagle (§6.8) |
| `SO_KEEPALIVE` + `TCP_KEEPIDLE/INTVL/CNT` | Detect dead peers (§6.9) |
| `SO_RCVBUF` / `SO_SNDBUF` | Fixed buffers; setting them **disables autotuning**, which is usually a regression |
| `SO_LINGER {1, 0}` | Close with RST instead of FIN; skips TIME_WAIT; loses unsent data |
| `TCP_USER_TIMEOUT` | Max time unacknowledged data may sit before the connection is dropped; the right way to bound a write to a dead peer |
| `TCP_FASTOPEN`, `TCP_CONGESTION`, `TCP_QUICKACK`, `TCP_CORK` | Per-socket versions of the sysctls discussed above |
| `SO_BINDTODEVICE`, `IP_TRANSPARENT` | Proxies that spoof client addresses |

## 12.2 Blocking, non-blocking, and readiness: from C10K to C10M

A blocking `read()` parks the thread. One thread per connection worked for hundreds of connections and collapsed at ten thousand (the **C10K problem**, Kegel 1999): memory for stacks, context-switch cost, and scheduler thrash.

The answer is **readiness notification**: put every socket in non-blocking mode, ask the kernel "which of these can I read or write without blocking?", and handle only those. `select`/`poll` scan the whole list on every call (O(n)); **epoll** (Linux), **kqueue** (BSD/macOS), and **IOCP** (Windows, a completion rather than readiness model) are O(ready). Every high-performance server is an epoll loop: nginx, HAProxy, Envoy, Redis, Node.js (via libuv), Netty, and Go's runtime (the **netpoller** hides it behind goroutines that look blocking but park on epoll).

Edge-triggered vs level-triggered epoll, the thundering herd on `accept()` (solved by `SO_REUSEPORT` or `EPOLLEXCLUSIVE`), and reading until `EAGAIN` are the details that separate a correct event loop from one that stalls under load.

**io_uring** (Linux 5.1+) goes further: submission and completion ring buffers shared with the kernel, batching syscalls and supporting true asynchronous file and network I/O. It is the state of the art for user-space servers that stay in the kernel stack.

## 12.3 The receive path, end to end

What happens between a frame hitting the NIC and your `read()` returning (see Computer Architecture §9 for the hardware half):

1. NIC receives the frame, validates the CRC, and **DMAs** it into a pre-allocated buffer in a **receive ring** in host memory. With **RSS (Receive Side Scaling)** it hashes the 5-tuple to pick one of several rings, each tied to a CPU, so flows are spread across cores and each flow stays on one core (cache locality).
2. NIC raises a **hardware interrupt** (coalesced: one interrupt per N packets or per T microseconds, tunable with `ethtool -C`).
3. The interrupt handler does almost nothing except schedule a **softirq**; **NAPI** then *polls* the ring in a loop, disabling further interrupts while there is work. This is why a saturated server shows `ksoftirqd` burning CPU.
4. **GRO (Generic Receive Offload)** merges consecutive segments of the same flow into one large `sk_buff` so the rest of the stack runs once per ~64 KB instead of once per 1.5 KB. The transmit-side twins are **TSO/GSO**: the stack hands the NIC a 64 KB segment and the NIC (or kernel) cuts it into MTU-sized packets. These offloads are the reason kernel TCP can do 100 Gbps at all; `ethtool -k` shows them, and a VM or container with them disabled will be mysteriously slow.
5. `netif_receive_skb` → optional XDP/eBPF hook → `ip_rcv` (checksum, routing decision: local or forward) → netfilter hooks (`iptables`/`nftables`, conntrack) → `tcp_v4_rcv`: look up the socket by 5-tuple hash, run the TCP state machine, ACK, place data on the socket receive queue.
6. Wake any thread blocked in `read()` or mark the fd ready in epoll.
7. `read()` copies from the kernel buffer into user space. That copy is the main per-byte cost; **zero-copy** paths avoid it: `sendfile()`/`splice()` for file-to-socket, `MSG_ZEROCOPY` for large sends, `TCP_ZEROCOPY_RECEIVE` for receives, and **kTLS** to encrypt in the kernel so `sendfile` works over TLS.

**Bypassing the kernel** entirely, **DPDK** and **AF_XDP** hand the ring buffers to a user-space process that polls them, at the cost of dedicating cores and reimplementing TCP (or using UDP). Trading firms, load balancers (Katran uses XDP), and some storage systems do this. For everyone else, the kernel plus offloads is faster to operate and fast enough.

## 12.4 eBPF and XDP

**eBPF** lets you load verified, sandboxed programs into kernel hooks. For networking that means: **XDP** programs that run on the raw frame before `sk_buff` allocation (drop DDoS traffic at 20+ Mpps per core, or load-balance), **tc** programs for policy and telemetry, socket-level programs for redirecting connections (Cilium replaces `kube-proxy` and most of iptables this way), and tracing (`bpftrace`, `tcplife`, `tcpretrans`, `tcpconnlat` from BCC) that observes every connection without a packet capture. If you operate Linux at scale, eBPF tooling is now the primary way to *see* the network.

## 12.5 The sysctls that matter

| Knob | Default (Linux) | When to change |
|---|---|---|
| `net.core.somaxconn` | 4096 (128 pre-5.4) | Busy accept loops; must also raise `listen()` backlog in the app |
| `net.ipv4.tcp_max_syn_backlog` | scaled by memory | SYN-heavy servers |
| `net.ipv4.ip_local_port_range` | 32768–60999 | Clients making many outbound connections |
| `net.ipv4.tcp_tw_reuse` | 2 (loopback only) → 1 | Outbound-heavy clients hitting TIME_WAIT exhaustion |
| `net.ipv4.tcp_fin_timeout` | 60 | Rarely; only affects FIN_WAIT_2 orphan timeout, not TIME_WAIT |
| `net.core.rmem_max` / `wmem_max`, `net.ipv4.tcp_rmem` / `tcp_wmem` | 6 MB / 4 MB max | High-BDP paths (§11.2) |
| `net.ipv4.tcp_congestion_control` | cubic | bbr for lossy WAN serving |
| `net.core.default_qdisc` | fq_codel (fq for BBR) | Pacing, bufferbloat |
| `net.ipv4.tcp_slow_start_after_idle` | 1 | Set to 0 for long-lived RPC connections so an idle pool member does not restart slow start |
| `net.ipv4.tcp_keepalive_time` | 7200 | 60 for anything behind NAT/LB |
| `net.netfilter.nf_conntrack_max` | scaled | NAT/firewall hosts and Kubernetes nodes hitting "table full, dropping packet" |
| `net.ipv4.tcp_mtu_probing` | 0 | 1 on hosts behind tunnels to survive ICMP black holes |
| `net.ipv4.tcp_notsent_lowat` | unlimited | Small value for latency-sensitive servers to keep the send queue short |

Change one, measure, and keep a written reason. Tuning folklore copied from a 2012 blog post is a leading cause of confusing production behaviour.

---

# 13 — Datacenter and Cloud Networking

## 13.1 The Clos fabric

Old datacenters were three tiers of increasingly large switches (access → aggregation → core) with spanning tree blocking half the links and a giant layer-2 domain. Traffic patterns were "north–south" (client to server). Modern workloads are **east–west** (server to server: microservices, storage, MapReduce, and now all-reduce for model training), and the design is a **Clos / leaf-spine** fabric:

```
          [spine 1] [spine 2] [spine 3] [spine 4]
             \  \  /   /  \  \  /   /  ...
   [leaf/ToR 1] [leaf/ToR 2] ... [leaf/ToR N]
     | | | |      | | | |          | | | |
     servers      servers          servers
```

- Every leaf (top-of-rack) connects to every spine. Any two servers are at most leaf → spine → leaf apart: a deterministic, low latency.
- **ECMP** spreads flows across all spines. All links carry traffic; no STP.
- Everything is **layer 3**: each rack is its own subnet, and the ToR runs **BGP** (yes, the internet's protocol, chosen for its maturity, per-neighbour policy, and vendor support; RFC 7938 describes the practice) to advertise it.
- **Oversubscription** ratio is the design choice: 3:1 means the racks' server-facing bandwidth is three times the uplink bandwidth. Hyperscalers run 1:1 for training clusters.
- Scale by adding a third tier (pods of leaf-spine connected by super-spines). Meta's F16 and Google's Jupiter are this shape at hundreds of thousands of ports.

Why it matters to a software engineer: latency inside a region is uniform and tiny, bandwidth between any two servers is enormous, and the failure of one spine reduces capacity by 1/N rather than partitioning anything. Also: **cross-AZ traffic costs money and adds ~1 ms**; cross-region costs more of both.

## 13.2 Overlays, SDN, and the virtual network

Cloud tenants want their own private IP space, and containers want an IP per pod, on top of a physical fabric that knows nothing about them. The answer is an **overlay**: encapsulate the tenant packet inside a UDP packet on the physical ("underlay") network.

- **VXLAN** (UDP 4789) wraps an Ethernet frame with a 24-bit Virtual Network Identifier: 16 million isolated networks. **Geneve** (UDP 6081) is its extensible successor and what most cloud and Kubernetes overlays use now.
- The **50–60 bytes of overhead** reduce the effective MTU: a pod on a 1500-byte underlay gets ~1450, and if the underlay is itself a cloud VPC at 1500 (or 9001 jumbo on AWS), you must configure it consistently or meet the black hole of §4.4.
- **EVPN** (BGP carrying MAC/IP reachability) is the control plane for VXLAN in enterprise fabrics.
- **SDN (Software-Defined Networking)** separates the control plane (a central controller computes forwarding state) from the data plane (switches or hypervisor vSwitches like OVS just apply it). Every cloud VPC is SDN: the "router" and "security group" are distributed rules programmed into every hypervisor by a controller. That is why cloud networking has no ARP flooding, no broadcast, and "a subnet" is a database entry.
- **SmartNICs / DPUs** (AWS Nitro, Azure's FPGA, NVIDIA BlueField) run the overlay, encryption, and security-group logic in hardware so the hypervisor CPU serves tenants.

## 13.3 The cloud VPC vocabulary

| Concept (AWS names; others analogous) | What it is |
|---|---|
| VPC | A private IPv4/IPv6 address space (a `/16` typically) in one region |
| Subnet | A `/24`-ish slice pinned to one availability zone. "Public" just means its route table points `0.0.0.0/0` at an Internet Gateway |
| Route table | Per subnet; longest-prefix match to local, IGW, NAT GW, peering, TGW, or an instance |
| Internet Gateway | 1:1 NAT for instances with public IPs |
| NAT Gateway | Outbound-only NAPT for private subnets; per-AZ; billed per GB; port limits (§4.3) |
| Security group | Stateful, allow-only, instance-level firewall |
| NACL | Stateless subnet-level filter |
| VPC peering | Non-transitive L3 connection between two VPCs |
| Transit Gateway | A regional router hub connecting many VPCs and on-prem links, transitively |
| PrivateLink / Private Service Connect | Expose a service as an ENI *inside* the consumer's VPC; no peering, no public IP, one-directional |
| VPC endpoints | Reach S3/DynamoDB etc. without traversing the internet or paying NAT GW fees |
| Direct Connect / ExpressRoute / Interconnect | Private physical circuit to your on-prem network |
| Elastic Network Interface | A virtual NIC with a fixed private IP and security groups; what an instance or pod actually "has" |

Bandwidth is per instance size and often bursts; per-flow throughput inside a VPC caps around 5–10 Gbps (single-flow) regardless of the instance's aggregate limit, because of ECMP hashing. Parallel flows are the answer, again.

## 13.4 Kubernetes networking

The Kubernetes model has three rules: every pod gets its own IP; every pod can reach every other pod without NAT; a node can reach every pod on it. The **CNI plugin** implements it:

- **Overlay CNIs** (Flannel VXLAN, Calico VXLAN/IPIP, Cilium tunnel) wrap pod traffic between nodes; simple, MTU-sensitive.
- **Native routing** (Calico BGP, Cilium native, AWS VPC CNI which gives pods real VPC IPs from ENIs) has no encapsulation and better performance, at the cost of IP consumption and IPAM limits (the AWS "max pods per node" table is the ENI/IP limit).
- **Services** are virtual IPs (ClusterIP) that do not exist on any interface. **kube-proxy** programs iptables (a linear rule chain that gets slow past ~5,000 services) or IPVS (hash-based, scales) on every node to DNAT ClusterIP:port to a random ready pod. Cilium replaces this with eBPF. The consequence: **a Service is per-connection load balancing at L4**. Long-lived HTTP/2 or gRPC connections pin to one pod forever; you need an L7 balancer (Ingress/Gateway, mesh, or client-side balancing) for those.
- **NodePort** opens a port on every node; **LoadBalancer** asks the cloud for an external L4 balancer that targets NodePorts (or, with modern controllers, pods directly). **Ingress** / **Gateway API** describes L7 routing implemented by nginx, Envoy, HAProxy, Traefik, or the cloud's ALB.
- **DNS**: CoreDNS answers `svc.ns.svc.cluster.local` from the API server's endpoint state; headless Services return pod IPs directly (StatefulSets, client-side balancing). Remember `ndots` (§8.3).
- **NetworkPolicy** is the pod-level firewall, enforced by the CNI (not by kube-proxy; Flannel alone does nothing with it).
- **conntrack** on every node tracks every flow through a Service; the table filling up or the race on `INVALID` packets producing sporadic 1 s and 5 s stalls are the two most famous Kubernetes networking bugs.

## 13.5 Service mesh

A **service mesh** (Istio, Linkerd, Consul Connect; Cilium mesh without sidecars) inserts a proxy (usually Envoy) next to every pod, intercepts its traffic with iptables or eBPF, and provides mTLS identity, retries, timeouts, circuit breaking, traffic splitting, and telemetry uniformly, with no application code. The price is a proxy hop (0.5–2 ms and CPU per request), a control plane to run, and a great deal of YAML. Newer "ambient"/sidecar-less designs move L4 into a per-node component and L7 into a shared waypoint proxy to reduce the cost. Adopt one when you have more than a handful of services *and* cannot standardise a client library across your languages; otherwise a good client library does most of it more cheaply.

## 13.6 Networks for AI training and HPC

Distributed training runs **collective operations** (all-reduce, all-to-all) across thousands of GPUs every few hundred milliseconds; the network is on the critical path of every step. This is a different regime:

- **RDMA (Remote Direct Memory Access)** lets a NIC write directly into a remote machine's GPU or host memory with no CPU or kernel involvement, at microsecond latencies. Delivered over **InfiniBand** (its own lossless L2, NVIDIA/Mellanox) or **RoCEv2** (RDMA over UDP/IP on Ethernet).
- RoCE requires a **lossless fabric**: **Priority Flow Control** pauses the upstream port when a queue fills (with head-of-line blocking and pause-storm risks), and **DCQCN** does ECN-based congestion control. Getting PFC/ECN thresholds right is a specialised art, and the **Ultra Ethernet Consortium** is redesigning the transport to avoid needing losslessness.
- Topology is rail-optimised: each GPU's NIC connects to a dedicated "rail" of switches so that the collective's ring or tree maps onto physical paths with no oversubscription. NCCL discovers the topology and chooses ring, tree, or hierarchical algorithms accordingly.
- Bandwidth per GPU is 400–800 Gbps of network plus NVLink inside the node. A 10,000-GPU cluster has more east–west bandwidth than the public internet's largest exchange points.

---

# 14 — Load Balancers, Proxies, and CDNs

## 14.1 L4 versus L7

| | L4 (transport) balancer | L7 (application) balancer / reverse proxy |
|---|---|---|
| Sees | IPs, ports, (QUIC connection ID) | Full HTTP request: path, headers, cookies |
| Decision | Per connection (or per packet for UDP) | Per request |
| TLS | Passes through (or terminates without inspecting) | Terminates; can re-encrypt |
| Throughput | Millions of pps, terabits (in software with XDP/DPDK, or ASIC) | Bounded by parsing; tens to hundreds of Gbps per box |
| Features | Health checks, DSR, consistent hashing, ECMP integration | Routing rules, retries, rate limits, auth, header rewriting, WAF, caching, observability |
| Examples | AWS NLB, Google Maglev, Meta Katran, IPVS, HAProxy in TCP mode | nginx, Envoy, HAProxy HTTP mode, AWS ALB, Traefik, Caddy |

A typical stack is **anycast → L4 (stateless, ECMP-scaled) → L7 (stateful, feature-rich) → application**, with each tier scaling independently.

## 14.2 How the packets actually get to the backend

- **NAT / full proxy**: the LB terminates the connection and opens a new one to the backend. The backend sees the LB's IP as the client, hence **`X-Forwarded-For`** (L7) and the **PROXY protocol** (L4, a header line HAProxy invented that prefixes the client's real address to the stream). Trust `X-Forwarded-For` only from your own proxies, and only the rightmost value they appended; the rest is attacker-controlled.
- **DSR (Direct Server Return)**: the LB rewrites only the destination MAC (L2) or encapsulates (IPIP/GUE) and the backend replies **directly to the client** with the VIP as source. Return traffic, which is the bulk, never touches the LB. Maglev and Katran work this way; the backend must have the VIP configured on a loopback.
- **Consistent hashing / Maglev hashing**: map each 5-tuple to a backend so that when a backend is added or removed only ~1/N of flows move and, crucially, **any LB instance computes the same mapping** without sharing state. This is what makes the L4 tier stateless and ECMP-scalable.

## 14.3 Balancing algorithms and health

- **Round robin** and **weighted round robin**: fine for homogeneous, short requests.
- **Least connections / least outstanding requests**: better for variable request cost.
- **Power of two choices** (pick two at random, take the less loaded): near-optimal with no global state; Envoy's default.
- **Peak EWMA** (Linkerd, Finagle): route by exponentially weighted latency; avoids slow backends automatically.
- **Sticky sessions** (cookie or IP hash): needed only when the app keeps state in memory, which is a design smell to eliminate rather than support.

**Health checking** is where outages hide. Active checks (`GET /healthz` every N seconds) detect a dead backend in N × failure-threshold seconds. Passive checks (outlier detection: eject a backend after consecutive 5xx or timeouts) react faster. Two rules: a health endpoint must **check what the LB needs to know and nothing that can fail for unrelated reasons** (a health check that fails when the database is slow turns a slow database into a total outage as every backend is ejected simultaneously, see Facebook in §5.4), and **panic thresholds** (if more than X% of backends are unhealthy, ignore health and route to all) prevent the cascade.

**Connection draining / graceful shutdown**: mark the backend unhealthy, stop sending new connections, wait for in-flight requests (bounded), then stop. Kubernetes: readiness probe fails → endpoints updated → `preStop` sleep gives kube-proxy/LB time to reprogram → SIGTERM → app drains → exit. Skipping any step drops requests during every deploy.

## 14.4 Global load balancing

At the widest scope, three tools combine: **DNS-based GSLB** picks a region by geography and health (§8.4); **anycast** lets one IP land at the nearest PoP by BGP; and **Global Accelerator / Cloud Load Balancing**-style products anycast the front door and carry traffic on the provider's backbone to the chosen region. Stateful failover across regions is the hard part and is a data-layer problem, not a networking one.

## 14.5 Forward proxies, egress, and API gateways

A **forward proxy** sits in front of clients (corporate egress filtering, Squid, egress gateways in a mesh) and is how you give a fleet a small, fixed set of egress IPs for an allowlist. An **API gateway** is an L7 reverse proxy with product features (auth, quotas, transformation, developer portal). A **service proxy** (Envoy) is the general-purpose programmable L7 data plane that most of these are now built on.

## 14.6 CDNs

A CDN is a globally distributed reverse proxy with a cache, an anycast front door, and a private backbone:

- **Edge PoPs** in hundreds of cities terminate TLS within ~10–30 ms of most users, absorbing the round-trip cost of §11.1 and keeping warm, high-cwnd connections to the origin.
- **Cache hierarchy**: edge → regional/"origin shield" → origin. The shield collapses many edges' misses into one origin fetch (request coalescing) and protects the origin from thundering herds on cache expiry.
- **Cache key**: normally scheme + host + path + query; tune it (ignore tracking parameters, include a header via `Vary`) or you get cache fragmentation or, worse, cross-user leakage.
- **Invalidation**: purge by URL, tag/surrogate key, or everything; propagation takes seconds. Versioned asset URLs (`/app.3f9a2c.js`, `Cache-Control: immutable`) sidestep it entirely for static content.
- **Origin protection**: authenticated origin pulls (mTLS or a shared header) so nobody bypasses the CDN and hits the origin directly; IP allowlists of the CDN's ranges.
- Beyond caching, edges now run **compute** (Workers, Lambda@Edge, Fastly Compute), WAFs, bot management, image optimisation, and DDoS absorption; they are the modern "L7 at the edge" tier.

---

# 15 — Wireless and Mobile: Why the Last Hop Is Different

Everything above assumes a stable, low-loss link. Mobile breaks that assumption in specific, engineerable ways.

- **Radio Resource Control state machine.** A cellular radio idles in a low-power state and takes tens to hundreds of milliseconds to promote to a connected state before the first packet can be sent; after a few seconds of inactivity it demotes again. A chatty app that sends a heartbeat every 10 s keeps the radio permanently active and drains the battery; a burst-and-sleep pattern is far cheaper. Batch and coalesce network activity.
- **RTT variability.** 4G RTTs range 30–100 ms and 5G 10–30 ms in good conditions, but handovers between cells, congestion, and signal fade spike them to seconds. TCP's RTO estimator and CUBIC cope; BBR copes better; fixed short timeouts in the application do not.
- **Loss is not congestion.** Radio loss is random. Loss-based TCP interprets it as congestion and halves cwnd, which is why BBR and link-layer retransmission (HARQ) both exist.
- **NAT everywhere.** Every mobile carrier runs CGNAT (§4.3) with short UDP timeouts (often 30–60 s); long-lived connections need keepalives, and inbound is impossible without a relay.
- **Address changes.** Walking from Wi-Fi to cellular changes the client's IP. Every TCP connection dies; QUIC connections migrate (§7.2). **Happy Eyeballs** handles v4/v6 racing on the same transition.
- **Middleboxes and transparent proxies.** Carriers have historically run transparent HTTP proxies, image recompressors, and header injectors; TLS everywhere ended most of that, and HTTP/3's encrypted transport ends the rest.
- **Multipath TCP** (RFC 8684) uses Wi-Fi and cellular simultaneously as subflows of one connection; Apple uses it for Siri and Maps. **MASQUE**/iCloud Private Relay tunnels HTTP/3 through two proxies to hide the client IP from the origin.

Design implications: idempotent, resumable requests; exponential backoff with jitter; offline-first data models; small first payloads; avoid dozens of small connections; and test on a throttled, lossy network profile, not office Wi-Fi.

---

# 16 — Observability and Debugging: Method, Tools, and Failure Signatures

## 16.1 The method

1. **Establish the symptom precisely.** Timeout, refused, reset, slow, intermittent, one client or all, one path or all. Each word narrows the layer.
2. **Bisect the path.** Client → DNS → LB → node → pod. Test from each vantage point with the simplest tool that exercises the next hop.
3. **Ascend the layers** from the bottom: link up? (`ip link`), address and route? (`ip addr`, `ip route get`), neighbour resolves? (`ip neigh`), reachable? (`ping`, `mtr`), port open? (`nc -zv`, `ss -ltn`), TLS ok? (`openssl s_client`), HTTP ok? (`curl -v`).
4. **Look at the counters before the packets.** `ss -ti`, `nstat`, `ethtool -S`, LB metrics, and conntrack counts explain most problems without a capture.
5. **Capture on both ends** when you must. A capture on one side tells you what it *saw*; the diff between the two sides tells you where the packets died.

## 16.2 The toolbox

| Tool | Layer | Use |
|---|---|---|
| `ip link / addr / route / neigh` | 2–3 | Interface state, addresses, routes, ARP/NDP cache |
| `ethtool -S / -k / -C` | 1–2 | NIC counters (drops, CRC), offloads, coalescing |
| `ping` | 3 | Reachability and RTT. `ping -M do -s 1472` finds the path MTU |
| `traceroute` / `mtr` | 3 | Per-hop path and loss; `mtr` runs it continuously and shows where loss starts. Remember hops that rate-limit ICMP show false loss; only loss that *persists to the destination* is real |
| `dig +trace`, `dig @resolver`, `resolvectl` | DNS | Watch the full delegation walk; compare resolvers; see TTLs |
| `ss -tanp`, `ss -ti`, `ss -ltn` | 4 | Socket states, per-connection TCP info (rtt, cwnd, retrans, `rwnd`), listen queue depth |
| `nstat -az`, `netstat -s` | 4 | Stack-wide counters: `TcpRetransSegs`, `TcpExtListenOverflows`, `TcpExtTCPTimeouts`, `TcpExtSyncookiesSent` |
| `conntrack -L / -S` | 3–4 | NAT/firewall state table, `insert_failed` counts |
| `nc -zv host port`, `nc -l port` | 4 | Is the port open? Fake a server |
| `curl -v`, `curl -w`, `curl --resolve` | 7 | Full request trace, timings per phase, test a specific backend IP with the right SNI/Host |
| `openssl s_client -connect -servername` | TLS | Handshake, chain, ALPN, versions |
| `tcpdump -i any -nn -s0 -w out.pcap 'host X and port Y'` | all | The ground truth. `-nn` no DNS, `-s0` full packets; write to file and read in Wireshark |
| Wireshark | all | `tcp.analysis.retransmission`, `tcp.analysis.zero_window`, `Statistics → TCP Stream Graphs`, `Follow TCP Stream`; decrypt TLS with `SSLKEYLOGFILE` |
| `iperf3` | 4 | Raw throughput; `-P 8` for parallel streams; `-u` for UDP loss/jitter |
| `bpftrace`, BCC `tcpretrans`, `tcplife`, `tcpconnlat`, `gethostlatency` | 4–7 | Per-event kernel tracing without captures |
| `tc qdisc show`, `tc -s qdisc` | 2–3 | Queue discipline and drops; `tc netem` to *inject* loss and delay for testing |
| `nmap` | 3–4 | Port scans (with authorisation) |

## 16.3 Failure signatures

| Symptom | What it almost always means |
|---|---|
| **Connection refused** (`ECONNREFUSED`, RST to SYN) | Reached the host; nothing listening on that port (or a firewall that *rejects*). Wrong port, service not started, listening on 127.0.0.1 instead of 0.0.0.0 |
| **Connection timed out** on connect | SYN got no answer: wrong IP/route, silent-drop firewall / security group, host down, accept queue full |
| **Connects instantly, then hangs** on the first request | Accept-queue overflow (client thinks it connected), MTU black hole (small packets pass, large hang), or the server accepted and is stuck |
| **Works for small responses, hangs for large** | MTU black hole. Test with `ping -M do -s 1472` |
| **Sporadic `ECONNRESET` on the first request from a pool** | Server or LB idle timeout closed the connection; client reused a dead one. Fix idle timeouts (client < LB < server) |
| **`ECONNRESET` mid-transfer** | Peer crashed, process exited with unread data, LB/NAT dropped state, a middlebox injected RST |
| **Exactly 1 s / 3 s / 7 s connect latency** | SYN retransmit timers (1, 2, 4 s backoff): first SYN was dropped. Loss on connect, SYN flood, or conntrack race |
| **Exactly 5 s or 10 s latency, intermittent** | DNS timeout and retry (§8.3) |
| **40 ms or 200 ms latency quanta** | Nagle + delayed ACK (§6.8) |
| **200 ms+ tail spikes inside a datacenter** | RTO after a single loss (§6.4); look at `TcpExtTCPTimeouts` and switch buffer drops |
| **Throughput flat at a number unrelated to link speed** | Window/BDP cap (§11.2), single-flow ECMP cap, LACP member cap, or loss × RTT per Mathis |
| **`TcpExtListenOverflows` climbing** | Application not calling `accept()` fast enough |
| **Thousands of CLOSE_WAIT** | Application fd leak: it never calls `close()` |
| **Thousands of TIME_WAIT, then `EADDRNOTAVAIL`** | No connection reuse; ephemeral port exhaustion |
| **`nf_conntrack: table full, dropping packet`** | Node conntrack limit; raise it or reduce flow churn |
| **502 from the proxy** | Backend closed or reset the connection, or sent an unparseable response; often a backend keepalive timeout shorter than the proxy's |
| **503** | No healthy backend, or overload / rate limit |
| **504** | Backend accepted but did not respond within the proxy timeout |
| **Works with IP, fails with hostname** | DNS. Or SNI/Host mismatch on a shared IP |
| **Works from the node, fails from the pod** | CNI, NetworkPolicy, pod MTU, or `ndots` |
| **Works over v4, slow over v6 (or ~250 ms slower everywhere)** | Broken IPv6 path; Happy Eyeballs is saving you |
| **One host to one destination is slow, everything else fine** | ECMP/LACP put that flow on a bad link; look for CRC errors on that path |

## 16.4 Capturing safely in production

`tcpdump` is cheap in CPU when filtered (BPF runs in the kernel) but a busy 25 Gbps host can fill a disk in seconds. Always filter, always limit (`-c 100000` or `-W`/`-G` rotation), use `-s 128` (headers only) unless you need payload, write to a file rather than the terminal, and prefer `ss -ti`/`nstat`/eBPF first. Never capture payload of production traffic without thinking about what personal data is in it; TLS makes most of it opaque anyway, which is the point.

---

# 17 — Design Principles and War Stories

## 17.1 The fallacies of distributed computing

Peter Deutsch's list (1994) is the networking section of every system-design review:

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

Every one of them maps to a mechanism in this note: (1) retransmission, timeouts, idempotency; (2) round-trip counting, CDNs; (3) BDP, compression; (4) TLS, mTLS, zero trust; (5) DNS TTLs, connection migration, health checks; (6) BGP policy, peering; (7) egress and cross-AZ bills; (8) MTU, IPv6, middleboxes.

## 17.2 Timeouts, retries, backoff, idempotency

The four rules of talking to anything over a network:

- **Every call has a deadline**, and the deadline propagates: an incoming request with 800 ms left must not make a downstream call with a 2 s timeout. Separate **connect timeout** (short: 1–3 s; the SYN either arrives or it does not) from **request timeout** (based on the p99 of the downstream plus margin).
- **Retry only what is safe and only what might succeed.** A connect timeout or a 503 is retryable; a 500 after the server *may* have done the work is only safe if the operation is **idempotent** (idempotency keys make writes retryable). Retry a bounded number of times, never a timeout on a request that has a deadline shorter than the retry budget.
- **Exponential backoff with full jitter** (`sleep = random(0, min(cap, base × 2^attempt))`). Without jitter, every client that failed together retries together: a **retry storm** or thundering herd that keeps a recovering service down. Cap retries per client as a *budget* (e.g. retries ≤ 10% of requests) rather than per request.
- **Circuit breakers and load shedding.** When a downstream is failing, stop calling it for a while (fail fast) so that its recovery is not prevented by your retries, and shed load at the front when queues grow rather than letting latency climb unbounded (§1.4's queueing curve).

## 17.3 Ossification and why the internet changes so slowly

Any header field a middlebox can see, some middlebox will make assumptions about, and then the field can never change. TCP options, ECN bits, IP fragmentation, new protocol numbers, and even TLS 1.3's version field (which had to pretend to be 1.2 with an extension because middleboxes rejected 1.3) have all been frozen this way. The design rules that came out of it: **encrypt everything you do not need the network to see** (QUIC), **grease** unused values so that middleboxes cannot learn to depend on them being absent (TLS GREASE, QUIC versions), and **use existing wrappers** (UDP, HTTP, TLS) as the substrate for new protocols. A staff engineer proposing a new wire format should assume that the first version's quirks will be permanent.

## 17.4 War stories worth retelling

- **YouTube, February 2008**: BGP more-specific hijack (§5.4). Lesson: longest-prefix match plus no origin validation means anyone can take your traffic; deploy RPKI and monitor your prefixes.
- **Dyn, 21 October 2016**: the Mirai IoT botnet DDoSed a major managed-DNS provider; Twitter, GitHub, Spotify, Reddit, and Netflix became unreachable for hours because their DNS was single-homed. Lesson: DNS is on your availability path; use two providers with synchronised zones, and remember that resolvers cache only as long as your TTL.
- **Cloudflare / Verizon route leak, 24 June 2019**: a small ISP's BGP optimiser leaked more-specifics, a big transit provider accepted them without filters. Lesson: filtering and max-prefix limits are boring and they are the actual defence.
- **Facebook, 4 October 2021**: backbone withdrawal → DNS servers withdrew their anycast routes as designed → total outage, tools and badge readers included (§5.4). Lesson: correlated health checks amplify failures; out-of-band access to the control plane is not optional.
- **AWS us-east-1, 7 December 2021**: an automated scaling event flooded the internal network connecting the main AWS network to its control-plane network; congestion collapse, retry storms, and the monitoring needed to diagnose it lived on the same congested network. Lesson: retry storms turn a degradation into an outage; observability must not share fate with the thing it observes.
- **Rogers (Canada), 8 July 2022**: a core-router configuration update removed a routing filter; the resulting flood of routes overwhelmed the core routers, and both wireless and wireline service went down nationally for a day, including 911 access. Lesson: control-plane changes need staged rollout and an independent management path.
- **Slack, 4 January 2021**: AWS Transit Gateway capacity did not scale fast enough as users returned from holiday; packet loss led to monitoring blindness and a scaling feedback loop. Lesson: your cloud provider's network has scaling limits too, and "everyone comes back on the same morning" is a load pattern to test.
- **Every company, always**: the expired certificate; the security group that dropped the ephemeral-port return path; the VPN MTU; the Go client with two idle connections; the JVM caching DNS forever; the Kubernetes `ndots` storm; the load balancer idle timeout one second shorter than the client's.

---

# 18 — The Staff Engineer's Decision Checklist

The decisions you are actually expected to make, and the default answer with the reason.

| Decision | Default | Change it when |
|---|---|---|
| **TCP or UDP/QUIC for a new service?** | TCP (HTTP/2 or gRPC) | Real-time media, connection migration, or you serve the lossy public internet at scale and can afford the CPU (HTTP/3 at the edge, TCP inside) |
| **HTTP/1.1, /2, or /3 to backends?** | HTTP/2 with L7 balancing | /1.1 if only L4 balancing is available and pinning is a problem; /3 essentially never inside a datacenter |
| **Where does TLS terminate?** | Edge, re-encrypt to backend, mTLS in the mesh | Regulated data that must be end-to-end: terminate in the app and accept L4-only balancing |
| **L4 or L7 load balancer?** | Both: L4 in front for scale, L7 for routing | Pure L4 for non-HTTP protocols or extreme throughput |
| **DNS TTL?** | 60–300 s for endpoints that fail over; 1 h+ for stable names | Shorter only if you have measured the resolver load and your provider's SLA |
| **Connect / request timeouts?** | Connect 1–3 s; request = downstream p99 × 1.5, propagated as a deadline | Never "30 s default" copied from a library |
| **Retries?** | Idempotent only, ≤ 2, exponential backoff with full jitter, per-client retry budget | Non-idempotent writes need idempotency keys first |
| **Connection pool size?** | Little's law: concurrency = QPS × latency, plus 25% | Serverless with no warm pools: prefer fewer, larger instances |
| **Idle timeouts?** | Client < LB < server, all set explicitly | |
| **Keepalives?** | On, 30–60 s, for anything crossing NAT or a cloud LB | |
| **MTU?** | 1500 externally; 9000 inside a fabric you fully control; pod MTU = underlay − overlay overhead | Tunnels: clamp MSS and enable `tcp_mtu_probing` |
| **IPv6?** | Dual-stack from day one; bind `::` | Never "we'll add it later" |
| **Congestion control?** | CUBIC | BBR for edge servers to the public internet; DCTCP inside an ECN-enabled fabric |
| **Service mesh?** | No, until >~20 services in >2 languages | Then yes, and budget the operational cost |
| **Multi-region?** | Active-passive with DNS/anycast failover and a tested runbook | Active-active only when the data layer supports it |
| **Health check content?** | Process is alive and can serve; *not* downstream dependencies | Deep checks belong in alerting, not in the LB |
| **Egress?** | NAT gateway or egress proxy with fixed IPs; VPC endpoints for cloud services | |
| **DDoS?** | Front everything public with a provider that has more capacity than any botnet | |
| **Custom binary protocol?** | No; use HTTP/2, gRPC, or an existing framing over TLS | Only with a measured need and the ossification lesson in mind |

Design-review questions to ask of any system diagram: What is the RTT budget of the slowest user? How many round trips before first byte? What happens when this link drops 0.1% of packets? Where is the state that a NAT, LB, or conntrack table holds, and what is its timeout? What is idempotent? Which health check can fail for a reason unrelated to the thing it protects? What is the blast radius of a bad route announcement, a bad DNS change, an expired certificate?

---

# 19 — Mental Models Worth Keeping

1. **Count round trips, not bytes.** Latency is the RTT count times the RTT; bandwidth only matters once the pipe is full.
2. **Loss × RTT sets your throughput ceiling** (`MSS / (RTT·√p)`). Shorten the RTT or fix the loss; more bandwidth will not help.
3. **Every queue is a latency bomb.** Utilisation above ~70–80% makes delay explode; big buffers make it worse, not better.
4. **State in the middle is a timeout waiting to happen.** NAT, LB, conntrack, and firewall tables all expire; keepalives and idle-timeout ordering are how you survive them.
5. **The endpoints own correctness; the network owns best effort.** Retransmission, ordering, and integrity are yours. Anything the network adds is an optimisation you cannot depend on.
6. **Longest prefix wins, and money beats distance.** Inside a router, the most specific route; on the internet, the most profitable one.
7. **Silent drops cost hours; loud rejects cost seconds.** Design internal networks and APIs to fail fast and visibly.
8. **Anything a middlebox can see, it will freeze.** Encrypt the transport; grease the rest.
9. **DNS is a database with a caching layer you do not control.** Plan changes around TTLs and the clients that ignore them.
10. **Correlated health checks amplify failures.** A check that fails everywhere at once is an outage, not a safeguard.
11. **The 40 ms, 200 ms, 1 s, 3 s, 5 s latency quanta are protocol timers, not mysteries.** Nagle/delayed ACK, min RTO, SYN retries, DNS.
12. **The tail is where the network lives.** Medians look fine on every graph; p99 shows the retransmits.
13. **Encapsulation is 50 bytes of MTU you forgot about.** Every tunnel and overlay needs the arithmetic done.
14. **The datacenter is the opposite regime from the internet**: microsecond RTTs, enormous bandwidth, lossless expectations, and a 200 ms RTO that is a thousand RTTs long.
15. **Read the counters before the packets, and capture at both ends when you must.**

---

# 20 — A Reading and Practice Path

## Books

- **Kurose & Ross, *Computer Networking: A Top-Down Approach*** — the undergraduate text; read it once for the vocabulary.
- **Tanenbaum & Wetherall, *Computer Networks*** — bottom-up, stronger on the physical and link layers.
- **W. Richard Stevens, *TCP/IP Illustrated, Vol. 1*** (2nd ed. by Kevin Fall) — TCP explained through packet traces. Still the best way to actually understand TCP.
- **Ilya Grigorik, *High Performance Browser Networking*** — free online; the performance chapters (§11 here) in full depth.
- **Beej's Guide to Network Programming** — the socket API, free.
- **Marcel Gagné / Brendan Gregg, *Systems Performance*** and *BPF Performance Tools* — the host-stack observability of §12 and §16.
- **Russ White & Ethan Banks, *Computer Networking Problems and Solutions*** — the "why" behind routing design.

## RFCs worth reading in full

RFC 793/9293 (TCP), 791 (IP), 8200 (IPv6), 4271 (BGP-4), 1034/1035 (DNS), 9110–9114 (HTTP semantics, /1.1, /2, /3), 8446 (TLS 1.3), 9000/9001/9002 (QUIC), 5681 (TCP congestion control), 6298 (RTO), 7938 (BGP in datacenters), 8305 (Happy Eyeballs), 1918 (private addresses), 6890 (special-purpose addresses).

## Papers

- Saltzer, Reed, Clark, "End-to-End Arguments in System Design" (1984).
- Jacobson & Karels, "Congestion Avoidance and Control" (1988).
- Dean & Barroso, "The Tail at Scale" (2013).
- Cardwell et al., "BBR: Congestion-Based Congestion Control" (2016).
- Eisenbud et al., "Maglev: A Fast and Reliable Software Network Load Balancer" (2016).
- Singh et al., "Jupiter Rising" (Google's datacenter fabric, 2015).
- Langley et al., "The QUIC Transport Protocol: Design and Internet-Scale Deployment" (2017).
- Alizadeh et al., "Data Center TCP (DCTCP)" (2010).
- Lapukhov, Premji, Mitchell, "Use of BGP for Routing in Large-Scale Data Centers" (RFC 7938).

## Labs that build real intuition

1. **Watch a handshake.** `tcpdump -i any port 443 -w h.pcap` while running `curl https://example.com`; open in Wireshark; find SYN/SYN-ACK/ACK, the ClientHello, the ALPN result, the FINs. Count the round trips.
2. **Break the MTU.** On a Linux VM, `ip link set dev eth0 mtu 1300` on one side of a tunnel with ICMP blocked; observe the hang; fix it with MSS clamping.
3. **Inject loss and delay.** `tc qdisc add dev eth0 root netem delay 50ms loss 0.1%`; run `iperf3`; compare CUBIC and BBR (`-C bbr`); verify the Mathis formula roughly holds.
4. **Exhaust the accept queue.** Write a server that `listen()`s and never `accept()`s; connect with `nc` in a loop; watch `ss -ltn` and `nstat | grep Listen`.
5. **Walk DNS by hand.** `dig +trace www.example.com`; then `dig +norecurse` at each server yourself.
6. **Run BGP.** Two FRRouting containers peering over a Docker network; announce a prefix; withdraw it; watch convergence; add a bogus more-specific and see it win.
7. **Build a VXLAN overlay** between two hosts with `ip link add vxlan0 type vxlan ...`; measure the MTU it needs.
8. **Trace with eBPF.** `tcpconnlat`, `tcpretrans`, `tcplife` from BCC on a busy box; correlate with `ss -ti`.
9. **Reproduce the 40 ms stall.** A client that writes a header then a body with Nagle on, against a server with delayed ACK; time it; set `TCP_NODELAY`.
10. **Deploy a Kubernetes cluster with two different CNIs** and compare `ip route` on a node, pod MTU, and how a Service is implemented (`iptables-save | grep KUBE-SVC` vs `cilium bpf lb list`).

When you can predict the outcome of each lab before running it, explain it with the numbers in this note, and name the production incident it corresponds to, you are operating at the level this document was written for.
