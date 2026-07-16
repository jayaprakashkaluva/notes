# Presto (prestodb) — Technical Architecture & Implementation Deep Dive

> **Audience:** Staff / Principal engineers evaluating, extending, or operating Presto at scale.
> **Codebase analyzed:** `prestodb/presto` @ `master` (v0.299-SNAPSHOT, ~124 Maven modules, Java 17 + C++20).
> **Package namespace:** `com.facebook.presto.*` (foundation libraries: `com.facebook.airlift.*`, `com.facebook.drift.*`).

---

## 1. Executive Summary

Presto is a **distributed, in-memory, pipelined MPP SQL query engine** designed for interactive and batch analytics over federated data sources (data lakes, RDBMS, NoSQL, streaming stores). Its defining architectural choices:

1. **Storage/compute separation via a Connector SPI** — the engine owns parsing, planning, scheduling, and execution; all data access is behind a plugin boundary (`presto-spi`). One query can join data across catalogs (e.g., Hive ⋈ MySQL ⋈ Kafka).
2. **Pipelined, streaming execution (no materialized shuffle by default)** — stages stream pages between each other over HTTP; the query is admitted as a whole into cluster memory. This gives low latency but historically weaker fault tolerance than Spark-style stage-checkpointing (addressed separately by Presto-on-Spark and recoverable grouped execution).
3. **A dual-runtime strategy** — the classic Java evaluation engine (`presto-main-base`) and **Prestissimo** (`presto-native-execution`), a C++ worker built on [Velox](https://velox-lib.io) that speaks the exact same coordinator↔worker protocol. The stated long-term direction (see `ARCHITECTURE.md`) is full migration of evaluation to native workers, with Java retained on the coordinator/control plane.
4. **Everything is a plugin** — connectors, functions, types, security, resource-group policies, session-property providers, event listeners, TTL fetchers, and even router scheduling policies load through `Plugin`/`CoordinatorPlugin` with classloader isolation.

---

## 2. Repository & Module Topology

~124 Maven modules under a single reactor (`pom.xml`, artifact `presto-root`). The important layering, from most-depended-upon outward:

```
┌────────────────────────────────────────────────────────────────────┐
│  presto-common          Types, Block/Page (columnar data), config  │  ← shared by client+server+SPI; zero engine deps
├────────────────────────────────────────────────────────────────────┤
│  presto-spi             Connector/Plugin/security/eventing SPI     │  ← the plugin ABI; extreme back-compat discipline
├────────────────────────────────────────────────────────────────────┤
│  presto-parser          ANTLR4 grammar → AST (Statement tree)      │
│  presto-analyzer        Semantic analysis abstractions             │
│  presto-expressions     RowExpression IR utilities                 │
│  presto-matching        Pattern-matching DSL for optimizer rules   │
│  presto-bytecode        Runtime ASM bytecode generation DSL        │
│  presto-memory-context  Hierarchical memory accounting             │
├────────────────────────────────────────────────────────────────────┤
│  presto-main-base       THE ENGINE: analyzer, planner/optimizer,   │
│                         metadata, scheduler, task execution,       │
│                         operators, memory mgmt, HTTP endpoints     │
│  presto-main            Thin assembly on top of main-base          │
│  presto-server          RPM/tar.gz server packaging (launcher)     │
├────────────────────────────────────────────────────────────────────┤
│  presto-client          REST protocol client (StatementClient)     │
│  presto-cli / presto-jdbc                                          │
├────────────────────────────────────────────────────────────────────┤
│  Connectors: presto-hive, -iceberg, -delta, -hudi, -mysql,         │
│  -postgresql, -kafka, -cassandra, -mongodb, -elasticsearch,        │
│  -pinot, -druid, -kudu, -bigquery, -clickhouse, -lance, ...        │
│  presto-base-jdbc / presto-base-arrow-flight (connector toolkits)  │
├────────────────────────────────────────────────────────────────────┤
│  Alternate runtimes / control planes:                              │
│  presto-native-execution (C++/Velox "Prestissimo" worker)          │
│  presto-native-sidecar-plugin (Java coordinator ↔ native sidecar)  │
│  presto-spark-* (Presto-on-Spark: engine as a Spark job)           │
│  presto-router (query gateway / load balancer, pluggable)          │
├────────────────────────────────────────────────────────────────────┤
│  File formats: presto-orc, presto-parquet, presto-rcfile           │
│  (engine-owned readers/writers — a deliberate vertical-            │
│   integration choice for predicate/lazy-read performance)          │
└────────────────────────────────────────────────────────────────────┘
```

**Key layering invariants:**

- `presto-common` and `presto-spi` must not depend on engine internals; connectors compile only against these (plus toolkits). SPI evolution is additive; breaking changes are a big deal because third-party plugins bind to it.
- `presto-main-base` vs `presto-main` is a relatively recent split to separate the engine library from its assembly, easing reuse (e.g., Presto-on-Spark and tests link the base without dragging server packaging).
- File-format readers (ORC/Parquet) are **engine modules, not connector internals** — they implement lazy materialization directly into engine `Block`s, predicate pushdown into stripe/row-group pruning, and dictionary-aware filtering. This vertical integration is a core performance decision.
- Foundation is **Airlift** (`com.facebook.airlift`): Guice `Bootstrap` DI, JAX-RS HTTP server, JSON codecs, discovery, config binding (`@Config` setters), stats/JMX. Every server process is an Airlift app assembled from Guice modules (`ServerMainModule`, `CoordinatorModule`, `WorkerModule`).

---

## 3. Process & Cluster Architecture

### 3.1 Server roles

One binary, role selected by config (`coordinator=true|false`, `node-scheduler.include-coordinator`, `resource-manager-enabled`, etc.):

| Role | Responsibility |
|---|---|
| **Coordinator** | Client protocol endpoint, parse/analyze/plan/optimize, admission (queueing), scheduling, cluster metadata, transaction management |
| **Worker** | Task execution: scan/filter/agg/join operators, exchanges, spill |
| **Resource Manager** (optional) | Disaggregated-coordinator mode: aggregates cluster load + queue state across *multiple* coordinators (thrift-based `ResourceManagerServer`), enabling coordinator horizontal scaling and HA |
| **Catalog Server** (optional) | Offloads catalog metadata serving (`presto-main-base/.../catalogserver`) |
| **Router** (`presto-router`) | Standalone query gateway in front of N clusters; pluggable scheduling (round-robin, weighted, custom via `presto-router-example-plugin-scheduler`; `presto-plan-checker-router-plugin` routes based on native-engine plan compatibility) |
| **Sidecar** (native) | A Prestissimo process paired with a Java coordinator that serves function metadata/session properties/expression optimization for the native evaluation engine (`presto-native-sidecar-plugin`) |

Membership/liveness: Airlift **discovery service** (typically embedded in coordinator); workers heartbeat and announce; `failureDetector` package tracks suspect nodes; `presto-node-ttl-fetchers` + `presto-cluster-ttl-providers` integrate scheduled-instance TTL (e.g., spot instances) into scheduling decisions.

### 3.2 Query lifecycle (control plane)

```
Client ──POST /v1/statement──▶ Coordinator
  1. DispatchManager (dispatcher/) — create QueryId, session (SessionPropertyManager,
     SessionPropertyDefaults), queue via ResourceGroupManager (admission control)
  2. Parse: presto-parser (ANTLR4, SqlParser) → AST `Statement`
  3. Analyze: Analyzer → Analysis (name resolution, type checking, coercions,
     view expansion, MV rewrite, lambda/subquery analysis) against Metadata SPI
  4. Plan: LogicalPlanner → PlanNode DAG (IR), initially TableScan/Filter/Project/
     Aggregation/Join/... with `VariableReferenceExpression`s
  5. Optimize: Optimizer runs PlanOptimizers list — a mix of legacy whole-plan
     visitors and the IterativeOptimizer (rule/pattern-based, memo/Lookup,
     presto-matching) with cost comparator (CBO) + HBO
  6. Fragment: PlanFragmenter cuts the plan at exchange boundaries into
     PlanFragments (each = a stage), assigning PartitioningScheme
  7. Schedule: SqlQueryScheduler → per-section SectionExecution → SqlStageExecution
     → HttpRemoteTask instances placed by NodeScheduler/NodePartitioningManager;
     splits streamed from ConnectorSplitManager via SplitSource
  8. Execute: workers run SqlTaskExecution; results stream back stage-by-stage;
     final stage buffered by the coordinator and paged to client via nextUri
```

Every entity is governed by an explicit **state machine** (`QueryStateMachine`, `StageExecutionStateMachine`, `TaskStateMachine`) with listener-driven transitions — the codebase is heavily async (`ListenableFuture` everywhere, no blocking RPC in control paths).

### 3.3 Client protocol

`presto-client` implements the REST paging protocol: `POST /v1/statement` returns JSON with `nextUri`; the client long-polls `GET nextUri` receiving incremental result batches + query stats; `DELETE` cancels. Results are JSON (or binary with `presto-common-arrow`/Arrow Flight work in progress via `presto-flight-shim`). JDBC/CLI/UI (`presto-ui`, React) all sit on this. Session state (catalog/schema/session properties/transaction id/prepared statements) round-trips via HTTP headers — the protocol is stateless per request, which is what makes router/gateway insertion trivial.

---

## 4. Data Plane: Pages, Blocks, Types

The columnar in-memory format lives in **`presto-common`** (shared with SPI so connectors produce engine-native data with zero conversion):

- **`Block`** — immutable columnar vector for one column: primitive fixed-width (`LongArrayBlock`, `IntArrayBlock`...), variable-width (`VariableWidthBlock` with offsets + slice), and *composable wrappers*: `DictionaryBlock` (indices into a dictionary — preserved through operators to defer materialization), `RunLengthEncodedBlock` (RLE constants), nested `ArrayBlock`/`MapBlock`/`RowBlock`. Null handling via boolean valueIsNull arrays.
- **`Page`** — a horizontal batch of Blocks (all same positionCount). The unit of data flow between operators and over the wire (serialized + optionally compressed/encrypted as `SerializedPage`).
- **Type system** — `Type` implementations (also in `presto-common`) define physical representation (`javaType`: long/double/Slice/Block) + read/write into blocks. Parametric types (DECIMAL, VARCHAR(n), ARRAY, MAP, ROW) are first-class; semantic types layered above.
- **`TupleDomain<T>` / `Domain`** — the universal predicate lattice (per-column value sets/ranges) used for pushdown negotiation between optimizer and connectors, partition pruning, and file-format filtering. It is deliberately *lossy but sound* (supersets), so pushdown never changes results — the engine can always re-apply the residual filter.

**Memory discipline:** every Block reports `getRetainedSizeInBytes()`; all operator allocations are accounted through **`presto-memory-context`** — a tree of `AggregatedMemoryContext`/`LocalMemoryContext` rolling up operator → driver → pipeline → task → query → pool. This user-space accounting (not GC-based) is what enables deterministic admission control, per-query kill decisions (`ClusterMemoryManager`, OOM killer policies), and spill triggering.

---

## 5. Optimizer

Located in `presto-main-base/.../sql/planner`. Two coexisting frameworks:

1. **Legacy `PlanOptimizer`s** — whole-tree visitors run in a fixed sequence (`PlanOptimizers` defines the ordered list, ~60+ passes). Examples: `PredicatePushDown`, `UnaliasSymbolReferences`, `HashGenerationOptimizer` (injects precomputed hash columns), `AddExchanges` (the *physical* distribution decider), `AddLocalExchanges` (intra-node parallelism), `IndexJoinOptimizer`, `ApplyConnectorOptimization` (delegates plan subtrees to `ConnectorPlanOptimizer` for connector-specific rewrites — this is how Pinot/Druid/JDBC aggregate pushdown works).
2. **`IterativeOptimizer`** — Cascades-lite: a `Memo` of plan groups, `Rule<T>`s matched via the `presto-matching` pattern DSL, applied to fixpoint with a cost comparator. CBO rules include join reordering (`ReorderJoins`, bounded exhaustive up to `optimizer.max-reordered-joins`) and join distribution selection (`DetermineJoinDistributionType`: PARTITIONED vs REPLICATED/broadcast).

**Statistics & cost:**
- `StatsCalculator` derives per-node row counts/NDV/nullFraction from connector-provided `TableStatistics` (`ConnectorMetadata.getTableStatistics`) through scalar-expression stat propagation; `CostCalculator` converts to CPU/memory/network cost (`cost/` package).
- **HBO (History-Based Optimization)** — `HistoricalStatisticsEquivalentPlanMarkingOptimizer` canonicalizes plans, and prior executions' actual runtime stats (served by a pluggable store, e.g. `redis-hbo-provider`) override estimates for recurring queries. This is a major Meta-driven differentiator for batch reliability (right-sizing join strategies/partition counts from history instead of error-prone estimates).
- Runtime adaptivity is limited by design (pipelined execution); Presto-on-Spark and grouped execution carry the adaptive/batch load instead.

**Physical properties framework:** `ActualProperties`/`StreamPropertyDerivations` track partitioning, ordering, grouping and constant-ness through the plan; `AddExchanges` inserts remote exchanges only where required properties aren't already satisfied (classic property-based optimization à la Volcano). `PartitioningHandle` abstracts *how* data is partitioned — including **connector-defined partitioning** (e.g., Hive bucketing) so the engine can do bucket-aligned joins with zero shuffle, and materialized-CTE temporary tables (`CteProjectionAndPredicatePushDown`, temp-table utilities) for batch CTE reuse.

**Plan IR notes:** expressions are progressively lowered from AST `Expression` to **`RowExpression`** (typed, function-handle-resolved IR in `presto-expressions`/SPI) — all post-analysis optimizer rules and execution operate on RowExpression; this is also what enables shipping expression evaluation to the native sidecar for Prestissimo clusters.

---

## 6. Scheduling & Distributed Execution

### 6.1 Fragmentation and stages

`PlanFragmenter` cuts at `ExchangeNode` boundaries → `SubPlan` tree of `PlanFragment`s. Each fragment declares:
- a **partitioning handle** (SOURCE_DISTRIBUTED for leaf scan stages; FIXED_HASH_DISTRIBUTION; SINGLE for final aggregation/output; SCALED_WRITER; COORDINATOR_ONLY for DDL),
- an **output partitioning scheme** (how its output pages are bucketed for the consumer stage).

### 6.2 Scheduler

`SqlQueryScheduler` splits the stage tree into **sections** (subtrees separated by *materializing* exchanges, relevant for recoverable execution) → `SectionExecution` → `SqlStageExecution` per stage. Placement:
- **`NodeScheduler`/`NodeSelector`** — leaf-stage split→node assignment honoring locality hints (`ConnectorSplit.getPreferredNodes`), max-splits-per-node backpressure, topology-aware selection, and TTL-aware filtering.
- **`NodePartitioningManager`** — for hash-distributed stages, builds the bucket→node map (either system round-robin or connector-supplied, e.g. Hive bucket mapping).
- Splits are pulled **lazily and asynchronously** from `SplitSource` (batched `getNextBatch`) so planning-time metadata calls don't block scheduling; `SqlStageExecution` drip-feeds splits to tasks based on queue depth (dynamic split assignment).

Tasks are managed via **`HttpRemoteTask`**: task creation/update is an idempotent POST of the fragment + split assignments + output buffer configuration; continuous long-poll GETs stream `TaskStatus`/`TaskInfo` back. Internal RPC is JSON-over-HTTP by default, with **Thrift (Drift)** codecs available for hot paths (task status/info) — `presto-internal-communication`, `thrift/` packages. SMILE binary JSON is also supported.

### 6.3 Worker execution model

On each worker, `SqlTaskManager` hosts `SqlTask` → `SqlTaskExecution`:

```
PlanFragment → LocalExecutionPlanner → pipelines of OperatorFactories
Task
 ├─ Pipeline 0:  TableScanOperator → FilterAndProject → PartialAgg → LocalExchangeSink
 ├─ Pipeline 1:  LocalExchangeSource → FinalAgg → TaskOutputOperator
 └─ each pipeline × driverInstances = Drivers (one thread-slice unit each)
```

- **`Driver`** = a chain of `Operator`s pulling `Page`s (`getOutput`/`addInput`), one split per leaf driver.
- **`TaskExecutor`** implements **cooperative time-slicing**: a fixed pool of threads runs `PrioritizedSplitRunner`s for ~1s quanta with multilevel feedback queues (short-running queries get priority — this is the mechanism behind Presto's interactive latency fairness under mixed workloads). Operators must be non-blocking; anything waiting (exchange data, memory, IO) returns a `ListenableFuture` blocking the driver, which is descheduled — no thread is ever parked on IO.
- **Local exchange** (`LocalExchange`) re-partitions between pipelines inside a task (e.g., N scan drivers → M agg drivers).
- **Remote exchange:** producer side buffers pages in `OutputBuffer` (partitioned/broadcast/arbitrary flavors) with client-acknowledged, position-based paging; consumer side `ExchangeClient` concurrently pulls from all upstream tasks with memory-bounded prefetch. Backpressure is end-to-end: full output buffers block producing drivers; slow consumers throttle the whole pipeline.
- **Code generation:** `presto-bytecode` (ASM DSL) JIT-compiles filter/projection expressions, joins' hash-lookup loops (`JoinCompiler`), and ordering comparators into specialized classes — avoiding interpreter overhead and enabling JVM inlining. Expression evaluation also has a constant-folding interpreter (`InterpretedFunctionInvoker`).
- **Joins:** partitioned hash join builds `PagesIndex`→`LookupSource` per driver, shared via `LookupSourceFactory`; broadcast joins replicate build side to all nodes. Index joins (`ConnectorIndex`) support key-lookup sources.
- **Spill** (`spiller/` package): hash agg, join build, order-by, and window can spill to local disk when memory contexts hit revocation — a `MemoryRevokingScheduler` requests revocable-memory release; operators serialize state and merge on unspill. Prestissimo delegates the equivalent to Velox's spilling.
- **Grouped / recoverable execution:** for bucketed tables, execution can proceed bucket-group-by-bucket-group (bounding memory to a group at a time), and with materialized exchanges enables **per-section retry** — the precursor of full batch fault tolerance (which Presto-on-Spark provides more completely).

### 6.4 Memory management (cluster level)

Per-query memory tracked in pools (`memory/` package): user vs system vs revocable; `ClusterMemoryManager` on the coordinator aggregates worker pool snapshots, enforces `query.max-memory`/`query.max-total-memory`, and runs a pluggable **low-memory killer** policy (kill largest query / progress-blocking query) on cluster OOM. Admission (resource groups) gates on cluster memory before dispatch.

---

## 7. Extension Architecture (SPI)

`presto-spi` is the contract; plugins are loaded from `plugin/` dirs each in an **isolated classloader** (parent-last except for SPI/common classes) — the reason connectors can ship conflicting Guava/Jackson versions. `Plugin` (worker+coordinator) and `CoordinatorPlugin` (coordinator-only, used by native sidecar function providers) are the entry points.

**Connector SPI (the big one)** — a connector supplies factories for:

| Interface | Role |
|---|---|
| `ConnectorMetadata` | Schemas/tables/columns, table layouts, statistics, DDL/DML handles, view/MV definitions |
| `ConnectorSplitManager` | Split enumeration (`ConnectorSplitSource`) with pushed-down layout/predicate |
| `ConnectorPageSourceProvider` / `ConnectorPageSinkProvider` | Read/write `Page`s for a split (columnar, lazy blocks supported) |
| `ConnectorNodePartitioningProvider` | Expose storage bucketing to the engine (co-located joins, grouped execution) |
| `ConnectorPlanOptimizer` | Connector-side plan rewrites (aggregate/limit/topN pushdown into e.g. Pinot/JDBC) |
| `ConnectorAccessControl`, procedures, session properties, `ConnectorIndex`... | |

The **table layout** mechanism (`getTableLayouts` with `Constraint<ColumnHandle>`) is how predicate pushdown is negotiated: connector returns (possibly narrowed) layouts + *unenforced* residual predicate; optimizer keeps the residual filter in-plan. Handles (`ConnectorTableHandle`, `ColumnHandle`, `ConnectorSplit`) are opaque JSON-serializable tokens — the engine round-trips them without interpretation (and with Thrift/custom serde support via `ConnectorTypeSerde` for native workers).

**Other SPI families:** `SystemAccessControl` (file/ranger-style), `PasswordAuthenticator`/JWT/Kerberos, `EventListener` (query completion events → `presto-openlineage-event-listener`, custom loggers), `ResourceGroupConfigurationManager` (static file / DB-backed in `presto-resource-group-managers`), session property defaults (`presto-db-session-property-manager`), `TypeManager` extensions, `BlockEncoding`s, and **function namespaces**.

**Function architecture** deserves a callout: `FunctionAndTypeManager` resolves functions through pluggable **`FunctionNamespaceManager`s** (`presto-function-namespace-managers`) — built-in namespace (`presto.default.*`) plus remote namespaces where UDFs execute out-of-process via Thrift (`presto-thrift-testing-udf-server`) or gRPC, or in a dedicated **function server** (`presto-function-server`). SQL-invoked functions (`presto-sql-helpers`) define functions in SQL that inline into plans. Built-in functions register via annotation-driven reflection (`@ScalarFunction`, `@SqlType`, `@AggregationFunction` with generated accumulator state) and are compiled with `presto-bytecode`.

---

## 8. Prestissimo: Native Execution (`presto-native-execution`)

The strategic bet (per `ARCHITECTURE.md`): Java coordinator + **C++ workers on Velox**.

- `presto_cpp` implements the *entire worker protocol surface*: task create/update/status/results HTTP endpoints, `SerializedPage` wire format, memory/spill config, announcement — so the Java coordinator schedules native workers identically to Java ones (a clean seam: the protocol, not the code, is the contract).
- Plan fragments arrive as JSON-serialized `PlanNode`/`RowExpression` IR and are translated into Velox plans (`presto_cpp/main/types` translators). Velox provides vectorized evaluation, columnar operators, memory arenas, and spilling.
- **Sidecar model** (`presto-native-sidecar-plugin`, `etc_sidecar/`): because the native engine's function catalog/semantics differ from Java built-ins, a sidecar native process serves function metadata, session property definitions, and expression optimization to the coordinator (via `CoordinatorPlugin`), keeping planning consistent with native evaluation semantics. `presto-plan-checker-router-plugin` lets a router pre-flight whether a query is native-compatible and route accordingly — the operational migration path for mixed fleets.
- Worker config supports GFlags passthrough; build via CMake/Makefile with optional adapters (S3, HDFS, GCS...). Functional parity is tracked by `presto-native-tests`.

**Implication for extension authors:** connector `ConnectorPageSourceProvider` Java code does *not* run on native workers — native clusters need Velox-side connectors (Hive/Iceberg/TPCH exist); Java-only connectors remain coordinator-reachable only for metadata or unsupported. This asymmetry is the central migration constraint.

## 9. Presto-on-Spark (`presto-spark-*`)

Runs Presto's planner + operators *inside* Spark executors for very large batch ETL: Spark provides shuffle materialization, retries, and speculative execution; Presto provides SQL semantics, functions, and operator performance. Notable engineering: `presto-spark-classloader-interface` is the only module loaded in Spark's classloader — everything else is reflectively loaded in an isolated classloader to survive Spark's dependency hell (an instructive pattern for embedding one large system in another). Fragments become RDD stages; exchanges become Spark shuffles.

---

## 10. Cross-Cutting Concerns

- **Transactions:** `TransactionManager` coordinates per-query (auto-commit) or explicit transactions; each connector enlists via `ConnectorTransactionHandle` with declared isolation. Metadata operations are transaction-scoped; distributed writes commit via `ConnectorPageSink` finish + metadata `finishInsert/finishCreateTable` (single-coordinator commit point).
- **Security:** authentication (password/LDAP, Kerberos, JWT, certificate), authorization layered as `AccessControlManager` → system access control + per-connector access control; row filters/column masks supported in analysis. Internal cluster auth via shared-secret / certificates.
- **Resource groups:** hierarchical admission trees (concurrency + memory + queue limits, weighted/fair scheduling policies, per-user/source selectors), configurable static (JSON) or dynamic (DB-backed with reload).
- **Observability:** `QueryStats`/`TaskStats`/operator-level stats flow through `TaskInfo` continuously (the UI's live query view is just this data); `EventListener` emits lifecycle events (+ OpenLineage integration); `presto-open-telemetry` adds tracing spans; JMX beans everywhere (also queryable via the `jmx` connector — the engine introspects itself with SQL).
- **Verification & testing:** `presto-verifier` replays production query pairs against two clusters and diffs results (checksum-based) — the guardrail for engine upgrades. `presto-tests` has full in-process multi-node `DistributedQueryRunner`s; `presto-product-tests` run dockerized environments (Tempto); TPC-H/TPC-DS connectors generate data in-engine for benchmarks (`presto-benchmark`, `presto-benchto-benchmarks`).

---

## 11. Design Assessment (Staff+ Lens)

**Strengths / load-bearing decisions**
1. **The SPI boundary is the product.** Presto's federation moat comes from a disciplined, additive-only SPI with classloader isolation; the `TupleDomain` + table-layout negotiation is a sound (never wrong, sometimes suboptimal) pushdown contract that has scaled to ~40 connectors in-tree.
2. **Protocol-as-contract enabled the native pivot.** Because coordinator↔worker interaction is a versioned HTTP/JSON(+Thrift) protocol rather than shared Java objects, an entirely different runtime (C++/Velox) could be swapped under an unchanged control plane — the highest-leverage architectural decision in the codebase's last five years.
3. **Cooperative scheduling + user-space memory accounting** give multi-tenant fairness and deterministic OOM behavior that thread-per-task engines struggle to match.
4. **Async-everything control plane** (state machines + ListenableFutures + long-poll) keeps a single coordinator driving tens of thousands of concurrent tasks.

**Structural tensions to be aware of**
1. **Coordinator is still the scalability bottleneck** for split-heavy queries (split enumeration, task status fan-in, final-stage results all funnel through it); disaggregated coordinators + resource managers mitigate cluster-level but not per-query limits.
2. **Two optimizer frameworks** (sequential visitors + iterative memo rules) coexist; ordering-sensitive passes make optimizer changes risky — HBO exists partly because estimate-based CBO is fragile at Meta scale.
3. **Dual-runtime parity tax:** every function/operator/semantic behavior now needs Java + Velox parity (see sidecar machinery, plan-checker router, native test suites) — a deliberate, temporary cost of the migration strategy.
4. **Pipelined execution ties fault tolerance to query granularity** — a worker loss kills in-flight queries unless grouped/recoverable execution or Presto-on-Spark is used; this is the fundamental latency-vs-recoverability trade the architecture accepts.

**Where to start reading code**
`SqlQueryExecution.start()` (the whole lifecycle in one file), `PlanOptimizers` (optimizer sequence), `AddExchanges` (distribution decisions), `SqlTaskExecution` + `Driver` + `TaskExecutor` (execution core), `HiveMetadata`/`HivePageSourceProvider` (canonical connector), `presto_cpp/main/TaskManager.cpp` (native worker entry).

---

*Generated from repository analysis on 2026-07-15; version references: `presto-root` 0.299-SNAPSHOT, Java 17 toolchain.*
