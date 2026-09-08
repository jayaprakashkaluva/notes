# The OSI and TCP/IP Models: Why Networks Are Layered

A working engineer's guide to the two reference models every network conversation is built on. The goal is not to memorise seven names for an exam but to understand **what problem layering solves**, **what the world looked like before it**, **why two models exist and how they relate**, and most importantly **which layer you are actually standing on** when you write an HTTP handler, open a socket, configure a load balancer, or debug a "the service is down" page at 3 a.m.

Throughout, the models are treated as what they are: *vocabulary and a division of responsibility*, not physical law. Real protocols leak across layers, and the second-to-last section is devoted to exactly where and why.

Companion notes: [Computer Architecture](../computer-architecture/computer-architecture.md) (§9 Networking Hardware covers the NIC, DMA rings, and the kernel receive path), [REST API Best Practices](../software-engineering/rest-api-best-practices.md) (lives entirely at layer 7).

---

## Table of Contents

1. [The Problem Before Layering](#1--the-problem-before-layering)
2. [What Layering Actually Buys You](#2--what-layering-actually-buys-you)
3. [A World Without These Models](#3--a-world-without-these-models)
4. [The OSI Reference Model](#4--the-osi-reference-model)
5. [The TCP/IP Model](#5--the-tcpip-model)
6. [OSI vs TCP/IP: Mapping, Differences, and Why Both Survive](#6--osi-vs-tcpip-mapping-differences-and-why-both-survive)
7. [Encapsulation: How a Byte Becomes a Frame](#7--encapsulation-how-a-byte-becomes-a-frame)
8. [Layer-by-Layer Map to Real Applications and Products](#8--layer-by-layer-map-to-real-applications-and-products)
9. [End-to-End Walkthroughs Through the Stack](#9--end-to-end-walkthroughs-through-the-stack)
10. [The Application Developer's View: Which Layers Do You Actually Touch?](#10--the-application-developers-view-which-layers-do-you-actually-touch)
11. [Debugging With the Model: One Tool per Layer](#11--debugging-with-the-model-one-tool-per-layer)
12. [Where the Models Leak: Layer Violations in the Real World](#12--where-the-models-leak-layer-violations-in-the-real-world)
13. [Mental Models Worth Keeping](#13--mental-models-worth-keeping)
14. [Further Reading](#14--further-reading)

---

# 1 — The Problem Before Layering

In the 1970s every serious computer vendor shipped its own complete networking stack, top to bottom, and none of them talked to each other.

| Vendor stack | Owner | Era | Talked to |
|---|---|---|---|
| SNA (Systems Network Architecture) | IBM | 1974 | IBM mainframes and terminals |
| DECnet | Digital Equipment Corp | 1975 | VAX / PDP machines |
| XNS (Xerox Network Systems) | Xerox PARC | late 1970s | Xerox workstations; later cloned as Novell IPX |
| AppleTalk | Apple | 1985 | Macintoshes and LaserWriters |
| NetBEUI / NetBIOS | IBM, then Microsoft | 1985 | PCs on a single LAN segment |
| ARPANET NCP, then TCP/IP | US DoD / universities | 1970 → 1983 | research networks |

Each of these was a *monolith*: the wire signalling, the addressing, the error recovery, the session handling, and the application encoding were designed together and could not be separated. Consequences a customer of the time lived with:

- **Vendor lock-in was total.** An IBM 3270 terminal could not attach to a DEC VAX. Buying a second vendor's machine meant buying a second network.
- **Every application re-implemented the plumbing.** A file-transfer program and a terminal program on the same machine might carry their own retransmission logic, their own framing, their own addressing.
- **Hardware and software were welded together.** Moving from coaxial cable to twisted pair, or from 10 Mbit/s to 100 Mbit/s, could require changing application-visible behaviour because nothing insulated the application from the physical medium.
- **Interoperability meant gateways that translated everything.** Because the stacks disagreed at every level, a "gateway" between two networks was a full application-level translator that understood both ends completely, and it broke whenever either side changed.
- **Nobody could specialise.** A cable engineer, a router designer, and an application programmer had to understand the whole thing, because a change at one level rippled through all of them.

Two responses emerged in parallel:

1. **The pragmatic one.** Vint Cerf and Bob Kahn's 1974 paper *"A Protocol for Packet Network Intercommunication"* split ARPANET's monolithic Network Control Program into a host-to-host reliability protocol (TCP) over a minimal packet-forwarding protocol (IP). This became the TCP/IP suite, mandated on ARPANET on 1 January 1983 ("flag day") and formalised for hosts in RFC 1122/1123 in 1989.
2. **The formal one.** ISO and the CCITT (now ITU-T) began work in 1977 on a vendor-neutral *Open Systems Interconnection* architecture, published as ISO 7498 in 1984 (and as ITU-T X.200). It defined **seven layers**, each with a precise service definition, and a full protocol suite to implement them (X.25, X.400 mail, X.500 directory, FTAM file transfer, and so on).

The OSI *protocols* lost the market to TCP/IP in the early 1990s. The OSI *model* won the language: nobody says "the host-to-host layer" or "the internetwork layer", everybody says "layer 4" and "layer 3". Understanding both models, and how they map, is therefore unavoidable.

---

# 2 — What Layering Actually Buys You

Layering is the networking instance of a general engineering principle: **decompose a large problem into a stack of services, each of which uses only the service directly beneath it and provides a well-defined service to the one above.** Each layer on one machine converses *logically* with the same layer on the peer machine, using a **protocol**; *physically*, data only ever moves down the stack on the sender, across the wire, and up the stack on the receiver.

```
   Host A                                          Host B
 ┌─────────────┐   Layer-N protocol (logical)    ┌─────────────┐
 │  Layer N    │ ◄ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ► │  Layer N    │
 ├─────────────┤  service interface (real call)  ├─────────────┤
 │  Layer N-1  │ ◄ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ► │  Layer N-1  │
 ├─────────────┤                                 ├─────────────┤
 │     ...     │                                 │     ...     │
 ├─────────────┤                                 ├─────────────┤
 │  Layer 1    │ ═══════════ physical medium ════│  Layer 1    │
 └─────────────┘                                 └─────────────┘
```

The properties this buys, each of which is a concrete problem solved:

| Property | The problem it solves | Concrete example |
|---|---|---|
| **Separation of concerns** | Nobody has to understand the whole system | A Django developer never thinks about Ethernet CRCs; a NIC firmware engineer never thinks about JSON |
| **Interoperability by contract** | Vendor lock-in | Any device that speaks IP can forward packets for any application, regardless of who built either |
| **Independent evolution** | Changing one thing breaks everything | Wi-Fi 6, 5G, and 400G Ethernet were all deployed under an unchanged IPv4/TCP/HTTP stack; HTTP/3 was deployed over unchanged Wi-Fi |
| **Reuse** | Every application re-implements reliability, routing, framing | One TCP implementation in the kernel serves every process on the machine |
| **Substitutability** | Cannot swap a component | TCP ↔ QUIC under HTTP; Ethernet ↔ Wi-Fi ↔ LTE under IP; JSON ↔ Protobuf over HTTP |
| **Hiding heterogeneity** | The path crosses many different networks | A packet from a phone traverses LTE, fibre, an IX, a data-centre fabric, and a VM's virtual NIC. Only IP is common to all of them. |
| **Testability and fault isolation** | "It's broken" with no way to localise | Ping works (L3 OK), `telnet host 443` connects (L4 OK), TLS handshake fails (L6 problem) |
| **Standardisation surface** | Where do you write the RFC? | The IETF can standardise HTTP without saying anything about copper vs fibre |
| **Clear addressing at every scope** | One flat namespace for everything | MAC (this link) → IP (this internet) → port (this process) → URL (this resource) |

The cost, which is real and comes back in §12: every layer adds header bytes, a processing step, and a place where information is hidden from the layer that might need it. Layering is a trade of efficiency and global knowledge for manageability and evolution. In the 50 years since, that trade has been decisively vindicated: the Internet grew by roughly nine orders of magnitude in hosts and speed without a redesign.

---

# 3 — A World Without These Models

Take the properties above and remove them. This is not hypothetical; it is a description of 1978, and of any codebase that quietly reinvents the stack.

## 3.1 The N × M explosion

Suppose there are *N* kinds of application (web, mail, file transfer, video, remote shell, database access…) and *M* kinds of physical network (Ethernet, Wi-Fi, LTE, fibre, satellite, Bluetooth…). Without a common middle:

- Each application must contain code for each network: **N × M implementations**.
- Each new network technology requires touching **every application**.
- Each new application requires understanding **every network**.

With a common internetwork layer (IP) in the middle, this collapses to **N + M**: each application targets IP once, each network carries IP once. This is the "narrow waist" or **hourglass** of the Internet, and it is the single most important architectural fact about it:

```
             HTTP  SMTP  DNS  SSH  RTP  gRPC  BGP  NTP ...      ← many applications
                \    |    |    |    |    /    /    /
                 TCP      UDP      QUIC     SCTP                ← few transports
                       \   |   |   /
                            IP                                  ← ONE waist
                       /   |   |   \
             Ethernet  Wi-Fi  LTE/5G  PPP  DOCSIS  Fibre ...    ← many links
```

Anything can change above the waist or below it without coordination across the waist. That is why IPv6 (a change *at* the waist) has taken 25 years, while HTTP/3 and Wi-Fi 7 (changes away from it) took a few.

## 3.2 Every application carries its own stack

Without a shared transport, each program must solve, alone and inconsistently:

- packet loss detection and retransmission
- reordering
- flow control (do not overwhelm the receiver)
- congestion control (do not overwhelm the network; without this the network *collapses*. The 1986 ARPANET congestion collapse, when throughput fell by a factor of ~1000, is what produced Van Jacobson's TCP congestion control)
- multiplexing many conversations onto one machine
- fragmentation to fit link sizes
- error detection

Each would do it slightly differently, with different bugs, and none would be tuned against the others. The network as a whole would be unstable because there is no shared notion of fairness.

## 3.3 No independent innovation

Wi-Fi was standardised in 1997, long after the web existed. Because the web sat above IP and Wi-Fi sat below it, **no web server or browser was modified** to run over Wi-Fi. In a monolithic world, "web over wireless" would have been a new product. Equivalently: HTTP/2 (2015) and HTTP/3 (2022) were deployed without any changes to routers. That is only possible because routers never look above layer 3.

## 3.4 Debugging becomes guesswork

A layered stack gives you a **binary search over the failure**: is it the cable, the link, the address, the route, the port, the handshake, the encoding, or the application logic? Each layer has its own tool (§11). Without layers there is one symptom ("it doesn't work") and one tool (staring at the whole thing).

## 3.5 No division of labour or industry

The networking industry is *organised* by layer. Cable and optics companies (L1), switch ASIC vendors (L2), router vendors and ISPs (L3), operating-system kernels (L4), CDN and application companies (L5–L7), each with their own standards bodies (IEEE 802 for L1/L2, IETF for L3–L7, W3C above that). Remove the model and you remove the interfaces that let these be separate businesses, separate teams, and separate skill sets.

## 3.6 Applications leak the medium

Without a layer boundary, application behaviour depends on the physical network: message sizes tied to a particular frame length, timeouts tied to a particular cable's latency, addresses tied to a particular vendor's scheme. Moving the application to a different network means rewriting it. This is exactly what happened when organisations migrated from SNA to TCP/IP in the 1990s: it was a rewrite, not a reconfiguration.

---

# 4 — The OSI Reference Model

ISO 7498 (1984) defines seven layers. Each layer is specified by (a) the **service** it offers to the layer above, (b) the **protocol** peer entities use, and (c) the name of the unit of data it handles, the **PDU** (Protocol Data Unit). Memorise it top-down with *"All People Seem To Need Data Processing"* or bottom-up with *"Please Do Not Throw Sausage Pizza Away"*; then forget the mnemonic and learn what each one is *for*.

```
 ┌───┬──────────────┬─────────────────────────────────────────────┬────────────┐
 │ 7 │ Application  │ the network-facing part of the program      │ data       │
 ├───┼──────────────┼─────────────────────────────────────────────┼────────────┤
 │ 6 │ Presentation │ syntax: encoding, compression, encryption   │ data       │  "host layers"
 ├───┼──────────────┼─────────────────────────────────────────────┼────────────┤  (end systems only)
 │ 5 │ Session      │ dialogue control, checkpoints, resumption   │ data       │
 ├───┼──────────────┼─────────────────────────────────────────────┼────────────┤
 │ 4 │ Transport    │ process-to-process delivery, reliability    │ segment    │
 ├───┼──────────────┼─────────────────────────────────────────────┼────────────┤
 │ 3 │ Network      │ host-to-host across networks: addressing,   │ packet     │
 │   │              │ routing                                     │            │  "media layers"
 ├───┼──────────────┼─────────────────────────────────────────────┼────────────┤  (every hop)
 │ 2 │ Data Link    │ node-to-node on one link: framing, MAC      │ frame      │
 ├───┼──────────────┼─────────────────────────────────────────────┼────────────┤
 │ 1 │ Physical     │ bits on a medium: voltages, light, radio    │ bit/symbol │
 └───┴──────────────┴─────────────────────────────────────────────┴────────────┘
```

The single most useful distinction in the diagram is the split between **media layers (1–3)**, which exist on every device along the path, and **host layers (4–7)**, which exist only at the two end systems. A router has layers 1–3. A switch has layers 1–2. Your laptop and the server have all seven. This is the *end-to-end principle* drawn as a picture.

## 4.1 Layer 1 — Physical

**Job:** move raw bits between two directly connected devices. Defines the medium (copper, fibre, air), connectors, voltage levels or light wavelengths, modulation, bit timing, and line coding. It has no concept of a "frame" or an "address": it just turns 1s and 0s into signals and back.

| Aspect | Examples |
|---|---|
| Media | Cat6/Cat6a twisted pair, single-mode / multi-mode fibre, coaxial, 2.4/5/6 GHz radio, mmWave |
| Standards | IEEE 802.3 PHY sub-layers (1000BASE-T, 10GBASE-SR, 400GBASE-DR4), 802.11 PHY (OFDM, OFDMA in Wi-Fi 6), USB, Bluetooth radio, SONET/SDH, DWDM, 3GPP NR physical layer |
| Encoding | Manchester (old Ethernet), 8b/10b, 64b/66b, PAM4 (100G+ lanes), QAM-1024 (Wi-Fi 6), NRZ |
| Devices | Cables, transceivers (SFP+/QSFP), hubs, repeaters, media converters, antennas |
| Failure looks like | Link light off, CRC errors climbing, "no carrier", Wi-Fi signal too weak to hold a modulation rate |

## 4.2 Layer 2 — Data Link

**Job:** deliver frames between nodes on the **same physical link or LAN**, and decide who may transmit when (medium access control). Provides framing (where does one frame end?), hardware addressing (MAC), error detection (CRC), and on some media, flow control and retransmission.

IEEE splits it into two sub-layers: **MAC** (Media Access Control: addressing, CSMA/CD for classic Ethernet, CSMA/CA for Wi-Fi) and **LLC** (Logical Link Control, 802.2: identifies the layer-3 protocol; largely vestigial on Ethernet II, which uses the EtherType field instead).

| Aspect | Examples |
|---|---|
| Protocols | Ethernet (802.3), Wi-Fi (802.11 MAC), PPP / PPPoE, 802.1Q VLAN tagging, ARP (usually placed here; strictly it glues L2 and L3), LACP, STP/RSTP, LLDP, 802.1X port authentication, MACsec |
| Addressing | 48-bit MAC address, meaningful only on the local segment; rewritten at every router hop |
| Devices | Switches, bridges, wireless access points, NICs |
| Data-centre relevance | VXLAN encapsulates L2 frames in UDP to stretch a LAN across an L3 fabric; every Kubernetes CNI plugin decides whether pods share an L2 domain |
| Failure looks like | Wrong VLAN, duplicate MAC, spanning-tree loop, ARP cache poisoning, MTU mismatch on a trunk |

## 4.3 Layer 3 — Network

**Job:** deliver packets from a source host to a destination host **across multiple networks**, which requires (a) an address scheme that is globally meaningful, not just link-local, and (b) **routing**: choosing the next hop at each intermediate router. Also handles fragmentation when a packet is larger than the next link's MTU (IPv4 routers fragment; IPv6 routers don't, they return an ICMP "packet too big").

The Internet's layer 3, IP, is deliberately **unreliable and connectionless**: best-effort datagrams, no guarantees of delivery, order, or non-duplication. That minimalism is what lets it run over anything.

| Aspect | Examples |
|---|---|
| Protocols | IPv4, IPv6, ICMP / ICMPv6 (ping, traceroute, error reporting, neighbour discovery), IPsec (AH/ESP), IGMP |
| Routing protocols (control plane; they run *over* IP but decide L3 behaviour) | OSPF, IS-IS, BGP (the protocol that holds the global Internet together), RIP, EIGRP |
| Addressing | 32-bit IPv4, 128-bit IPv6, CIDR prefixes, subnets, public vs RFC 1918 private, NAT |
| Devices | Routers, layer-3 switches, firewalls (packet filters), the Linux kernel routing table on every host |
| Cloud relevance | A VPC is an L3 construct; route tables, NAT gateways, internet gateways, peering, security groups (mostly L3/L4 filters) |
| Failure looks like | No route to host, TTL exceeded, asymmetric routing breaking stateful firewalls, overlapping subnets after a merger, BGP route leak |

## 4.4 Layer 4 — Transport

**Job:** deliver data **process-to-process**, not just host-to-host. Layer 3 gets a packet to the right machine; layer 4 gets it to the right *program* on that machine (via **ports**) and, depending on the protocol, adds reliability, ordering, flow control, and congestion control. Also provides **segmentation**: breaking an application's byte stream into segments sized for the network.

This is the layer where the fundamental design choice of the Internet is expressed: put the intelligence at the **end hosts** (TCP in the OS kernel) and keep the middle (IP routers) dumb and fast.

| Aspect | Examples |
|---|---|
| Protocols | **TCP** (reliable, ordered byte stream; connection-oriented; congestion-controlled), **UDP** (unreliable datagrams; no connection; minimal header), **QUIC** (reliable, multiplexed, encrypted streams over UDP; used by HTTP/3), SCTP (multi-streaming, used in telecom signalling and WebRTC data channels), DCCP |
| Addressing | 16-bit port numbers; a *socket* = (IP, port); a *connection* = the 4-tuple (src IP, src port, dst IP, dst port) plus protocol |
| Devices | End hosts. Also **L4 load balancers** (AWS NLB, Linux IPVS, HAProxy in TCP mode, Maglev), stateful firewalls tracking connections, NAT devices rewriting ports |
| Key mechanisms | 3-way handshake, sequence/ACK numbers, sliding window, retransmission timers, slow start, CUBIC / BBR congestion control, Nagle, delayed ACK, selective ACK |
| Failure looks like | Connection refused (nothing listening on the port), connection timed out (SYN dropped by a firewall), RST storms, port exhaustion behind NAT, head-of-line blocking, bufferbloat |

## 4.5 Layer 5 — Session

**Job:** establish, manage, and tear down **dialogues** between applications: who talks when (half/full duplex), synchronisation points (checkpoints so a long transfer can resume from the last checkpoint rather than the start), and grouping related exchanges into one logical session.

OSI made this a separate layer because its designers were thinking about long-running mainframe transactions. In the TCP/IP world there is no distinct session layer; its responsibilities are scattered across the transport (a TCP connection *is* a session), the application (HTTP cookies, JWTs, database sessions), and a handful of specific protocols. The concept is still useful as vocabulary.

| Aspect | Examples |
|---|---|
| Protocols with explicit session semantics | NetBIOS session service, RPC binding (SunRPC, DCE-RPC), PPTP / L2TP tunnel control, SIP (sets up and tears down VoIP calls), RTCP (session control for RTP), SOCKS, TLS *session* resumption tickets, H.245 |
| Application-level session concepts | HTTP cookies and server-side session stores, OAuth/OIDC tokens, WebSocket lifecycle, gRPC channels, database connection sessions with transaction state, SSH channel multiplexing, MQTT persistent sessions, resumable uploads (tus, S3 multipart) |
| Failure looks like | Session expired, sticky-session load balancing sending a user to the wrong backend, a resumable upload restarting from zero |

## 4.6 Layer 6 — Presentation

**Job:** make sure both sides interpret the bytes the same way. Handles **data representation** (character sets, integer endianness, floating point formats, structured serialisation), **compression**, and, in OSI's placement, **encryption**. OSI defined ASN.1 as the abstract syntax and BER/DER as the transfer encodings; ASN.1 survives today inside X.509 certificates, SNMP, LDAP, Kerberos, and 5G signalling.

In the TCP/IP world this layer's functions are done by libraries linked into the application, not by a protocol layer. TLS is conventionally placed here (or between 4 and 7), since it transforms the representation of the byte stream without changing the application semantics.

| Aspect | Examples |
|---|---|
| Serialisation formats | JSON, Protocol Buffers, Apache Avro, Thrift, MessagePack, CBOR, XML, ASN.1 DER, FlatBuffers, Cap'n Proto, Parquet (at rest), Arrow (in memory / Flight) |
| Character encoding | ASCII, UTF-8, UTF-16, ISO-8859-1, EBCDIC; `Content-Type: text/html; charset=utf-8` is a presentation-layer statement |
| Compression | gzip, Brotli, zstd (`Content-Encoding`), HPACK/QPACK header compression, JPEG / H.264 / Opus (media codecs are presentation) |
| Encryption | TLS 1.2 / 1.3, DTLS (for UDP), SSH's transport-layer encryption, SRTP for media |
| Developer touchpoints | `JSON.parse`, `proto.Marshal`, `Content-Type` and `Accept` negotiation, `Content-Encoding`, `.proto` / OpenAPI schemas, `SSLContext` / `tls.Config` |
| Failure looks like | Mojibake (wrong charset), "Unexpected token < in JSON" (got HTML, expected JSON), certificate validation failure, schema-evolution break between producer and consumer, endianness bug in a binary protocol |

## 4.7 Layer 7 — Application

**Job:** the protocols that applications themselves speak: the semantics of requests and responses, resource naming, methods, status codes, message formats. This is *not* "the application"; it is the **network-facing protocol** the application uses. A browser is an application; HTTP is its layer-7 protocol.

| Category | Protocols / products |
|---|---|
| Web | HTTP/1.1, HTTP/2, HTTP/3, WebSocket, WebDAV, REST and GraphQL conventions on top of HTTP, gRPC (HTTP/2 + Protobuf), Server-Sent Events |
| Naming | DNS, mDNS / DNS-SD, LLMNR |
| Mail | SMTP, IMAP, POP3 |
| Remote access | SSH, Telnet, RDP, VNC |
| File transfer & sharing | FTP / SFTP / FTPS, SMB/CIFS, NFS, TFTP, BitTorrent, rsync |
| Infrastructure | DHCP, NTP / PTP, SNMP, Syslog, LDAP, Kerberos, RADIUS |
| Messaging & streaming | AMQP (RabbitMQ), MQTT (IoT), Kafka wire protocol, NATS, STOMP, XMPP, SIP + RTP (VoIP, WebRTC signalling and media) |
| Databases | PostgreSQL wire protocol, MySQL protocol, MongoDB wire protocol, Redis RESP, Cassandra CQL native protocol, TDS (SQL Server) |
| Devices | Web servers (nginx, Apache, Envoy), API gateways, **L7 load balancers** (AWS ALB, Cloudflare, Envoy, HAProxy in HTTP mode), web application firewalls, proxies, CDN edges |
| Failure looks like | HTTP 4xx/5xx, DNS NXDOMAIN, SMTP 550, authentication failure, malformed request, API version mismatch |

---

# 5 — The TCP/IP Model

The TCP/IP model (also called the *Internet model* or *DoD model*) was not designed on a whiteboard first; it was **extracted** from a working protocol suite. RFC 1122 (1989), *Requirements for Internet Hosts*, describes four layers. Many textbooks (Kurose & Ross, Tanenbaum's later editions) present a five-layer hybrid that splits the bottom layer in two; that hybrid is what most engineers actually carry in their heads.

```
   RFC 1122 (4 layers)            Common 5-layer hybrid           OSI equivalents
 ┌────────────────────┐        ┌────────────────────┐
 │    Application     │        │    Application     │            7 + 6 + 5
 ├────────────────────┤        ├────────────────────┤
 │     Transport      │        │     Transport      │            4
 ├────────────────────┤        ├────────────────────┤
 │      Internet      │        │      Internet      │            3
 ├────────────────────┤        ├────────────────────┤
 │        Link        │        │        Link        │            2
 │  (incl. physical)  │        ├────────────────────┤
 │                    │        │      Physical      │            1
 └────────────────────┘        └────────────────────┘
```

## 5.1 The four layers

| Layer | Responsibility | Core protocols | PDU |
|---|---|---|---|
| **Application** | Everything above the transport: protocol semantics, encoding, sessions, and encryption libraries. Lives in user space (mostly). | HTTP, DNS, SMTP, SSH, TLS, gRPC, and everything in OSI 5–7 | message |
| **Transport** | Process-to-process delivery via ports; reliability and congestion control if TCP/QUIC; nothing if UDP. Lives in the kernel (TCP/UDP) or a user-space library (QUIC). | TCP, UDP, QUIC, SCTP | segment (TCP) / datagram (UDP) |
| **Internet** | Host-to-host delivery across any number of networks with a single global address space and best-effort forwarding. The narrow waist. | IPv4, IPv6, ICMP, IPsec; routing protocols decide its tables | packet |
| **Link** (network access / network interface) | Get a packet onto the directly attached network, whatever it is. RFC 1122 deliberately says almost nothing here: it is "whatever the underlying network provides." | Ethernet, Wi-Fi, PPP, ARP/NDP, cellular, and their physical layers | frame |

## 5.2 The design principles baked into it

These are not in the OSI model and they matter more than the layer count:

- **End-to-end principle** (Saltzer, Reed, Clark 1984): functions like reliability and security should be implemented at the end hosts, because only they know what "correct" means; the network should not try to do them on the hosts' behalf. TCP in the endpoints, IP in the routers.
- **Best-effort datagram network:** IP may drop, reorder, duplicate, or delay packets. Anything that needs better than that builds it on top. This is what made IP cheap enough to run everywhere.
- **Robustness principle** (RFC 1122 §1.2.2, "Postel's law"): be conservative in what you send, liberal in what you accept. Credited with the Internet's interoperability and, later, blamed for protocol ossification.
- **Rough consensus and running code:** the IETF standardises what already works. The model documents practice; OSI attempted to prescribe it.
- **No strict layering:** RFC 1122 explicitly allows layers to be bypassed or merged when useful. ICMP is carried inside IP yet is considered part of the Internet layer. Routing protocols run over TCP or UDP yet configure layer 3.

## 5.3 Why OSI 5, 6, 7 collapse into one layer

In TCP/IP, session, presentation, and application functions are all performed **inside the application process**, by libraries it links, not by separate protocol entities with their own headers. When a Go service calls `http.Post` with a JSON body over TLS:

- JSON encoding (presentation) is a function call.
- TLS (presentation/session) is a library the HTTP client wraps around the socket.
- The HTTP request (application) is written to that TLS writer.
- Cookies or bearer tokens (session) are just HTTP headers.

There are no layer boundaries between these in the sense of a header being added by an independent entity; they are all "the application" as far as the OS and the network are concerned. That is why the TCP/IP model has one layer where OSI has three, and why the OSI names for 5 and 6 are still useful *descriptively* (to say "that's a presentation problem") even though nothing enforces them.

---

# 6 — OSI vs TCP/IP: Mapping, Differences, and Why Both Survive

## 6.1 Side-by-side mapping

| OSI | OSI layer | TCP/IP layer | Where it runs on a Linux host | Representative protocols | Address / identifier |
|---|---|---|---|---|---|
| 7 | Application | Application | User-space process | HTTP, DNS, SMTP, SSH, gRPC | URL, hostname, email address |
| 6 | Presentation | Application | User-space library | TLS, JSON, Protobuf, gzip, UTF-8 | (content-type / cipher suite) |
| 5 | Session | Application | User-space library / app state | cookies, tokens, SIP, RPC binding | session ID |
| 4 | Transport | Transport | Kernel (`net/ipv4/tcp*.c`), or user-space for QUIC | TCP, UDP, QUIC, SCTP | port number |
| 3 | Network | Internet | Kernel (`net/ipv4`, `net/ipv6`, netfilter) | IPv4, IPv6, ICMP, IPsec | IP address |
| 2 | Data Link | Link | Kernel driver + NIC firmware | Ethernet, Wi-Fi, ARP, VLAN | MAC address |
| 1 | Physical | Link (or Physical in 5-layer) | NIC PHY, cable, transceiver | 1000BASE-T, 802.11ax PHY | (none) |

## 6.2 The real differences

| Dimension | OSI | TCP/IP |
|---|---|---|
| Origin | Designed by committee (ISO/CCITT) *before* implementation | Extracted from a running network *after* implementation |
| Layer count | 7 | 4 (or 5) |
| Layer boundaries | Strict; each layer is a distinct entity with its own protocol and header | Loose; upper three collapsed; RFC 1122 permits bypassing |
| Services vs protocols | Cleanly separates the service definition, the interface, and the protocol; this is its lasting intellectual contribution | Protocols came first; the "model" describes them |
| Connection model | Originally connection-oriented at L3 (X.25 virtual circuits); connectionless added later | Connectionless at L3 from day one; connections are an L4 concept |
| Security placement | Presentation layer | Not in the original model; TLS/IPsec/802.1X added at whichever layer suited |
| Session/presentation | Explicit layers | Absorbed into the application |
| Physical layer | Explicit | Deliberately unspecified in RFC 1122 |
| Protocol suite | X.25, CLNP, TP0–TP4, X.400, X.500, FTAM: essentially dead outside IS-IS (still a dominant ISP interior routing protocol, and it runs directly over layer 2, not IP), ASN.1, and X.509 | The Internet |
| Used for | Teaching, vocabulary, troubleshooting, product categorisation ("L4 vs L7 load balancer") | Actually building and running networks |

## 6.3 Why OSI protocols lost

- **Timing:** TCP/IP was running and free (BSD Unix shipped it in 1983); OSI specifications were published in 1984 and implementations lagged years behind.
- **Complexity:** seven layers with five transport classes and multiple options per layer meant two "conforming" OSI stacks often could not interoperate. GOSIP (US government mandate for OSI, 1990) was quietly dropped in 1995.
- **Cost:** ISO standards were paid documents; RFCs were free. Students and startups built on what they could read.
- **Politics:** OSI was driven by telecom carriers who wanted circuit-like, billable virtual circuits; the datagram model was cheaper and fit the computing industry's needs better.

## 6.4 Why the OSI model survives anyway

Because it is the better **teaching and diagnostic tool**. The seven-layer split gives a name to distinctions the four-layer model blurs:

- "That's a **layer 2** problem" (wrong VLAN) vs "a **layer 1** problem" (bad cable) is a distinction TCP/IP's single Link layer cannot express.
- "**L4** load balancer" vs "**L7** load balancer" is the industry's standard product vocabulary.
- "**Layer 6** bug" (charset/serialisation mismatch) vs "**layer 7** bug" (wrong HTTP method) is a useful distinction in an incident review even though both are "application" in TCP/IP.
- The ironic "**layer 8** problem" (the human) shows how deeply the numbering has embedded itself.

The practical synthesis, and the one this document uses: **think in OSI numbers, build with TCP/IP protocols.**

---

# 7 — Encapsulation: How a Byte Becomes a Frame

Each layer treats what it receives from above as an opaque payload and wraps it in its own header (and, at layer 2, a trailer). The receiver peels them off in reverse. The nesting is literal: an Ethernet frame *contains* an IP packet which *contains* a TCP segment which *contains* TLS records which *contain* HTTP bytes.

```
 Application  │                                            │ GET /api/users HTTP/1.1 ... │
              │                                            └─────────────────────────────┘
 Presentation │                                 │ TLS rec hdr │ encrypted(HTTP)            │
              │                                 └─────────────┴────────────────────────────┘
 Transport    │                      │ TCP hdr  │            TLS record(s)                 │   segment
              │                      └──────────┴──────────────────────────────────────────┘
 Network      │           │ IP hdr   │                 TCP segment                         │   packet
              │           └──────────┴─────────────────────────────────────────────────────┘
 Data link    │ Eth hdr   │                     IP packet                                  │ FCS │ frame
              └───────────┴────────────────────────────────────────────────────────────────┴─────┘
 Physical     ░░░░░░░░░░░░░░░░░░░░░░░░ preamble + symbols on the wire ░░░░░░░░░░░░░░░░░░░░░░░░░░░░
```

Typical header sizes on the wire for an HTTPS request over Ethernet:

| Layer | Header | Bytes | Key fields |
|---|---|---|---|
| 2 | Ethernet II | 14 + 4 FCS (+4 if VLAN-tagged) | dst MAC, src MAC, EtherType (0x0800 = IPv4, 0x86DD = IPv6) |
| 3 | IPv4 | 20 (no options) | version, total length, TTL, protocol (6 = TCP, 17 = UDP), src IP, dst IP, checksum |
| 3 | IPv6 | 40 | flow label, payload length, next header, hop limit, 128-bit src/dst |
| 4 | TCP | 20–60 | src port, dst port, seq, ack, flags (SYN/ACK/FIN/RST), window, options (MSS, SACK, timestamps) |
| 4 | UDP | 8 | src port, dst port, length, checksum |
| 6 | TLS 1.3 record | 5 + 16 AEAD tag | content type, version, length |
| 7 | HTTP/1.1 | variable text | request line, headers |

Two consequences developers hit constantly:

- **MTU and MSS.** Ethernet's default MTU is 1500 bytes. After 20 (IP) + 20 (TCP) that leaves an MSS of 1460 bytes of TCP payload. Tunnels (VPN, VXLAN, GRE) add their own headers *inside* the 1500, so the effective MTU drops; a misconfigured tunnel yields the classic "ping works, SSH hangs after login" symptom (small packets pass, full-size ones are silently dropped when Path MTU Discovery is blocked).
- **Every layer's header is overhead.** A 64-byte payload in a TCP/IPv4/Ethernet frame is 118 bytes on the wire (46% overhead), which is why small-message protocols (DNS, game state, telemetry) use UDP's 8-byte header and why HTTP/2 introduced HPACK.

**The demultiplexing chain** on receipt is the encapsulation in reverse, and every field that names "the next protocol" is a pointer up the stack:

```
EtherType 0x0800 → IP protocol 6 → TCP dst port 443 → TLS ALPN "h2" → HTTP/2 :path /api/users → your handler
```

---

# 8 — Layer-by-Layer Map to Real Applications and Products

This section answers "where does X live?" for the things an engineer actually deploys. Assignments follow common industry usage; where something spans layers, it is listed at the layer of its *primary* function and the overlap is noted in §12.

## 8.1 One table, every layer

| Layer | What you deploy, configure, or code | Cloud / infra products | Everyday consumer things |
|---|---|---|---|
| **7 Application** | REST/GraphQL APIs, gRPC services, web servers, DNS zones, SMTP relays, Kafka topics, SQL over the DB wire protocol, OAuth flows, webhooks | AWS ALB / API Gateway / CloudFront, GCP HTTP(S) LB, Cloudflare Workers & WAF, nginx, Envoy, Kong, Route 53 / Cloud DNS, SES, Istio's L7 policy | Browser, Gmail, Slack, Zoom's signalling, Spotify's API calls, a smart bulb's MQTT messages |
| **6 Presentation** | `Content-Type` negotiation, JSON/Protobuf/Avro schemas, gzip/Brotli, TLS termination, certificates, UTF-8 handling, image/video codecs | ACM / Let's Encrypt certs, Schema Registry (Confluent), TLS termination at ALB/CloudFront, KMS envelope encryption of payloads | The padlock icon, an emoji rendering correctly, a JPEG loading, Netflix's H.265 stream |
| **5 Session** | Login sessions, cookies, JWT refresh, WebSocket lifecycle, DB connection pools, sticky sessions, resumable uploads, RPC channel state | ALB sticky sessions, ElastiCache / Redis session store, Cognito / Auth0 tokens, RDS Proxy (pools DB sessions), SIP trunks in Amazon Connect | Staying logged in, a Zoom call that survives switching Wi-Fi → LTE, a resumed Google Drive upload |
| **4 Transport** | Choosing TCP vs UDP vs QUIC, port numbers, socket options (`SO_REUSEPORT`, `TCP_NODELAY`, keepalives), timeouts, connection pooling, backlog sizing, kernel tuning (`tcp_congestion_control`, buffer sizes) | AWS NLB, GCP TCP/UDP LB, Linux IPVS / kube-proxy, security groups' port rules, Cloudflare Spectrum, HAProxy in `mode tcp` | Whether a video call stutters (UDP) vs a download pauses (TCP retransmit), online game netcode, a torrent client |
| **3 Network** | IP addressing plans, subnets, VPC route tables, NAT, firewall rules, VPN/IPsec, BGP peering, pod CIDRs, IPv6 rollout, ICMP-based health checks | VPC / VNet, Transit Gateway, NAT Gateway, Internet Gateway, Direct Connect / ExpressRoute, Calico / Cilium (L3 CNI), Cloud Router (BGP), AWS Network Firewall | Home router, "obtain IP automatically", VPN apps, why two devices on different networks can't see each other |
| **2 Data Link** | VLANs, switch port config, MAC address tables, LACP bonding, Wi-Fi SSIDs and channel plans, VXLAN overlays, ARP tables, 802.1X NAC | Nitro / virtual NICs (ENI), VXLAN in every cloud overlay, Wi-Fi controllers (Meraki, Aruba), top-of-rack switches (Arista, Cisco Nexus), Open vSwitch, SR-IOV | Wi-Fi password, "connected, no internet", Bluetooth pairing, Ethernet cable into the TV |
| **1 Physical** | Cabling standards, optics choice (SR/LR/DR), fibre plant, PoE budgets, RF surveys, link speed/duplex, structured cabling | Data-centre cross-connects, DWDM long-haul, submarine cables, 5G radio, AWS Outposts racks, Snowmobile (truck-as-L1) | Cat6 cable, Wi-Fi signal bars, HDMI/USB cables, the fibre to your house |

## 8.2 Where common software components sit

| Component | Primary layer | Notes |
|---|---|---|
| Browser (Chrome) | 7, plus its own 6 (TLS via BoringSSL) and 4 (QUIC in user space) | Chrome ships a full QUIC transport in the process |
| `curl` | 7 (HTTP) + 6 (TLS) | `curl -v` prints L3/L4 connect, L6 handshake, L7 request: it is a stack tour |
| nginx / Apache | 7 | Also terminates TLS (6) and accepts TCP (4) via the kernel |
| Envoy / Istio sidecar | 7 (HTTP filters) and 4 (TCP proxy) | Service mesh moves 5–7 concerns (retries, auth, mTLS) out of app code |
| HAProxy | 4 or 7 depending on `mode` | The canonical example of the L4/L7 distinction |
| Kubernetes Service (ClusterIP) | 4 | kube-proxy / IPVS rewrites (IP, port); knows nothing about HTTP |
| Kubernetes Ingress / Gateway API | 7 | Routes on Host and path |
| Kubernetes NetworkPolicy | 3/4 | Selects by IP/pod and port |
| Kubernetes CNI (Calico, Cilium, Flannel) | 2/3 | Cilium uses eBPF and also does L7 policy |
| Docker bridge network | 2 (`docker0` is a Linux bridge) + 3 (NAT via iptables) | |
| WireGuard / OpenVPN / IPsec | 3 (tunnels IP packets) | OpenVPN's *transport* is UDP or TCP (4); the tunnelled payload is L3 |
| SSH | 7 (protocol), 6 (its own encryption), 5 (channels) | `ssh -L` port forwarding tunnels arbitrary L4 streams |
| PostgreSQL / MySQL client | 7 (wire protocol) over 4 (TCP) with optional 6 (TLS) | Connection pools (PgBouncer) are session-layer infrastructure |
| Kafka client | 7 (Kafka protocol), with its own batching and compression (6) | Consumer groups are a session concept |
| Redis | 7 (RESP protocol) | Pipelining is an L7 optimisation over one L4 connection |
| gRPC | 7 (HTTP/2 framing + method semantics) + 6 (Protobuf) | Deadlines and metadata ride in HTTP/2 headers |
| WebRTC | 7 (signalling, app-defined), 6 (SRTP/DTLS), 5 (SDP/ICE session), 4 (UDP/SCTP) | The most layer-spanning thing a web developer touches |
| DNS resolver (`systemd-resolved`, Unbound) | 7 | Usually UDP/53, TCP fallback, DoH/DoT wrap it in TLS/HTTPS |
| DHCP client | 7 (protocol), but *configures* 3 | Runs over UDP before the host has an IP address |
| ARP / NDP | 2/3 boundary | Maps L3 address → L2 address |
| Wireshark | Displays all of them | `tcpdump` captures at L2 |

---

# 9 — End-to-End Walkthroughs Through the Stack

Reading the models is one thing; watching a request fall through every layer is what makes them stick. Each walkthrough names the layer at each step.

## 9.1 A browser loads `https://api.example.com/users/42`

```
 Layer 7  Browser builds:  GET /users/42 HTTP/2   :authority api.example.com   accept: application/json
          ↓ but first it needs an IP:
 Layer 7  DNS query  A? api.example.com  → (over UDP/53, L4, over IP, L3) → stub resolver → recursive resolver → 203.0.113.10
 Layer 5  Browser checks: existing HTTP/2 connection to that origin in the pool? If yes, reuse (session reuse). If no:
 Layer 4  connect(203.0.113.10:443)  → kernel sends SYN, gets SYN-ACK, sends ACK (3-way handshake). (QUIC would skip this.)
 Layer 3  Each of those TCP segments is wrapped in an IP packet: src 192.168.1.5 → dst 203.0.113.10, TTL 64
 Layer 2  Host ARPs (or checks cache) for the default gateway's MAC; frame: src laptop-MAC → dst router-MAC, EtherType 0x0800
 Layer 1  NIC encodes the frame as symbols on Wi-Fi 6 OFDMA or Cat6 copper
          ── every router on the path: L1 → L2 (strip frame) → L3 (lookup dst prefix, decrement TTL) → L2 (new frame) → L1 ──
          ── NAT at the home router rewrites src 192.168.1.5:51234 → 198.51.100.7:40001 (a layer 3/4 mutation) ──
 Layer 6  TLS 1.3 handshake: ClientHello (SNI = api.example.com, ALPN = h2), ServerHello, certificate chain, key exchange. 1 RTT.
 Layer 7  HTTP/2 stream 1: HEADERS frame with the request; server responds HEADERS + DATA
 Layer 6  Response body is gzip'd JSON, UTF-8 → browser inflates and parses
 Layer 5  Set-Cookie: session=...; browser stores it for the next request
 Layer 7  JavaScript receives a parsed object
```

Where the latency goes, per layer, for a first connection over a 50 ms RTT path: DNS ~1 RTT (if not cached), TCP handshake 1 RTT, TLS 1.3 handshake 1 RTT, request/response 1 RTT. Four round trips ≈ 200 ms before the first byte of JSON. HTTP/3 over QUIC merges the transport and TLS handshakes into one RTT (0-RTT on resumption), which is a **layer 4 + 6 optimisation** and why it matters to a **layer 7** developer.

## 9.2 A Zoom / Google Meet call

| Step | Layer | Mechanism |
|---|---|---|
| Sign in, join meeting, negotiate who is in the room | 7 | HTTPS / WebSocket to the vendor's signalling servers |
| Describe media capabilities and candidate addresses | 5 | SDP offer/answer; ICE gathers host/STUN/TURN candidates (NAT traversal, an L3/L4 problem solved with L5/L7 help) |
| Encrypt media and key exchange | 6 | DTLS handshake, then SRTP |
| Carry audio/video | 4 | **UDP**: a lost packet is worthless 100 ms later, so retransmission (TCP) would only add delay; RTP adds sequence numbers and timestamps at L7/L5 |
| Compress media | 6 | Opus audio, VP9/AV1/H.264 video, adapted to bandwidth |
| Get through the network | 3 | Usually via a relay (SFU) in the vendor's cloud because two clients behind NATs cannot reach each other directly |
| Switch from Wi-Fi to LTE mid-call | 2/1 changes, 3 changes (new IP), 5 preserved | ICE restart re-negotiates candidates; the *session* survives because it is identified above the transport |

Every "why is my call bad?" question maps to a layer: choppy audio with a good link = congestion at 3/4; garbled audio = codec/bitrate at 6; can't join = 7 signalling or 5 ICE failure; drops when the microwave runs = 1.

## 9.3 A microservice calls another inside Kubernetes

```
 L7  Service A:  grpcClient.GetUser(ctx, req)          → HTTP/2 POST /users.UserService/GetUser
 L6  Protobuf marshal; (with Istio) sidecar adds mTLS
 L7  DNS: user-svc.default.svc.cluster.local → CoreDNS → ClusterIP 10.96.12.5 (virtual, exists in no NIC)
 L4  connect(10.96.12.5:50051) → kube-proxy's IPVS/iptables rule DNATs to a real pod: 10.244.3.17:50051
 L3  packet routed by the CNI: same node → veth pair on the Linux bridge; other node → VXLAN/Geneve overlay (an L2 frame in a UDP/IP packet) or native BGP-advertised pod routes (Calico)
 L2  veth → bridge → host NIC; on the wire it is an ordinary Ethernet frame between nodes
 L1  ToR switch, fibre
 L3/4 NetworkPolicy (Cilium eBPF) checks src pod, dst port → allow
 L7  Pod B's Envoy sidecar: authz policy, retry budget, metrics; forwards to the app container on localhost
 L6/7 Protobuf unmarshal; handler runs
```

Note how many separate layer-3 and layer-4 identities one call passes through: cluster DNS name → ClusterIP → pod IP, and the overlay adds a whole second L2/L3/L4 stack around the first. Debugging "service unreachable" means knowing which of those hops you are looking at.

## 9.4 Sending an email

| Step | Layer |
|---|---|
| Compose in the client; MIME encodes attachments as base64, declares `Content-Type: text/html; charset=utf-8` | 6 |
| Client submits via SMTP on port 587 with STARTTLS, authenticates with a token | 7 (SMTP), 6 (TLS), 5 (auth session) |
| Sending server looks up the recipient domain's MX record | 7 (DNS) |
| Server-to-server SMTP over TCP/25, opportunistic TLS, DKIM signature verified, SPF checks sender IP | 7, 6, 3 (SPF is a policy about *IP addresses*) |
| Recipient's client fetches via IMAP over TLS on 993 | 7, 6 |
| Message is stored until the client's IMAP session next synchronises | 5 |

## 9.5 `ssh user@host`

| Step | Layer |
|---|---|
| Resolve `host` | 7 (DNS) |
| TCP connect to port 22 | 4 |
| SSH version exchange, key exchange, host-key check | 6 (SSH transport layer, encryption) |
| Password / public-key authentication | 5/7 (SSH authentication protocol) |
| Open a "session" channel; request a PTY; multiplex `-L` port forwards as additional channels over the *same* TCP connection | 5 (SSH connection protocol) |
| Keystrokes and terminal output | 7 |
| Wi-Fi drops for 30 s; SSH freezes then dies | 1/2 failure surfaces as a 4 timeout that kills the 5 session. `mosh` fixes this by moving the session above UDP and re-keying on address change, exactly the layer separation SSH lacks |

---

# 10 — The Application Developer's View: Which Layers Do You Actually Touch?

Most application code sits at the top of the stack and reaches down through a small number of well-defined interfaces. Knowing exactly where those interfaces are tells you what you own, what the OS owns, and what the network owns.

```
 ┌──────────────────────────────────────────────────────────────────────────┐
 │  YOUR CODE                    handlers, business logic, serialisation    │  L7 / L6 / L5
 │  ─────────────────────────────────────────────────────────────────────── │
 │  HTTP client/server library   (net/http, requests, axios, Spring, ASGI)  │  L7
 │  Serialisation library        (encoding/json, protobuf, msgpack)         │  L6
 │  TLS library                  (OpenSSL, BoringSSL, rustls, SChannel)     │  L6
 │  Auth / session middleware    (cookies, JWT, OAuth client)               │  L5
 │  Connection pool              (http.Transport, HikariCP, PgBouncer)      │  L5/L4
 ╞════════════════════ the socket API: socket() connect() send() recv() ════╡  ← L4/L5 boundary, the line you cross into the kernel
 │  KERNEL                       TCP/UDP state machines, congestion control │  L4
 │                               IP routing table, netfilter, NAT           │  L3
 │                               NIC driver, bridge, VLAN, ARP              │  L2
 ╞════════════════════ DMA descriptor rings / PCIe ═════════════════════════╡
 │  HARDWARE                     NIC MAC + PHY, cable, switch               │  L2 / L1
 └──────────────────────────────────────────────────────────────────────────┘
```

## 10.1 What each kind of developer owns

| Role | Layers you write code at | Layers you configure | Layers you must understand to debug | Layers you can safely ignore |
|---|---|---|---|---|
| Frontend / mobile developer | 7 (HTTP calls), 6 (JSON) | 5 (token storage, cookie flags) | 6 (TLS errors; CORS is L7), 4 (timeouts, why the app is slow on LTE) | 1–3 |
| Backend / API developer | 7, 6, 5 | 4 (ports, timeouts, keepalives, pool sizes) | 4 (connection resets, `TIME_WAIT` exhaustion), 3 (why the DB is unreachable from this subnet) | 1–2 |
| Platform / SRE / DevOps | 7 (ingress, DNS) | 3–7 (VPCs, security groups, LBs, service mesh, certificates) | Everything, most days | 1 (unless on-prem) |
| Network engineer | 2–3 (routing policy) | 1–4 | 1–4 | 5–7 (until the app team says "the network is slow") |
| Database / streaming developer | 7 (wire protocol), 6 (encoding), 5 (sessions, transactions) | 4 (TCP tuning is a big deal for Kafka/Cassandra) | 3–4 | 1–2 |
| Game / real-time media developer | 7, 5, and frequently **4** (custom reliability over UDP) | 4 | 3–4 (NAT traversal, jitter, loss) | 1–2 |
| Embedded / IoT developer | Often all of them (lwIP stack in the firmware, MQTT on top, LoRa/BLE radio config) | 1–7 | 1–7 | none |

## 10.2 Decisions that are secretly layer decisions

- **REST vs gRPC vs GraphQL** is an L7 (semantics) plus L6 (encoding) decision. It has no bearing on L4 except that gRPC requires HTTP/2 and therefore long-lived TCP connections, which matters for L4 load balancers that balance per connection (one hot backend) rather than per request. This is the classic "gRPC behind an NLB is unbalanced" incident; the fix is an L7 balancer.
- **"Should I use WebSockets or polling?"** is an L5 question (long-lived dialogue vs request/response) with L4 consequences (one connection held open per client; proxies and NATs time idle connections out; you need heartbeats).
- **"Which load balancer?"** L4 (NLB) preserves the client's TCP connection and can forward any protocol, is faster, and cannot see HTTP; L7 (ALB) terminates TCP and TLS, can route on path/header, retry, and rewrite, but only for protocols it understands.
- **Timeouts.** A single user request can hit a DNS timeout (L7 resolver), a TCP connect timeout (L4, SYN lost), a TLS handshake timeout (L6), an HTTP response timeout (L7), and an idle-connection timeout in a pool (L5). Setting only one of them is why services hang.
- **"Why is it slow only on mobile?"** Higher RTT makes every layer's round trips more expensive: 4 RTTs × 150 ms LTE = 600 ms before first byte. Reducing round trips (HTTP/3, TLS 1.3, connection reuse, DNS prefetch) is a bigger win than optimising the handler.
- **Retries.** TCP already retries lost segments (L4). Retrying at L7 on top is correct for idempotent requests and dangerous for non-idempotent ones; retrying at both layers with no jitter creates retry storms. Knowing which layer already retries prevents double-retrying.
- **Security controls per layer:** 802.1X / MACsec (2), IPsec / security groups / firewalls (3/4), TLS / mTLS (6), OAuth / API keys / WAF rules (7). Defence in depth means one control at each, not three at layer 7.

## 10.3 The socket API is the contract

`socket(AF_INET, SOCK_STREAM, 0)` picks the Internet layer (`AF_INET` = IPv4, `AF_INET6` = IPv6) and the transport (`SOCK_STREAM` = TCP, `SOCK_DGRAM` = UDP). `connect()` triggers the L4 handshake and, implicitly, L3 route lookup and L2 ARP. `send()` hands bytes to the kernel; the return value means "queued," not "delivered," which is why application-level acknowledgements exist. `recv()` returns whatever contiguous bytes have arrived, with no message boundaries, which is why every L7 protocol over TCP defines its own framing (`Content-Length`, chunked encoding, length-prefixed Protobuf, newline-delimited JSON). Every network library you use is a wrapper around these five calls plus `setsockopt`.

---

# 11 — Debugging With the Model: One Tool per Layer

The model's most concrete value on the job is turning "it doesn't work" into a bisection. Work **bottom-up** when the symptom is total failure and **top-down** when the symptom is partial or slow.

| Layer | Question | Linux / macOS tools | What a failure looks like |
|---|---|---|---|
| 1 | Is there a link? | `ip link` / `ethtool eth0` (speed, duplex, link detected), `iwconfig` / `nmcli dev wifi` (signal), switch port LEDs, `ethtool -S` (CRC/symbol errors) | `NO-CARRIER`, negotiated 100 Mb/s on a gigabit port, rising `rx_crc_errors` |
| 2 | Can I reach my neighbour? | `ip neigh` / `arp -a`, `bridge fdb`, `tcpdump -e` (shows MACs), `lldpctl` | ARP entry `FAILED`/`INCOMPLETE`, wrong VLAN, MAC flapping on the switch |
| 3 | Do I have an address and a route, and does the far end answer? | `ip addr`, `ip route get 8.8.8.8`, `ping`, `traceroute` / `mtr`, `tcpdump icmp` | `Destination Host Unreachable`, `No route to host`, traceroute stops at a hop, TTL exceeded, 100% loss after a NAT |
| 4 | Is the port open and does the handshake complete? | `ss -tlnp` (what is listening), `ss -tan` (connection states), `nc -zv host 443`, `tcpdump 'tcp[tcpflags] & (tcp-syn\|tcp-rst) != 0'`, `ss -ti` (RTT, cwnd, retransmits) | `Connection refused` (RST: nothing listening), `Connection timed out` (SYN dropped by a firewall), thousands of `TIME_WAIT`, high `retrans` |
| 5 | Does the session exist / persist? | Application logs, `redis-cli` on the session store, browser devtools → Application → Cookies, LB access logs with the stickiness cookie | Random logouts, requests landing on a backend with no state, idle connections killed by an LB after 60 s |
| 6 | Do the two sides agree on encoding and encryption? | `openssl s_client -connect host:443 -servername host`, `curl -v` (prints the TLS handshake), `file`, `iconv`, `jq .` on the body, protobuf decode, Wireshark's "Decode As" | Certificate expired / wrong SNI / unknown CA, `SSL_ERROR_...`, mojibake, JSON parse error, schema mismatch |
| 7 | Is the request semantically right and does the service answer correctly? | `curl -v`, `dig` / `nslookup`, `httpie`, browser devtools Network tab, `grpcurl`, `psql`, application logs, distributed tracing | 4xx/5xx, `NXDOMAIN`, wrong response body, timeout in the handler, auth rejected |
| all | Show me the packets | `tcpdump -i any -w cap.pcap host X and port Y`, then Wireshark | Wireshark's pane literally displays the frame as Ethernet → IP → TCP → TLS → HTTP, one expandable row per layer |

**Worked bisection.** A deploy goes out; users report "the site is down."

1. `dig site.example.com` returns the right IP: **L7 DNS OK.**
2. `ping` that IP succeeds: **L1–L3 OK** end-to-end.
3. `nc -zv ip 443` → connected: **L4 OK**, something is listening.
4. `openssl s_client` → `certificate has expired`: **L6 failure.** The new deploy shipped an old cert bundle.

Four commands, one per layer, and the incident is localised in under a minute without reading a line of application code. Without a layered model the natural move is to start reading application logs, which are empty because the request never reached layer 7.

---

# 12 — Where the Models Leak: Layer Violations in the Real World

The models are abstractions; the protocols are engineering. Every one of these is a deliberate trade of purity for a practical win, and knowing them stops you from being surprised.

| Violation | What happens | Why it is tolerated |
|---|---|---|
| **NAT** (L3/L4 middlebox) | A router rewrites IP addresses *and* TCP/UDP ports, breaking the end-to-end principle; hosts behind NAT are unreachable without help (STUN/TURN/ICE, port forwarding) | IPv4 exhaustion; NAT bought 25 years |
| **Stateful firewalls & deep packet inspection** | Network devices read L4 and L7 fields to make L3 forwarding decisions | Security; also the reason for protocol ossification below |
| **TLS** | Not cleanly L5, L6, or L7: it has a session (resumption), presentation (encryption), and application-visible extensions (SNI, ALPN pick the L7 protocol). Textbooks put it at 6; the IETF calls it "transport layer security" | It was slotted in between existing layers rather than redesigning them |
| **QUIC** | Merges transport (L4), encryption (L6), and stream multiplexing (L5) into one user-space protocol over UDP, invisible to the kernel | Kernel TCP cannot be changed quickly across billions of devices; UDP gets through middleboxes; HTTP/2's head-of-line blocking needed a transport fix |
| **MPLS** ("layer 2.5") | Label switching between L2 and L3; routers forward on labels, not IPs | ISP traffic engineering and VPNs; cheaper forwarding than IP lookups in the 1990s |
| **Tunnels & overlays** (VXLAN, GRE, IPsec, WireGuard, VPNs) | An entire L2 or L3 stack nested inside L4 or L3 of another; a packet can be Ethernet-in-VXLAN-in-UDP-in-IP-in-Ethernet | Stretching networks over networks you don't control; multi-tenancy in clouds |
| **ARP / NDP** | Sits between 2 and 3: an L3 address resolved to an L2 one, carried directly in an Ethernet frame with no IP header | Something has to bridge the two address spaces |
| **ICMP** | Defined as part of the Internet layer yet carried *inside* IP packets like a transport | Errors about the network need to travel through the network |
| **BGP / OSPF** | Control-plane protocols that run over TCP (BGP, port 179) or directly over IP (OSPF, protocol 89) to program layer-3 forwarding | Routing needs reliable delivery; reusing TCP is simpler than inventing one |
| **DHCP** | An L7 protocol whose job is to configure L3, running before the host has an L3 address (broadcast from 0.0.0.0) | Bootstrapping |
| **Path MTU Discovery** | The transport needs to know a link-layer property (MTU) of every hop | Fragmentation is worse |
| **ECN, DSCP/QoS marking** | Application intent (this is voice) written into L3 header bits read by L2 queues | Latency-sensitive traffic needs the whole path's cooperation |
| **TCP offload (TSO/LRO/GRO), RDMA, DPDK, kernel bypass** | L4 segmentation done by the L2 NIC; or the whole kernel stack skipped by user space | Performance at 100G+ |
| **L7 load balancers, CDNs, service meshes** | "The network" now terminates TLS, parses HTTP, retries requests, and injects headers: layers 4–7 implemented in the middle, not just the ends | Operational reality: you want retries, auth, and observability without rewriting every service |
| **Ossification** | Middleboxes that parse TCP/IP so rigidly that new options or protocols are dropped; SCTP never deployed on the public Internet for this reason, QUIC had to hide inside UDP and encrypt its own headers | Postel's law applied by a million devices |
| **Cross-layer optimisation in wireless** | 5G and Wi-Fi schedulers use application QoS hints and TCP feedback to allocate radio resources | Radio is the bottleneck and strict layering wastes it |

The lesson is not that the models are wrong. It is that a layer boundary is a **default**, and every violation is somebody paying the price of coupling in exchange for something they valued more. When you are tempted to violate one in your own design (e.g. having application code make decisions based on the client's IP address, which is an L3 fact that NAT and proxies routinely lie about), ask what you are buying and whether the coupling will hold when the layer below changes.

---

# 13 — Mental Models Worth Keeping

1. **Hourglass, not stack.** The point of the design is the narrow waist. Everything above IP can change without the network noticing; everything below can change without applications noticing. Protect the waist.
2. **Media layers are hop-by-hop; host layers are end-to-end.** L1–L3 are re-done at every router (new frame, new MAC, TTL−1); L4–L7 exist only in the endpoints. Any device that touches L4+ in the middle is a middlebox, and middleboxes are where surprises live.
3. **Each layer has its own address.** MAC (this link), IP (this internetwork), port (this process), URL/name (this resource). Most connectivity bugs are one of these being wrong, and the tool for each is different.
4. **Every layer trades bytes and a round trip for a guarantee.** Headers cost bandwidth; handshakes cost latency. Performance work is mostly removing round trips (connection reuse, TLS 1.3, QUIC, DNS caching) rather than shaving CPU.
5. **Reliability is a per-layer choice, not a global property.** Ethernet drops frames with bad CRCs; IP drops packets freely; TCP makes it reliable; UDP doesn't; the application may retry again. Know which layers retry before adding another.
6. **OSI is vocabulary; TCP/IP is reality.** Say "L4 load balancer," build with TCP. Say "presentation bug," fix the `charset`.
7. **When debugging, bisect by layer.** One tool per layer, bottom-up for outages, top-down for slowness. This is the model's real job.
8. **Layer violations are loans.** NAT, QUIC, tunnels, service meshes, kernel bypass: each borrows against layering's flexibility to buy performance, security, or deployability. The interest is paid at the next protocol transition.

---

# 14 — Further Reading

- Zimmermann, H. *OSI Reference Model: The ISO Model of Architecture for Open Systems Interconnection*, IEEE Trans. Comm., 1980. The original rationale for seven layers.
- ISO/IEC 7498-1:1994 / ITU-T X.200. *The Basic Reference Model.* The spec itself, freely available from ITU.
- Cerf, V. & Kahn, R. *A Protocol for Packet Network Intercommunication*, IEEE Trans. Comm., 1974. Where TCP/IP began.
- RFC 1122 / RFC 1123, *Requirements for Internet Hosts.* The four-layer model in its own words, plus Postel's law.
- RFC 3439, *Some Internet Architectural Guidelines and Philosophy.* Includes the "layering considered harmful" section: the IETF's own critique.
- Saltzer, Reed, Clark. *End-to-End Arguments in System Design*, 1984. The principle behind TCP-in-the-hosts, IP-in-the-network.
- Clark, D. *The Design Philosophy of the DARPA Internet Protocols*, SIGCOMM 1988. Why the Internet made the choices it did, in priority order.
- Russell, A. *OSI: The Internet That Wasn't*, IEEE Spectrum, 2013. The history of why OSI lost.
- Jacobson, V. *Congestion Avoidance and Control*, SIGCOMM 1988. What happens without a shared transport (§3.2), and the fix.
- Kurose & Ross, *Computer Networking: A Top-Down Approach.* The standard textbook; uses the five-layer hybrid.
- Tanenbaum & Wetherall, *Computer Networks.* Covers both models and the history of the "OSI vs TCP/IP" debate.
- Stevens, *TCP/IP Illustrated, Vol. 1.* Packet-level walkthroughs of every protocol in §4–§5.
- RFC 9000 (QUIC), RFC 8446 (TLS 1.3), RFC 9110 (HTTP semantics). The modern top of the stack, and the layer violations it embraces.
