# Apache Flink — Internal Implementation Details

> Based on analysis of the Flink source tree at `C:\workspace\opensource\flink` (version **2.4-SNAPSHOT**).
> Big-picture / control-plane companion: [flink-architecture.md](flink-architecture.md)
> Paths below are relative to the repo root; most code is under
> `flink-runtime/src/main/java/org/apache/flink/`.

---

## 1. Task Execution Layer

### 1.1 `Task` — the container (`runtime/taskmanager/Task.java`)

One `Task` = one execution attempt of one parallel subtask, run by **one dedicated thread**. It is
deliberately ignorant of the rest of the job: it only knows its own code (a `TaskInvokable` class
name, instantiated via the user-code classloader), its configuration, and the IDs of the
intermediate result partitions it consumes/produces. The Task:

- drives the execution-state machine (`CREATED → DEPLOYING → INITIALIZING → RUNNING →
  FINISHED/CANCELED/FAILED`) using lock-free atomic updates and reports transitions to the
  JobMaster;
- wires the invokable to the network stack (`ResultPartitionWriter`s / `InputGate`s from the
  `ShuffleEnvironment`), the broadcast variable manager, the state backends, and the checkpoint
  responder;
- supervises cancellation with a watchdog (`TaskCancelerWatchDog`) that can kill the whole
  TaskManager if a task thread refuses to die (fail-fast rather than leak).

### 1.2 `StreamTask` — the streaming invokable (`streaming/runtime/tasks/StreamTask.java`)

The base class of every streaming task. Specializations by head-operator shape:
`OneInputStreamTask`, `TwoInputStreamTask`, `MultipleInputStreamTask`,
`SourceOperatorStreamTask` (FLIP-27 sources), plus iteration head/tail.

Lifecycle inside `Task`'s thread: `restore()` (build the `OperatorChain`, initialize operator
state — possibly reading recovered in-flight channel data) → `invoke()` (run the mailbox loop) →
`finish()`/`cleanUp()`. Recent work (FLINK-39522, visible in the latest commits) restructured
channel-state recovery into an async future chain and defers task finish until recovery completes.

### 1.3 The mailbox threading model (`streaming/runtime/tasks/mailbox/`)

The single most important concurrency idea in the data plane. From `MailboxProcessor`'s Javadoc:
the mailbox loop "continuously executes the provided `MailboxDefaultAction` in a loop; on each
iteration it also checks for pending actions in the mailbox … ensuring single-threaded execution
between the default action (record processing) and mailbox actions (checkpoint trigger, timer
firing, …)."

- **Default action** = process one unit of input (pull records from the input processor, push them
  through the operator chain).
- **Mails** = control actions enqueued by other threads (checkpoint trigger RPC, processing-time
  timer firings from `SystemProcessingTimeService`, async operator completions, watermark
  emission…). They run *between* records on the same thread — so operator code never needs
  locking (the pre-1.10 "checkpoint lock" is gone).
- Hot-path optimization: the loop only re-checks control flags when `hasMail` is true, keeping
  the record path branch-cheap.
- When input is temporarily unavailable (e.g. no network buffers), the default action is
  **suspended** and the thread parks on availability futures instead of spinning.
- `MailboxExecutor` exposes this to operators (e.g. `AsyncWaitOperator` and the async state
  runtime submit result-handling mails).

### 1.4 Operator chain (`streaming/runtime/tasks/OperatorChain.java`)

Chained operators execute synchronously in the task thread: the head operator's output is a
`ChainingOutput` that directly calls `processElement` of the next operator
(`CopyingChainingOutput` if object reuse is disabled — it deep-copies via the serializer to
protect against mutation). Chain-external edges end in `RecordWriterOutput`s which serialize into
network buffers. The chain also fans out via `BroadcastingOutputCollector` when an operator has
multiple downstream chained consumers. `RegularOperatorChain` vs `FinishedOperatorChain` handles
the "restored as already finished" case (a task whose operators all finished before the restoring
checkpoint just re-emits MAX_WATERMARK/EndOfData without running user code).

Operators themselves implement `StreamOperator` (`streaming/api/operators/`), usually via
`AbstractStreamOperator` / `AbstractUdfStreamOperator`, and get: keyed-state access scoped by the
current record's key, timer services, output collectors, and latency-marker/watermark plumbing.

---

## 2. Network Stack (`runtime/io/network/`)

Flink's data plane is its own credit-based streaming shuffle on top of Netty, behind the pluggable
`ShuffleEnvironment` SPI (`runtime/shuffle/`, default `NettyShuffleEnvironment`).

### 2.1 Data structures

```
Producer task                                      Consumer task
─────────────                                      ─────────────
RecordWriter                                       StreamInputProcessor
  │ serialize record → BufferBuilder                 ▲ deserialize (spanning records supported)
  ▼                                                  │
ResultPartition                                    InputGate (SingleInputGate / UnionInputGate)
  ├─ ResultSubpartition (one per consumer)  ══► InputChannel (Local / Remote)
  │    queue of Buffer/BufferConsumer          LocalInputChannel: reads producer's subpartition
  │                                            RemoteInputChannel: Netty client + credits
```

- **`MemorySegment`** (`flink-core/…/core/memory/MemorySegment.java`) — Flink's page abstraction
  (default 32 KB): one final class (deliberately no inheritance, so calls devirtualize) wrapping
  either a heap `byte[]` or off-heap memory, accessed with `Unsafe` including explicit
  endian-aware multi-byte ops, binary compare/swap/copy helpers, and collapsed bounds+liveness
  checks.
- **Buffer pools** (`io/network/buffer/`): a global `NetworkBufferPool` (off-heap, sized by the
  network memory fraction) hands segments to per-partition/per-gate `LocalBufferPool`s.
  `BufferBuilder`/`BufferConsumer` form a single-producer single-consumer view over a segment so a
  partially-written buffer can already be consumed (low latency). `BufferCompressor/Decompressor`
  optionally compress (LZ4 etc.) per buffer.
- **Partition types** (`ResultPartitionType`): `PIPELINED(_BOUNDED)` for streaming;
  `BLOCKING` for batch (written fully, then consumed — `BoundedBlockingResultPartition`,
  sort-merge shuffle via `SortMergeResultPartition`); **hybrid/tiered shuffle**
  (`partition/hybrid/tiered/`) for batch — spills between memory/disk/remote tiers and lets
  consumers start while producers still run.

### 2.2 Credit-based flow control (`io/network/netty/`)

One TCP connection is multiplexed between all channels of a TaskManager pair, so per-channel flow
control cannot rely on TCP backpressure (head-of-line blocking). Instead:

- Each `RemoteInputChannel` has **exclusive buffers** (default 2) and a shared pool of **floating
  buffers** per gate. It announces its available buffers to the producer as **credits**.
- The producer (`CreditBasedSequenceNumberingViewReader` + `PartitionRequestQueue`) sends one
  buffer per credit, decrementing; it also piggybacks its **backlog** (queued buffers) so the
  consumer can proactively request floating buffers to match.
- Events like checkpoint barriers can be sent without credit, so control flow is never blocked by
  data backpressure.

**Backpressure** is therefore explicit and non-blocking: no credits → producer's subpartition
queues grow → its `LocalBufferPool` exhausts → `RecordWriter`'s availability future is
incomplete → the task's mailbox loop parks. The Web UI backpressure monitoring simply samples
these "hard/soft backpressured" flags (`throughput/`, task metrics), no stack sampling needed.

**Buffer debloating** (`TaskManagerOptions.BUFFER_DEBLOAT_*`): dynamically shrinks the in-flight
buffer memory per gate to hold ≈ a configured time's worth of data at the measured throughput —
this bounds barrier travel time (and thus checkpoint duration) under backpressure without manual
buffer tuning.

### 2.3 Record serialization path

`RecordWriter` (`io/network/api/writer/`) serializes a `StreamRecord` via
`SerializationDelegate` → `TypeSerializer` into the current `BufferBuilder`; records can **span**
buffers. The channel is picked by the `StreamPartitioner`
(`streaming/runtime/partitioner/`): `ForwardPartitioner`, `RebalancePartitioner` (round-robin),
`RescalePartitioner`, `KeyGroupStreamPartitioner` (hash by key → key group → channel),
`BroadcastPartitioner`, … On the consumer side the deserializer reassembles spanning records
(spilling oversized ones), and `StatusWatermarkValve` merges per-channel watermarks/idleness.

---

## 3. Memory Management

TaskManager memory (`runtime/memory/`, config in `TaskManagerOptions`) is budgeted up front into:
framework/task heap, framework/task off-heap, **network memory** (the `NetworkBufferPool`),
**managed memory**, and JVM metaspace/overhead. Managed memory is off-heap, allocated as
`MemorySegment`s by the `MemoryManager` per slot, and is *reserved* (not JVM-allocated) for
consumers like RocksDB/ForSt block caches + write buffers, batch sort/hash/join operators
(`runtime/operators/` still contains the classic sort/hash implementations with normalized-key
sorting on binary data), and Python UDF workers. Because everything is budgeted, a TaskManager's
actual memory use is predictable — the design principle is "work on binary data in `MemorySegment`s,
spill when it doesn't fit, never OOM."

---

## 4. Time, Watermarks, Timers

- **Watermarks** are special `StreamElement`s flowing with the data
  (`streaming/runtime/watermarkstatus/`, `api/common/eventtime/`). Sources (FLIP-27
  `WatermarkStrategy`/`SourceOperator`) generate them; multi-input operators/gates take the **min**
  across inputs, with `WatermarkStatus` (idle/active) to keep idle channels from stalling
  progress; watermark **alignment** can throttle sources that run too far ahead.
- **Timers**: `InternalTimeServiceManagerImpl` (`streaming/api/operators/`) keys timers by
  (key, namespace) in `InternalTimerServiceImpl`; queues are `KeyGroupedInternalPriorityQueue`s —
  heap-based or RocksDB-backed — partitioned by key group so they snapshot/rescale with keyed
  state. Event-time timers fire when the operator's watermark advances; processing-time timers are
  scheduled on `SystemProcessingTimeService` (a `ScheduledThreadPoolExecutor`) whose callbacks are
  enqueued as mails into the mailbox — so all timer code still runs in the task thread.

---

## 5. State Backends (`runtime/state/`, `flink-state-backends/`)

The task-facing abstraction is `StateBackend` → creates a `CheckpointableKeyedStateBackend` (keyed
state: `ValueState`, `ListState`, `MapState`, …, scoped to the *current key* set by the runtime
before each `processElement`) and an `OperatorStateBackend` (operator/broadcast state, always
heap). Where checkpoints go is a separate concern: `CheckpointStorage` (JM-memory vs filesystem).

**Key groups** are the unit of state rescaling: the key space is hashed into
`maxParallelism` groups; each subtask owns a contiguous `KeyGroupRange`, and rescaling reassigns
whole groups — snapshots are indexed by key group so restore reads only the owned ranges.

| Backend | Where state lives | Snapshot |
|---|---|---|
| `HashMapStateBackend` (`runtime/state/heap/`) | JVM heap, nested `StateTable`s with **copy-on-write** structures | async full snapshot; supports incremental via changelog wrapper |
| `EmbeddedRocksDBStateBackend` (`flink-statebackend-rocksdb`) | embedded RocksDB (frocksdb), one column family per state, keys = serialized (keygroup‑prefix, key, namespace) | **incremental**: uploads only new SST files; leverages RocksDB snapshots/checkpoint API |
| `ForStStateBackend` (`flink-statebackend-forst`) | Flink 2.x **disaggregated** RocksDB-derived store that can keep SSTs directly on DFS; designed for the **async state API** (`runtime/asyncprocessing/`) which batches/parallelizes state reads out of the task thread while preserving per-key ordering | incremental, checkpoint can be a near-metadata-only operation since files already live on DFS |
| Changelog (`flink-statebackend-changelog` + `flink-dstl`) | wraps another backend; every mutation is also appended to a durable short-term log (DSTL) | checkpoint = tiny log offset → sub-second, decoupled from materialization which runs in background |

TTL (`runtime/state/ttl/`) wraps state accessors with timestamped values, filtering on read and
cleaning up via compaction filters (RocksDB) or incremental iteration (heap).

---

## 6. Checkpointing — the full path

### 6.1 Coordinator side (`runtime/checkpoint/CheckpointCoordinator.java`)

Per Javadoc: it "triggers the checkpoint by sending messages to the relevant tasks and collects
the checkpoint acknowledgements" plus the reported state handles. Flow:

1. Timer fires (`CheckpointRequestDecider` enforces max-concurrent / min-pause).
2. `CheckpointPlanCalculator` picks tasks to trigger (sources, or — for jobs with finished tasks —
   the running frontier; finished subtasks are recorded in the checkpoint).
3. `OperatorCoordinator`s (e.g. source enumerators, FLIP-27) snapshot **first**, then RPC
   `triggerCheckpoint` goes to trigger tasks.
4. Tasks ack with their state handles → `PendingCheckpoint` completes →
   `CompletedCheckpoint` persisted to the `CompletedCheckpointStore` (HA), checkpoint ID from
   `CheckpointIDCounter`, old checkpoints subsumed (`CheckpointsCleaner` deletes asynchronously).
5. `notifyCheckpointComplete` RPC to all tasks (this is what commits two-phase-commit sinks).

### 6.2 Task side

- **Barrier injection**: source tasks receive the trigger as a mail; non-source tasks react to
  `CheckpointBarrier`s arriving in-band in their input channels, handled by
  `SingleCheckpointBarrierHandler` (`streaming/runtime/io/checkpointing/`) — an explicit state
  machine (`BarrierHandlerState` implementations) covering aligned, unaligned, and
  aligned-with-timeout modes.
- **Aligned (exactly-once)**: on the first barrier, input channels that already delivered theirs
  are **blocked** (data buffered upstream, not read) until barriers arrive on all channels; then
  the task snapshots and forwards the barrier. Under backpressure alignment can take long — hence:
- **Unaligned checkpoints**: the barrier **overtakes** in-flight buffers: it jumps the queues, the
  task snapshots immediately, and the bypassed in-flight buffers themselves become part of the
  snapshot, captured by the `ChannelStateWriter` (`runtime/checkpoint/channel/`). On restore,
  channel state is read back (`SequentialChannelStateReader`) and re-injected into
  gates/partitions before processing resumes. `execution.checkpointing.aligned-checkpoint-timeout`
  starts aligned and switches to unaligned if alignment exceeds the timeout.
- **Snapshot execution**: `SubtaskCheckpointCoordinatorImpl` runs the synchronous part in the
  mailbox (operators snapshot state — for RocksDB essentially flush + hard-link SSTs; for heap a
  COW snapshot), then hands an `AsyncCheckpointRunnable` to a background executor to upload state
  and ack the JobMaster. The record-processing pause is only the sync part (milliseconds).
- **File merging** (`runtime/checkpoint/filemerging/`): small state files of multiple
  checkpoints/subtasks are merged into shared physical files to stop small-file explosion on DFS.

### 6.3 Savepoints vs checkpoints, recovery modes

Savepoints (`SavepointType`) are user-triggered, self-contained, canonical-format (portable
across backends), never auto-subsumed. Checkpoints are engine-owned and may be incremental.
Restore honors `RecoveryClaimMode` (CLAIM / NO_CLAIM) governing who owns and may delete the
restored files. `CheckpointType.FULL_CHECKPOINT` forces a non-incremental one.

### 6.4 Exactly-once end-to-end

Within the graph: barriers/alignment. At the sinks: `TwoPhaseCommitSinkFunction` / modern
`Sink` with committables — pre-commit on snapshot, commit on `notifyCheckpointComplete`
(e.g. Kafka transactions). Source offsets are part of source operator state, so replay +
transactional commit ⇒ end-to-end exactly-once.

---

## 7. Serialization (`flink-core`)

`TypeInformation` is extracted from user function signatures at graph-build time
(`TypeExtractor`); it yields `TypeSerializer`s: dedicated ones for primitives/POJOs
(`PojoSerializer` with per-field serializers), tuples/rows, and Kryo only as fallback for opaque
types. Serializer **snapshots** (`TypeSerializerSnapshot`) are written into every state snapshot
and drive schema-compatibility checks/migration on restore. The Table runtime bypasses most of
this with its own `RowData` binary format (`BinaryRowData`: fixed-length null bits + fields,
variable-length section) operated on directly in `MemorySegment`s by generated code.

---

## 8. Cross-cutting: what makes it fast

- **Single-writer principle everywhere**: RPC endpoints have main threads; tasks have the mailbox;
  buffers have SPSC builder/consumer pairs — almost no lock contention on hot paths.
- **Binary, page-based data**: serialize once into `MemorySegment`s, compare/sort on binary where
  possible, spill in pages.
- **Async by default**: snapshots, state uploads, RPC, channel recovery (FLINK-39522), and the new
  async state API for disaggregated backends — the task thread almost never blocks on I/O.
- **Flow control that degrades gracefully**: credit-based transport + buffer debloating +
  unaligned checkpoints mean backpressure slows the pipeline but does not break checkpointing.
