# Consensus in Production: Raft Mechanics and Spanner's Transaction Stack

> Part 3 of the realtime distributed systems series. Index: [realtime-high-scale-distributed-systems.md](realtime-high-scale-distributed-systems.md)
> **Scope:** Raft at the level you'd need to review an implementation — election, log replication, the commit rule and its famous hazard, membership change, snapshots, client semantics — then Spanner's composition of Paxos + 2PC + TrueTime, and the architectural rule about where consensus belongs.
> **Primary sources:** [Ongaro & Ousterhout, "In Search of an Understandable Consensus Algorithm" (extended)](https://raft.github.io/raft.pdf) ([USENIX ATC 2014, Best Paper](https://www.usenix.org/conference/atc14/technical-sessions/presentation/ongaro); [raft.github.io](https://raft.github.io/); [close reading](https://www.ncameron.org/blog/raft-paper/)) · [Corbett et al., "Spanner", OSDI 2012](https://research.google/pubs/spanner-googles-globally-distributed-database-2/).

---

## 1. What Consensus Buys, Restated

Consensus maintains a **replicated log**: if any state machine applies command `c` as entry *n*, no other state machine ever applies a different entry *n* ([raft.github.io](https://raft.github.io/)). Everything else — leader election, terms, quorums — is machinery to preserve that invariant while still making progress when minorities fail. Per FLP ([file 01 §3](01-foundations-time-and-ordering.md)), safety is unconditional; liveness holds only when timing cooperates.

A group of 2f+1 members tolerates f crash failures. Raft and Multi-Paxos are equivalent in result and efficiency; Raft's contribution is *decomposition* — leader election / log replication / safety as separable subproblems — which measurably improves human understanding (the paper's user study) and therefore implementation correctness ([paper](https://raft.github.io/raft.pdf)).

## 2. Raft, Mechanism by Mechanism

### 2.1 Terms and states

Time divides into **terms**, each beginning with an election; terms are a Lamport clock over leadership eras. Every RPC carries the sender's term: a server receiving a *higher* term immediately adopts it (and steps down to follower if it was leader/candidate); messages from *lower* terms are rejected. This single rule retires stale leaders without coordination ([close reading](https://www.ncameron.org/blog/raft-paper/)). Servers are always in one of three states: **follower** (passive), **candidate** (electing), **leader** (sole handler of client writes).

### 2.2 Leader election

- A follower that hears no heartbeat (empty `AppendEntries`) within its **election timeout** becomes candidate: increments term, votes for itself, sends `RequestVote` to all.
- **One vote per term**, first-come-first-served, persisted to disk (a server that forgets its vote can double-vote after restart — this is why `currentTerm` and `votedFor` are on the durable state list).
- Majority of votes → leader; a higher-term message → step down; timeout with no winner → new term, retry.
- **Randomized election timeouts** (the paper's example range: 150–300 ms) desynchronize candidates so split votes are rare and resolve quickly — this randomness is precisely the FLP escape hatch ([paper](https://raft.github.io/raft.pdf); [close reading](https://www.ncameron.org/blog/raft-paper/)).

**Election restriction** (the safety linchpin): a voter refuses any candidate whose log is less *up-to-date* than its own — compared by (last entry's term, then log length). Since a committed entry is on a majority, and a candidate needs a majority of votes, the winner's log must contain every committed entry — this is how Raft gets **Leader Completeness** without any post-election log transfer to the leader ([paper](https://raft.github.io/raft.pdf)).

### 2.3 Log replication

The leader appends the client command, then sends `AppendEntries(prevLogIndex, prevLogTerm, entries[], leaderCommit)` to followers:

- **Consistency check:** a follower rejects the append unless its log contains an entry at `prevLogIndex` with `prevLogTerm`. On rejection the leader decrements that follower's `nextIndex` and retries, walking back to the divergence point, then overwrites the follower's conflicting suffix. Induction on this check yields the **Log Matching property**: if two logs share an entry (same index and term), they are identical up through it ([paper](https://raft.github.io/raft.pdf)).
- An entry is **committed** once the leader has it replicated on a majority; the leader advances `commitIndex`, piggybacks it on subsequent RPCs, and everyone applies committed entries in order. Committed entries are never removed ([close reading](https://www.ncameron.org/blog/raft-paper/)).

### 2.4 The commit rule and the Figure-8 hazard

The subtlest rule in Raft: **a leader may only commit entries of its own current term by counting replicas.** Entries from previous terms are never directly committed by replica-counting — they become committed *implicitly* when a current-term entry that follows them commits ([paper](https://raft.github.io/raft.pdf) §5.4.2, the "Figure 8" scenario; [close reading](https://www.ncameron.org/blog/raft-paper/)).

Why: an old-term entry can be present on a majority and *still* be overwritten — a leader from a newer term that never saw it can win election (its log is "more up-to-date" by term) and replicate a different entry over those indexes. Counting replicas of an old-term entry therefore proves nothing about its durability. Practical corollary: a freshly elected leader commits a **no-op entry of its own term** immediately, which flushes commitment of all its inherited entries.

### 2.5 Membership change

Naively switching from configuration C_old to C_new is unsafe: during the transition, disjoint majorities of C_old and C_new could elect two leaders for the same term. Raft's original answer is **joint consensus**: an intermediate configuration C_old,new in which every election and every commit requires *majorities of both* configurations; once the C_old,new entry commits, the leader replicates C_new and the old configuration becomes irrelevant ([paper](https://raft.github.io/raft.pdf); [close reading](https://www.ncameron.org/blog/raft-paper/)). Ongaro's dissertation later gives the simpler **single-server change** algorithm (add/remove one member at a time — any two adjacent configurations share a majority automatically) plus a TLA+ specification of Raft ([raft.github.io](https://raft.github.io/)).

### 2.6 Log compaction and client semantics

- **Snapshots:** the state machine periodically snapshots at an applied index; the log prefix is discarded; a leader whose follower has fallen behind the snapshot horizon ships the snapshot (`InstallSnapshot`) instead of entries ([paper](https://raft.github.io/raft.pdf)).
- **Exactly-once client writes:** committing a command twice (client retried after a lost response) would double-apply. The paper's remedy: clients hold **session/serial numbers**; the state machine tracks the latest serial per client and answers duplicates from cache — exactly-once *effect* via at-least-once delivery + dedup, the same identity Kafka uses ([file 05](05-exactly-once-and-stream-processing.md); [close reading](https://www.ncameron.org/blog/raft-paper/)).
- **Linearizable reads:** a "leader" answering reads may be deposed and not know it. The paper requires the leader to confirm incumbency before answering — exchange heartbeats with a majority for the read (or maintain a leader lease bounded by clock assumptions) ([paper](https://raft.github.io/raft.pdf); [close reading](https://www.ncameron.org/blog/raft-paper/)). Reviewing an implementation? This is the first place to look for a linearizability bug.

### 2.7 Raft's durable-state checklist (implementation review aid)

Must survive crash before responding to any RPC: `currentTerm`, `votedFor`, `log[]`. Everything else (`commitIndex`, `lastApplied`, leader's `nextIndex[]`/`matchIndex[]`) is reconstructible volatile state. Fsync placement relative to RPC replies is where real-world implementations quietly lose safety.

## 3. Spanner: Consensus *in* the Data Path, Made Affordable

Spanner is the counterexample to "keep consensus out of the data path" — it puts a Paxos group under every shard and still serves global OLTP ([OSDI 2012](https://research.google/pubs/spanner-googles-globally-distributed-database-2/)). The stack, bottom-up:

1. **Paxos group per tablet** — data is range-partitioned into tablets/directories; each replicates across datacenters via a Paxos state machine with **long-lived leaders and pipelining** (amortizing election and filling the WAN pipe). Directories move between groups for load balance and locality.
2. **Transactions within one group** — go through that group's Paxos leader; the common case pays one consensus round.
3. **Cross-group transactions: 2PC layered over Paxos.** Classic 2PC blocks when the coordinator dies; Spanner removes the blocking by making *every participant and the coordinator itself a Paxos group* — the commit/abort decision is replicated, so coordinator "death" is just a leader failover ([paper](https://research.google/pubs/spanner-googles-globally-distributed-database-2/)). The textbook 2PC objection is an objection to unreplicated coordinators, not to 2PC.
4. **TrueTime timestamps + commit wait** give externally consistent commit order globally, and **lock-free snapshot reads** at any timestamp — mechanics in [file 01 §5.3](01-foundations-time-and-ordering.md) ([TrueTime docs](https://docs.cloud.google.com/spanner/docs/true-time-external-consistency)).

CAP position: technically **CP** — a partitioned minority blocks — with availability engineered up via the network rather than the protocol ([Brewer 2017](https://research.google.com/pubs/archive/45855.pdf)).

## 4. Where Consensus Belongs — the Architectural Rule

Consensus serializes through a leader and a quorum: per-group throughput is bounded and every write pays a quorum round-trip. High-traffic architectures therefore place it deliberately:

- **Control plane only (most systems):** shard maps, membership, schema, leases, leader election — small state, low rate, correctness-critical. Data path uses cheaper replication ([file 02](02-partitioning-and-replication.md)). Examples: Kafka's KRaft metadata quorum; Cassandra using Paxos only for opt-in lightweight transactions and (5.1+) cluster metadata (`../datastores/cassandra-architecture.md`).
- **Data plane, sharded (Spanner-style):** consensus per shard scales horizontally because groups are independent; the cost shows up as per-key write latency, bought back with pipelining, long-lived leaders, and TrueTime reads. YugabyteDB is this shape (`../datastores/yugabytedb-architecture.md`).

**Leases and fencing:** a common hybrid is consensus-elected leadership with time-bounded leases for cheap local reads/decisions. Any lease scheme is only as sound as its clock-skew bound — Raft's incumbency check (§2.6) is the skew-free alternative; TrueTime is the bounded-skew alternative made rigorous. When reviewing a design that says "the lock holder does X," ask what fences a *former* holder that doesn't yet know it — the answer must be an epoch/term number validated by the resource, which is exactly what Raft terms and Kafka producer epochs ([file 05 §2](05-exactly-once-and-stream-processing.md)) provide.
