# Apache Spark — Structured Streaming Internals

> Based on analysis of the Spark source tree at `C:\workspace\opensource\spark`, version
> **5.0.0-SNAPSHOT**, HEAD `ee11a92a9f1`.
> Companions: [spark-architecture.md](spark-architecture.md) ·
> [spark-core-internals.md](spark-core-internals.md) ·
> [spark-sql-catalyst-internals.md](spark-sql-catalyst-internals.md)
>
> Code lives under `sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/`,
> organized into `runtime/`, `operators/stateful/`, `state/`, `checkpointing/`, `sources/`,
> `sinks/`, and `continuous/`. The legacy DStream API is the separate `streaming` module and
> is effectively frozen.

---

## 1. The Central Idea

Structured Streaming is **not** a separate engine. A streaming query is a Catalyst logical
plan executed repeatedly, once per micro-batch, by a variant of `QueryExecution` called
`IncrementalExecution`. Each execution:

- reads a *bounded range* of each source's offsets (rather than the whole source),
- carries state forward from the previous batch through a `StateStore` keyed by
  (operator ID, partition ID, batch version),
- and commits atomically to a write-ahead log so the whole thing is replayable.

The consequence that drives everything else: **the physical plan must be stable across
batches**. If the operator tree changed shape between batches, state store IDs would not
line up and state would be lost or misattributed. This is why streaming queries reject most
plan changes on restart, and why AQE is disabled for streaming.

---

## 2. Execution Runtime (`runtime/`)

### 2.1 Class hierarchy

```
 StreamingQuery (user-facing)
   └─ StreamingQueryWrapper
        └─ StreamExecution                    (abstract; the query thread + WAL protocol)
             ├─ MicroBatchExecution           (the default: Trigger.ProcessingTime / Once /
             │    └─ AsyncProgressTrackingMicroBatchExecution   AvailableNow)
             └─ ContinuousExecution           (experimental, ~ms latency, at-least-once)
```

`StreamExecution` owns:
- the **query execution thread** (a `QueryExecutionThread extends UninterruptibleThread` —
  uninterruptible because HDFS clients corrupt state if interrupted mid-write),
- `offsetLog: OffsetSeqLog` and `commitLog: CommitLog` (the WAL pair),
- the state machine `INITIALIZING → ACTIVE → {RECONFIGURING} → TERMINATED`,
- `awaitOffset`/`awaitTermination` synchronization for tests and for `processAllAvailable`.

### 2.2 The micro-batch loop (`MicroBatchExecution.runActivatedStream`, line 664)

```
 populateStartOffsets(execCtx)        ← recovery: read offsetLog + commitLog
 loop while active:
   ┌ constructNextBatch(execCtx, noDataBatchesEnabled)      (line 977)
   │    · getOffset / latestOffset from each source
   │    · apply ReadLimit (maxOffsetsPerTrigger, maxFilesPerTrigger, …)
   │    · if there is new data (or a no-data batch is needed for eviction):
   │         markMicroBatchStart → offsetLog.add(batchId, availableOffsets)   ← WAL WRITE #1
   │         source.commit(previousOffsets) for sources needing it
   └ runBatch(execCtx, sparkSessionForStream)               (line 1099)
        · replace StreamingExecutionRelation with the batch's actual data
        · new IncrementalExecution(...) → executedPlan
        · run the job; write to the sink
        · commitLog.add(batchId, CommitMetadata(watermark, stateUniqueIds))  ← WAL WRITE #2
        · advance committedOffsets, update StreamingQueryProgress
   sleep per TriggerExecutor
```

**The two-log protocol is the whole fault-tolerance story:**

| Log | Written | Meaning |
|---|---|---|
| `offsets/<batchId>` | *before* the batch runs | "batch N will process exactly this offset range" |
| `commits/<batchId>` | *after* the batch's sink write succeeds | "batch N is durably complete" |

On restart, `populateStartOffsets` (line 806) reads the latest offset-log entry:
- if `latestCommittedBatchId == latestBatchId`, the last batch finished — start batch N+1;
- if `latestCommittedBatchId == latestBatchId - 1`, batch N was planned but not committed —
  **re-run batch N with exactly the same offset range**. This is what makes replay
  deterministic, and it is why sources must support replay by offset range and sinks must be
  idempotent for end-to-end exactly-once;
- if `latestCommittedBatchId < latestBatchId - 1`, the logs are inconsistent — hard error.

`OffsetSeqLog` also persists the **`OffsetSeqMetadata`**: the batch watermark, batch
timestamp, and a snapshot of the SQL configs that affect state (shuffle partitions, state
store provider class, …). Those configs are then *restored* on restart rather than taken
from the current session, because changing `spark.sql.shuffle.partitions` would repartition
state and silently lose it.

`HDFSMetadataLog` is the base implementation: one file per batch, written to a temp path and
atomically renamed, with `CompactibleFileStreamLog` periodically compacting the sequence
(default every 10 batches) so the directory doesn't grow unbounded. `AsyncOffsetSeqLog` /
`AsyncCommitLog` / `AsyncProgressTrackingMicroBatchExecution` move these writes off the
critical path when `asyncProgressTrackingEnabled` is set — trading a small
at-least-once window for latency. `ChecksumCheckpointFileManager` adds integrity checking
over `CheckpointFileManager` (the FS abstraction that handles S3-vs-HDFS atomicity
differences).

### 2.3 Triggers (`TriggerExecutor.scala`)

| Trigger | Executor | Semantics |
|---|---|---|
| `ProcessingTime(interval)` | `ProcessingTimeExecutor` | fire every interval; skip (and warn) if a batch overruns |
| `Trigger.Once` | `SingleBatchExecutor` | exactly one batch, ignoring all rate limits |
| `Trigger.AvailableNow` | `MultiBatchExecutor` + `AvailableNowDataStreamWrapper` | drain everything available *now*, in multiple rate-limited batches, then stop. The correct replacement for `Once` |
| `Continuous(interval)` | `ContinuousExecution` | see §7 |

`AvailableNowSourceWrapper` / `AvailableNowMicroBatchStreamWrapper` freeze each source's
latest offset at query start so "now" doesn't chase a moving target.

### 2.4 `IncrementalExecution` (`runtime/IncrementalExecution.scala`)

A `QueryExecution` subclass that adds a preparation rule set (`state` rules) run after the
normal ones:

- **`StatefulOperatorStateInfo` assignment** — each stateful operator gets a stable
  `operatorId` (assigned by post-order traversal of the plan), the `checkpointLocation`, the
  `batchId` as `storeVersion`, and `numPartitions`. The operator ID must be stable across
  batches; this is the invariant that forbids plan changes.
- **Watermark wiring** — `WatermarkPropagator` computes per-operator input watermarks (§4).
- **`StateStoreCheckpointIdRule`** — for checkpoint format V2, threads
  `currentStateStoreCkptId` (operatorId → array of per-partition checkpoint UUIDs) so a
  restart binds to exactly the state files the last commit produced, rather than trusting
  "whatever version N is in the directory". This closes a real correctness gap with
  eventually-consistent object stores.
- **`StateSchemaBroadcast`** — broadcasts the state schema per operator so executors don't
  each re-read it.
- **Shuffle partitioning override** — `StatefulOperatorPartitioning` forces
  `HashPartitioning` on the grouping keys with the *persisted* partition count.

`UnsupportedOperationChecker` (in `sql/catalyst/analysis/`) is the gate that rejects
operations with no incremental semantics: multiple aggregations without a watermark, sorting
a non-complete-mode stream, certain outer-join shapes without watermark+time constraints,
`limit` in update mode, and so on. Its error messages are the main user-facing surface of
the streaming semantics model.

---

## 3. State Store (`state/`)

### 3.1 The contract

```scala
trait ReadStateStore {                     // versioned, per (operator, partition)
  def id: StateStoreId; def version: Long
  def get(key, colFamilyName): UnsafeRow
  def iterator(colFamilyName): StateStoreIterator[UnsafeRowPair]
  def prefixScan(prefixKey, colFamilyName): StateStoreIterator[UnsafeRowPair]
  def abort(): Unit
}
trait StateStore extends ReadStateStore {
  def put(key, value, colFamilyName): Unit
  def remove(key, colFamilyName): Unit
  def commit(): Long                        // returns the new version
  def metrics: StateStoreMetrics
}
```

Key properties:
- **Versioned and MVCC-ish.** `StateStoreProvider.getStore(version)` returns a store at a
  specific version. A task loads version N and commits version N+1. A retried task loads N
  again — so task retry and speculation are safe.
- **Task-scoped.** `StateStoreRDD` acquires the store in `compute()` and registers an
  `abort` on task-completion failure. Commit happens only on the success path.
- **Column families** — a later addition (needed by `transformWithState`) that lets one
  store hold several logically separate keyspaces (value state, list state, map state,
  timers, TTL indexes) without a store per variable.

`StateStore.scala` also contains the **maintenance machinery**: a background thread pool per
executor that performs snapshotting and cleanup for loaded providers, plus
`unloadedProvidersToClose` tracking so a provider isn't closed while maintenance ops remain.

### 3.2 `HDFSBackedStateStoreProvider` (the original)

- In-memory `HDFSBackedStateStoreMap` per version, plus per-version **delta files** on DFS.
- Every `minDeltasForSnapshot` (10) versions, a background snapshot writes a full `.snapshot`.
- Recovery = load the newest snapshot ≤ target version, then replay deltas.
- **The problem**: the entire state for a partition must fit in executor heap. State size is
  bounded by memory, and GC pressure grows with state.

### 3.3 `RocksDBStateStoreProvider` (the default direction)

`RocksDB.scala` (3,000+ lines) wraps an embedded RocksDB instance per (operator, partition):

> "Class representing a RocksDB instance that checkpoints version of data to DFS.
> After a set of updates, a new version can be committed by calling `commit()`.
> Any past version can be loaded by calling `load(version)`. … This class is not
> thread-safe, so use it only from one thread." — `RocksDB.scala:58`

State lives on **local disk** (the executor's `spark.local.dir`), with the LSM tree giving
memory-bounded operation on state far larger than heap. `RocksDBFileManager` handles the
DFS side: on commit it uploads new SST files and a manifest as a versioned `.zip`, and on
load it downloads only the SSTs not already present locally (SST files are immutable and
content-addressed, so the incremental upload/download is genuinely incremental).

**Changelog checkpointing** (`StateStoreChangelog.scala`) is the latency fix. Instead of
uploading a snapshot every commit, each commit writes a small changelog file:

```
 put    record: | key length | key content | value length | value content |
 delete record: | key length | key content | -1 |
 file:          | record | record | ... | -1 |
```

Full snapshots are then produced asynchronously by the maintenance thread every
`minDeltasForSnapshot` versions. Commit latency drops from "upload the whole state" to
"upload a few KB", at the cost of a longer replay on recovery. `AutoSnapshotLoader` and the
lineage tracking in `RocksDB.scala:3052` (an array of `(version, uniqueId)` pairs) manage
which snapshot+changelog chain reconstitutes a given version — necessary once checkpoint
IDs are UUIDs rather than plain version numbers.

`RocksDBStateEncoder` handles the key/value encoding, including the **range-scan encoder**
that makes timer and TTL indexes ordered-scannable by writing big-endian, sign-flipped
prefixes so `UnsafeRow` byte order matches numeric order.

`RocksDBMemoryManager` gives all instances on an executor a shared block cache and
write-buffer manager, so N partitions don't each allocate their own — without this, RocksDB
memory would be O(partitions).

`RocksDBStateMachine` guards the legal operation sequence (load → update → commit/abort),
because the failure mode of getting this wrong is silent state corruption rather than an
exception.

### 3.4 Supporting infrastructure

- **`StateStoreCoordinator`** — a driver RPC endpoint tracking which executor holds which
  (operator, partition) store, so the next batch's task is scheduled with that executor as
  its preferred location. Getting this wrong means reloading state from DFS every batch.
- **`StateSchemaCompatibilityChecker`** — persists the state key/value schema and validates
  it on restart. Incompatible changes fail the query rather than misinterpreting bytes.
- **`OperatorStateMetadata`** — persists operator IDs and their properties so restarts can
  detect plan changes.
- **`StateStoreRowChecksum`** — per-row integrity checking.
- **State Data Source** (`sql/core/.../datasources/v2/state/`) — read a checkpoint's state as
  a DataFrame (`spark.read.format("statestore")`), including changelog/CDC reads. The
  debugging tool that made stateful streaming tractable to operate.
- **`OfflineStateRepartitionRunner`** — repartitions an existing checkpoint's state to a new
  partition count. This directly addresses the long-standing "you can never change
  `spark.sql.shuffle.partitions`" constraint.

---

## 4. Event Time, Watermarks, and Late Data

### 4.1 The watermark

`EventTimeWatermark` (logical) / `EventTimeWatermarkExec` (physical) tracks the max observed
event time per partition via an accumulator; at batch end `WatermarkTracker` computes the
global watermark as `max_observed_event_time - delayThreshold`, using a configurable
multi-source policy (`min` by default, `spark.sql.streaming.multipleWatermarkPolicy`).

The watermark is **monotonic** (never moves backward) and is persisted in the offset log's
metadata, so a restart resumes with the right watermark rather than re-admitting old data.

Late data — rows with event time below the watermark — is **dropped** by stateful operators
(counted in `numLateInputs` / `numRowsDroppedByWatermark` in `StreamingQueryProgress`).
Stateless operators pass late rows through unchanged.

### 4.2 Multi-operator propagation (`WatermarkPropagator.scala`, SPARK-40925)

The subtle problem: with two stateful operators chained, the downstream one must not use the
*input* watermark, because the upstream operator's output is delayed relative to it —
using the input watermark downstream would drop rows the upstream operator legitimately
emitted late.

`PropagateWatermarkSimulator` solves it by simulating propagation over the physical plan in
post-order:

> - Input watermark for a node = `min(output watermarks of all children)`, excluding children
>   providing no watermark.
> - Output watermark: watermark nodes emit the origin value; **stateless nodes** pass the
>   input watermark through; **stateful nodes** return `op.produceOutputWatermark(input)`.
>
> — `WatermarkPropagator.scala:128-148`

A time-window aggregation, for instance, emits results *at* the window end, so its output
watermark lags its input watermark by the window length — which
`produceOutputWatermark` encodes. Redefining a watermark mid-plan throws.

`UseSingleWatermarkPropagator` preserves the pre-SPARK-40925 global-watermark behavior for
compatibility.

---

## 5. Stateful Operators (`operators/stateful/`)

### 5.1 Aggregation

`StateStoreRestoreExec` → `HashAggregateExec` → `StateStoreSaveExec`. Restore reads the
prior aggregate buffer for each key and merges it into the batch's partial aggregates; save
writes the merged buffer back and emits output according to output mode:

| Output mode | `StateStoreSaveExec` behavior |
|---|---|
| `Append` | emit only keys whose window has closed (watermark passed); requires event-time watermark |
| `Update` | emit every key updated in this batch |
| `Complete` | emit the entire state; state is never evicted |

`StreamingAggregationStateManager` abstracts the key/value split (v2 stores only the
non-key portion of the buffer, saving space over v1).

Session windows (`StreamingSessionWindowStateManager`, `MergingSessionsExec`,
`UpdatingSessionsExec`) need merging of adjacent sessions and therefore a prefix-scan-capable
store — this is why `prefixScan` is in the `ReadStateStore` contract.

### 5.2 Stream-stream joins (`operators/stateful/join/`)

`StreamingSymmetricHashJoinExec` keeps **four** state stores per partition: for each side, a
`keyToNumValues` and a `keyWithIndexToValue` (the multimap encoding). Each arriving row on
one side probes the other side's state and is then added to its own.

State eviction requires a bound on how long a row can still match. That bound comes from a
watermark plus either a time-range join condition (`leftTime BETWEEN rightTime AND
rightTime + INTERVAL 1 HOUR`) or a watermark on the join keys.
`StreamingJoinHelper` extracts the bound from the join condition. Without it, state grows
forever — hence the analyzer's insistence on a watermark for outer joins.

Outer joins additionally must emit unmatched rows only once it is certain no match will
arrive, which is exactly at eviction time — so outer-join results are *delayed* by the
watermark, a frequent source of "where are my rows" confusion.

### 5.3 `flatMapGroupsWithState` and `transformWithState`

`FlatMapGroupsWithStateExec` (`operators/stateful/flatmapgroupswithstate/`) is the original
arbitrary-stateful API: one opaque state object per key, plus processing-time or event-time
timeouts.

`TransformWithStateExec` (`operators/stateful/transformwithstate/`) is the newer, richer
model and is where the investment is going:

- **Typed state variables** (`statevariables/`): `ValueState`, `ListState`, `MapState` — each
  backed by its own RocksDB column family rather than one serialized blob per key. This means
  appending to a list is O(1) rather than read-modify-write of the whole list.
- **Timers** (`timers/`): registered per key, stored in an ordered column family so due
  timers are found by range scan.
- **TTL** (`ttl/`): per-variable expiration with an index column family, so expiry is a scan
  of due entries rather than a full state scan.
- **Schema evolution** (`StateStoreColumnFamilySchemaUtils`) — state variables carry Avro
  schemas so state can survive a class change.
- **Initial state** and a **`StatefulProcessor` lifecycle** (`init`/`handleInputRows`/
  `handleExpiredTimer`/`close`).
- Available to PySpark via `TransformWithStateInPySparkStrategy`.

This is a materially better design than `flatMapGroupsWithState`: granular state access
instead of whole-object serialization, and expiry that doesn't require scanning everything.

### 5.4 Deduplication and limits

`StreamingDeduplicateExec` stores seen keys; `StreamingDeduplicateWithinWatermarkExec` bounds
that state by the watermark. `streamingLimits.scala` implements `StreamingGlobalLimitExec`
with a counter in state (only legal in Append mode).

---

## 6. Sources and Sinks

**Sources** implement `Source` (V1) or `MicroBatchStream`/`ContinuousStream` (V2). The V2
contract is:
- `latestOffset(startOffset, readLimit)` — how far can we read, subject to rate limits;
- `planInputPartitions(start, end)` — the batch's partitions;
- `commit(end)` — the source may release resources up to this offset (Kafka commits offsets,
  `FileStreamSource` cleans up its log).

`FileStreamSource` + `FileStreamSourceLog` track seen files in a compactible log; this log
is the reason a long-running file-stream query on a directory with millions of files
eventually degrades, and why `maxFileAge`/archival options exist.

**Sinks** implement `Sink` (V1) or `StreamingWrite` (V2, via `MicroBatchWrite`). Only sinks
that are idempotent per batch ID give end-to-end exactly-once; `FileStreamSink` achieves it
with `ManifestFileCommitProtocol` (a per-batch manifest in `_spark_metadata` that readers
consult, so a re-run batch's duplicate files are ignored). `ForeachBatchSink` and
`ForeachWriterTable` hand the batch to user code with the batch ID, making idempotency the
user's responsibility — which is why the batch ID is passed at all.

---

## 7. Continuous Processing (`continuous/`)

An experimental low-latency mode: long-running tasks that never finish, reading and writing
continuously, with **epochs** rather than batches.

- `EpochCoordinator` (driver RPC endpoint) advances an epoch counter every
  `continuous.checkpointInterval`.
- Each task reports its offset at each epoch boundary; the coordinator writes the offset log
  once all partitions report.
- `ContinuousDataSourceRDD` / `ContinuousQueuedDataReader` decouple the reader thread from
  the compute thread through a queue.
- Latency ~1 ms, but **at-least-once only**, and it supports only map-like operations — no
  aggregations, no joins.

It has not converged; the practical low-latency path in this tree appears to be the
"real-time mode" machinery visible in `RealTimeModeAllowlist.scala`,
`LowLatencyMemoryStream`, and `RealTimeRowWriterFactory`, which keeps micro-batch semantics
while shrinking the batch boundary cost.

---

## 8. Observability

`ProgressReporter` emits `StreamingQueryProgress` per batch:
- `inputRowsPerSecond` / `processedRowsPerSecond`,
- `durationMs` broken into `triggerExecution`, `getOffset`/`latestOffset`, `addBatch`,
  `queryPlanning`, `walCommit`,
- per-source `startOffset`/`endOffset`/`numInputRows`,
- per-stateful-operator `numRowsTotal`, `numRowsUpdated`, `numRowsRemoved`,
  `memoryUsedBytes`, `numRowsDroppedByWatermark`, plus provider-specific custom metrics
  (RocksDB compaction time, SST file sizes, cache hit rates).

`StreamingQueryListener` receives start/progress/idle/terminated events through
`StreamingQueryListenerBus` (on its own `AsyncEventQueue`, so a slow listener can't stall the
query thread — but can drop events; see
[architecture §6](spark-architecture.md#6-event-bus-status-tracking-and-the-ui)).

The three numbers that matter operationally: `walCommit` duration (checkpoint FS latency),
`numRowsTotal` per operator (unbounded state growth), and the gap between
`inputRowsPerSecond` and `processedRowsPerSecond` (falling behind).

---

## 9. Design Assessment (Staff+ Lens)

**Strong**

- **Reusing the batch engine wholesale.** Every Catalyst optimization, every codegen
  improvement, every new source benefits streaming for free. The alternative — a separate
  streaming engine, as Flink and the old DStream API both are — costs double implementation
  forever. This is the single best decision in the design.
- **The two-log WAL protocol** is minimal and correct: offsets before, commits after,
  replay the uncommitted batch verbatim. Everything else about exactly-once follows from
  source replayability and sink idempotency, which are stated as explicit contracts rather
  than assumed.
- **Persisting configs in the offset log** is an unglamorous detail that prevents a whole
  class of silent state loss.
- **`transformWithState`'s column-family design** is a genuine advance over
  `flatMapGroupsWithState`: granular state access, indexed timers, indexed TTL.
- **Changelog checkpointing plus incremental SST upload** correctly separates commit latency
  from state size.
- **The state data source** turned an opaque binary checkpoint into something operators can
  query. Retrofitting observability at that level is rare and valuable.

**Tensions**

- **The stable-plan constraint is a heavy tax.** Operator IDs are assigned positionally, so
  adding a filter can invalidate a checkpoint. `OfflineStateRepartitionRunner` and state
  schema evolution are chipping at this, but the fundamental coupling of *plan shape* to
  *state identity* remains, and it is what makes streaming query upgrades an operational
  event rather than a deployment.
- **Watermark semantics are correct but not intuitive.** Multi-operator propagation
  (SPARK-40925) fixed a real correctness bug, but the resulting model — where a downstream
  operator's watermark lags its input by an amount that depends on the upstream operator's
  type — is hard for users to predict. Outer-join output delay is the most common
  manifestation.
- **Two arbitrary-state APIs.** `flatMapGroupsWithState` and `transformWithState` will
  coexist for a long time, as will two state store providers and two checkpoint formats.
- **Continuous processing is unfinished.** It has been experimental for years with a
  restricted operator set and weaker guarantees, while the actual latency work moved
  elsewhere. Carrying it costs a parallel execution path (`ContinuousExecution`,
  `EpochCoordinator`, continuous variants of sources and RDDs).
- **Micro-batch latency floor.** Even with async progress tracking, per-batch overhead
  (planning, WAL, task launch) puts a floor around the low hundreds of milliseconds. That is
  a structural consequence of the reuse decision above — the right trade, but a real one.
