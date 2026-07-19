# Realtime, High-Traffic, Highly Available Distributed Systems — Concepts & Practices

> **Audience:** Engineers designing or operating systems that must serve extreme traffic with low latency while surviving machine, network, and datacenter failures.
> **Scope:** Foundational theory, core building blocks (partitioning, replication, consensus, time), realtime data processing, traffic & overload management, resilience patterns, high-availability architecture, and operational practice.
> **Grounding:** Every major claim is tied to a primary source — the original paper, official documentation, or a first-party engineering publication. All sources are linked inline and collected in [§14 References](#14-references). Nothing here is invented; where a topic is folklore without a citable primary source, it is omitted.

---

## Table of Contents

1. [Definitions & Goals](#1-definitions--goals)
2. [Foundational Theory](#2-foundational-theory)
3. [Time, Clocks & Ordering](#3-time-clocks--ordering)
4. [Partitioning & Sharding](#4-partitioning--sharding)
5. [Replication](#5-replication)
6. [Consensus & Coordination](#6-consensus--coordination)
7. [Failure Detection & Membership](#7-failure-detection--membership)
8. [Delivery Guarantees, Idempotency & Distributed Transactions](#8-delivery-guarantees-idempotency--distributed-transactions)
9. [Realtime Stream Processing](#9-realtime-stream-processing)
10. [Traffic Management at Scale](#10-traffic-management-at-scale)
11. [Overload & Resilience Patterns](#11-overload--resilience-patterns)
12. [High-Availability Architecture Patterns](#12-high-availability-architecture-patterns)
13. [Operational Practice: Observability, SRE & Chaos](#13-operational-practice-observability-sre--chaos)
14. [References](#14-references)

---

## 1. Definitions & Goals

The three properties in the title are distinct and must be engineered separately:

| Property | Meaning | Measured by |
|---|---|---|
| **High traffic** | The system sustains very large request/event rates by scaling *out* (adding machines), not just *up* | Throughput (RPS/events-sec), saturation headroom |
| **Highly available** | The system keeps answering despite failures of components | Availability (the "nines"), error rate against an SLO |
| **Fault tolerant** | Failure of individual machines, disks, links, or whole zones does not cause data loss or extended outage | RPO/RTO, blast radius of a single failure |
| **Realtime (soft)** | Results are produced with bounded, low latency — milliseconds to seconds, not batch windows | Latency *percentiles* (p50/p99/p999), watermark lag |

Two framing facts shape everything below:

- **At scale, tail latency is the product, not the average.** In a request that fans out to 100 backends where each backend is slow 1% of the time, ~63% of user requests hit at least one slow backend — the end-to-end latency is the *max* over the fan-out, not the mean ([Dean & Barroso, "The Tail at Scale", CACM 2013](https://www.barroso.org/publications/TheTailAtScale.pdf)).
- **At scale, failure is a permanent condition, not an event.** The Dynamo paper opens from exactly this premise: at Amazon's scale there are always failed components, so the system is designed to treat "failure handling as the normal case without impacting availability or performance" ([DeCandia et al., "Dynamo", SOSP 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)).

---

## 2. Foundational Theory

These results define the *possibility space*. Every architecture decision in later sections is a position taken inside these constraints.

### 2.1 CAP Theorem

Eric Brewer conjectured (PODC 2000 keynote) and Gilbert & Lynch proved (2002) that a distributed read/write register cannot simultaneously guarantee **C**onsistency (linearizability), **A**vailability (every request to a non-failed node gets a response), and **P**artition tolerance. In an asynchronous network, when a partition occurs you must choose: refuse some requests (CP) or answer with possibly stale data (AP) ([Gilbert & Lynch, "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services"](https://en.wikipedia.org/wiki/CAP_theorem); [survey of the proof and its refinements](http://muratbuffalo.blogspot.com/2015/02/paper-summary-perspectives-on-cap.html)).

Important nuances that are frequently misstated:

- CAP is about behavior *during a partition*; it says nothing about normal operation.
- "Choosing AP or CP" is not system-wide — it can be chosen per operation (e.g., Cassandra/Dynamo tunable consistency per request).
- Spanner is often described as "effectively CA," but Google's own paper is explicit that it is technically CP: during a partition it sacrifices availability, and its very high availability in practice comes from engineering the network (private fiber, redundancy), not from beating the theorem ([Brewer, "Spanner, TrueTime & The CAP Theorem", 2017](https://research.google.com/pubs/archive/45855.pdf)).

### 2.2 PACELC

Daniel Abadi's PACELC (2010) extends CAP with the observation that the tradeoff exists even *without* partitions: **if Partition, choose Availability vs Consistency; Else, choose Latency vs Consistency**. Synchronous replication buys consistency at the cost of latency; asynchronous replication buys latency at the cost of consistency. PACELC was later given a formal treatment connecting it to known latency lower bounds for distributed objects ([Golab, "Proving PACELC", 2018](https://uwaterloo.ca/distributed-algorithms-systems-lab/sites/default/files/uploads/files/proving_pacelc.pdf)).

This is the theorem that actually governs day-to-day design of high-traffic systems, because partitions are rare but the latency/consistency tradeoff is paid on *every request*.

### 2.3 FLP Impossibility

Fischer, Lynch & Paterson (JACM 1985) proved that in a fully asynchronous system, **no deterministic consensus protocol can guarantee termination if even one process may crash** ([FLP paper record](https://paperswelove.org/papers/impossibility-of-distributed-consensus-with-one-fa-80e1c33b/); [proof walkthrough](https://team.inria.fr/antique/the-impossibility-result-of-fischer-lynch-and-paterson-and-its-proof/); it received the [2001 PODC Influential Paper Award](https://www.podc.org/influential/2001-influential-paper/)).

Practical consequence: real consensus systems (Paxos, Raft, ZAB) guarantee *safety* always but *liveness* only under partial synchrony — they use timeouts and randomized/leader-based election to make progress "eventually, in practice," which is why they can stall (but never corrupt state) under pathological network conditions.

### 2.4 Consistency Models (what "consistency" means per system)

A spectrum, strongest to weakest — stronger models cost more coordination (and thus latency, per PACELC):

- **Linearizability / external consistency** — operations appear to occur atomically at some point between invocation and response, consistent with real time. Spanner provides external consistency globally via TrueTime ([Corbett et al., "Spanner", OSDI 2012](https://research.google/pubs/spanner-googles-globally-distributed-database-2/)).
- **Sequential / serializable** — some global order exists, not necessarily matching real time.
- **Causal** — only causally-related operations are ordered; concurrent ones may be seen in different orders. This is what vector clocks capture (see §3).
- **Eventual consistency** — replicas converge if updates stop; the model Dynamo explicitly adopts to maximize availability, pushing conflict resolution to reads and to the application ([Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)).

### 2.5 Harvest & Yield / Graceful Degradation as Theory

Rather than a binary up/down, availability can be decomposed: **yield** (fraction of requests answered) and **harvest** (fraction of the data reflected in each answer). Under stress you can trade harvest for yield — e.g., a search service answering from a subset of shards. Google's SRE book documents this as *graceful degradation*: serve degraded responses (search only an in-memory cache subset, use a cheaper ranking algorithm) rather than failing entirely ([SRE Book, Ch. 22 "Addressing Cascading Failures"](https://sre.google/sre-book/addressing-cascading-failures/)).

---

## 3. Time, Clocks & Ordering

There is no shared "now" across machines. Everything about ordering in distributed systems flows from that.

### 3.1 Logical Clocks & Happens-Before

Lamport's 1978 paper ("Time, Clocks, and the Ordering of Events in a Distributed System", CACM 21(7)) introduced the **happens-before** partial order and **logical clocks**: event ordering can be established through message-passing causality alone, with no synchronized physical clocks ([Lamport timestamps](https://en.wikipedia.org/wiki/Lamport_timestamp)). Lamport clocks give: if a → b then L(a) < L(b) — but not the converse.

### 3.2 Vector Clocks

Vector clocks strengthen this: they capture causality exactly, so the system can distinguish "A supersedes B" from "A and B are concurrent (true conflict)." Dynamo uses vector clocks to version each object; on read, causally-ordered versions are collapsed automatically and truly concurrent versions are returned to the client for **semantic reconciliation** (e.g., merging two shopping-cart versions). To bound their size, Dynamo caps the clock length and evicts the oldest entry (by wall-clock timestamp) when exceeded ([Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf); [annotated summary](https://mwhittaker.github.io/papers/html/decandia2007dynamo.html)).

### 3.3 TrueTime: Engineering Bounded Clock Uncertainty

Spanner's **TrueTime** API returns an *interval* `[earliest, latest]` guaranteed to contain the true time, backed by GPS and atomic clocks in every datacenter. By waiting out the uncertainty (*commit wait*) before making a transaction's timestamp visible, Spanner achieves **external consistency** — globally, if T1 commits before T2 starts, T1's timestamp is smaller — enabling lock-free consistent snapshot reads at global scale ([Spanner, OSDI 2012](https://research.google/pubs/spanner-googles-globally-distributed-database-2/); [Cloud Spanner TrueTime docs](https://docs.cloud.google.com/spanner/docs/true-time-external-consistency)).

The general lesson: you cannot eliminate clock skew, but you can **measure and expose the uncertainty** and design protocols around the bound.

### 3.4 Event Time vs Processing Time

In realtime pipelines the distinction between **event time** (when the event occurred) and **processing time** (when the system sees it) is fundamental — data arrives out of order and late. The Dataflow model's answer is the **watermark**: a moving lower bound on the event times still in flight, letting the system decide when a time window is "complete enough" to emit ([Dataflow model overview](https://www.infoq.com/articles/dataflow-apache-beam/); [Confluent on watermarks and the Dataflow model](https://www.confluent.io/blog/watermarks-tables-event-time-dataflow-model/); formal comparison of Flink vs Dataflow watermark semantics in [Begoli et al., VLDB 2021](http://www.vldb.org/pvldb/vol14/p3135-begoli.pdf)). MillWheel introduced the same idea as the **low watermark** ([Akidau et al., "MillWheel", VLDB 2013](https://research.google.com/pubs/archive/41378.pdf)). Details in §9.

---

## 4. Partitioning & Sharding

Scaling *traffic* horizontally means splitting the keyspace/workload across machines.

### 4.1 Consistent Hashing & Virtual Nodes

Dynamo partitions data on a **consistent hash ring**: each node owns arcs of the ring, so adding/removing a node only moves the keys adjacent to it, not a full reshuffle. Plain consistent hashing distributes load unevenly, so Dynamo assigns each physical node many **virtual nodes** (tokens) — this smooths load and lets heterogeneous machines take proportional shares, and when a node fails its load disperses evenly across the remaining ones ([Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)).

### 4.2 Hash vs Range Partitioning

- **Hash partitioning** (Dynamo, Cassandra) — uniform load distribution, no efficient range scans.
- **Range partitioning** (Bigtable/Spanner directories/tablets) — efficient scans, but hot ranges must be detected and split/moved; Spanner moves directories between Paxos groups for load balancing ([Spanner paper](https://research.google/pubs/spanner-googles-globally-distributed-database-2/)).

Hot keys ("celebrity problem") are a hazard of any scheme: a single key hotter than one partition's capacity requires key-splitting, caching, or replication of that key — partitioning alone cannot save you.

### 4.3 Shuffle Sharding (Isolation via Combinatorics)

AWS's **shuffle sharding** assigns each customer/tenant a small random subset of the fleet's resources (e.g., Lambda hashing each customer to a small subset of a fixed number of queues). Two tenants rarely share their *entire* subset, so a poison workload from one tenant degrades only the small overlap, not the whole fleet — blast-radius reduction by combinatorics ([AWS Builders' Library on overload/queue isolation](https://aws.amazon.com/builders-library/avoiding-overload-in-distributed-systems-by-putting-the-smaller-service-in-control/); [queue backlog analysis](https://lumigo.io/blog/amazon-builders-library-in-focus-4-avoiding-insurmountable-queue-backlogs/)).

---

## 5. Replication

Replication serves both fault tolerance (survive node loss) and read scaling. The design axis is *who may accept writes* and *how divergence is reconciled*.

### 5.1 Leader-Based (Primary/Backup)

One replica accepts writes and ships them to followers, synchronously (no data loss on failover, higher latency — the PACELC "C" side) or asynchronously (lower latency, bounded data loss on failover — the "L" side). This is the model of classic RDBMS replication and of each Spanner Paxos group (writes go through the group's Paxos leader) ([Spanner paper](https://research.google/pubs/spanner-googles-globally-distributed-database-2/)).

### 5.2 Leaderless / Quorum Replication (Dynamo Model)

Every key is replicated to **N** nodes (its *preference list*); any replica can coordinate. Consistency is tuned by quorum arithmetic: a write must be acknowledged by **W** nodes and a read by **R**; setting **R + W > N** makes read and write quorums overlap so a read sees the latest acknowledged write ([Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)).

Mechanisms Dynamo layers on top — all now standard vocabulary:

- **Sloppy quorum** — during failures, writes land on the next healthy nodes on the ring beyond the preference list, so availability doesn't drop with strict membership.
- **Hinted handoff** — the stand-in node stores the write tagged with its intended owner and forwards it when the owner recovers.
- **Read repair** — a coordinator that sees stale replicas during a read pushes the newest version back to them.
- **Anti-entropy with Merkle trees** — replicas periodically compare hash trees of their key ranges and sync only the divergent branches, catching what read repair misses.

(All four: [Dynamo paper §4.6, §4.7](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf); [annotated version](https://mwhittaker.github.io/papers/html/decandia2007dynamo.html).)

### 5.3 CRDTs: Coordination-Free Convergence

**Conflict-free Replicated Data Types** (Shapiro, Preguiça, Baquero & Zawirski, 2011) are data structures whose merge operation is designed so replicas can accept updates **independently and concurrently, without coordination**, and are mathematically guaranteed to converge (strong eventual consistency). Examples: counters, sets (G-Set, OR-Set), sequences for collaborative editing ([original paper, SSS 2011](https://www.lip6.fr/Marc.Shapiro/papers/2011/CRDTs_SSS-2011.pdf); [survey](https://arxiv.org/abs/1805.06358); [overview & production uses — collaborative editing, chat, SoundCloud](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)). CRDTs are the principled replacement for ad-hoc last-write-wins where the data type permits it.

### 5.4 Conflict Resolution Spectrum

From cheapest to most correct: last-write-wins (loses data silently, but commutative/idempotent — what Cassandra uses per-cell) → CRDTs (automatic, lossless for supported types) → application semantic merge (Dynamo shopping cart) → consensus/serialization (no conflicts ever occur, at the cost of coordination on every write).

---

## 6. Consensus & Coordination

When you need a single agreed value/order (leader election, membership, metadata, transactions), quorum overlap is not enough — you need consensus.

### 6.1 Paxos and Raft

- **Paxos** is the classic algorithm; Spanner runs "Paxos state machines... to support replication" with long-lived leaders and pipelining, one Paxos group per data shard ([Spanner paper](https://research.google/pubs/spanner-googles-globally-distributed-database-2/)).
- **Raft** (Ongaro & Ousterhout, USENIX ATC 2014 — Best Paper) was designed explicitly for **understandability**: it decomposes consensus into **leader election, log replication, and safety**, and is equivalent in result and efficiency to (multi-)Paxos. A user study showed students learn Raft more easily than Paxos ([raft.github.io](https://raft.github.io/); [paper (extended version)](https://raft.github.io/raft.pdf); [USENIX page](https://www.usenix.org/conference/atc14/technical-sessions/presentation/ongaro)). Raft's dissertation adds a cluster-membership-change algorithm and a TLA+ formal spec. Raft powers etcd, TiKV, CockroachDB, Consul, and Kafka's KRaft mode.

Key operational properties of quorum consensus: a group of 2f+1 nodes tolerates f failures; safety holds under any asynchrony (FLP only threatens liveness, §2.3); all writes traverse the leader, so consensus groups are kept *small and per-shard* (as in Spanner) rather than system-wide.

### 6.2 What Consensus Is Used For at Scale

High-traffic systems deliberately keep consensus **out of the data path** where possible and use it for the *control plane*: cluster metadata, shard maps, leader election, distributed locks/leases, configuration. (Compare Cassandra: quorum arithmetic for data, Paxos only for opt-in lightweight transactions and — since 5.1 — cluster metadata.) Spanner is the notable system that puts Paxos *in* the data path and makes it fast enough via long-lived leaders, pipelining, and TrueTime-based lock-free reads ([Spanner paper](https://research.google/pubs/spanner-googles-globally-distributed-database-2/)).

---

## 7. Failure Detection & Membership

You cannot fail over from a node you don't know is dead — and in an asynchronous network, "dead" and "slow" are indistinguishable (this is FLP's practical face).

### 7.1 Heartbeats and Their Limits

All-to-all heartbeating is O(N²) in network load and produces false positives under transient congestion. Two refinements dominate practice:

### 7.2 SWIM

**SWIM** (Das, Gupta & Motivala, 2002) separates **failure detection** from **membership dissemination**. Each protocol period, a member pings one random peer; on timeout it asks *k* other members to ping the target indirectly ("outsourced heartbeats") before suspecting it. The **suspicion sub-protocol** marks a non-responsive node *suspected* rather than immediately *failed*, drastically reducing false positives. Membership updates are **piggybacked on the ping/ack traffic and spread infection-style (gossip/epidemic)**, so dissemination latency grows only logarithmically with group size and per-node load is constant ([SWIM paper, Cornell](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf); [protocol walkthrough](https://www.brianstorti.com/swim/)). SWIM descendants (HashiCorp memberlist/Serf/Consul) run production membership for large fleets.

### 7.3 Phi Accrual Failure Detection

Instead of a binary alive/dead verdict from a fixed timeout, the **φ (phi) accrual failure detector** outputs a continuous *suspicion level* computed from the statistical distribution of recent heartbeat inter-arrival times; the application picks its own threshold per use case. Cassandra and Akka use this to adapt to varying network conditions ([overview in SWIM/gossip literature](https://en.wikipedia.org/wiki/SWIM_Protocol); Cassandra's implementation is in its `gms/` gossip subsystem — see the Cassandra architecture note in this repo).

### 7.4 Gossip for State Dissemination

Beyond membership, gossip/epidemic protocols disseminate any cluster state (load, schema versions, tokens) with no leader and probabilistic-but-fast convergence — Dynamo uses gossip to propagate membership and partitioning state so every node can route any request locally ([Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)).

---

## 8. Delivery Guarantees, Idempotency & Distributed Transactions

### 8.1 The Delivery-Semantics Ladder

For any producer→broker→consumer chain ([Kafka delivery semantics docs](https://docs.confluent.io/kafka/design/delivery-semantics.html)):

- **At-most-once** — fire and forget; loss on failure.
- **At-least-once** — retry until acknowledged; duplicates on retry.
- **Exactly-once** — every record is processed once *in effect*, achieved by combining at-least-once delivery with **deduplication or idempotency**, never by the network alone.

### 8.2 How Kafka Implements Exactly-Once

Two mechanisms, both introduced in [KIP-98](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging):

- **Idempotent producer** — each producer gets a Producer ID; every message carries a per-partition sequence number; on retry the broker detects the duplicate sequence and acknowledges without re-writing ([Confluent: "Exactly-once Semantics are Possible"](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)).
- **Transactions** — a producer atomically writes a batch across multiple partitions (including consumer-offset commits): consumers in `read_committed` mode see all of the batch or none. Kafka transactions cover data *within Kafka only* — no external systems ([Confluent: "Transactions in Apache Kafka"](https://www.confluent.io/blog/transactions-apache-kafka/); [Kafka transactions course](https://developer.confluent.io/courses/architecture/transactions/)).

The transferable pattern: **exactly-once = at-least-once + idempotence**, with idempotency keys/sequence numbers as the deduplication mechanism ([Idempotent Writer pattern](https://developer.confluent.io/patterns/event-processing/idempotent-writer/)). MillWheel reached the same design a decade earlier: idempotent per-record processing plus checkpointing so "from the user's perspective, record delivery occurs exactly once" ([MillWheel paper](https://research.google.com/pubs/archive/41378.pdf)).

### 8.3 Distributed Transactions

- **Two-phase commit (2PC)** provides atomic commit across shards but blocks if the coordinator dies at the wrong moment. Spanner's fix: run **2PC on top of Paxos groups**, so both participants and the coordinator's state are themselves replicated and non-blocking, and use TrueTime for externally-consistent timestamps ([Spanner paper](https://research.google/pubs/spanner-googles-globally-distributed-database-2/)).
- Systems that can't afford coordination avoid cross-shard transactions structurally: design aggregates to fit one shard, use single-partition atomic batches, or accept saga-style compensation at the application layer (an application pattern; the primitive that makes it safe is again idempotency, §8.2).

---

## 9. Realtime Stream Processing

The realtime half of "realtime distributed systems": continuous computation over unbounded, out-of-order event streams.

### 9.1 The Log as the Backbone

High-traffic realtime architectures decouple producers from consumers with a partitioned, replicated **commit log** (Kafka): producers append; consumer groups read independently at their own pace; partitions provide the ordering unit and the parallelism unit; replication (leader + ISR) provides fault tolerance ([Kafka design docs](https://docs.confluent.io/kafka/design/delivery-semantics.html)). The log absorbs traffic spikes (buffering) and makes downstream reprocessing/replay possible — the property that enables the "kappa" style of serving both realtime and reprocessing from one stream pipeline.

### 9.2 Out-of-Order Data: Watermarks, Windows, Triggers

The **Dataflow model** (Google, VLDB 2015 — the model behind Apache Beam/Cloud Dataflow and adopted by Flink) starts from the observation that unbounded data cannot be globally ordered on arrival, so correctness must be defined over **event time**:

- **Windowing** slices unbounded data into finite buckets (fixed, sliding, session windows).
- **Watermarks** are the system's moving estimate of "no events with event time ≤ T remain in flight," used to decide window completeness ([Dataflow model insights](https://www.hemantkgupta.com/p/insights-from-paper-google-the-dataflow); [InfoQ on Dataflow/Beam](https://www.infoq.com/articles/dataflow-apache-beam/)).
- **Triggers + late-data policies** control emitting early/on-time/late results and how much lateness to tolerate, making the latency-vs-completeness tradeoff explicit and tunable ([Confluent on watermarks](https://www.confluent.io/blog/watermarks-tables-event-time-dataflow-model/); formal semantics comparison Flink vs Dataflow: [VLDB 2021](http://www.vldb.org/pvldb/vol14/p3135-begoli.pdf)).

MillWheel pioneered the production version of this: its **low watermark** gives "a tight bound on the timestamp of all events still in the system," substituting for in-order delivery ([MillWheel, VLDB 2013](https://research.google.com/pubs/archive/41378.pdf)).

### 9.3 Fault-Tolerant State: Distributed Snapshots

Stateful streaming needs consistent recovery. The theoretical base is the **Chandy–Lamport algorithm** (1985): record a consistent global state of a running asynchronous system by flowing **marker messages** through the channels, without halting processing ([Chandy–Lamport](https://en.wikipedia.org/wiki/Chandy%E2%80%93Lamport_algorithm); [Princeton precept notes](https://www.cs.princeton.edu/courses/archive/fall16/cos418/docs/P8-chandy-lamport.pdf)).

**Flink's checkpointing is an adaptation of Chandy–Lamport**: barriers injected into the stream align operator state snapshots; on failure the job rewinds all operators to the last completed checkpoint and replays the source from that point, giving exactly-once *state* semantics ([Flink stateful stream processing docs](https://nightlies.apache.org/flink/flink-docs-master/docs/concepts/stateful-stream-processing/); [checkpointing docs](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/fault-tolerance/checkpointing/)). **End-to-end** exactly-once additionally requires transactional/idempotent sinks — Flink's two-phase-commit sink coordinated with checkpoints, e.g., with Kafka transactions ([Flink blog: end-to-end exactly-once](https://flink.apache.org/2018/02/28/an-overview-of-end-to-end-exactly-once-processing-in-apache-flink-with-apache-kafka-too/)). MillWheel instead uses per-record **strong productions** (checkpoint-before-emit) and upstream backup ([MillWheel paper](https://research.google.com/pubs/archive/41378.pdf)).

### 9.4 Backpressure

A pipeline is only as fast as its slowest stage; without flow control, queues between stages grow without bound and the system falls over far from the bottleneck. Streaming engines propagate **backpressure** upstream (Flink via bounded network buffers; Kafka consumers naturally via pull-based consumption). The failure mode this prevents — unbounded queue backlogs that outlive the original overload — is analyzed in AWS's Builders' Library ([avoiding insurmountable queue backlogs](https://lumigo.io/blog/amazon-builders-library-in-focus-4-avoiding-insurmountable-queue-backlogs/)) and is one trigger of metastable failure (§11.6).

*Deep dives on Flink and Kafka-adjacent internals: see `analytics/flink-architecture.md` and `analytics/flink-internals.md` in this repo.*

---

## 10. Traffic Management at Scale

### 10.1 Load Balancing Layers

Traffic sheds through multiple tiers, each with its own balancing decision: DNS/anycast (global, coarse) → L4 (connection-level) → L7 (request-level, content-aware) → client-side/service-mesh balancing between internal services. Google's SRE book dedicates chapters to frontend and datacenter load balancing and, critically, to **subsetting and balancing policies** that avoid both connection explosion and load imbalance ([SRE book table of contents](https://sre.google/sre-book/table-of-contents/); [Handling Overload chapter](https://sre.google/sre-book/handling-overload/)).

Two grounded, non-obvious practices:

- **Client-side adaptive throttling**: Google's clients self-limit when a backend rejects requests — each client tracks its requests vs. accepts and probabilistically rejects *locally* beyond a multiple of the accept rate, so rejection cost isn't paid at the overloaded server ([SRE Ch. 21 "Handling Overload"](https://sre.google/sre-book/handling-overload/)).
- **Load is not QPS**: Google models utilization in abstract cost units (CPU), because "queries per second" varies wildly in per-request cost; quota and shedding decisions based on QPS alone misfire ([same chapter](https://sre.google/sre-book/handling-overload/)).

### 10.2 Caching

Caches convert read traffic into O(1) memory lookups and are the single biggest lever for read-heavy high traffic — but they create their own failure modes documented in the SRE cascading-failures chapter: a cold cache after restart means the backend sees the *full* uncached load, which it may no longer be provisioned for; a system that *requires* its cache to meet capacity has turned the cache into a hard dependency ([SRE Ch. 22](https://sre.google/sre-book/addressing-cascading-failures/)). Mitigations covered there: cache warming before serving, request coalescing, and provisioning for cache-miss load.

### 10.3 Rate Limiting & Admission Control

Per-tenant quotas and rate limiters protect shared services from any single client; AWS frames the general principle as *the smaller service must stay in control* of how much work larger fleets can push into it — via polling (pull) rather than push, and explicit admission control at the front door ([AWS Builders' Library: avoiding overload by putting the smaller service in control](https://aws.amazon.com/builders-library/avoiding-overload-in-distributed-systems-by-putting-the-smaller-service-in-control/)).

### 10.4 Autoscaling — With Care

Elastic capacity handles diurnal and spike load, but the SRE cascading-failures chapter warns that autoscaling interacts badly with failure: health-check-based systems can remove "unhealthy" (actually overloaded) instances, shrinking capacity exactly when load rises ([SRE Ch. 22](https://sre.google/sre-book/addressing-cascading-failures/)). Static stability (§12.3) is the AWS-side answer.

---

## 11. Overload & Resilience Patterns

The heart of fault tolerance in the request path. These patterns are individually simple; the discipline is applying them *everywhere consistently*.

### 11.1 Timeouts

Every remote call needs a deadline; without one, a slow dependency consumes threads/connections until the caller dies too. Amazon's guidance: set timeouts from observed latency distributions (not guesses), and beware both too-high (resource exhaustion) and too-low (spurious failures on healthy-but-slow calls) ([AWS Builders' Library: "Timeouts, retries, and backoff with jitter"](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)).

### 11.2 Retries, Exponential Backoff & Jitter

Retries are "selfish" — they spend server capacity to improve one client's odds — and synchronized retries after a blip re-create the overload that caused the failures. The remedy chain, per the same AWS article: retry a *bounded* number of times → back off *exponentially* → add **jitter** (randomness) so retries spread over time rather than arriving in waves — and apply jitter not just to retries but to any periodic/delayed work ([AWS: timeouts/retries/jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/); quantitative comparison of jitter strategies: [AWS Architecture Blog, "Exponential Backoff and Jitter"](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)). A further rule from the SRE book: retry at *one* level of the stack — retries amplify multiplicatively across layers ([SRE Ch. 22](https://sre.google/sre-book/addressing-cascading-failures/)).

### 11.3 Load Shedding & Graceful Degradation

An overloaded server that tries to serve everyone serves no one: clients time out and retry, so all work done past the client's deadline is wasted ("a vicious cycle"). **Load shedding** rejects excess requests cheaply at admission so accepted requests keep meeting latency goals — deliberately sacrificing some requests to raise overall availability ([Lumigo's analysis of AWS's load-shedding article](https://lumigo.io/blog/amazon-builders-library-in-focus-2-using-load-shedding-to-avoid-overload/); [SRE Ch. 21](https://sre.google/sre-book/handling-overload/)). **Graceful degradation** goes further: reduce the *cost* of each response (partial data, cheaper algorithm) instead of rejecting outright ([SRE Ch. 22](https://sre.google/sre-book/addressing-cascading-failures/)). Criticality-based shedding (drop batch/speculative traffic first, user-facing last) is the SRE-documented refinement ([SRE Ch. 21](https://sre.google/sre-book/handling-overload/)).

### 11.4 Circuit Breakers & Bulkheads

When a dependency is failing, continuing to call it wastes resources and propagates latency. A **circuit breaker** trips after an error threshold and fails fast (optionally probing periodically to close again); a **bulkhead** partitions resources (thread pools, connection pools) per dependency so one bad dependency can't drain shared capacity. These patterns were popularized in production by Netflix's Hystrix library and are discussed as standard remediations in the SRE cascading-failures chapter (fail fast, bounded queues, per-dependency limits) ([SRE Ch. 22](https://sre.google/sre-book/addressing-cascading-failures/); [chaos-engineering history covering Netflix's resilience tooling](https://www.gremlin.com/community/tutorials/chaos-engineering-the-history-principles-and-practice)).

### 11.5 Cascading Failures

The unifying failure model: one replica fails or slows → its load shifts to the rest → they saturate → "a domino effect that takes down all the replicas for a service." Positive-feedback loops (retries, cache misses after restarts, health-checkers removing overloaded-but-working nodes) are the fuel. Documented triggers and mitigations — including the counterintuitive first-response of *adding capacity and reducing load before debugging* — are in [SRE Ch. 22 "Addressing Cascading Failures"](https://sre.google/sre-book/addressing-cascading-failures/).

### 11.6 Metastable Failures

A formalized class of outage (HotOS 2021, expanded at OSDI 2022): a trigger pushes the system into an overloaded state, and a **sustaining effect** (retry storms, cache-cold misses, queue backlogs) keeps it there **even after the trigger is fixed** — recovery requires deliberately breaking the feedback loop (shedding drastically, restarting with caches warmed, disabling retries). The papers document such outages at AWS, Azure, and Google Cloud ([metastable failures overview + paper links](https://sites.psu.edu/timothyz/metastable-failures/)). This is the theoretical umbrella over §11.2–11.5: most "mystery" large-scale outages are metastable states.

### 11.7 Tail-Latency Techniques

From [Dean & Barroso, "The Tail at Scale"](https://www.barroso.org/publications/TheTailAtScale.pdf) ([CACM version](https://cacm.acm.org/research/the-tail-at-scale/)) — techniques that create "a predictably responsive whole out of less-predictable parts," usually reusing capacity already provisioned for fault tolerance:

- **Hedged requests** — send the request to a second replica after the first hasn't answered within (say) the p95 latency; take the first response, cancel the other. Bounds added load to ~5% while collapsing the tail ([paper](https://www.barroso.org/publications/TheTailAtScale.pdf); [practitioner treatment](https://dev.to/gabrielanhaia/request-hedging-the-tail-at-scale-technique-most-teams-skip-2j37)).
- **Tied requests** — enqueue on two servers simultaneously, each request carrying the identity of the other; whichever dequeues first cancels its twin, cutting queueing-delay variance without doubling work.
- **Micro-partitioning & selective replication** — many more partitions than machines (cf. virtual nodes, §4.1) enables fine-grained load shifting; hot partitions get extra replicas.
- **Latency-induced probation** — temporarily stop sending to a slow replica (observe: a *slow* node is worse than a dead one, because failure detectors don't remove it).
- **Good-enough responses & canary requests** — return partial fan-out results when the stragglers aren't worth waiting for; send novel queries to one or two servers first to avoid fleet-wide crashes from a poison request.

Root causes of latency variability the paper catalogs: shared resources, background daemons, queueing, garbage collection, power/thermal throttling — worth internalizing because they explain *why* the tail exists even in healthy fleets.

---

## 12. High-Availability Architecture Patterns

### 12.1 Redundancy Across Failure Domains

Replicas are placed across independent failure domains — machines, racks, availability zones, regions. Dynamo's preference list spans multiple datacenters ([Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)); Spanner replicates across datacenters/continents with Paxos, letting clients trade replication distance for latency ([Spanner paper](https://research.google/pubs/spanner-googles-globally-distributed-database-2/)). The quorum math determines the cost: surviving a zone failure with consensus requires ≥3 zones.

### 12.2 Cell-Based Architecture & Blast Radius

Instead of one giant fleet, partition the service into many self-contained **cells**, each serving a slice of customers; a bad deployment, poison workload, or metastable collapse takes out one cell, not the service. Shuffle sharding (§4.3) is the probabilistic variant; both are core AWS blast-radius doctrine ([AWS Builders' Library](https://aws.amazon.com/builders-library/avoiding-overload-in-distributed-systems-by-putting-the-smaller-service-in-control/) develops the workload-isolation reasoning; the Builders' Library is AWS's first-party collection of these practices).

### 12.3 Static Stability

A statically stable system keeps operating correctly when its *control plane* (the thing that reconfigures it) is down: pre-provisioned capacity in each AZ rather than reactive scale-up during an AZ loss; data planes that don't require the control plane on the request path. The SRE cascading-failures chapter makes the corresponding point from the Google side: systems provisioned assuming reactive replacement/scale-up fail when the failure itself prevents the reaction ([SRE Ch. 22](https://sre.google/sre-book/addressing-cascading-failures/)).

### 12.4 Event Sourcing / Log-Centric Design

Making the replicated log the source of truth (Kafka-centric architectures, §9.1; Cassandra's "log-structured everything"; Raft's replicated log) gives HA properties for free: replayability after failure, decoupled consumers that can lag and catch up, and natural audit. The log is the common abstraction under Raft (§6.1), Kafka (§8.2), and stream processing (§9).

### 12.5 Data-Layer HA Reference Points

The systems documented elsewhere in this repo instantiate these patterns; cross-references:

- `datastores/cassandra-architecture.md` — Dynamo-style leaderless replication + LSM storage, tunable consistency, gossip membership.
- `datastores/yugabytedb-architecture.md` — Raft-per-shard (Spanner-style) strongly consistent SQL.
- `datastores/redis-architecture.md` — in-memory serving layer, sentinel/cluster failover.
- `analytics/clickhouse-architecture-internals.md`, `analytics/presto-technical-architecture.md` — realtime analytics serving.

---

## 13. Operational Practice: Observability, SRE & Chaos

High availability is achieved operationally as much as architecturally.

### 13.1 Distributed Tracing

At scale, a request's latency is spread across dozens of services; logs per-service cannot reconstruct causality. Google's **Dapper** established production-scale distributed tracing with three design goals — **low overhead, application-level transparency, and ubiquitous deployment** — via sampling and trace/span context propagated through RPCs ([Dapper paper summary/notes](https://mwhittaker.github.io/papers/html/sigelman2010dapper.html); [Semantic Scholar record](https://www.semanticscholar.org/paper/Dapper,-a-Large-Scale-Distributed-Systems-Tracing-Sigelman-Barroso/003d5a65de0ac72daaf105ded903cb3eb88585b3)). Dapper directly inspired OpenTelemetry, Zipkin, and Jaeger. Traces are also the primary tool for *finding* the tail-latency causes of §11.7.

### 13.2 SLOs, Error Budgets & Measuring by Percentile

The SRE model: define **SLIs** (measured indicators — latency percentiles, error rate), set **SLOs** (targets), and run on the **error budget** (1 − SLO) — spending it on velocity, freezing risk when it's exhausted. Availability targets and monitoring philosophy are laid out in the SRE book's foundational chapters ([SRE book, table of contents](https://sre.google/sre-book/table-of-contents/)). Monitor latency as a distribution, never a mean — the mean is precisely what "The Tail at Scale" shows to be misleading ([Dean & Barroso](https://www.barroso.org/publications/TheTailAtScale.pdf)).

### 13.3 Chaos Engineering

Testing cannot enumerate the emergent failure modes of a distributed system; **chaos engineering** finds them empirically: define a measurable **steady state**, hypothesize it persists under injected real-world events (instance kill, network loss), run the experiment (ideally in production, with bounded blast radius), and disprove the hypothesis ([principlesofchaos.org](https://principlesofchaos.org/)). The practice started with Netflix's **Chaos Monkey** (2011), which randomly disables production instances during business hours to verify redundancy actually works ([history & principles](https://www.gremlin.com/community/tutorials/chaos-engineering-the-history-principles-and-practice); [overview](https://en.wikipedia.org/wiki/Chaos_engineering)). Chaos experiments are the standing verification for everything in §11–12: if killing a node moves your p99, your tail-tolerance is fiction.

### 13.4 Safe Change: The Dominant Cause of Outages Is You

Most large outages are triggered by changes (deploys, config pushes), not hardware. Grounded practices: progressive rollouts with automated rollback and **canarying** (send a small slice of traffic to the new version, compare SLIs before proceeding) — treated at length in Google's SRE material on release engineering and canarying ([SRE book/workbook index](https://sre.google/sre-book/table-of-contents/)); and **testing for the overload regime itself** — the SRE overload chapter stresses load-testing to the breaking point so the failure mode at saturation is known, not discovered ([SRE Ch. 21](https://sre.google/sre-book/handling-overload/)).

### 13.5 Formal Verification Where It Counts

For the small protocols everything else depends on (consensus, membership change, replication), model-checking pays: Raft ships a **TLA+ formal specification** in Ongaro's dissertation ([raft.github.io](https://raft.github.io/)). The pattern — formally specify the ~1% of the system that is a distributed protocol — is established industry practice for exactly the components whose bugs are unreproducible in tests.

---

## 14. References

### Papers (primary sources)

| Topic | Reference |
|---|---|
| Logical clocks, happens-before | Lamport, ["Time, Clocks, and the Ordering of Events in a Distributed System"](https://en.wikipedia.org/wiki/Lamport_timestamp), CACM 1978 |
| Consensus impossibility | Fischer, Lynch, Paterson, ["Impossibility of Distributed Consensus with One Faulty Process"](https://paperswelove.org/papers/impossibility-of-distributed-consensus-with-one-fa-80e1c33b/), JACM 1985 · [proof notes](https://team.inria.fr/antique/the-impossibility-result-of-fischer-lynch-and-paterson-and-its-proof/) |
| Distributed snapshots | Chandy & Lamport, ["Distributed Snapshots"](https://en.wikipedia.org/wiki/Chandy%E2%80%93Lamport_algorithm), 1985 |
| CAP theorem | Gilbert & Lynch, 2002 — [overview & context](https://en.wikipedia.org/wiki/CAP_theorem) · [Brewer's 12-year retrospective context](http://muratbuffalo.blogspot.com/2015/02/paper-summary-perspectives-on-cap.html) |
| PACELC | Abadi 2010; formalized in [Golab, "Proving PACELC"](https://uwaterloo.ca/distributed-algorithms-systems-lab/sites/default/files/uploads/files/proving_pacelc.pdf) |
| Dynamo (leaderless replication, consistent hashing, quorums) | DeCandia et al., [SOSP 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) · [annotated](https://mwhittaker.github.io/papers/html/decandia2007dynamo.html) |
| Raft | Ongaro & Ousterhout, [USENIX ATC 2014](https://www.usenix.org/conference/atc14/technical-sessions/presentation/ongaro) · [extended paper](https://raft.github.io/raft.pdf) · [raft.github.io](https://raft.github.io/) |
| Spanner / TrueTime | Corbett et al., [OSDI 2012](https://research.google/pubs/spanner-googles-globally-distributed-database-2/) · [TrueTime docs](https://docs.cloud.google.com/spanner/docs/true-time-external-consistency) · [Brewer on Spanner & CAP](https://research.google.com/pubs/archive/45855.pdf) |
| Tail latency | Dean & Barroso, ["The Tail at Scale"](https://www.barroso.org/publications/TheTailAtScale.pdf), CACM 2013 · [ACM page](https://cacm.acm.org/research/the-tail-at-scale/) |
| CRDTs | Shapiro et al., [SSS 2011](https://www.lip6.fr/Marc.Shapiro/papers/2011/CRDTs_SSS-2011.pdf) · [survey](https://arxiv.org/abs/1805.06358) |
| SWIM membership | Das, Gupta, Motivala, [2002](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf) · [walkthrough](https://www.brianstorti.com/swim/) |
| MillWheel (streaming, low watermark) | Akidau et al., [VLDB 2013](https://research.google.com/pubs/archive/41378.pdf) |
| Watermark semantics (Flink vs Dataflow) | Begoli et al., [VLDB 2021](http://www.vldb.org/pvldb/vol14/p3135-begoli.pdf) |
| Dapper (tracing) | Sigelman et al., 2010 — [notes](https://mwhittaker.github.io/papers/html/sigelman2010dapper.html) |
| Metastable failures | Bronson et al. HotOS 2021 / Huang et al. OSDI 2022 — [overview & links](https://sites.psu.edu/timothyz/metastable-failures/) |

### Official documentation & first-party engineering sources

- Google SRE Book — [Ch. 21 Handling Overload](https://sre.google/sre-book/handling-overload/), [Ch. 22 Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/), [full index](https://sre.google/sre-book/table-of-contents/)
- AWS Builders' Library — [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/), [Avoiding overload by putting the smaller service in control](https://aws.amazon.com/builders-library/avoiding-overload-in-distributed-systems-by-putting-the-smaller-service-in-control/) · [Exponential Backoff and Jitter (Architecture Blog)](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) · analyses of [load shedding](https://lumigo.io/blog/amazon-builders-library-in-focus-2-using-load-shedding-to-avoid-overload/) and [queue backlogs](https://lumigo.io/blog/amazon-builders-library-in-focus-4-avoiding-insurmountable-queue-backlogs/)
- Kafka / Confluent — [Delivery semantics](https://docs.confluent.io/kafka/design/delivery-semantics.html), [Exactly-once semantics](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/), [Transactions](https://www.confluent.io/blog/transactions-apache-kafka/), [KIP-98](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging)
- Apache Flink — [Stateful stream processing](https://nightlies.apache.org/flink/flink-docs-master/docs/concepts/stateful-stream-processing/), [Checkpointing](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/fault-tolerance/checkpointing/), [End-to-end exactly-once with Kafka](https://flink.apache.org/2018/02/28/an-overview-of-end-to-end-exactly-once-processing-in-apache-flink-with-apache-kafka-too/)
- Chaos engineering — [Principles of Chaos Engineering](https://principlesofchaos.org/) · [history (Gremlin)](https://www.gremlin.com/community/tutorials/chaos-engineering-the-history-principles-and-practice) · [overview](https://en.wikipedia.org/wiki/Chaos_engineering)
