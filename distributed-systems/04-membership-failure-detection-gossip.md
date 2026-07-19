# Membership, Failure Detection & Gossip

> Part 4 of the realtime distributed systems series. Index: [realtime-high-scale-distributed-systems.md](realtime-high-scale-distributed-systems.md)
> **Scope:** Why failure detection is necessarily imperfect, why naive heartbeating doesn't scale, SWIM in protocol-level detail, phi-accrual detection, and the membership-vs-detection separation that production systems converge on.
> **Primary sources:** [Das, Gupta & Motivala, "SWIM", DSN 2002](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf) · [protocol walkthrough](https://www.brianstorti.com/swim/) · [Dynamo, SOSP 2007](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf).

---

## 1. The Problem Statement, Honestly

In an asynchronous network, "crashed" and "slow/partitioned" are indistinguishable — this is the operational face of FLP ([file 01 §3](01-foundations-time-and-ordering.md)). Every failure detector is therefore a bet, characterized by two properties in tension:

- **Completeness** — crashed members are eventually detected (achievable).
- **Accuracy** — live members aren't falsely accused (not perfectly achievable; only probabilistically).

A detector's false positives aren't cosmetic: a false accusation can trigger rebalancing (data movement), leadership churn, or cascading load shifts. Meanwhile "The Tail at Scale" adds the inverse concern: a node that is *slow but alive* is worse than a dead one, precisely because failure detectors won't remove it ([Dean & Barroso](https://www.barroso.org/publications/TheTailAtScale.pdf) — hence latency-induced probation, [file 07](07-tail-latency-and-ha-architecture.md)). Detection tuning is thus a three-way trade: detection time × false-positive rate × network load.

## 2. Why All-to-All Heartbeating Dies at Scale

Everyone-pings-everyone costs O(N²) messages per period: at 1,000 nodes with 1-second periods, that is on the order of **1,000,000 messages per second** fleet-wide ([walkthrough](https://www.brianstorti.com/swim/)). Centralized heartbeat collectors fix the message count but concentrate failure and load. The structural insight of SWIM: **decouple failure detection (local, constant cost) from membership dissemination (epidemic, piggybacked)** — the two jobs have different scaling laws, so give each its own mechanism ([SWIM paper](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf)).

## 3. SWIM, Protocol-Level

### 3.1 Failure detection component

Each member, once per **protocol period T**:

1. Picks a target and sends `ping`; waits for `ack` within a timeout.
2. On timeout, does **not** accuse yet — it sends `ping-req(target)` to **k** randomly chosen members, who ping the target and relay any `ack` ("outsourced heartbeats").
3. Only if direct and all k indirect probes fail does the member move against the target.

The indirect probe is the underrated move: it distinguishes "target is down" from "*my* path to the target is down," filtering out asymmetric/local network failures that dominate false positives in naive detectors ([SWIM paper](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf); [walkthrough](https://www.brianstorti.com/swim/)).

Per-node probe load is **constant** — one ping plus occasional ping-reqs per period, independent of N.

### 3.2 Suspicion sub-protocol

Failing the probes marks the target **suspected**, not dead — the suspicion is disseminated, and the suspect (which keeps receiving pings from others) can **refute** it; in the full protocol, refutations carry a per-member **incarnation number** that the accused increments, so a fresh "alive, incarnation i+1" overrides any "suspect, incarnation ≤ i" regardless of gossip arrival order ([SWIM paper](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf) — the incarnation ordering is the paper's mechanism for making alive/suspect/dead messages commute). Only after a suspicion timeout with no refutation is the member confirmed dead ([walkthrough](https://www.brianstorti.com/swim/)).

This converts the accuracy problem from "never be wrong" (impossible) to "be wrong briefly and reversibly" — the false-positive cost drops from spurious rebalance to a transient flag.

### 3.3 Dissemination component: infection-style piggybacking

Membership updates (join/suspect/confirm/alive) are not multicast; they **piggyback on the ping/ping-req/ack traffic that is flowing anyway**. Each member maintains a buffer of recent updates and attaches them to outgoing protocol messages; updates spread epidemically, reaching the full group with latency growing **logarithmically** in N, with zero dedicated dissemination traffic and no reliance on IP multicast ([SWIM paper](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf); [walkthrough](https://www.brianstorti.com/swim/)).

### 3.4 Round-robin probing — bounding detection time

Pure random target selection makes worst-case first-detection unbounded (a dead node may just not get picked). The refinement: probe members in **round-robin order over a randomly shuffled local list**, inserting joiners at random positions. Worst-case time to *first* detection becomes deterministic: at most `T × N` — while expected detection stays fast and load stays constant ([SWIM paper](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf); [walkthrough](https://www.brianstorti.com/swim/)).

### 3.5 Properties summary

| Property | SWIM | All-to-all heartbeat |
|---|---|---|
| Per-node network load | Constant in N | O(N) send + O(N) receive |
| Expected first detection | ~one protocol period (some member probes the dead node) | one period |
| Worst-case first detection | ≤ T × N (round-robin) | one period |
| Dissemination latency | O(log N), piggybacked | n/a (implicit) |
| False-positive dampening | indirect probes + suspicion + incarnation refutation | none |

SWIM descendants (HashiCorp's memberlist, underlying Serf and Consul) run cluster membership for large production fleets; the modern Rust take is documented in [Quickwit's Chitchat writeup](https://quickwit.io/blog/chitchat).

## 4. Phi-Accrual Detection: Suspicion as a Continuous Value

Fixed timeouts encode one guess about network behavior. The **φ accrual failure detector** instead maintains the statistical distribution of recent heartbeat inter-arrival times and outputs a continuous suspicion level φ for "how likely is it that this silence means death, given observed history"; each *consumer* of the detector picks its own φ threshold — a cheap action (skip routing to the node) can trigger at low φ, an expensive one (evict, rebalance) at high φ ([overview in the SWIM/membership literature](https://en.wikipedia.org/wiki/SWIM_Protocol)). Cassandra's gossip subsystem ships exactly this design (`gms/` — see `../datastores/cassandra-architecture.md`). The design idea generalizes: **report evidence, let policies own their thresholds** — the same shape as TrueTime returning an interval instead of a timestamp ([file 01 §5.3](01-foundations-time-and-ordering.md)).

## 5. Separating Detection from Membership (the Dynamo lesson)

Dynamo splits the two concerns explicitly ([paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf); [annotated](https://mwhittaker.github.io/papers/html/decandia2007dynamo.html)):

- **Failure detection** is local and transient — a node that gets no response routes around the peer and retries later; no global agreement needed or wanted.
- **Membership** (ring ownership — which drives *data placement*) changes only by **explicit operator action**, gossiped through the ring, with **seed nodes** as guaranteed gossip partners to prevent logically split rings during concurrent joins.

Rationale: a transient blip must never trigger mass data movement. The general rule — worth applying to any system you review: **the more expensive the reaction, the more durable and deliberate its trigger must be.** Route-around: automatic, milliseconds. Traffic drain: automatic with hysteresis. Data rebalance / membership change: consensus-recorded or operator-confirmed.

Anti-pattern from the SRE cascading-failures chapter that encodes the same lesson: health-check-driven systems that *remove* "unhealthy" (actually overloaded) instances shrink capacity exactly when load spikes — detection wired directly to an expensive reaction with a shared-fate signal ([SRE Ch. 22](https://sre.google/sre-book/addressing-cascading-failures/); more in [file 06 §6](06-overload-control-and-resilience.md)).
