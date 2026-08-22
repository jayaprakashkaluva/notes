# Apache Spark — Core Execution Kernel Internals

> Based on analysis of the Spark source tree at `C:\workspace\opensource\spark`, version
> **5.0.0-SNAPSHOT**, HEAD `ee11a92a9f1`.
> Big-picture companion: [spark-architecture.md](spark-architecture.md)
> Query layer: [spark-sql-catalyst-internals.md](spark-sql-catalyst-internals.md)
>
> Paths are relative to the repo root; most code here is under
> `core/src/main/{scala,java}/org/apache/spark/`.

---

## 1. The RDD Model (`rdd/RDD.scala`, 2,290 lines)

An RDD is defined by exactly five properties, and every RDD subclass in Spark, MLlib,
GraphX, and third-party connectors implements some subset of them:

| Property | Signature | Purpose |
|---|---|---|
| Partitions | `getPartitions: Array[Partition]` | The unit of parallelism |
| Dependencies | `getDependencies: Seq[Dependency[_]]` | Lineage edges |
| Compute | `compute(split, context): Iterator[T]` | Produce one partition's rows |
| Partitioner | `partitioner: Option[Partitioner]` | Declared key layout (enables shuffle elision) |
| Preferred locations | `getPreferredLocations(split): Seq[String]` | Locality hints |

Two things about `compute` are load-bearing. First, it returns an **`Iterator`**: the whole
narrow-dependency chain within a stage is a composed iterator, so `map`→`filter`→`map`
does not materialize intermediates. Second, it is called on the executor with a
`TaskContext`, which is how a partition-level computation reaches metrics, accumulators,
completion listeners, and the `TaskMemoryManager`.

### 1.1 Dependencies (`Dependency.scala`)

```
Dependency
 ├─ NarrowDependency          (one child partition depends on a bounded set of parents)
 │    ├─ OneToOneDependency   (map, filter, mapPartitions)
 │    ├─ RangeDependency      (union)
 │    └─ PruneDependency      (partition pruning)
 └─ ShuffleDependency         (STAGE BOUNDARY — carries partitioner, serializer,
                               keyOrdering, aggregator, mapSideCombine, shuffleId,
                               shuffleHandle, shuffleWriterProcessor)
```

`ShuffleDependency` is the only thing the DAGScheduler cuts on. It registers itself with
the `ShuffleManager` at construction time (obtaining a `ShuffleHandle`), which is why
merely *building* a shuffling RDD allocates a shuffle ID even if the job never runs.

### 1.2 Determinism levels (`RDD.scala:2288`)

```scala
object DeterministicLevel extends Enumeration {
  val DETERMINATE, UNORDERED, INDETERMINATE = Value
}
```

- **DETERMINATE** — recomputation yields the same rows in the same order.
- **UNORDERED** — same rows, possibly different order (the normal result of a shuffle).
- **INDETERMINATE** — recomputation may yield *different rows* (`sample` without a fixed
  seed, `zipWithIndex` over an unordered parent, a shuffle whose parent is indeterminate).

`getOutputDeterministicLevel` (`RDD.scala:2187`) propagates this up the lineage: a shuffle
over an INDETERMINATE parent stays INDETERMINATE; a shuffle over a DETERMINATE parent
becomes UNORDERED. `RDD.isReliablyCheckpointed` forces DETERMINATE because the data is
frozen on durable storage. This enum drives the rollback machinery in §7 and is the
mechanism by which Spark avoids silently producing wrong answers after a partial retry.

### 1.3 Checkpointing

Two flavors, both in `rdd/`:
- **Reliable** (`ReliableRDDCheckpointData`) — writes partitions to a durable FS and
  *truncates the lineage*. Used for cutting unbounded lineage in iterative jobs, and for
  restoring determinism.
- **Local** (`LocalRDDCheckpointData`) — persists to executor local disk. Faster, but a lost
  executor is unrecoverable.

`RDDCheckpointData.synchronized` guards the read of `rdd.partitions` and the task-binary
serialization in `DAGScheduler.submitMissingTasks` (line 2778), because a concurrent job
checkpointing the same RDD would otherwise produce a task binary inconsistent with the
partition array it was built against.

---

## 2. DAGScheduler (`scheduler/DAGScheduler.scala`, 4,922 lines)

The DAGScheduler is a **single-threaded event-loop state machine**. All mutation happens on
`DAGSchedulerEventProcessLoop`'s thread; public methods (`taskEnded`, `executorLost`,
`submitJob`, …) only *post events*. This is why the class holds plain `HashMap`s rather
than concurrent structures, and why the class comment carries an explicit checklist
(lines 116–122) demanding that every new data structure be cleared on job end and added to
`DAGSchedulerSuite.assertDataStructuresEmpty`.

### 2.1 Core state

```scala
jobIdToStageIds     : HashMap[Int, HashSet[Int]]
stageIdToStage      : HashMap[Int, Stage]
shuffleIdToMapStage : HashMap[Int, ShuffleMapStage]   // only for in-flight jobs
jobIdToActiveJob    : HashMap[Int, ActiveJob]
waitingStages       : HashSet[Stage]   // parents not done
runningStages       : HashSet[Stage]
failedStages        : HashSet[Stage]   // to resubmit after fetch failure
cacheLocs           : HashMap[Int, IndexedSeq[Seq[TaskLocation]]]
failedEpoch         : HashMap[String, Long]  // executorId -> last handled failure epoch
```

Note the comment on `shuffleIdToMapStage` (line 154): once the jobs needing a shuffle
finish, the stage object is dropped and **the only record of the shuffle data is in the
`MapOutputTracker`**. A later job reusing the same shuffle reconstructs a `ShuffleMapStage`
whose `findMissingPartitions` immediately returns empty.

`cacheLocs` has a documented lock-ordering rule (line 290, SPARK-4454): synchronize on the
RDD *before* the map, never the reverse.

### 2.2 Stage construction

`handleJobSubmitted` → `createResultStage` → `getOrCreateParentStages` →
`getOrCreateShuffleMapStage`. The graph walk (`traverseRDDGraph`,
`getShuffleDependenciesAndResourceProfiles`) is iterative with an explicit stack, not
recursive — deep lineages (thousands of `union`s, or an unchecked iterative algorithm)
would otherwise blow the driver stack.

`getMissingAncestorShuffleDependencies` walks *past* already-registered stages so that a
shuffle whose stage object was garbage-collected is rediscovered.

`getMissingParentStages` decides what must run first: a parent `ShuffleMapStage` is missing
iff `!stage.isAvailable`, i.e. `mapOutputTracker` does not hold all of its map outputs. This
is the mechanism behind "skipped stages" in the UI.

### 2.3 `submitStage` / `submitMissingTasks`

`submitStage` (line 2331) is the recursive scheduler kernel:

```
if not waiting/running/failed:
    if attempts exhausted (STAGE_MAX_ATTEMPTS)  -> abortStage
    missing = getMissingParentStages(stage).sortBy(_.id)
    if missing.isEmpty -> submitMissingTasks(stage, jobId)
    else                -> submit each parent, park this stage in waitingStages
```

Note line 2364: after recursing into parents, it re-checks `stageIdToStage.contains(stage.id)`
because aborting a parent tears down the whole job's stage state mid-recursion. Re-parking a
job-less stage in `waitingStages` would be a permanent scheduler-state leak.

`submitMissingTasks` (line 2659) does the real work, in this order:

1. **Pre-emptive rollback** for statically indeterminate stages being retried (§7).
2. `stage.findMissingPartitions()` — for a `ResultStage`, partitions whose result the
   `ActiveJob` hasn't received; for a `ShuffleMapStage`, partitions with no `MapStatus`.
3. Clone the job `Properties` **per stage** (line 2702) so per-resource-profile values don't
   leak into sibling stages sharing the job's mutable `Properties`.
4. `outputCommitCoordinator.stageStart(...)` — arms the commit arbiter (§8).
5. Compute preferred locations per partition via `getPreferredLocs`.
6. **Serialize the task binary once and broadcast it.** For a `ShuffleMapStage` this is
   `(rdd, shuffleDep)`; for a `ResultStage`, `(rdd, func)`. Each task deserializes its own
   copy — deliberately, so tasks that mutate closure-referenced objects (Hadoop `JobConf`
   is not thread-safe) are isolated (line 2764 comment).
7. Build `ShuffleMapTask`/`ResultTask` objects and hand a `TaskSet` to the `TaskScheduler`.

Failure modes here are all `abortStage`: non-serializable tasks, failed partition
computation, failed preferred-location computation.

### 2.4 Task completion

`handleTaskCompletion` (line 3250) is the largest method in the class. The shape:

- **`Success` on a `ShuffleMapTask`** → register `MapStatus` with `MapOutputTracker`;
  if the stage now has all outputs, `markMapStageJobsAsFinished` and submit waiting children.
  A stale attempt's status is ignored (`task.stageAttemptId != stage.latestInfo.attemptNumber`).
- **`Success` on a `ResultTask`** → `job.finished(idx) = true`; when `numFinished == numPartitions`,
  the `JobWaiter` is completed and the job is cleaned up.
- **`FetchFailed`** (line 3490) → the interesting path:
  1. Mark the *consuming* stage failed and the *producing* map stage as needing
     resubmission (`mapOutputTracker.unregisterMapOutput` or `unregisterAllMapOutput`).
  2. If the failure indicates an executor/host is gone, `handleExecutorLost` bumps the
     **epoch** and removes all that executor's map outputs.
  3. Add both stages to `failedStages` and schedule `resubmitFailedStages` after
     `DAGScheduler.RESUBMIT_TIMEOUT` (a hard-coded 200 ms, `DAGScheduler.scala:4873` — not
     configurable). The delay batches the storm of fetch failures that a single lost node
     produces into one resubmission.
  4. If the map stage is indeterminate, roll back succeeding stages (§7).
- **`ExceptionFailure` / `TaskKilled` / `Resubmitted`** → generally handled by the
  TaskScheduler; DAGScheduler only updates accumulators and metrics.

### 2.5 Epochs — the fencing mechanism

`MapOutputTracker` maintains a monotonically increasing `epoch` (`MapOutputTracker.scala:651`).
Every `TaskDescription` carries the epoch at which it was created. When an executor is lost,
the driver increments the epoch. Executors call `updateEpoch` (line 1684) on task start; a
higher epoch clears the executor's cached map statuses.

`failedEpoch: HashMap[String, Long]` on the driver records the epoch at which each
executor's failure was fully processed, so a *late* `ExecutorLost` event for an executor
already handled at an equal-or-higher epoch is ignored. Without this, out-of-order RPC
delivery would cause repeated, redundant unregistration of map outputs.

This is a Lamport-clock-style fence, and it is the reason shuffle-failure handling is
correct despite the driver receiving events from many threads out of order.

### 2.6 Pipelined shuffle (in-flight, not in released Spark)

This tree adds `PipelinedShuffleDependency` and co-scheduling: a consumer stage may be
submitted **while its producer is still running**, on the theory that a pipelined shuffle is
incrementally readable. The machinery:

- `dependentStageMap: HashMap[Stage, DependentStageInfo]` (line 188) buffers the consumer's
  successful `CompletionEvent`s until every pipelined producer has finished. Without this
  buffering, the consumer's job could complete and cancel a still-running producer, or
  expose output before the producer's side effects landed.
- **Gang admission**: `rejectUnadmittablePipelinedGroup` decides up front, at job submission,
  whether the whole producer+consumer group fits in available slots. Deciding per-stage
  mid-flight would re-measure a moving target and risk deadlock (producer and consumer each
  holding half the slots).
- Co-scheduling only fires when *every* missing parent is pipelined **and** already running
  (line 2384); a mixed pipelined/regular parent set parks the stage normally.
- Speculation, dynamic allocation, fan-out to multiple consumers, and mixing pipelined with
  regular shuffles in one group are all rejected fail-fast at submission.

Architecturally this is a partial dissolution of the stage barrier — the most significant
change to the DAGScheduler's core model in years, and correspondingly hedged with
conservative admission checks.

---

## 3. TaskSchedulerImpl and TaskSetManager

### 3.1 `TaskSchedulerImpl` (1,409 lines)

Sits between the DAGScheduler (which knows about stages) and the `SchedulerBackend` (which
knows about executors). Its class comment enumerates the threads that call it: the
DAGScheduler event loop, RPC handler threads processing status updates, the periodic
`reviveOffers` timer, and task-result-getter threads. Every public method is `synchronized`,
with an explicit **lock-ordering warning**: some backends synchronize on themselves before
calling in, so the scheduler must never call into a backend while holding its own lock.

`resourceOffers(offers, isAllFreeResources)` is the matching loop:

1. Filter offers against the `HealthTracker` (excluded executors/nodes).
2. Shuffle the offer list to avoid always loading the same executor.
3. Ask the root `Pool` for `sortedTaskSets` — FIFO or FAIR depending on
   `spark.scheduler.mode` (`SchedulingAlgorithm.scala`, `SchedulableBuilder.scala`;
   FAIR reads `fairscheduler.xml` for pool weights and minShare).
4. For each task set, in increasing locality order, ask its `TaskSetManager` for a task per
   offer. Break out of a locality level as soon as it yields nothing.
5. Enforce **barrier semantics**: a barrier task set is only launched if *all* its tasks can
   be launched in this round (gang scheduling); otherwise nothing is launched.

The `isAllFreeResources` flag is the input to the delay-scheduling heuristic described in
the class comment (lines 71–81): "delay" is measured as time since the TaskSetManager last
launched a task *and* has not rejected a full-share offer. The legacy behavior (reset the
timer on any task launch) is behind `spark.locality.wait.legacyResetOnTaskLaunch`.

### 3.2 `TaskSetManager` (1,587 lines)

Owns one stage attempt's tasks. Not thread-safe — it must be called under the
TaskScheduler's lock.

**Locality levels** (`TaskLocality.scala`):
```scala
val PROCESS_LOCAL, NODE_LOCAL, NO_PREF, RACK_LOCAL, ANY = Value
```
`myLocalityLevels` is computed from the *actual* pending-task index (which executors/hosts
have local tasks), so a task set with no locality preferences skips straight to `ANY` rather
than burning `spark.locality.wait` (default 3s) per level. `getAllowedLocalityLevel` walks
down the levels as the wait expires; any successful launch at a tighter level resets it.

Pending tasks are indexed four ways (`pendingTasks.forExecutor`, `.forHost`, `.noPrefs`,
`.forRack`, `.all`) so offer matching is O(1) lookup rather than a scan.

**Retries.** `maxTaskFailures` (default `spark.task.maxFailures` = 4) counts failures *per
task index*, not per attempt. A `FetchFailed` does not count against this budget — it is a
stage-level problem, handled by the DAGScheduler.

**Speculation.** `checkSpeculatableTasks` runs on a timer. A task is speculatable when its
runtime exceeds `spark.speculation.multiplier` × the median of successful tasks, once
`spark.speculation.quantile` of tasks have finished. `spark.speculation.task.duration.threshold`
adds an absolute trigger for small stages where the quantile never fills. Speculative
copies race; the first to finish wins, the loser is killed with `TaskKilled("another attempt
succeeded")`. This is safe only because of the `OutputCommitCoordinator` (§8) for writing
tasks and because accumulators from killed tasks are discarded.

**Exclusion.** `TaskSetExcludeList` handles stage-scoped exclusion (this executor failed
this task twice → don't offer it this task; failed N distinct tasks → don't offer it this
stage). `HealthTracker` handles application-scoped exclusion with periodic expiry. The
class comment (`HealthTracker.scala:35`) is candid about the design tension: bad user code
produces many failures that must *not* be blamed on executors, while flaky executors
produce sparse failures spread across many small stages that must be aggregated to be
visible.

---

## 4. Executor (`executor/Executor.scala`, 1,668 lines)

One `Executor` per JVM, driven by `CoarseGrainedExecutorBackend` over RPC.

- **Thread pool** — a cached pool named `Executor task launch worker-<tid>`. Each
  `TaskRunner` is a `Runnable`.
- **Setup per task**: set the thread's context classloader (per-`JobArtifactSet`, so
  session-scoped jars in Connect don't collide), `updateDependencies` (fetch jars/files/
  archives, with a timestamp cache), deserialize the task, create a `TaskMemoryManager`.
- **Run**: `task.run(taskAttemptId, attemptNumber, metricsSystem, ...)` sets up the
  `TaskContextImpl`, runs `runTask`, then fires completion/failure listeners.
- **Result handling**: serialize the value; if `resultSize` fits in
  `spark.task.maxDirectResultSize` (and under `spark.rpc.message.maxSize`), send it inline
  in a `StatusUpdate`; otherwise store it as a `TaskResultBlockId` in the BlockManager and
  send an `IndirectTaskResult`. Results over `spark.driver.maxResultSize` are discarded and
  the task is failed — the guard that prevents a `collect()` from OOMing the driver.
- **Interruption**: `spark.job.interruptOnCancel` controls whether `Thread.interrupt` is
  used. `TaskReaper` (`spark.task.reaper.enabled`) escalates: if a killed task doesn't die
  within `spark.task.reaper.killTimeout`, it can kill the JVM. This exists because user code
  in a tight non-interruptible loop is otherwise unkillable.
- **Heartbeats**: `Heartbeater` ships accumulator updates and executor metrics to the driver
  every `spark.executor.heartbeatInterval`. Missing `spark.network.timeout` worth of
  heartbeats makes the driver declare the executor lost.
- **Metrics polling**: a separate poller samples JVM/GC/memory metrics so the peak-memory
  columns in the UI aren't sampled only at task boundaries.

---

## 5. Memory Management

### 5.1 `UnifiedMemoryManager` (`memory/UnifiedMemoryManager.scala`)

```
 usableMemory   = heap - RESERVED_SYSTEM_MEMORY_BYTES (300 MB, line 264)
 maxMemory      = usableMemory * spark.memory.fraction        (0.6)
 storageRegion  = maxMemory   * spark.memory.storageFraction  (0.5)
 → default: 30% of heap "guaranteed" storage, 60% shared pool, 40% user/JVM overhead
```

The boundary is **soft and asymmetric**:

- Storage may borrow all free execution memory, but is evicted when execution reclaims it.
- Execution may borrow all free storage memory, and **is never evicted**. The class comment
  (line 47) is explicit that this is a complexity trade-off, not an oversight.

The asymmetry is correct: evicting cached blocks costs a recompute; failing an execution
allocation costs a spill or a task failure. But it means a job that eats storage with
execution memory will silently degrade `cache()` to no-op — the classic "my cached DataFrame
isn't cached" symptom.

`ExecutionMemoryPool.acquireMemory` implements **fair sharing among active tasks**: each of
N active tasks is entitled to between `1/2N` and `1/N` of the pool. A task requesting memory
blocks (on the pool's monitor) until it can get at least `1/2N`, and the pool is notified
whenever a task releases memory or the active-task count changes. This prevents the first
task on an executor from grabbing everything.

Off-heap (`spark.memory.offHeap.enabled` / `.size`) is a separate, statically sized pair of
pools using `sun.misc.Unsafe` allocation.

### 5.2 `TaskMemoryManager` (`core/src/main/java/.../memory/TaskMemoryManager.java`)

The address-encoding problem: to store pointers *inside* off-heap structures you need a
stable 64-bit address, but on-heap Java addressing is `(Object base, long offset)` and the
base moves under GC.

Solution — a **page table**:

```
 on-heap 64-bit encoded address:
   [ 13 bits page number ][ 51 bits offset within page ]
   → 8,192 pages; page size capped at (2^31 - 1) * 8 bytes ≈ 17 GB
   → ~140 TB addressable per task in principle
 off-heap: the raw address is stored directly
```

`pageTable: MemoryBlock[8192]` maps page number → base object (null off-heap). `encodePageNumberAndOffset`
/ `getPage` / `getOffsetInPage` do the translation. `allocatePage` acquires from the
`MemoryManager`, and on failure calls `spill()` on the task's registered `MemoryConsumer`s
in increasing order of memory used — the **cooperative spilling** protocol.

`MemoryConsumer` is the SPI every spillable operator implements: `ShuffleExternalSorter`,
`UnsafeExternalSorter`, `BytesToBytesMap`, `LongToUnsafeRowMap`, `ExternalAppendOnlyUnsafeRowArray`.

### 5.3 `PackedRecordPointer` — a tighter encoding for shuffle

For the shuffle sorter specifically, records are addressed with 8 bytes total including the
partition ID (`shuffle/sort/PackedRecordPointer.java`):

```
 [ 24 bit partition id ][ 13 bit page number ][ 27 bit offset in page ]
 → max 16,777,216 partitions; max page size 128 MB; ~1 TB per task
```

Packing partition ID into the pointer is what makes the shuffle sort a **radix/binary sort
over an array of longs** with no pointer chasing and no object headers — 8 bytes per record
in the sort array rather than an object per record.

---

## 6. Shuffle

### 6.1 Writer selection (`shuffle/sort/SortShuffleManager.scala`)

`registerShuffle` picks one of three handles, checked in order:

1. **`BypassMergeSortShuffleHandle`** — if `!mapSideCombine` and
   `numPartitions <= spark.shuffle.sort.bypassMergeThreshold` (200).
   `BypassMergeSortShuffleWriter` opens one file per reduce partition, writes records
   directly with no sorting at all, then concatenates. Hash-shuffle semantics with
   sort-shuffle file layout. Fast for small fan-out; catastrophic for large fan-out because
   of the open-file count.
2. **`SerializedShuffleHandle`** ("Tungsten sort") — if `canUseSerializedShuffle`:
   serializer supports relocation of serialized objects (Kryo, and Spark SQL's
   `UnsafeRowSerializer`), no map-side combine, and `numPartitions <= 16,777,216`.
   `UnsafeShuffleWriter` serializes each record immediately and sorts **packed pointers**,
   never touching the objects. Spill merging concatenates compressed byte ranges — with a
   concatenable codec (LZ4, Snappy, LZF, ZSTD) the merge is a `transferTo` with no
   decompression at all.
3. **`BaseShuffleHandle`** — everything else. `SortShuffleWriter` uses `ExternalSorter` with
   a `PartitionedAppendOnlyMap` (with map-side combine) or `PartitionedPairBuffer` (without),
   spilling sorted runs to disk and k-way merging.

The three-way split is a clean example of specializing for the cases that dominate: modern
DataFrame shuffles almost always take path 2, because `UnsafeRow` is relocatable and SQL
aggregation does its own combining.

### 6.2 File layout (`shuffle/IndexShuffleBlockResolver.scala`)

One map task produces exactly **two files** (plus optionally a checksum file):

```
 shuffle_<shuffleId>_<mapId>_0.data    ← all partitions, concatenated in partition order
 shuffle_<shuffleId>_<mapId>_0.index   ← (numPartitions + 1) longs: offsets into .data
 shuffle_<shuffleId>_<mapId>_0.checksum.<algo>   ← per-partition CRC32/Adler32 (optional)
```

The reduce ID in the filename is fixed at 0 — the index file supplies the per-reducer
byte range. This "consolidated" layout replaced the O(mappers × reducers) file explosion of
hash shuffle and is why Spark can shuffle at 10k × 10k without exhausting file descriptors.

Index/data files are written to a temp name and atomically renamed together, with a
verification read on commit, so a speculative duplicate writer can't produce a torn pair.

### 6.3 Read path (`storage/ShuffleBlockFetcherIterator.scala`)

Returns `(BlockId, InputStream)` tuples lazily so the reducer pipelines against the network.
The throttles, all simultaneously enforced:

| Config | Meaning |
|---|---|
| `spark.reducer.maxSizeInFlight` (48m) | total bytes of outstanding remote fetches |
| `spark.reducer.maxReqsInFlight` | number of concurrent fetch requests |
| `spark.reducer.maxBlocksInFlightPerAddress` | per-remote-host block count (protects a hot mapper node) |
| `spark.maxRemoteBlockSizeFetchToMem` | above this, stream the block to disk instead of heap |
| `spark.shuffle.detectCorrupt` | decompress-verify small blocks; retry once on corruption |

`maxBytesInFlight` is deliberately split into ~5 concurrent requests
(`maxBytesInFlight / 5`) so the reducer streams from several mappers at once rather than
serializing on one.

**Corruption diagnosis**: when `spark.shuffle.checksum.enabled` is on and a block fails to
decompress, the reducer asks the *writer side* to recompute the checksum, which distinguishes
"disk corrupted the file" from "network corrupted the transfer" from "a bug in the
serializer". The result appears in the `FetchFailed` message.

**Batch fetch** (`doBatchFetch`, enabled by `spark.sql.adaptive.fetchShuffleBlocksInBatch`)
coalesces contiguous partition ranges from the same mapper into one request — essential for
AQE's coalesced partitions, which by construction read many adjacent reducer IDs.

### 6.4 Push-based shuffle (SPARK-30602)

`ShuffleBlockPusher` on the map side pushes blocks to remote **shuffle merge services**,
which merge blocks for the same reduce partition from many mappers into one sequential file.
Reducers then read a few large merged chunks instead of thousands of small ones.

Driver-side coordination lives in the DAGScheduler: `prepareShuffleServicesForShuffleMapStage`
picks merger locations, `scheduleShuffleMergeFinalize` waits
`spark.shuffle.push.finalize.timeout` for stragglers before finalizing, and `MergeStatus`
joins `MapStatus` in the `MapOutputTracker`. `PushBasedFetchHelper` on the read side falls
back to the original unmerged blocks when a chunk fetch fails.

Note the interaction with determinism (`DAGScheduler.scala:3002`): for a statically
indeterminate stage, partial merge results are treated as size 0 so they are never
registered — the stage's output may change on retry, and merged data cannot be selectively
invalidated.

### 6.5 `MapOutputTracker` (1,986 lines)

Driver-side `MapOutputTrackerMaster` holds `shuffleStatuses: Map[Int, ShuffleStatus]`, where
each `ShuffleStatus` holds `mapStatuses: Array[MapStatus]` plus merge statuses. Executors
run `MapOutputTrackerWorker` with a local cache invalidated by epoch.

Two `MapStatus` representations:
- **`CompressedMapStatus`** — one byte per reduce partition, sizes encoded on a logarithmic
  scale (1.1^x). Lossy, ~10% error, fine for throttling decisions.
- **`HighlyCompressedMapStatus`** — used above `spark.shuffle.minNumPartitionsToHighlyCompress`
  (2000). Stores the *average* size of non-empty blocks, an exact set of huge blocks
  (> 2× average), and a `RoaringBitmap` of empty blocks. Turns O(numPartitions) per map
  status into O(1) plus outliers, which is what keeps driver memory bounded at 10k × 10k.

Serialized statuses are broadcast when they exceed `spark.shuffle.mapOutput.minSizeForBroadcast`,
because otherwise every reducer fetching the full status array over RPC would saturate the
driver. `getStatistics` on a `ShuffleDependency` produces the `MapOutputStatistics`
(bytes per reduce partition) that AQE consumes for coalescing and skew detection.

---

## 7. Indeterminate Stages and Rollback

The correctness hazard: stage B reads stage A's shuffle output. A partition of A is lost.
A is recomputed — but A is INDETERMINATE, so the recomputed partition holds *different rows*.
Partitions of B that already succeeded read the old data; partitions retried read the new
data. The result is silently wrong.

Spark's answer (`rollbackSucceedingStages`, `DAGScheduler.scala:3027`):

- When a fetch failure forces retry of an INDETERMINATE map stage, **all succeeding stages
  are rolled back entirely** — their shuffle outputs cleared, in-flight results marked to be
  ignored, and every task rerun.
- If a succeeding stage cannot be rolled back — most importantly a `ResultStage` whose
  partitions already ran a side-effecting action, or a stage whose output was already
  consumed — **the job is aborted** rather than silently producing wrong output.
- `submitMissingTasks` (line 2673) does this *pre-emptively* for statically indeterminate
  stages on retry, rather than waiting for a task-completion event: cheaper, and it
  guarantees `findMissingPartitions` returns *all* partitions rather than a partial set.
- A separate runtime path exists for checksum-mismatch detection, where indeterminacy is
  only discovered at completion time and must be handled reactively.
- `ReliableRDDCheckpointData` forces DETERMINATE, so checkpointing an indeterminate RDD is
  the user-level escape hatch.

This is a good example of Spark choosing "fail loudly" over "recover optimistically" at the
one place where lineage recomputation is unsound.

---

## 8. `OutputCommitCoordinator`

Speculation plus file output plus at-most-once commit semantics is a distributed consensus
problem in miniature. The coordinator solves it centrally:

- Driver endpoint tracks, per (stage, stage attempt, partition), which task attempt has been
  *authorized* to commit.
- `TaskContext.attemptNumber` is used by `HadoopMapReduceCommitProtocol`/
  `SQLHadoopMapReduceCommitProtocol` to call `canCommit(stage, attempt, partition)` before
  moving files from staging to the final location.
- First requester wins; all others are denied and fail with `CommitDeniedException`, which
  the TaskSetManager treats as a non-counting failure.
- `stageStart(stage, maxPartitionId)` (called in `submitMissingTasks`) resets authorization
  state per stage attempt — necessary so a rolled-back stage can re-authorize.

Without this, two speculative copies of the same write task could both rename into the
output directory.

---

## 9. Storage Layer Details

### 9.1 `BlockInfoManager`

Readers-writer locks per block, **scoped to a task attempt**:

- `lockForReading(blockId, blocking)` / `lockForWriting` / `unlock`.
- `registerTask(taskAttemptId)` and `releaseAllLocksForTask(taskAttemptId)` — the latter is
  invoked from the task's completion listener, so a task that dies holding a block lock
  cannot deadlock the executor.
- Write locks are exclusive and are what make concurrent `cache()` of the same partition by
  two tasks safe: one wins and computes, the other blocks and then reads.

### 9.2 `BlockManager` (2,600 lines)

Responsibilities: `putSingle`/`putIterator`/`putBytes` into memory or disk per
`StorageLevel`, `get` with local-then-remote resolution, replication, and serving blocks to
peers via `NettyBlockTransferService`.

Notable subsystems:
- **`BlockManagerDecommissioner`** — on graceful decommission (spot-instance preemption, K8s
  drain), proactively migrates cached RDD blocks and shuffle blocks to peers or to
  `FallbackStorage` (a durable path) so the executor's loss doesn't trigger recomputation.
  `MigratableResolver` is the shuffle-side hook.
- **Shuffle-block serving** — the executor's BlockManager can serve shuffle blocks directly,
  or the External Shuffle Service (`common/network-shuffle`) can, which is what allows
  dynamic allocation to release executors without losing their shuffle output.
- **`RDDBlockId` cache tracking** feeds back to `DAGScheduler.getCacheLocs`, closing the loop
  between caching and locality-aware scheduling.

### 9.3 `DiskBlockManager`

Creates `spark.local.dir/blockmgr-<uuid>/<2-hex-subdir>/` trees and hashes block IDs across
them (`subDirsPerLocalDir`, default 64) so no single directory holds millions of entries and
I/O spreads across configured volumes. A shutdown hook deletes the tree; `deleteFilesOnStop`
governs whether the ESS-managed directories survive.

---

## 10. Dynamic Allocation (`ExecutorAllocationManager`)

A control loop, re-evaluated on a timer, per `ResourceProfile`:

- **Scale up** when there are pending or backlogged tasks for longer than
  `spark.dynamicAllocation.schedulerBacklogTimeout`, growing exponentially (1, 2, 4, 8…)
  on each subsequent `sustainedSchedulerBacklogTimeout`.
- **Scale down** an executor idle for `executorIdleTimeout`, unless it holds shuffle data
  needed by a running stage — in which case `cachedExecutorIdleTimeout` (default infinite)
  applies. This coupling is exactly why the External Shuffle Service (or
  `spark.dynamicAllocation.shuffleTracking.enabled`, or decommissioning-based migration) is
  a practical prerequisite for aggressive downscaling.
- Target = `max(pending + running tasks / tasksPerExecutor)` clamped to
  `[minExecutors, maxExecutors]`, with `initialExecutors` as the starting point.

`ExecutorAllocationClient` is the backend-facing interface (`requestTotalExecutors`,
`killExecutors`); YARN and K8s implement it differently but present the same contract.

---

## 11. Barrier Execution (`BarrierTaskContext`, `BarrierCoordinator`)

For gang-scheduled workloads (distributed deep learning via Horovod/TensorFlow):

- `rdd.barrier().mapPartitions(f)` marks the stage as a barrier stage.
- The TaskScheduler launches **all or none** of the tasks in one offer round.
- `BarrierTaskContext.barrier()` performs a global sync via `BarrierCoordinator` on the
  driver; `allGather` exchanges small messages between tasks.
- If any task fails, the whole stage restarts — there is no partial retry, because peer
  tasks hold state.
- Incompatible with dynamic allocation and with several RDD chain shapes; the DAGScheduler
  validates these up front (`checkBarrierStageWithRDDChainPattern`,
  `checkBarrierStageWithDynamicAllocation`, `checkBarrierStageWithNumSlots`) and fails fast
  with `BarrierJobAllocationFailed` rather than deadlocking.

---

## 12. Design Assessment

**Strong points**

- The event-loop design of the DAGScheduler makes an enormously stateful component
  *reasonable*: there is exactly one writer, so every invariant is a single-threaded
  invariant. The cost is a throughput ceiling and 4,900 lines in one file.
- Epoch fencing is a small, correct mechanism that neutralizes an entire class of
  out-of-order-event bugs.
- The three-way shuffle writer split and the `PackedRecordPointer`/page-table encodings are
  textbook systems engineering: measure what dominates, specialize for it, keep a general
  fallback.
- `HighlyCompressedMapStatus` and broadcasting of map statuses show a consistent instinct to
  keep driver memory O(1) in the shuffle dimensions.
- Cooperative spilling via `MemoryConsumer` is the right abstraction — operators negotiate
  rather than being killed.

**Weak points and tensions**

- **`DAGScheduler.scala` is doing too much.** Stage construction, failure handling, push-based
  shuffle coordination, resource-profile merging, barrier validation, pipelined-group
  admission, and (in this tree) test-only fault injection all live in one class. The
  fault-injection maps at lines 205–243 sitting in production code, with a long comment
  explaining why they can't be `null`, is a smell that the class has no room left.
- **Execution memory is never evicted for storage.** Documented and deliberate, but it makes
  caching behavior non-deterministic from the user's point of view.
- **Delay scheduling is heuristic on top of heuristic.** Two different reset semantics behind
  a legacy flag, tuned by a single `spark.locality.wait` knob that must serve both a 3-node
  and a 3,000-node cluster.
- **Shuffle remains the system's hard barrier.** Push-based shuffle, ESS, decommission
  migration, and pipelined shuffle are four separate mechanisms all working around the same
  structural fact. A more fundamental redesign (disaggregated shuffle storage as a
  first-class citizen, rather than `ShuffleDataIO` as a plugin point) is the obvious
  unfinished business.
