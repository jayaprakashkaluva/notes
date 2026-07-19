# Foundations: Impossibility Results, Consistency Models, Time & Ordering

> Part 1 of the realtime distributed systems series. Index: [realtime-high-scale-distributed-systems.md](realtime-high-scale-distributed-systems.md)
> **Scope:** The theory that bounds what any design can achieve — CAP/PACELC/FLP with proof intuition — plus the machinery of time and ordering: logical clocks, vector clocks, and TrueTime.

---

## 1. CAP, Precisely

### 1.1 The formal statement

Gilbert & Lynch's 2002 proof formalizes Brewer's PODC 2000 conjecture with precise definitions ([context and formalization](https://en.wikipedia.org/wiki/CAP_theorem); [analysis of the proof and its refinements](http://muratbuffalo.blogspot.com/2015/02/paper-summary-perspectives-on-cap.html)):

- **C** = linearizability of a single read/write register (every operation appears atomic at a point between invocation and response, in real-time order).
- **A** = every request received by a *non-failing* node must eventually return a response.
- **P** = the network may lose arbitrarily many messages between two groups of nodes.

**Proof intuition** (the whole theorem is this simple): partition the nodes into two sides G1 and G2 with all messages between them lost. A client writes `v1` to G1; the write must complete (availability). A client then reads from G2; the read must return (availability) but cannot have learned of `v1` (partition) — so it returns stale data, violating linearizability. Therefore during a partition you choose: respond possibly-stale (**AP**) or refuse/block (**CP**).

### 1.2 What CAP does *not* say — the misreadings that matter at staff level

1. **CAP constrains behavior only while a partition is active.** It says nothing about latency, throughput, or consistency in the healthy case. The theorem that governs the healthy case is PACELC (§2).
2. **The C/A choice is per-operation, not per-system.** Dynamo-style stores expose it as per-request `(R, W)` quorum settings ([Dynamo, SOSP 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)); a single system can serve CP reads for one endpoint and AP reads for another.
3. **In an asynchronous network you cannot "choose CA."** Partition tolerance is not optional for a system whose nodes communicate over a real network; "CA" only describes single-node or perfectly-reliable-network fictions.
4. **Availability in CAP is not the availability in your SLA.** A CP system that fails 0.001% of requests during rare partitions can be *operationally* more available than an AP system with a buggy conflict-resolution path. Google's position paper on Spanner makes exactly this argument: Spanner is **technically CP** — during a partition it sacrifices availability — yet achieves better than 5-nines *measured* availability because Google engineered the partition rate down (private fiber backbone, redundant paths), so users "effectively" experience CA ([Brewer, "Spanner, TrueTime & The CAP Theorem", 2017](https://research.google.com/pubs/archive/45855.pdf)). The staff-level takeaway: **the partition rate is an engineering variable, not a constant of nature** — you can spend money on the network instead of weakening the consistency model.

## 2. PACELC — the theorem you pay for on every request

Abadi's PACELC (2010): **if Partition → Availability vs Consistency; Else → Latency vs Consistency.** The "ELC" half is where high-traffic systems live, because it is paid continuously: any replication scheme that acknowledges a write before all replicas have it is trading consistency for latency; any scheme that waits is trading latency for consistency. Golab's 2018 formalization connects PACELC to proven latency lower bounds for distributed shared objects — the tradeoff is not folklore but theorem ([Golab, "Proving PACELC"](https://uwaterloo.ca/distributed-algorithms-systems-lab/sites/default/files/uploads/files/proving_pacelc.pdf)).

Positioning the systems in this series by the definitions:

| System | Partition behavior | Healthy-case behavior |
|---|---|---|
| Dynamo/Cassandra (leaderless, sloppy quorum) | PA — keeps accepting writes | EL — async convergence, tunable per request |
| Spanner (Paxos + TrueTime) | PC — minority side blocks | EC — pays commit-wait and quorum latency for external consistency |
| Kafka (ISR replication) | tunable via `acks`/`min.insync.replicas` | tunable: `acks=all` is EC-flavored, `acks=1` is EL-flavored |

## 3. FLP Impossibility

**Statement** (Fischer, Lynch & Paterson, JACM 1985): in a fully asynchronous message-passing system, no *deterministic* consensus protocol can guarantee termination if even a single process may crash-fail ([paper record](https://paperswelove.org/papers/impossibility-of-distributed-consensus-with-one-fa-80e1c33b/); [proof walkthrough](https://team.inria.fr/antique/the-impossibility-result-of-fischer-lynch-and-paterson-and-its-proof/); [2001 PODC Influential Paper Award](https://www.podc.org/influential/2001-influential-paper/)).

**Proof shape** (valence argument): call a system configuration *bivalent* if both decision values are still reachable. FLP shows (a) some initial configuration is bivalent, and (b) from any bivalent configuration, the adversarial scheduler can always deliver messages in an order that leads to another bivalent configuration — because no process can distinguish "the decisive process is crashed" from "its message is merely delayed." So an infinite non-deciding execution always exists.

**Engineering consequences** — this is why real systems look the way they do:

- Consensus implementations (Raft, Multi-Paxos, ZAB) guarantee **safety unconditionally** and **liveness only under partial synchrony**: they use timeouts (an implicit failure detector) to elect leaders, and randomization (Raft's randomized election timeouts) to escape the adversarial schedules FLP constructs ([Raft paper](https://raft.github.io/raft.pdf); details in [file 03](03-consensus-raft-and-spanner.md)).
- A stalled consensus group under network chaos is *the algorithm working as designed* — it is refusing to trade safety for liveness. Alert on it, but don't "fix" it by weakening quorum requirements.
- Failure detection can never be perfect (§ [file 04](04-membership-failure-detection-gossip.md)); every timeout is a wager that slow ≠ dead.

## 4. The Consistency Model Hierarchy

Ordered strongest → weakest. Each step down removes coordination from the write/read path and therefore latency (PACELC), at the cost of anomalies the application must tolerate.

| Model | Guarantee | Coordination cost | Canonical system |
|---|---|---|---|
| **External consistency / strict serializability** | Transactions appear to execute serially *in real-time order*: "if one transaction completes before another starts to commit, clients can never see a state that includes the effect of the second but not the first" ([Spanner docs](https://docs.cloud.google.com/spanner/docs/true-time-external-consistency)) | Consensus per write + commit-wait on bounded clock uncertainty | Spanner |
| **Linearizability** | Single-object real-time atomicity (the C in CAP) | Quorum/consensus per operation | etcd, ZooKeeper reads via leader |
| **Sequential consistency** | Some global order consistent with each process's program order; not tied to real time | Cheaper: no real-time ordering across clients | — |
| **Causal consistency** | Only causally-related ops are ordered; concurrent ops may be observed in different orders | Metadata (vector clocks / dependency tracking), no synchronous coordination | Dynamo's version model |
| **Eventual / strong eventual** | Replicas converge if updates stop; SEC (CRDTs) additionally guarantees *deterministic* convergence | None on the write path | Dynamo, CRDT stores |

Two practitioner notes:

- **"Read-your-writes", "monotonic reads" etc. are session guarantees** — client-relative weakenings of the above, often achievable cheaply (sticky routing, session tokens) and often *all the product actually needs*. Identify the anomaly the product cannot tolerate before buying a stronger model.
- **The model is a contract per operation, not per database.** Spanner offers strong reads and cheaper bounded-staleness snapshot reads side by side; snapshot reads "return the value of the most recent version prior to that timestamp and don't need to block writes" via MVCC ([Spanner TrueTime docs](https://docs.cloud.google.com/spanner/docs/true-time-external-consistency)).

## 5. Time and Ordering

There is no global "now". Every ordering mechanism below is a different answer to "what can we know about the order of two events on different machines?"

### 5.1 Happens-before and Lamport clocks

Lamport's 1978 CACM paper defines the **happens-before** relation `→`: `a → b` if same-process program order, or `a` is a send and `b` the matching receive, or transitively. Events unrelated by `→` are *concurrent* — the fundamental fact of distributed systems is that this is a **partial** order ([Lamport timestamps](https://en.wikipedia.org/wiki/Lamport_timestamp)).

Lamport clock algorithm: each process keeps counter `L`; increment on each local event; attach `L` to every message; on receive set `L := max(L_local, L_msg) + 1`. Guarantee: `a → b ⇒ L(a) < L(b)`. **The converse does not hold** — `L(a) < L(b)` tells you nothing; Lamport clocks can order events but cannot *detect concurrency*.

Uses: total-order tiebreaking (order by `(L, process_id)`), last-write-wins timestamps, term/epoch numbers in consensus (Raft's terms are a Lamport clock over leadership eras).

### 5.2 Vector clocks — detecting concurrency

Vector clock `V` = one counter per process; increment own entry on local event; on receive, element-wise max then increment own. Now causality is captured *exactly*: `a → b ⇔ V(a) < V(b)` (element-wise ≤ with at least one <); incomparable vectors ⇔ concurrent events — a **true conflict** requiring resolution.

Dynamo's production use ([paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf); [annotated](https://mwhittaker.github.io/papers/html/decandia2007dynamo.html)):

- Every object version carries a vector clock (entries per *coordinator node*, not per client).
- On read, versions whose clocks are ordered are collapsed automatically (the descendant wins); incomparable versions are **both returned to the client** for semantic reconciliation (the shopping-cart merge), and the reconciled write carries a clock that dominates both.
- **Truncation:** clocks have a maximum size; each entry is timestamped with wall-clock time, and the oldest entry is evicted when the cap is exceeded. This can lose causality information (false conflicts after truncation) — an explicit, documented tradeoff of bounded metadata vs. perfect causality.

Sizing intuition: vector clocks are O(writers). Systems that let *any* replica coordinate writes (Dynamo) pay O(N) worst case; systems that funnel writes through a leader pay O(1) (a term number suffices) — one more reason leader-based designs are metadata-cheap.

### 5.3 TrueTime — engineering a usable global clock

Spanner's insight: you cannot eliminate clock uncertainty, but you can **bound it and expose the bound**. TrueTime's `TT.now()` returns an **interval** `[earliest, latest]` guaranteed to contain true time, with `TT.after(t)`/`TT.before(t)` predicates; timestamps drawn from it are monotonically increasing ([Spanner TrueTime docs](https://docs.cloud.google.com/spanner/docs/true-time-external-consistency); [Spanner, OSDI 2012](https://research.google/pubs/spanner-googles-globally-distributed-database-2/) — the uncertainty is kept small by GPS and atomic-clock time masters in each datacenter, per the OSDI paper).

**Commit wait**, the core protocol move (OSDI paper §4): a read-write transaction takes its commit timestamp `s` from TrueTime, then the leader **waits until `TT.after(s)` is true** — i.e., until `s` is guaranteed to be in the past everywhere — before making the commit visible. After the wait, any transaction that starts later must receive a strictly larger timestamp, which yields **external consistency** with no communication between the two transactions. The cost is latency proportional to the uncertainty width — which is why Google spends hardware (atomic clocks) to keep the interval narrow. This is PACELC made physical: consistency purchased with latency, then the latency purchased back with clock engineering.

**Lock-free reads:** because every version is stamped with a TrueTime-derived timestamp, a read-only transaction picks a timestamp and reads that MVCC snapshot consistently across the whole database without locks and without blocking writers ([docs](https://docs.cloud.google.com/spanner/docs/true-time-external-consistency)).

### 5.4 Event time vs processing time

For *data* (as opposed to protocol state), the ordering problem reappears as **event time** (when it happened) vs **processing time** (when the system saw it), with unbounded skew between them. The resolution — watermarks as a moving lower bound on in-flight event times — belongs to the stream-processing file: [05-exactly-once-and-stream-processing.md](05-exactly-once-and-stream-processing.md) ([MillWheel, VLDB 2013](https://research.google.com/pubs/archive/41378.pdf); [Dataflow model](https://www.infoq.com/articles/dataflow-apache-beam/)).

---

## Design rules this file compresses to

1. During partitions you pick stale-or-stopped; the rest of the time you pick slow-or-stale — and you make both picks *per operation*.
2. Partition frequency is an engineering budget item (Spanner's network spend), not fate.
3. Consensus liveness is conditional (FLP); design operations and alerting for "safely stalled."
4. Choose the weakest consistency model whose anomalies the product can tolerate; name the anomaly first.
5. Use Lamport clocks to order, vector clocks to detect conflicts, bounded-uncertainty physical time (TrueTime-style) only when you can pay for it.
