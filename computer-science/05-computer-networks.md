# Computer Networks: How Machines Talk

Assumes [Operating Systems](operating-systems.md) (processes, syscalls, the socket preview in Part 13). This document explains how a URL typed in a browser becomes pixels — the full stack from copper and radio up to HTTPS — and builds the mental model behind every "why is it slow/down/insecure?" question about networked software.

---

## Table of Contents

1. [The Problem and the Big Idea: Layering](#1--the-problem-and-the-big-idea-layering)
2. [The Link Layer: One Hop](#2--the-link-layer-one-hop)
3. [The Network Layer: IP and Routing](#3--the-network-layer-ip-and-routing)
4. [The Transport Layer: UDP and TCP](#4--the-transport-layer-udp-and-tcp)
5. [Naming: DNS](#5--naming-dns)
6. [The Web: HTTP](#6--the-web-http)
7. [Security in Transit: TLS](#7--security-in-transit-tls)
8. [The Modern Stack: HTTP/2, QUIC, CDNs](#8--the-modern-stack-http2-quic-cdns)
9. [The Full Story of One Page Load](#9--the-full-story-of-one-page-load)
10. [Network Programming and APIs](#10--network-programming-and-apis)
11. [Performance and Debugging](#11--performance-and-debugging)
12. [The Road to Expertise](#12--the-road-to-expertise)

---

# 1 — The Problem and the Big Idea: Layering

Connecting billions of machines built by thousands of vendors over copper, fiber, and radio, owned by ~100,000 competing networks, with no central operator — and making it look to a program like "a readable/writable byte stream to anywhere." The solution is the curriculum's recurring move, applied ruthlessly: **layers, each solving one problem, each using only the layer below**.

```
  Application   HTTP, DNS, SMTP, SSH...      "what the bytes mean"
  Transport     TCP, UDP                     "process-to-process delivery, reliability"
  Network       IP                           "machine-to-machine across the world"
  Link          Ethernet, Wi-Fi, LTE/5G      "one hop between adjacent devices"
  Physical      signals on copper/fiber/air
```

Each layer **wraps** the layer above's data with its own header — like envelopes inside envelopes:

```
| Ethernet hdr | IP hdr | TCP hdr | HTTP request ... | Ethernet trailer |
```

The power of the design: layers are *independently replaceable*. Wi-Fi replaced Ethernet, fiber replaced copper, HTTP/3 is replacing HTTP/1.1 — and no other layer had to change. IP is the **narrow waist**: every application above, every technology below, one protocol in the middle — the minimal agreement that makes the whole zoo interoperate.

Two design principles worth knowing by name:

- **End-to-end principle**: keep the middle of the network dumb (just forward packets); put intelligence (reliability, encryption, ordering) at the endpoints. Why the internet scaled — routers don't track your conversations.
- **Best-effort delivery**: IP promises *nothing* — packets may be dropped, duplicated, reordered, corrupted. Every guarantee you experience is *constructed at the endpoints* on top of this chaos (§4.3). Internalizing "the network is allowed to drop anything" is the beginning of networking wisdom — and of distributed-systems wisdom ([../distributed-systems/](../distributed-systems/)).

---

# 2 — The Link Layer: One Hop

Delivers **frames** between devices on the same local network (LAN):

- **MAC addresses** — 48-bit burned-in hardware IDs (`8c:85:90:12:ab:34`); flat, no structure, meaningful only locally.
- **Ethernet**: framed bytes over cable to a **switch**, which learns which MAC lives on which port and forwards frames only where needed.
- **Wi-Fi (802.11)**: same job over shared radio — plus the problems of a shared medium: collisions (can't detect while transmitting → collision *avoidance*, waiting random backoffs), signal decay, and encryption of the air (WPA2/3). This is why Wi-Fi is inherently flakier than cable — physics, not your router.
- **ARP** (address resolution): "who has IP 192.168.1.7? Tell me your MAC" — broadcast on the LAN; bridges the IP world to the link world. (First protocol to check when "same network but can't reach it.")
- Frames carry checksums: corrupted frames are *dropped*, not fixed — recovery is someone else's job (§4.3, the end-to-end principle in action).

---

# 3 — The Network Layer: IP and Routing

## 3.1 IP addresses — location, not identity

An **IP address** (IPv4: 32 bits, `142.250.72.14`; IPv6: 128 bits, `2607:f8b0::200e`) is **hierarchical, like a postal address**: the prefix identifies a network, the rest a host within it. `192.168.4.0/24` means "first 24 bits are the network part" (CIDR notation). Hierarchy is what makes global routing *possible*: routers store routes to ~1M *prefixes*, not billions of hosts, and forwarding is **longest-prefix match** (a trie — doc 03 §10 pays off) on the destination address.

IPv4's 4.3 billion addresses ran out (formally exhausted 2011–2019 across registries). Two responses:

- **NAT** (network address translation): your whole home shares one public IP; the router rewrites private addresses (`192.168.x.x`, `10.x.x.x`) to its public one, tracking each connection in a table. Consequences programmers hit constantly: machines behind NAT can't be *reached* unsolicited (breaks peer-to-peer; hence STUN/TURN hole-punching in video calls), and "my IP" has two answers (private vs. public).
- **IPv6**: 3.4 × 10³⁸ addresses; deployed alongside IPv4 (dual-stack) at ~45–50% of traffic and rising; conceptually identical for this document's purposes.

## 3.2 Routing — how packets find their way

Each **router** examines the destination IP, looks up the longest matching prefix in its forwarding table, sends the packet out that interface; repeat hop by hop (typically 10–20 hops across the world — watch it live with `traceroute`/`tracert`). Where do tables come from?

- **Inside one organization** (an *autonomous system*, AS): link-state protocols (**OSPF**, IS-IS) — every router learns the full topology and runs **Dijkstra** (doc 04 §6.3, verbatim) to compute shortest paths.
- **Between organizations**: **BGP** — the ~76,000 ASes (ISPs, clouds, universities) *announce* to neighbors which prefixes they can reach; routes propagate by rumor, filtered by *business policy* (money and politics, not shortest path). BGP is the internet's actual glue and its softest spot: a bad announcement can (and repeatedly does) black-hole traffic — Facebook's 2021 six-hour global outage was a self-inflicted BGP withdrawal; hijacks have redirected national traffic. Trust-based since 1989; RPKI is slowly adding cryptographic validation.
- **ICMP** — IP's control channel: "destination unreachable," "time-to-live exceeded" (the mechanism behind traceroute: send packets with TTL 1, 2, 3... and collect the complaints), and ping's echo.

Key mental model: **the internet is not a thing but an agreement** — a mesh of rival networks voluntarily exchanging traffic (peering/transit) using BGP as the contract language.

---

# 4 — The Transport Layer: UDP and TCP

IP delivers (maybe) machine-to-machine. Transport adds the two things applications need: **which process** (ports — the 16-bit numbers from [OS doc Part 13](operating-systems.md); 443 = HTTPS, 53 = DNS, 22 = SSH), and optionally **reliability**.

## 4.1 UDP — IP with ports, nothing more

Eight bytes of header: source/dest port, length, checksum. Send a datagram; it arrives once, late, duplicated, or never. Zero setup latency, zero protocol overhead, and *you* own the reliability policy — perfect when old data is worthless (live audio/video, game state: retransmitting a stale frame is worse than skipping it), when one-shot request/reply fits (DNS), or when you're building a better TCP in userspace (QUIC, §8).

## 4.2 TCP — the reliable byte stream

TCP constructs, on top of lawless IP, the abstraction every application actually wants: **a two-way, ordered, reliable, congestion-controlled byte stream**. The mechanisms (each answering one failure mode):

- **Connection setup — the three-way handshake**: SYN → SYN-ACK → ACK, exchanging random starting **sequence numbers**. Costs one round-trip time (RTT) before any data — latency you'll meet again in §7 and §11.
- **Sequence numbers** on every byte → the receiver reorders out-of-order arrivals and discards duplicates. The stream you `read()` is a *reconstruction*.
- **ACKs + retransmission**: the receiver acknowledges what it has; the sender retransmits anything unacknowledged past a timeout (or on faster signals — duplicate/selective ACKs). Loss becomes *delay*, invisibly. (This is why loss doesn't corrupt your download; it just slows it.)
- **Flow control**: the receiver advertises how much buffer it has left; the sender never exceeds it. Protects the *receiver*.
- **Congestion control**: protects the *network* — the genuinely deep part. There's no signal saying "the internet is full"; TCP infers congestion from loss/delay and self-regulates: start small and double each RTT (*slow start* — actually exponential), then probe linearly, cut on loss (AIMD), governed today by algorithms like CUBIC and BBR (model the bottleneck's bandwidth-delay product instead of waiting for loss). Every TCP flow on earth voluntarily backing off is *why the internet doesn't collapse* — it nearly did in 1986 (congestion collapse), and this is the fix. Consequences you observe: downloads ramp up rather than starting fast; throughput on a lossy Wi-Fi link craters (TCP reads loss as congestion); one connection ≈ fair share of the bottleneck.
- Teardown: FIN/ACK each direction (plus TIME_WAIT — the source of "address already in use" for restarting servers).

## 4.3 The lesson

TCP is the end-to-end principle incarnate: *all* of this lives in the two endpoints' kernels ([OS doc Part 13](operating-systems.md)); routers in between know nothing of it. And the abstraction leaks at the edges — "reliable" means *retransmitted while the connection lives*, not *delivered-and-processed*: when a connection drops, a sender cannot know which of its recently-written bytes were seen by the application on the other side. Exactly-once processing is therefore an *application-level* construction (idempotency keys, dedup) — the bridge into [../distributed-systems/05-exactly-once-and-stream-processing.md](../distributed-systems/05-exactly-once-and-stream-processing.md).

---

# 5 — Naming: DNS

Humans use names (`www.wikipedia.org`); packets need IPs. **DNS** is the internet's phone book — and its largest distributed database, resolving trillions of queries daily with no central server.

The name space is a **hierarchy read right-to-left**: root → `org` → `wikipedia.org` → `www.wikipedia.org`, with each zone *delegated* to its owner's servers. Resolution (when nothing is cached): your **resolver** (ISP's, or 8.8.8.8/1.1.1.1) asks a **root server** ("who handles `org`?") → asks the `org` TLD servers ("who handles `wikipedia.org`?") → asks Wikipedia's **authoritative** servers → gets the answer, caches it, returns it. Caching at every layer (resolver, OS, browser), governed by each record's **TTL**, is why the system survives its load — and why DNS changes "take time to propagate" (they don't propagate; caches expire).

Record types worth knowing: **A**/**AAAA** (name → IPv4/IPv6), **CNAME** (alias to another name — how sites point at CDNs), **MX** (mail), **NS** (delegation), **TXT** (freeform — where email auth like SPF/DKIM lives). Tools: `nslookup`, `dig` (read its answer section once and DNS stops being magic).

Engineering notes: DNS runs on UDP (one-shot fits; falls back to TCP for big answers); it's a favorite attack surface (cache poisoning → DNSSEC signatures; plaintext queries reveal browsing → DNS-over-HTTPS/TLS); and it doubles as a *traffic-steering layer* — returning different IPs by geography/load is how global services direct you to a nearby datacenter (§8). When "the internet is down" but ping 8.8.8.8 works — it's DNS. It's always DNS.

---

# 6 — The Web: HTTP

The application protocol that ate the world. Plain text (through 1.1), stateless request/response over a TCP (now QUIC) connection:

```
GET /wiki/Cat HTTP/1.1               HTTP/1.1 200 OK
Host: en.wikipedia.org               Content-Type: text/html; charset=UTF-8
Accept: text/html                    Content-Length: 84732
                                     Cache-Control: max-age=3600
                                     <!DOCTYPE html><html>...
```

- **Methods** — the verb vocabulary: `GET` (fetch, no side effects, cacheable), `POST` (submit/create), `PUT` (replace), `PATCH` (modify), `DELETE`. The GET-is-safe-and-idempotent convention is load-bearing: caches, prefetchers, and crawlers rely on it.
- **Status codes**: 2xx success (200 OK, 201 Created), 3xx redirection (301 permanent, 304 "your cached copy is still good"), 4xx *your* fault (400 malformed, 401 unauthenticated, 403 forbidden, 404, 429 slow down), 5xx *server's* fault (500, 502/504 — some proxy's upstream failed, 503 overloaded).
- **Headers** carry everything else: content type/length, compression, caching directives, auth tokens, cookies.
- **Statelessness + cookies**: each request stands alone (this is *why* web servers scale horizontally — any server can answer any request; see [../distributed-systems/](../distributed-systems/)); continuity (logins, carts) is layered on via **cookies** — server-set key-values the browser attaches to every subsequent request, carrying your session ID. Hence also tracking, and the security problems in [doc 10](10-security-and-cryptography.md) (session theft, CSRF).
- **Caching** — HTTP's superpower: `Cache-Control`/`ETag` let browsers and intermediaries reuse responses (the memory-hierarchy lesson yet again, now at planetary scale — §8's CDNs are "L2 cache for the web").

**REST APIs** are HTTP's conventions applied to program-to-program interfaces: resources as URLs, verbs as methods, JSON bodies, status codes as outcomes (§10).

---

# 7 — Security in Transit: TLS

Plain HTTP is a postcard — every router, ISP, and coffee-shop Wi-Fi reads and *modifies* it. **TLS** (the S in HTTPS) wraps any TCP stream with three guarantees: **confidentiality** (encrypted), **integrity** (tamper-evident), **authenticity** (you're talking to who the certificate says).

The handshake in essence (full crypto in [doc 10](10-security-and-cryptography.md)): client and server run a **key exchange** (elliptic-curve Diffie-Hellman — deriving a shared secret over a public channel, the closest thing to magic in this curriculum) → the server proves identity with its **certificate** — its public key signed by a **Certificate Authority** your browser already trusts (a chain ending at ~150 root CAs shipped with your OS/browser; "Let's Encrypt" made these free, pushing HTTPS above 95% of page loads) → all traffic switches to fast symmetric encryption (AES/ChaCha20) under the derived keys. TLS 1.3 (2018) cut it to **one round trip** (0 for resumption) and removed a museum of broken options.

What TLS does *not* hide: that you connected to that server, when, and how much you transferred (metadata); and it's only as strong as the CA system (a rogue CA can mint certificates for anyone — mitigated by Certificate Transparency logs). The padlock means "encrypted to the named party" — not "the named party is honest."

---

# 8 — The Modern Stack: HTTP/2, QUIC, CDNs

How the 1990s web stack got rebuilt for a latency-dominated world:

- **HTTP/1.1's problem**: one request at a time per connection → browsers opened 6 parallel connections and it still stalled (*head-of-line blocking* at the HTTP layer).
- **HTTP/2** (2015): one connection, many concurrent **streams** multiplexed as binary frames + header compression. But TCP resurfaced the problem: one lost packet stalls *all* streams (TCP's in-order guarantee is connection-wide — HOL blocking moved down a layer).
- **QUIC + HTTP/3** (2021–22): rebuild transport **on UDP in userspace** — per-stream ordering (loss stalls only its own stream), TLS 1.3 fused into the transport (1-RTT cold, 0-RTT resumed connections), connections survive IP changes (Wi-Fi → cellular without dropping). Now ~30%+ of web traffic. The meta-lesson: TCP's kernel/router entrenchment ("ossification") made evolving it impossible — so the web *tunneled a new transport through UDP*, the narrow waist's escape hatch.
- **CDNs** (Cloudflare, Akamai, CloudFront): thousands of edge datacenters caching content near users and terminating TLS there. Because the speed of light is a hard floor (~100 ms RTT across an ocean — no technology fixes this), *moving the content* is the only cure for distance. DNS/anycast steer you to the nearest edge; the edge serves cached static content and proxies the rest over warm long-haul connections. Most of "the web feels fast" is CDNs.

---

# 9 — The Full Story of One Page Load

Every layer, in order, for `https://en.wikipedia.org` on coffee-shop Wi-Fi (the interview classic, and a genuine self-test — close the document and narrate it):

1. **Join the network** (once): Wi-Fi association + WPA handshake; **DHCP** lease (your IP, gateway, DNS resolver — link layer + config).
2. **DNS** (§5): resolver walk or cache hit → `en.wikipedia.org` = some CDN edge IP near you.
3. **TCP handshake** (§4.2) with that IP on port 443 — or QUIC skips straight to:
4. **TLS handshake** (§7): key exchange, certificate chain verification → encrypted channel.
5. **HTTP request** (§6): `GET /wiki/...` with headers and cookies.
6. Each packet en route: **ARP** to find the gateway's MAC (§2), **NAT** rewrite at your router (§3.1), then per-hop **longest-prefix forwarding** (§3.2) across ISP and backbone ASes glued by **BGP**, congestion-controlled end-to-end by **TCP/QUIC** (§4.2).
7. **Response**: HTML arrives; the browser parses it, discovers CSS/JS/images, fetches them (multiplexed, §8, mostly cache hits at the CDN edge), executes JS, renders. (The browser-internals half of this story — parsing, layout, paint — belongs to docs 08's parsing and beyond.)
8. Every subsequent visit is faster: DNS cached, TLS resumed, HTTP caches validated with 304s. The hierarchy of caches, end to end.

---

# 10 — Network Programming and APIs

What you actually write, and where it sits:

- **Sockets** — covered in [OS doc Parts 9 & 13](operating-systems.md): `connect`/`accept`/`read`/`write`, epoll event loops for many connections. Everything below is libraries over this.
- **Talking between services**: **REST/JSON over HTTP** (universal, human-readable, cache-friendly); **gRPC** (HTTP/2 + Protocol Buffers — binary, typed contracts, streaming; the intra-datacenter default); **WebSockets** (upgrade an HTTP connection to a persistent two-way stream — chat, live dashboards); **message queues** (Kafka, RabbitMQ — decouple sender and receiver in time; see [../distributed-systems/](../distributed-systems/)).
- **Serialization** — turning structures into wire bytes: JSON (text, universal, verbose), Protocol Buffers/Avro (binary, schema'd, compact — schemas also solve *versioning*: fields evolve without breaking old readers, a distributed-systems essential).
- **The failure catechism** every networked program must answer (this is where network programming diverges from programming): timeouts on *every* call (the default of "wait forever" is an outage); **retries with exponential backoff + jitter** (retry storms have flattened clouds); **idempotency** for anything retried (§4.3 — a retried "charge card" must not double-charge: idempotency keys); **circuit breakers** and graceful degradation. Full treatment: [../distributed-systems/06-overload-control-and-resilience.md](../distributed-systems/06-overload-control-and-resilience.md).

---

# 11 — Performance and Debugging

## 11.1 The two numbers

**Latency** (time for one round trip — bounded below by distance/speed-of-light; ~0.5 ms same-datacenter, ~100 ms cross-ocean) and **bandwidth** (bytes/second — grows with technology). They are *independent*: a truck of SSDs has colossal bandwidth and dreadful latency. Modern web performance is overwhelmingly **latency-dominated**: a page needing DNS + TCP + TLS + HTTP sequentially pays 3–4 RTTs before the first byte — which is exactly what §8's whole agenda (fewer round trips, closer servers) attacks. Corollaries: batch requests; avoid chatty sequential APIs (N+1 request patterns); put data near users; and know the **bandwidth-delay product** (bytes "in flight" a connection must keep unacknowledged to fill the pipe — why high-latency links need big windows to be fast).

## 11.2 The toolbox and the method

Debug bottom-up or top-down along the §9 chain, bisecting layers:

| Question | Tool |
|---|---|
| Do I have connectivity / where does it die? | `ping`, `traceroute`/`tracert`, `mtr` |
| Is it DNS? | `dig`/`nslookup` (it's often DNS) |
| Is the port open / who's listening? | `ss -tlnp` / `netstat`, `nc` (netcat), `telnet host port` |
| What's actually on the wire? | `tcpdump`, **Wireshark** (watch one HTTP request once — transformative) |
| What is the HTTP exchange? | `curl -v`, browser DevTools Network tab |
| Is TLS the problem? | `curl -v`, `openssl s_client` |
| Per-connection kernel stats? | `ss -ti` (congestion window, RTT live) |

---

# 12 — The Road to Expertise

## 12.1 Books

1. **Computer Networking: A Top-Down Approach** (Kurose & Ross) — the standard text; teaches in this document's order (application first), with superb labs.
2. **High Performance Browser Networking** (Grigorik) — *free online* (hpbn.co); the practitioner's book on latency, TCP/TLS/HTTP realities, and why the modern stack looks like §8. Read second.
3. **TCP/IP Illustrated, Vol. 1** (Stevens/Fall) — protocols traced packet-by-packet; the depth reference.
4. **Unix Network Programming** (Stevens) — the sockets bible, alongside [OS doc](operating-systems.md) material.
5. RFCs — the actual specs are readable (start: RFC 9293 TCP, 1034/1035 DNS, 9110 HTTP semantics, 9000 QUIC).

## 12.2 Doing

1. Run the §11.2 tool tour on a site you use daily; narrate §9 from your own captures.
2. **Build an HTTP server on raw sockets** (no framework — parse the request line and headers yourself, serve files). Then make it concurrent (threads, then epoll). This one project cements Parts 4–6 and OS Parts 9/13 simultaneously.
3. **Build a DNS client** from the RFC (UDP packet in, parse the answer — a weekend, deeply illuminating).
4. Implement a toy **reliable-transport over UDP** (sequence numbers, ACKs, retransmit timer, then a congestion window) — Stanford's CS144 labs (free) walk exactly this; the single best networks course online.
5. Wireshark a TLS handshake; identify the ClientHello, certificate, and where plaintext ends.
6. Then: [../distributed-systems/](../distributed-systems/) — networks are the substrate; distributed systems are what you build when you accept §1's failure model at application scale.

## 12.3 Ideas to retain forever

1. **Layers with narrow waists** — independence above and below IP is why the internet could evolve for 50 years.
2. **The network promises nothing** (best-effort); every guarantee is built at the endpoints (end-to-end principle) — and each guarantee leaks at the edges (delivered ≠ processed).
3. **Hierarchy scales**: IP prefixes and DNS zones are the same trick — aggregate names, delegate subtrees, cache aggressively.
4. **TCP = reliability + fairness constructed from chaos**; congestion control is a planetary cooperative act running in every kernel.
5. **The internet is an agreement between rivals** (BGP + peering), held together by trust and money — and it fails accordingly.
6. **Latency is physics, bandwidth is technology** — round trips are the currency of web performance; the modern stack (TLS 1.3, QUIC, CDNs) is one long war on RTTs.
7. **Statelessness scales; state is layered on** (cookies/sessions) — and caching, at every layer, is the web's real engine.
8. **Encrypt in transit, always** — and know precisely what TLS's padlock does and doesn't claim.
9. **Networked code = failure-handling code**: timeouts, backoff+jitter, idempotency, or eventual outage.
10. **When it's broken, bisect the layers** — and it's usually DNS.

---

*Next: [Databases](06-databases.md) — durable, queryable, concurrent state, built on everything so far.*
