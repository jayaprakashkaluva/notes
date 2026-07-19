# Apache Flink — Architecture

> Based on analysis of the Flink source tree at `C:\workspace\opensource\flink` (version **2.4-SNAPSHOT**).
> Companion doc with data-plane / implementation deep-dives: [flink-internals.md](flink-internals.md)

---

## 1. What Flink Is

Apache Flink is a distributed **stateful stream processing engine**. Its core abstraction is an
unbounded (or bounded) stream of events flowing through a dataflow graph of operators, where each
operator can hold fault-tolerant, partitioned **state**. Batch processing is treated as a special
case of streaming (bounded streams), so one runtime serves both.

The three pillars of the design:

1. **Dataflow graph execution** — a user program is compiled into a DAG of parallel operators
   connected by data exchanges, deployed onto a cluster of worker processes.
2. **Local state + global snapshots** — operator state lives locally (heap or embedded RocksDB),
   and consistency/fault tolerance comes from asynchronous distributed snapshots
   (Chandy–Lamport-style checkpoints with barriers).
3. **Event time** — watermarks flow through the graph as a measure of event-time progress,
   decoupling correctness from wall-clock arrival order.

---

## 2. Repository / Module Map

| Module | Role |
|---|---|
| `flink-core`, `flink-core-api` | Foundational APIs: `Configuration`, filesystems, `TypeSerializer`/`TypeInformation`, `MemorySegment`, common function interfaces |
| `flink-runtime` | The heart of the engine: JobManager components, TaskExecutor, scheduler, network stack, checkpointing, state abstraction, REST layer. **Since Flink 2.x the streaming runtime (`StreamTask`, operators, `org.apache.flink.streaming.runtime.*`) also lives here** |
| `flink-streaming-java` | Classic DataStream API (`StreamExecutionEnvironment`, `DataStream`, transformations) |
| `flink-datastream-api`, `flink-datastream` | The new **DataStream V2** API (FLIP-408 line) and its implementation |
| `flink-rpc/*` | RPC abstraction (`flink-rpc-core`) and its Apache **Pekko**-based implementation (`flink-rpc-akka`, loaded via `flink-rpc-akka-loader` in an isolated classloader) |
| `flink-table/*` | SQL/Table API: parser, Calcite-based planner, code generation, SQL Client/Gateway, JDBC driver |
| `flink-state-backends/*` | `flink-statebackend-rocksdb`, `flink-statebackend-forst` (disaggregated state, Flink 2.x), `flink-statebackend-changelog`, heap-spillable |
| `flink-dstl` | "Durable Short-term Log" — the changelog storage behind the changelog state backend |
| `flink-clients` | Client-side job submission (`CliFrontend`, `PipelineExecutor`s) |
| `flink-kubernetes`, `flink-yarn` | Active resource-manager integrations |
| `flink-connectors`, `flink-formats` | Base connector APIs (most connectors are externalized to separate repos) |
| `flink-runtime-web` | Web UI (served by the JobManager REST endpoint) |
| `flink-metrics`, `flink-queryable-state`, `flink-python` | Metrics reporters, queryable state, PyFlink |

---

## 3. Process Architecture

A running Flink cluster consists of two kinds of JVM processes plus a client:

```
              ┌────────────────────────  JobManager process  ───────────────────────┐
              │                                                                     │
 Client ──────┼─► REST endpoint ─► Dispatcher ──spawns──► JobMaster (one per job)   │
 (submit job) │  (WebMonitor)      - receives/persists      - owns ExecutionGraph   │
              │                      JobGraphs              - Scheduler             │
              │                    - recovers jobs          - CheckpointCoordinator │
              │                                             - SlotPool              │
              │                     ResourceManager ◄──── slot requests ────┘       │
              │                     - SlotManager: matches slot requests            │
              │                       to TaskManager slots                          │
              └───────────────────────────▲─────────────────────────────────────────┘
                                          │ registration / heartbeats / slot offers
              ┌────────────────  TaskManager process(es)  ──▼───────────────────────┐
              │  TaskExecutor                                                       │
              │   - task slots (unit of resource scheduling)                        │
              │   - runs Tasks (one thread each)                                    │
              │   - NettyShuffleEnvironment (network data plane)                    │
              │   - state backends, memory manager                                  │
              └─────────────────────────────────────────────────────────────────────┘
```

### 3.1 JobManager-side components (all RPC endpoints in `flink-runtime`)

- **`Dispatcher`** (`runtime/dispatcher/Dispatcher.java`) — "responsible for receiving job
  submissions, persisting them, spawning JobManagers to execute the jobs and to recover them in
  case of a master failure." One per cluster; the entry point for job submission over REST.
- **`JobMaster`** (`runtime/jobmaster/JobMaster.java`) — one per job; drives the execution of a
  single `ExecutionPlan`/`JobGraph`. Owns the scheduler, the `ExecutionGraph`, the
  `CheckpointCoordinator`, and the `SlotPool` (the job-local pool of slots obtained from the
  ResourceManager). Receives `updateTaskExecutionState` calls from TaskExecutors.
- **`ResourceManager`** (`runtime/resourcemanager/`) — cluster-wide resource broker. Its
  `SlotManager` matches slot requests coming from JobMasters against slots registered by
  TaskExecutors; in *active* deployments (YARN/Kubernetes) it also starts/stops TaskManager
  containers/pods on demand (`ActiveResourceManager`, with implementations in `flink-yarn` and
  `flink-kubernetes`).

### 3.2 TaskManager side

- **`TaskExecutor`** (`runtime/taskexecutor/TaskExecutor.java`) — the worker RPC endpoint. It
  registers its **task slots** with the ResourceManager, offers slots to JobMasters, and executes
  `Task`s. Each slot is a fixed slice of the TaskManager's resources; slot sharing lets one slot
  host one parallel subtask of *each* operator of a job (so one slot can run a whole pipeline
  slice).
- **`Task`** (`runtime/taskmanager/Task.java`) — one execution of one parallel subtask, run by a
  dedicated thread. Per its Javadoc, a Task "wraps a Flink operator and runs it, providing all
  services necessary to consume input data, produce its results and communicate with the
  JobManager." Tasks know nothing about the rest of the graph — only their own code, config, and
  the IDs of intermediate results to consume/produce. (Deep dive in flink-internals.md §1–2.)

### 3.3 Heartbeats and liveness

`runtime/heartbeat/` implements a generic `HeartbeatManager` used pairwise between
JobMaster ↔ TaskExecutor, JobMaster ↔ ResourceManager, and TaskExecutor ↔ ResourceManager.
Missed heartbeats trigger disconnects, slot releases and task failover.

---

## 4. RPC System

`flink-rpc-core` defines the actor-flavored programming model; the transport is Apache **Pekko**
(the Akka fork), hidden behind `RpcSystem` and loaded through a separate classloader by
`flink-rpc-akka-loader` so Pekko/Scala never leak onto the user classpath.

Key concepts (`org.apache.flink.runtime.rpc`):

- **`RpcEndpoint`** — a component with its own single-threaded **main thread executor**. All state
  of an endpoint (Dispatcher, JobMaster, TaskExecutor, ResourceManager) is mutated only from its
  main thread — this is Flink's actor model: no locks in the control plane, just
  `runAsync`/`callAsync` onto the main thread. `MainThreadValidatorUtil` asserts this invariant.
- **`RpcGateway`** — a Java interface whose methods are proxied to remote calls; methods returning
  `CompletableFuture` are asks, `void` methods are fire-and-forget tells.
- **`FencedRpcEndpoint`** — endpoints whose calls carry a **fencing token** (e.g. `JobMasterId`,
  `DispatcherId` = UUIDs tied to leadership). Messages with a stale token are rejected — this is
  how split-brain from an old leader is prevented.

---

## 5. High Availability

`runtime/highavailability/` + `runtime/leaderelection/`:

- **Leader election** — `DefaultLeaderElectionService` drives a pluggable
  `LeaderElectionDriver`: `ZooKeeperLeaderElectionDriver` (Curator) or a Kubernetes
  ConfigMap-lease based driver (in `flink-kubernetes`). Components (`LeaderContender`s:
  Dispatcher, ResourceManager, REST endpoint, each JobMaster) are granted leadership together with
  a fencing UUID that becomes their fencing token.
- **Leader retrieval** — `runtime/leaderretrieval/` lets clients and TaskExecutors discover the
  current leader's address.
- **Persistence for recovery** — HA storage keeps: submitted `JobGraph`s (`JobGraphStore`),
  completed checkpoint metadata (`CompletedCheckpointStore`), and the checkpoint ID counter
  (`CheckpointIDCounter`). Actual state payloads live on a DFS; ZK/K8s only store pointers.
  On JobManager failover the new Dispatcher leader re-reads the JobGraphs and restarts each job
  from its latest completed checkpoint.

---

## 6. From Program to Running Job (the compilation pipeline)

Four graph layers, each a refinement of the previous:

```
 User program (DataStream / Table API / SQL)
   │  builds a list of Transformations
   ▼
 StreamGraph            (streaming/api/graph/StreamGraph.java, built by StreamGraphGenerator)
   │  logical graph: StreamNodes (operators) + StreamEdges (partitioners)
   ▼
 JobGraph               (runtime/jobgraph/, built by StreamingJobGraphGenerator)
   │  operator CHAINING applied → JobVertex per chain; the unit the Dispatcher accepts
   ▼
 ExecutionGraph         (runtime/executiongraph/, built inside the JobMaster)
   │  parallelized: ExecutionJobVertex → ExecutionVertex (per subtask) → Execution (per attempt)
   │  IntermediateResult(Partition)s model produced data
   ▼
 Physical deployment    (TaskDeploymentDescriptor → TaskExecutor.submitTask → Task thread)
```

Notable details:

- **Operator chaining** (`StreamingJobGraphGenerator`): consecutive operators with a forward
  connection, same parallelism, and compatible chaining strategies are fused into one `JobVertex`
  so records pass between them as method calls (no serialization, no network). At runtime the
  chain is materialized as an `OperatorChain` inside a single `StreamTask`.
- **Deterministic operator IDs**: `StreamGraphHasherV2` hashes the graph structure so that
  operator state can be matched to operators across job versions when restoring savepoints
  (user-overridable via `uid()` → `StreamGraphUserHashHasher`).
- **Incremental/adaptive graphs**: `AdaptiveGraphManager` +
  `StreamGraphOptimizer` (batch) allow the JobGraph to be generated **progressively** while the job
  runs, so downstream vertices can be optimized (e.g. join strategy, parallelism) using statistics
  from already-finished upstream stages.
- **`ExecutionPlan`** is the newer abstraction the Dispatcher/JobMaster accept — either a
  `JobGraph` or a `StreamGraph` (for progressive generation).

---

## 7. Scheduling

The scheduler lives behind the `SchedulerNG` interface (`runtime/scheduler/`); the JobMaster just
drives it. Three families:

### 7.1 `DefaultScheduler` (streaming + classic batch)

- Uses **pipelined-region scheduling**: the graph is decomposed into regions of tasks connected by
  pipelined edges; a region is scheduled as a unit once its blocking inputs are ready
  (`runtime/scheduler/strategy/PipelinedRegionSchedulingStrategy`).
- **Slot allocation**: `SlotSharingExecutionSlotAllocator` groups one subtask of each vertex in a
  slot-sharing group into an `ExecutionSlotSharingGroup` and maps each group to one physical slot
  from the JobMaster's `SlotPool`. The SlotPool requests slots from the ResourceManager, which
  directs TaskExecutors to offer them to the JobMaster.
- **Failover**: `RestartPipelinedRegionFailoverStrategy` restarts only the failover region(s)
  affected by a failure (everything connected through pipelined exchanges, transitively), guided
  by restart strategies (fixed-delay, failure-rate, exponential-delay) in
  `runtime/executiongraph/failover/`.

### 7.2 `AdaptiveScheduler` (streaming, reactive/elastic)

`runtime/scheduler/adaptive/` — an explicit **state machine** (the files *are* the states:
`Created → WaitingForResources → CreatingExecutionGraph → Executing ⇄ Restarting / Failing /
StopWithSavepoint → Finished`). It decides the parallelism *from the resources actually
available*, can run a job under-parallelized when slots are missing, and **rescales in place**
when TaskManagers join or leave (basis of Reactive Mode and in-place autoscaling; recent versions
support rescaling from local state without a full restart).

### 7.3 `AdaptiveBatchScheduler` (batch)

`runtime/scheduler/adaptivebatch/` — derives each vertex's parallelism at runtime from the actual
size of its inputs, supports **speculative execution** of slow tasks
(`runtime/scheduler/slowtaskdetector/`), and cooperates with the incremental JobGraph generation
mentioned above.

---

## 8. Deployment Modes & Entrypoints

`runtime/entrypoint/ClusterEntrypoint` is the JVM main-class skeleton for the JobManager process;
concrete subclasses per mode/resource provider:

- **Session cluster** — long-running cluster, many jobs share the Dispatcher/ResourceManager
  (`StandaloneSessionClusterEntrypoint`, `YarnSessionClusterEntrypoint`,
  `KubernetesSessionClusterEntrypoint`).
- **Application mode** — cluster is started *for* one application; the user `main()` runs on the
  JobManager (`ApplicationClusterEntryPoint` in `runtime/application` + client-side support in
  `flink-clients`). This is the recommended production mode (per-job mode was removed in 2.x).
- **MiniCluster** (`runtime/minicluster/`) — everything in one JVM; used by IDE execution and
  tests.

The client side (`flink-clients`) resolves a `PipelineExecutor` from config
(`local`, `remote`, `yarn-application`, `kubernetes-application`, ...) and hands the pipeline to
the cluster over REST.

**REST layer**: `runtime/rest/` implements a Netty-based HTTP server with typed
`MessageHeaders`/handler pairs (`webmonitor/`); it serves both the programmatic API and the Web UI
bundle from `flink-runtime-web`. Job submission, savepoint triggering, rescaling, metrics
scraping — everything external goes through REST (the binary RPC is internal-only).

---

## 9. Fault Tolerance Model (summary)

- **State snapshots**: the `CheckpointCoordinator` (in the JobMaster) periodically injects
  **checkpoint barriers** into the sources; barriers flow with the data and trigger asynchronous
  snapshots of each task's state to durable storage; the coordinator collects acks and finalizes a
  globally-consistent checkpoint. Exactly-once semantics come from barrier alignment (or unaligned
  checkpoints) plus transactional/idempotent sinks. Details in flink-internals.md §6.
- **Task failure**: restart the failover region from the last completed checkpoint.
- **JobManager failure**: HA services recover job graphs + checkpoint pointers; a new leader
  resumes.
- **TaskManager failure**: detected by heartbeat timeout; its tasks fail over as above; in active
  deployments the ResourceManager requests replacement containers.

---

## 10. Table / SQL Stack (brief)

`flink-table/`:

1. SQL text → `flink-sql-parser` (Calcite parser with Flink extensions) → SqlNode.
2. Validation + logical planning on **Apache Calcite** (`flink-table-planner`): RelNodes,
   rule-based logical optimization + cost-based physical planning; streaming-specific rules handle
   changelog semantics (insert/update/delete streams), watermark pushdown, mini-batching, and
   state-TTL-aware joins/aggregations.
3. Physical plan = `ExecNode` DAG → **Janino-compiled generated Java code** for expressions and
   operators → `Transformation`s → the same StreamGraph/JobGraph pipeline as DataStream jobs.
4. `flink-sql-gateway` + `flink-sql-client` provide the interactive/remote SQL surface;
   `flink-sql-jdbc-driver` speaks to the gateway.

---

*See [flink-internals.md](flink-internals.md) for: task execution & the mailbox threading model,
the network stack (credit-based flow control, back pressure, buffer debloating), memory
management, watermarks & timers, checkpointing internals (aligned/unaligned, channel state), and
the state backend implementations (heap, RocksDB, ForSt, changelog).*
