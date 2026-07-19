# Exactly-Once Semantics & Realtime Stream Processing Internals

> Part 5 of the realtime distributed systems series. Index: [realtime-high-scale-distributed-systems.md](realtime-high-scale-distributed-systems.md)
> **Scope:** The delivery-guarantee ladder made precise; Kafka's idempotent producer and transaction protocol at the coordinator/log level; the event-time model (watermarks, triggers); Flink's Chandy-Lamport-derived checkpointing including alignment and unaligned checkpoints; end-to-end exactly-once composition; backpressure.
> Complements the engine deep-dives in `../analytics/flink-architecture.md` and `../analytics/flink-internals.md`.

---

## 1. The Ladder, Made Precise

Per the [Kafka delivery-semantics documentation](https://docs.confluent.io/kafka/design/delivery-semantics.html):

- **At-most-once** — send without retry; failures lose data.
- **At-least-once** — retry until acknowledged; failures duplicate data.
- **Exactly-once** — every record affects final state once.

The load-bearing fact: **exactly-once *delivery* over a lossy network is impossible; exactly-once *processing effect* is engineered as at-least-once delivery + deduplication/idempotence.** Every system in this file — Kafka, MillWheel, Flink, and Raft's client sessions ([file 03 §2.6](03-consensus-raft-and-spanner.md)) — implements that one identity with different bookkeeping.

## 2. Kafka: Idempotent Producer and Transactions

### 2.1 Idempotent producer (dedup by sequence number)

Each producer receives a **Producer ID (PID)**; every batch carries a **per-partition sequence number**. On retry after a lost acknowledgment, the broker recognizes the duplicate sequence and acknowledges *without re-appending* — the log gets one copy no matter how many sends ([Confluent, "Exactly-once Semantics Are Possible"](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/); [KIP-98](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging)). Scope: one producer session, one partition. Crossing sessions and partitions needs transactions.

### 2.2 The transaction protocol ([Confluent, "Transactions in Apache Kafka"](https://www.confluent.io/blog/transactions-apache-kafka/))

**Actors.** Every broker runs a **transaction coordinator** module. Transaction *state* (not data) lives in an internal replicated topic, **`__transaction_state`**; a producer's `transactional.id` hashes to one partition of it, which pins exactly one coordinator as its owner.

**Zombie fencing.** `initTransactions()` registers the `transactional.id`, completes/aborts any transaction left open by a previous incarnation, and **bumps the producer epoch**. Any surviving old instance ("zombie") with the same id but a stale epoch is thereafter rejected — the same fencing-token shape as Raft terms ([file 03 §4](03-consensus-raft-and-spanner.md)). The blog's stated precondition: fencing is sound only if "the input topics and partitions in the read-process-write cycle is always the same for a given transactional.id" — i.e., the transactional id must be bound to the *input partition assignment*, not to the process.

**Flow for read-process-write:**

1. `initTransactions()` — register, recover, bump epoch.
2. `send()` — first write to each new partition registers that partition with the coordinator; transaction start is implicit.
3. `sendOffsetsToTransaction()` — the consumed offsets are written *inside* the transaction (offsets are just messages in `__consumer_offsets`), making "processed input" and "produced output" atomic together. This is the step that turns the pattern into exactly-once *stream processing* rather than just atomic multi-partition produce.
4. `commitTransaction()` — two phases:
   - **Phase 1:** coordinator writes `prepare_commit` to `__transaction_state`. Once that record is replicated, **the commit is decided** — crash recovery rolls forward.
   - **Phase 2:** coordinator writes **transaction markers** (commit/abort control records) into every participating data partition, then marks the transaction complete.

This is 2PC with the classic blocking problem removed the same way Spanner removes it — the coordinator's decision lives in a replicated log, so coordinator failover resumes, not blocks ([file 03 §3](03-consensus-raft-and-spanner.md)).

**Consumer side.** `read_committed` consumers use the **Last Stable Offset (LSO)**: the broker withholds offsets beyond the first still-open transaction, so consumers see only non-transactional and *committed* transactional records, with aborted data filtered via the markers — no client-side buffering of uncommitted data ([same source](https://www.confluent.io/blog/transactions-apache-kafka/)).

**Cost.** Overhead is per-transaction, not per-message: partition-registration RPCs, one marker per participating partition, and transaction-log writes. Confluent's figure: a producer writing 1 KB records at max throughput committing every 100 ms loses ~**3% throughput**; transactional *consumers* see **no degradation** (zero-copy preserved; LSO logic is broker-side) ([same source](https://www.confluent.io/blog/transactions-apache-kafka/)). Design lever: commit interval trades end-to-end latency against amortization.

**Boundary.** Kafka transactions cover Kafka-stored state only — no external systems ([Confluent](https://www.confluent.io/blog/transactions-apache-kafka/)). Crossing that boundary is §5.

## 3. The Event-Time Model: Watermarks, Windows, Triggers

Unbounded data cannot be globally ordered on arrival — the Dataflow model's founding observation ([model insights](https://www.hemantkgupta.com/p/insights-from-paper-google-the-dataflow); [InfoQ overview](https://www.infoq.com/articles/dataflow-apache-beam/)). Its decomposition of "what does a correct realtime answer even mean":

- **What** is computed — the transformation/aggregation.
- **Where** in event time — **windowing**: fixed, sliding, or session windows over *event* time, not arrival time.
- **When** results materialize — **triggers** against the **watermark**, the system's moving lower bound on event times still in flight. MillWheel's production formulation: the **low watermark** "provides a tight bound on the timestamp of all events still in the system," substituting for in-order delivery — an aggregator emits a window when the watermark passes its end ([MillWheel, VLDB 2013](https://research.google.com/pubs/archive/41378.pdf)).
- **How** corrections happen — allowed lateness and refinement policy (discard/accumulate/retract) for data arriving behind the watermark.

Staff-level points: the watermark is a **heuristic estimate** wherever sources can't promise completeness — too fast drops late data, too slow adds latency; the trigger/lateness machinery exists precisely so this latency-vs-completeness tradeoff is an explicit application policy rather than an accident of implementation ([Confluent on watermark semantics](https://www.confluent.io/blog/watermarks-tables-event-time-dataflow-model/); formal comparative semantics of Flink vs Dataflow watermarks: [Begoli et al., VLDB 2021](http://www.vldb.org/pvldb/vol14/p3135-begoli.pdf)). Operationally, **watermark lag is the truthful "realtime-ness" metric** for a pipeline — throughput can look healthy while watermarks stall.

MillWheel pairs the watermark with per-record **exactly-once**: idempotent record processing plus checkpointing ("strong productions" — state and productions checkpointed before effects become visible) with upstream backup for replay ([MillWheel paper](https://research.google.com/pubs/archive/41378.pdf)).

## 4. Flink Checkpointing: Chandy-Lamport in Production

The theoretical base — [Chandy & Lamport 1985](https://en.wikipedia.org/wiki/Chandy%E2%80%93Lamport_algorithm): a consistent global snapshot of a running asynchronous system can be captured **without pausing it**, by flowing **marker messages** through the channels; the markers partition every channel's traffic into pre- and post-snapshot ([lecture treatment](https://www.cs.princeton.edu/courses/archive/fall16/cos418/docs/P8-chandy-lamport.pdf)).

Flink's adaptation ([stateful stream processing docs](https://nightlies.apache.org/flink/flink-docs-master/docs/concepts/stateful-stream-processing/); [checkpointing docs](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/fault-tolerance/checkpointing/)):

- **Barriers** carrying a checkpoint id are injected at sources at concrete positions (e.g., Kafka offsets, reported to the checkpoint coordinator) and "flow with the records as part of the data stream," never overtaking them — the stream is partitioned into "belongs to snapshot n" / "n+1".
- **Alignment:** a multi-input operator receiving barrier *n* on one input **buffers** that input's subsequent records until barrier *n* arrives on all inputs; then it snapshots its state, emits the barrier downstream, drains the buffers, resumes. Alignment is what keeps records of different snapshots unmixed — the exactly-once guarantee for state.
- **Asynchronous snapshots:** the operator snapshots "at the point in time when [it has] received all snapshot barriers from [its] input streams, and before emitting the barriers to [its] output streams," but the state upload to the backend proceeds asynchronously so processing isn't blocked.
- **Recovery:** restore all operators to checkpoint *n*'s state, rewind sources to *n*'s recorded positions, replay. Exactly-once for *internal state* falls out: every record's effect is either in the restored state or will be replayed, never both.
- **At-least-once mode:** skip alignment — operators keep processing during the barrier wait, so post-barrier records leak into snapshot *n*'s state and are re-processed after recovery (duplicates). Lower latency; acceptable for idempotent/parallel-independent operations.
- **Unaligned checkpoints:** for backpressured jobs where barriers crawl behind queued data, barriers may "overtake all in-flight data as long as the in-flight data becomes part of the operator state" — the overtaken buffers are persisted *into the checkpoint*. Checkpoint latency decouples from backpressure, at the price of larger checkpoints and I/O.
- **Savepoints** = manually triggered, non-expiring checkpoints — the operational primitive for redeploys/rescales/upgrades with state carried over.

## 5. End-to-End Exactly-Once: Composing the Pieces

Flink's checkpoint protects *internal* state; the sink is outside it. The composition — Flink's **two-phase-commit sink** coordinated with the checkpoint: pre-commit external writes during the checkpoint (e.g., an open Kafka transaction), commit them only when the checkpoint completes, so the external system's committed data always corresponds to a completed checkpoint ([Flink blog: end-to-end exactly-once with Kafka](https://flink.apache.org/2018/02/28/an-overview-of-end-to-end-exactly-once-processing-in-apache-flink-with-apache-kafka-too/)). The full chain:

```
replayable source (Kafka offsets in checkpoint)
  → checkpointed operator state (Chandy-Lamport barriers)
  → transactional/idempotent sink (Kafka transactions, or idempotent upsert)
```

Break any link — a non-replayable source, an eagerly-committing sink — and the pipeline silently degrades to at-least-once. When auditing a claimed exactly-once pipeline, verify each link separately.

## 6. Backpressure

A pipeline runs at its slowest stage; the design question is only *where the excess accumulates*. Streaming engines make accumulation bounded and propagating — Flink's bounded network buffers push backpressure upstream hop-by-hop until it reaches the source, which slows its consumption (and a pull-based source like a Kafka consumer simply lags, with the log absorbing the difference — [file 06](06-overload-control-and-resilience.md) covers the request/response analog, load shedding). Unbounded queues are the anti-pattern: the backlog itself becomes the overload that outlives the trigger — AWS's queue-backlog analysis ([Builders' Library analysis](https://lumigo.io/blog/amazon-builders-library-in-focus-4-avoiding-insurmountable-queue-backlogs/)) and the metastable-failure framework ([file 06 §7](06-overload-control-and-resilience.md)) both land here. Note the interaction documented in §4: backpressure also *slows barriers*, degrading checkpoint freshness — the exact coupling unaligned checkpoints exist to cut.
