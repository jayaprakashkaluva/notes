# Partitioning & Replication: The Dynamo Design Space and CRDTs

> Part 2 of the realtime distributed systems series. Index: [realtime-high-scale-distributed-systems.md](realtime-high-scale-distributed-systems.md)
> **Scope:** Consistent hashing and its production evolutions, quorum replication mechanics (sloppy quorums, hinted handoff, read repair, Merkle anti-entropy), the quorum-math design space, and CRDTs as the coordination-free end of the replication spectrum.
> **Primary source:** [DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store", SOSP 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) ([annotated version](https://mwhittaker.github.io/papers/html/decandia2007dynamo.html)) — still the densest single catalog of these techniques; Cassandra, Riak, and Voldemort are its direct descendants (see `../datastores/cassandra-architecture.md` for a codebase-level view).

---

## 1. The SLA That Shaped the Design

Dynamo's requirements are stated as a **tail-latency SLA, not an average**: e.g. 99.9% of responses within 500 ms. The paper's premise is that at Amazon's scale "there are always failed components," so the design treats failure handling as the normal case and optimizes the *worst* customer experiences, not the mean. Every mechanism below follows from those two commitments. (Both: [Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf).)

## 2. Consistent Hashing, For Real

### 2.1 Base scheme and its two problems

Keys and nodes hash onto a ring; each node owns the arc between its predecessor's token and its own; node arrival/departure moves only adjacent keys. Two production problems:

1. **Load skew** — random token placement gives nodes uneven arcs, and key popularity is itself skewed.
2. **Heterogeneity** — a fleet is never uniform hardware.

Both are addressed with **virtual nodes**: each physical node holds many tokens; more powerful machines take more tokens; when a node dies, its arcs disperse across *many* survivors instead of doubling one neighbor's load.

### 2.2 The three partitioning strategies (Dynamo §6.2) — an evolution worth memorizing

The paper documents its own redesign, which is the interesting part ([annotated summary](https://mwhittaker.github.io/papers/html/decandia2007dynamo.html)):

1. **Strategy 1 — random tokens, partition = arc between tokens.** Simple, but partition boundaries move whenever membership changes, so (a) data must be rescanned/rehashed on every join, and (b) Merkle trees (§5) must be recomputed because the ranges they cover changed.
2. **Strategy 2 — random tokens over *fixed* equal-size partitions.** Decouples partitioning from placement: the keyspace is pre-divided into Q equal partitions; tokens only decide which node *stores* each partition.
3. **Strategy 3 — fixed equal partitions, Q/S tokens per node.** Placement becomes "each of S nodes holds Q/S whole partitions." Won on all axes: fastest bootstrap/recovery (transfer whole partitions), trivial Merkle maintenance (one tree per fixed partition), simpler membership bookkeeping.

The general lesson: **decouple the unit of data placement (fixed partitions) from the unit of membership (nodes)** — the same conclusion "The Tail at Scale" reaches from the latency direction with micro-partitions ([Dean & Barroso](https://www.barroso.org/publications/TheTailAtScale.pdf); see [file 07](07-tail-latency-and-ha-architecture.md)), and the design Cassandra, Riak, and (as tablets/directories) Spanner all converged on.

## 3. Replication and the Write/Read Path

### 3.1 Preference lists and coordinators

Each key's **preference list** is the N nodes responsible for it — the first N *distinct physical nodes* walking clockwise from the key's hash (skipping virtual-node duplicates), spread across datacenters for survivability. Every node knows the preference list for any key (membership is fully gossiped, §6), so **any node can route, and any of the top-N can coordinate**: the coordinator writes locally and sends to the other N−1, acknowledging after W−1 replies ([Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)). Zero-hop routing is a deliberate latency choice — routing state is O(cluster) per node in exchange for no proxy hop.

### 3.2 Quorum arithmetic as a tuning surface

With N replicas, writes acknowledged by W, reads by R:

| Setting | Buys you | Costs you |
|---|---|---|
| `R + W > N` | Read and write quorums intersect → reads see the latest *acknowledged* write | Latency of the slower quorum member |
| `W = 1` | Fastest, most available writes | Reads must do the consistency work; durability rides on one node until repair |
| `R = 1` | Fastest reads | Staleness up to replication lag |
| `W = N` | Read-anywhere | Writes unavailable if any replica is down |

Staff-level caveats:

- **Quorum intersection is not linearizability.** Dynamo explicitly classifies itself as eventually consistent even with `R + W > N`, because sloppy quorums (§4) mean the W acknowledgers may not be in the canonical top-N, and concurrent coordinators can produce siblings that only vector clocks (see [file 01 §5.2](01-foundations-time-and-ordering.md)) can adjudicate ([paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)).
- **Latency is set by the W-th (R-th) fastest replica** — quorums are also a tail-latency tool: `W=2, N=3` masks one slow node per write. This is the fault-tolerance capacity that hedged requests reuse ([file 07](07-tail-latency-and-ha-architecture.md)).

### 3.3 Handling transient failure: sloppy quorum + hinted handoff

Under strict quorum, a key with 2 of 3 preference-list nodes unreachable is unwritable. Dynamo instead uses a **sloppy quorum**: the write goes to the next healthy nodes walking the ring past the failures. The stand-in stores the object **tagged with a hint** naming the intended owner and delivers it when the owner recovers (**hinted handoff**) ([paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)). Consequences to internalize:

- Availability for writes approaches "any W nodes anywhere alive" — this is the paper's "always writeable" goal (shopping carts must never refuse an add).
- The same mechanism *weakens* what a quorum overlap means (§3.2) — sloppiness is a knob you can disable per-keyspace where correctness demands it.

### 3.4 Repairing divergence: read repair and Merkle anti-entropy

Two convergence mechanisms with different reach:

- **Read repair** (opportunistic, hot data): after answering a read, the coordinator pushes the newest version to any replica that returned a stale one. Free for frequently-read keys; never touches cold keys.
- **Anti-entropy with Merkle trees** (systematic, cold data): each node keeps a hash tree per partition (leaves = hashes of key ranges, parents = hashes of children). Replicas compare roots; on mismatch they descend only into differing subtrees, so comparison traffic is proportional to *divergence*, not data size. With fixed partitions (Strategy 3) trees survive membership changes ([paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf); Cassandra's repair is this exact design — see `../datastores/cassandra-architecture.md`).

Design rule: opportunistic repair for the hot set, scheduled anti-entropy for the long tail, hinted handoff to shrink the window both must cover.

## 4. Membership: Explicit, Gossiped, Seeded

Dynamo's membership choices are deliberately conservative ([paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf); [annotated](https://mwhittaker.github.io/papers/html/decandia2007dynamo.html)):

- **Membership changes are initiated manually by an operator** — automatic ejection is dangerous because a network blip could trigger mass rebalancing (data movement), so *failure detection* (local, transient, drives request routing) is separated from *membership* (global, durable, drives data placement). This separation is the important idea; see [file 04](04-membership-failure-detection-gossip.md) for the detection half.
- Membership and token state spread by **gossip**; **seed nodes** that everyone eventually gossips with prevent logically split rings when nodes join concurrently.

## 5. CRDTs: The Coordination-Free Endpoint

If conflicts can be *merged automatically and losslessly*, replication needs no coordination at all. **Conflict-free Replicated Data Types** (Shapiro, Preguiça, Baquero & Zawirski, SSS 2011) formalize this: replicas accept updates independently and concurrently, and are mathematically guaranteed to converge — **Strong Eventual Consistency** (convergence is deterministic, not merely eventual) ([original paper](https://www.lip6.fr/Marc.Shapiro/papers/2011/CRDTs_SSS-2011.pdf); [comprehensive survey](https://arxiv.org/abs/1805.06358)).

Two equivalent constructions (both defined in the paper):

- **State-based (CvRDT):** replica states form a **join-semilattice**; updates are monotonic (inflationary); merge = least-upper-bound of two states. LUB is commutative, associative, idempotent — so replicas can merge in any order, any number of times, over any gossip topology. Ships whole (or delta) state.
- **Operation-based (CmRDT):** concurrent operations are designed to **commute**; requires reliable exactly-once *causal-order* delivery of operations from the transport. Ships small ops, but demands more from the messaging layer.

Workhorse types (from the paper's specifications): **G-Counter/PN-Counter** (grow-only / increment-decrement counters as per-replica vectors), **G-Set** (grow-only set), **OR-Set** (observed-remove set — add wins over concurrent remove by tagging adds with unique ids), **LWW-Register** (last-write-wins — convergent but *lossy*; a CRDT in mechanism, not in spirit), plus sequence CRDTs for collaborative text. Production use spans collaborative editing, chat systems, and SoundCloud, among others ([overview](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)).

**Limits** (why CRDTs don't end the consistency debate): they guarantee convergence, not *invariants across objects* — "balance never negative" or "at most one winner" are global preconditions that fundamentally require coordination. CRDTs remove coordination where the merge is semantically valid; they cannot manufacture validity.

## 6. The Conflict-Resolution Spectrum (unified view)

Ordered by coordination cost, with where each mechanism in this series sits:

1. **Last-write-wins** — cheapest, silently lossy; requires only a timestamp (Cassandra cell reconciliation).
2. **CRDT merge** — automatic and lossless *for supported types* ([§5](#5-crdts-the-coordination-free-endpoint)).
3. **Application semantic merge** — Dynamo returning sibling versions + vector clocks to the client (cart union) ([paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)).
4. **Single-leader serialization** — conflicts prevented per-shard by funneling writes (Raft/Spanner groups, [file 03](03-consensus-raft-and-spanner.md)).
5. **Global consensus / transactions** — conflicts prevented across shards (Spanner 2PC-over-Paxos, [file 03](03-consensus-raft-and-spanner.md)).

The architecture question is never "which is best" but **which invariants force you rightward**, and for which keys — high-traffic systems run several of these simultaneously, per data class.
