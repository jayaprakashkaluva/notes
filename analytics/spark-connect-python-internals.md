# Apache Spark — Spark Connect & Python Interop Internals

> Based on analysis of the Spark source tree at `C:\workspace\opensource\spark`, version
> **5.0.0-SNAPSHOT**, HEAD `ee11a92a9f1`.
> Companions: [spark-architecture.md](spark-architecture.md) ·
> [spark-core-internals.md](spark-core-internals.md) ·
> [spark-sql-catalyst-internals.md](spark-sql-catalyst-internals.md) ·
> [spark-streaming-internals.md](spark-streaming-internals.md)
>
> Code lives under `sql/connect/{common,server,client}`, `python/pyspark/`,
> `core/src/main/scala/org/apache/spark/api/python/`, and the newer
> `udf/worker/{proto,core,grpc}` modules.

---

## 1. Why Connect Exists

The classic architecture makes the **driver JVM** the API boundary: your application code
runs *inside* the driver, on the same classpath, in the same process. That has four
consequences that became untenable at scale:

1. **No isolation.** A user's dependency conflicts with Spark's; a user's `System.exit` kills
   the driver; one user's `collect()` OOMs everyone sharing the session.
2. **No thin clients.** Any client needs the full Spark JAR set and a compatible JVM.
3. **No upgrade decoupling.** Client and server versions are locked together.
4. **Poor interactivity.** A dropped network connection loses the query.

Spark Connect replaces the boundary with **gRPC + an unresolved logical plan expressed in
protobuf**. The client builds a plan; the server resolves, optimizes, executes it, and
streams results back as Arrow batches. The client never touches Catalyst.

The architectural enabler is the `sql/api` module split (see
[architecture §2.1](spark-architecture.md#21-the-sqlapi-split)): types, `Column`, `Row`,
encoders, and the `Dataset`/`SparkSession` *interfaces* live there with no Catalyst
dependency, so the Connect client can implement the same user-facing API against protobuf.

---

## 2. The Protocol (`sql/connect/common/src/main/protobuf/spark/connect/`)

Proto files: `base.proto` (service + requests), `relations.proto` (the plan tree),
`expressions.proto`, `types.proto`, `commands.proto`, `catalog.proto`, `ml.proto`,
`pipelines.proto`, `common.proto`.

### 2.1 Service surface (`base.proto:1314`)

```protobuf
service SparkConnectService {
  rpc ExecutePlan(ExecutePlanRequest)      returns (stream ExecutePlanResponse) {}
  rpc AnalyzePlan(AnalyzePlanRequest)      returns (AnalyzePlanResponse) {}
  rpc Config(ConfigRequest)                returns (ConfigResponse) {}
  rpc AddArtifacts(stream AddArtifactsRequest) returns (AddArtifactsResponse) {}
  rpc ArtifactStatus(ArtifactStatusesRequest)  returns (ArtifactStatusesResponse) {}
  rpc Interrupt(InterruptRequest)          returns (InterruptResponse) {}
  rpc ReattachExecute(ReattachExecuteRequest)  returns (stream ExecutePlanResponse) {}
  rpc ReleaseExecute(ReleaseExecuteRequest)    returns (ReleaseExecuteResponse) {}
  rpc ReleaseSession(ReleaseSessionRequest)    returns (ReleaseSessionResponse) {}
  rpc FetchErrorDetails(FetchErrorDetailsRequest) returns (FetchErrorDetailsResponse) {}
  rpc CloneSession(CloneSessionRequest)    returns (CloneSessionResponse) {}
  rpc GetStatus(GetStatusRequest)          returns (GetStatusResponse) {}
}
```

A `Plan` is either a `Relation` (a query, executed and streamed back) or a `Command`
(a side effect: write, DDL, register UDF, SQL with no result). `Relation` is a recursive
oneof over ~80 node kinds — `Read`, `Project`, `Filter`, `Join`, `Aggregate`, `Sort`,
`SetOp`, `Sample`, `Deduplicate`, `Range`, `LocalRelation`, plus the typed-Dataset,
pandas-API, ML, and streaming nodes.

Crucially, the transmitted plan is **unresolved**. The client does no name resolution, no
type checking, no optimization. It is a serialized DataFrame API call, not a plan in the
Catalyst sense.

`AnalyzePlan` covers everything that needs server knowledge without execution: `schema`,
`explain`, `isLocal`, `inputFiles`, `treeString`, `semanticHash`, `sameSemantics`. This is
why `df.schema` is a network round trip in Connect and is worth caching client-side.

### 2.2 Reattachable execution — the interesting part

A naive gRPC server streaming call dies with the connection. For a query that runs for
20 minutes over a flaky network, that is unacceptable. The protocol therefore makes
execution a **server-side resource with a resumable cursor**:

- `ExecutePlanRequest` carries `ReattachOptions.reattachable = true` and an
  `operation_id` (a client-generated UUID).
- Each `ExecutePlanResponse` carries a `response_id`.
- If the stream breaks, the client calls `ReattachExecute(operation_id, last_response_id)`
  and the server **resumes from the next response**, replaying from its cache.
- The stream is complete only when a `ResultComplete` message arrives. A stream that ends
  without it means "reattach for more" — an explicit, unambiguous end-of-stream marker
  rather than relying on transport semantics.
- `ReleaseExecute(operation_id, until_response_id)` lets the client acknowledge consumption
  so the server can free cached responses; releasing without an ID frees the whole execution.
- Non-reattachable executions are released automatically after the RPC returns.

This makes long-running queries survive load-balancer timeouts, proxy restarts, and laptop
sleep. It is the single most consequential design detail in Connect.

### 2.3 Error propagation

`FetchErrorDetails` returns the full server-side exception chain — error class, message
parameters, SQLSTATE, stack trace, and cause chain — so the client can reconstruct a
*typed* exception (`AnalysisException`, `ParseException`, `ArithmeticException`) rather than
a generic gRPC status. This depends entirely on the error-class framework described in
[architecture §9](spark-architecture.md#9-cross-cutting-concerns): because errors are
structured data rather than formatted strings, they survive a process boundary.

---

## 3. Server Implementation (`sql/connect/server/`)

### 3.1 Layering

```
 gRPC server (Netty)
   └─ interceptors: LoggingInterceptor, PreSharedKeyAuthenticationInterceptor,
                    RequestDecompressionInterceptor, LocalPropertiesCleanupInterceptor,
                    + user plugins via SparkConnectInterceptorRegistry
   └─ SparkConnectService  (one handler class per RPC)
        ├─ SparkConnectSessionManager   → SessionHolder   (per user+session UUID)
        └─ SparkConnectExecutionManager → ExecuteHolder   (per operation)
             └─ ExecuteThreadRunner (execution thread)
                  └─ SparkConnectPlanner: proto → Catalyst LogicalPlan
                  └─ SparkConnectPlanExecution: run + push Arrow batches
                       └─ ExecuteResponseObserver (cache)
                            └─ ExecuteGrpcResponseSender (consumer, attachable/detachable)
```

### 3.2 `SessionHolder`

Holds everything session-scoped: the `SparkSession` (with its own `SessionState`, temp
views, UDF registry, and SQL confs), the artifact manager and its isolated classloader,
cached DataFrames for the "DataFrame reference" mechanism, listener registrations, ML model
cache, and the streaming query cache. Sessions are keyed by `(userId, sessionId)` and expire
after `spark.connect.session.manager.defaultSessionTimeout` of inactivity.

`CloneSession` forks a session's state into a new session — useful for notebook branching
and for retry-with-modified-config without disturbing the original.

### 3.3 The producer/consumer split

This is the mechanism that makes reattach work, and it is worth reading directly:

> "This StreamObserver is running on the execution thread. Execution pushes responses to it,
> it caches them. ExecuteResponseGRPCSender is the consumer of the responses
> ExecuteResponseObserver 'produces'. It waits on the responseLock. New produced responses
> notify the responseLock. … A single ExecuteResponseGRPCSender can be attached to the
> ExecuteResponseObserver. Attaching a new one will notify an existing one that it was
> detached." — `execution/ExecuteResponseObserver.scala:36`

So: the execution thread never blocks on the network. It produces into a bounded cache
(`spark.connect.execute.reattachable.observerRetryBufferSize`); the gRPC sender drains it.
When the connection drops, the sender detaches and the *execution keeps running*. A
reattach attaches a new sender at the requested offset. Cached responses are trimmed on
`ReleaseExecute` or once the buffer limit is reached and the client has acknowledged past
them.

`ExecuteThreadRunner` documents its own state machine:

> `notStarted -> interrupted` · `notStarted -> started -> startedInterrupted -> completed`
> · `notStarted -> started -> completed`. "The thread can only be interrupted if the thread
> is in the startedInterrupted state." — `ExecuteThreadRunner.scala:346`

`SparkConnectExecutionManager` is the global registry and also the **abandoned-execution
reaper**: an execution with no attached consumer and no reattach for
`spark.connect.execute.reattachable.senderMaxStreamDuration` is interrupted and removed, so
a client that simply vanishes doesn't leak a running job forever.

### 3.4 `SparkConnectPlanner`

The proto→Catalyst translator: a large dispatch over the `Relation`/`Expression` oneofs
producing unresolved Catalyst nodes. `InvalidInputErrors` centralizes the rejection paths.
This is the trust boundary — every field arriving from a client is untrusted input, and the
planner is where malformed or malicious plans must be rejected rather than turned into
Catalyst nodes that blow up deeper in.

Result delivery (`SparkConnectPlanExecution`) collects the query as **Arrow record batches**
sized by `spark.connect.grpc.arrow.maxBatchSize`, streaming them as they are produced rather
than materializing the full result. Observed metrics
(`df.observe(...)`), execution progress (`ConnectProgressExecutionListener`), and the plan
are attached as additional response types on the same stream.

### 3.5 Artifacts

`AddArtifacts` streams JARs, Python files, `.zip`/`.whl` archives, and cached blobs to the
server; chunked for large files, single-batch for small ones, all CRC-checked.
`ArtifactStatus` lets the client skip re-uploading what's already there.

Server-side, `ArtifactManager` (`sql/core/.../sql/artifact/ArtifactManager.scala` — note it
lives in `sql/core`, not the Connect module, so the classic session can use it too) stores
them per session and builds an isolated `ClassLoader`. `JobArtifactSet` (in `core`) carries
the artifact set into task execution, so
executors resolve classes against the *session's* classloader. This is what makes two
sessions with conflicting library versions coexist in one Spark application — the isolation
property that the classic driver-embedded model could never provide.

### 3.6 Deployment shapes

- **Server plugin** — `SparkConnectPlugin` starts the gRPC server inside an existing
  Spark application via `spark.plugins`.
- **Standalone** — `SparkConnectServer` / `start-connect-server.sh`.
- **Local server** (`python/pyspark/sql/connect/local_server.py`, `local_server_pool.py`) —
  the client transparently spawns a local JVM server. HEAD's commit
  `[SPARK-58021][CONNECT] Add local server pool member claiming` adds pooling with claiming
  so repeated local sessions reuse a warm JVM instead of paying startup each time. This
  matters because it removes the last reason to use the classic local API for development.

---

## 4. Clients

### 4.1 Python (`python/pyspark/sql/connect/`)

Mirrors `pyspark.sql` module for module — `dataframe.py`, `column.py`, `functions/`,
`group.py`, `readwriter.py`, `catalog.py`, `streaming/`. `plan.py` builds the proto tree;
`client/` holds the gRPC channel, retry policy, and the reattach loop.

The split is not symmetric. `pyspark/sql/connect/` is a full parallel implementation;
`pyspark/sql/classic/` holds only the classes that needed a distinct classic variant
(`dataframe.py`, `column.py`, `window.py`, `table_arg.py`), with the rest of `pyspark.sql`
serving as the shared/classic surface. `SparkSession.builder` dispatches in
`pyspark/sql/session.py`: an `sc://` remote URL or `api_mode="connect"` selects
`pyspark.sql.connect.session.SparkSession`, otherwise the classic one.

The retry policy is worth noting: retries are classified by gRPC status and by whether the
operation is idempotent, with exponential backoff and jitter, and reattach is itself part of
the retry path rather than a separate mechanism.

### 4.2 JVM (`sql/connect/client/jvm/`)

Implements the `sql/api` `Dataset`/`SparkSession` interfaces against proto. Because the
interfaces are shared, most Scala user code compiles unchanged against either the classic or
the Connect implementation — the exceptions being anything that reaches for
`SparkContext`, `RDD`, or Catalyst internals, which is exactly the code Connect intends to
break.

`sql/connect/client/jdbc/` exposes a JDBC driver over Connect — a much lighter path than the
Hive Thrift Server.

---

## 5. Python Execution (Classic Path)

### 5.1 Driver-side bootstrap

`python/pyspark/java_gateway.py` launches `spark-submit` as a child process and connects to
it with **Py4J** over a local socket with a filesystem-delivered auth token. Every classic
`SparkContext`/`SparkSession` call is a Py4J reflective call into the JVM. This is why
classic PySpark driver code has noticeable per-call overhead, and why building a large
DataFrame plan in a loop is slow even before anything executes.

Connect replaces this entirely: no Py4J, no child JVM, no shared filesystem requirement.

### 5.2 Executor-side workers

Python UDFs run in **separate OS processes** on the executor:

```
 Executor JVM
   ├─ PythonWorkerFactory  ──fork/exec──▶  daemon.py  ──fork──▶  worker.py (one per task)
   └─ BasePythonRunner
        ├─ Writer thread : serialize input rows ──▶ socket ──▶ worker stdin
        └─ ReaderIterator: socket ◀── worker stdout ◀── serialized results
```

`daemon.py` is a pre-forking server: forking a worker from an already-imported daemon is far
cheaper than a fresh interpreter start. Workers are pooled and reused across tasks
(`spark.python.worker.reuse`), keyed by the Python exec and environment.

`BasePythonRunner` (`core/.../api/python/PythonRunner.scala:255`) runs the writer on a
separate thread from the reader so the JVM feeds and drains the worker concurrently rather
than lock-stepping. Memory for the worker is bounded by `spark.executor.pyspark.memory`,
enforced with `resource.setrlimit` inside the worker.

### 5.3 Eval types (`PythonEvalType`)

The `evalType` int selects the serialization contract and the worker-side loop:

| Range | Family | Examples |
|---|---|---|
| `100`–`106` | row/batched | `SQL_BATCHED_UDF` (100, pickle per row), `SQL_ARROW_BATCHED_UDF` (101), elementwise Arrow/pandas variants |
| `200`–`216` | pandas/Arrow grouped & scalar | `SQL_SCALAR_PANDAS_UDF` (200), `SQL_GROUPED_MAP_PANDAS_UDF` (201), `SQL_GROUPED_MAP_PANDAS_UDF_WITH_STATE` (208), `SQL_GROUPED_MAP_ARROW_UDF` (209), iterator variants (215, 216) |
| `300`+ | UDTFs, data sources, `transformWithState` in PySpark | |

The performance cliff is between 100 and everything else. `SQL_BATCHED_UDF` pickles one row
at a time; the Arrow/pandas types transfer columnar batches of
`spark.sql.execution.arrow.maxRecordsPerBatch` (10,000) rows with zero-copy on the Arrow
side. A pandas UDF is routinely an order of magnitude faster than a plain Python UDF for the
same logic, and `SQL_ARROW_BATCHED_UDF` (101) gives most of that win to unmodified
row-oriented UDF code by transporting with Arrow while keeping the per-row Python API.

Physical operators: `BatchEvalPythonExec`, `ArrowEvalPythonExec`, `FlatMapGroupsInPandasExec`,
`MapInPandasExec`, `AggregateInPandasExec`, `WindowInPandasExec`,
`TransformWithStateInPySparkExec` — all in `sql/core/.../execution/python/`.
`ExtractPythonUDFs` is the planner rule that pulls Python UDFs out of expressions into these
dedicated operators, because a Python UDF cannot participate in whole-stage codegen.

### 5.4 Arrow conversion

`toPandas()` / `createDataFrame(pandas_df)` go through Arrow when
`spark.sql.execution.arrow.pyspark.enabled` is on (default), with a fallback to the pickle
path on unsupported types. `ArrowConverters` (`sql/core/.../execution/arrow/`) does
`InternalRow` ↔ `ArrowRecordBatch`. Since `UnsafeRow` is already a columnar-friendly binary
format and Arrow is columnar, this conversion is far cheaper than round-tripping through
Python objects — though it is still a conversion, not a zero-copy reinterpretation.

### 5.5 Python data sources and UDTFs

`python/pyspark/sql/worker/` contains a set of single-purpose workers invoked at *planning*
time rather than execution time: `create_data_source.py`, `plan_data_source_read.py`,
`data_source_pushdown_filters.py`, `commit_data_source_write.py`, `analyze_udtf.py`,
`lookup_data_sources.py`. This lets a **data source written entirely in Python** participate
in DSv2 planning — including filter pushdown and partition planning — by calling back into
Python during query compilation. Same for polymorphic UDTFs, whose output schema is computed
by invoking Python's `analyze` method during analysis.

### 5.6 The `udf/worker` modules

New in this tree: `udf/worker/proto`, `udf/worker/core`, `udf/worker/grpc`. A protobuf- and
gRPC-based UDF worker protocol, separate from the ad-hoc socket protocol in
`PythonRunner`. The direction is clear — a versioned, language-agnostic, out-of-process UDF
contract that could host non-Python runtimes and decouple worker lifecycle from executor
internals.

### 5.7 pandas API on Spark (`python/pyspark/pandas/`)

A pandas-compatible API translating to Spark plans. The core abstraction is
`InternalFrame`, which maintains the mapping from pandas index/columns to Spark columns —
because pandas has an index and Spark does not. Operations that require positional
semantics (`iloc`, `sort_index`) require materializing a distributed sequence
(`DistributedSequenceID` / `ExtractDistributedSequenceID` in Catalyst), which is expensive
and is why `compute.default_index_type` exists as a tuning knob. `compute.ops_on_diff_frames`
gates operations that must join two frames on their indexes.

---

## 6. Design Assessment (Staff+ Lens)

**Strong**

- **The unresolved-plan-over-the-wire choice.** Sending an *unresolved* plan rather than a
  resolved one or SQL text is the right call: it keeps all catalog, resolution, and
  optimization logic server-side (so clients never need to track Catalyst semantics), while
  still letting the client express the full DataFrame API rather than being limited to what
  SQL can say.
- **Reattachable execution.** The `operation_id` + `response_id` + explicit `ResultComplete`
  design solves connection fragility properly instead of papering over it with retries. The
  producer/consumer split in `ExecuteResponseObserver` is the correct implementation: the
  execution thread is never coupled to the network.
- **Session-scoped artifact isolation.** Per-session classloaders finally deliver the
  multi-tenancy that the classic driver model structurally could not.
- **The `sql/api` extraction.** A large, unglamorous refactor that is the precondition for
  everything else. Being able to compile the same Scala user code against two entirely
  different implementations is a strong test of the abstraction.
- **Errors as structured data.** Error classes plus `FetchErrorDetails` mean the process
  boundary doesn't degrade exception quality — a common failure of RPC-ified APIs.

**Tensions**

- **Two implementations of the entire DataFrame API, per language.** `pyspark/sql/classic/`
  and `pyspark/sql/connect/`, plus the JVM pair. Every new function must be added in both,
  and behavioral drift is a persistent, low-grade tax. It is the unavoidable cost of a
  compatible migration, but it is real and it is large.
- **Chattiness.** `df.schema`, `df.columns`, and `df.explain` are network round trips.
  Idiomatic PySpark code that touches schema in a loop degrades badly. This is a client
  caching problem more than a protocol problem, but it surfaces as a Connect problem.
- **Protobuf schema evolution as a permanent constraint.** ~80 `Relation` node kinds, all of
  which must remain wire-compatible across versions. Every plan-shape change is now a
  protocol change with a compatibility review, which is precisely the discipline the project
  wanted — and precisely the friction it costs.
- **Python UDFs remain a process boundary.** Arrow batching narrowed the gap substantially,
  but a forked interpreter with a socket in the middle has a floor. The `udf/worker/grpc`
  work suggests this is being addressed structurally rather than incrementally, but it is
  early.
- **The classic path is not going away quickly.** RDD APIs, `SparkContext`, accumulators,
  broadcast variables, and anything reaching into Catalyst have no Connect equivalent by
  design. That leaves a long tail of workloads pinned to the classic driver, and therefore a
  long tail of dual maintenance.
