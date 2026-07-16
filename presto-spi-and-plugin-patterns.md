# Presto SPI — Service Provider Interface & Related Design Patterns

> Companion to [presto-technical-architecture.md](presto-technical-architecture.md).
> Source of truth: `presto-spi` module @ master (0.299-SNAPSHOT), package `com.facebook.presto.spi.*`.

---

## 1. What "SPI" Means

**SPI = Service Provider Interface.** It is the mirror image of an API:

| | API | SPI |
|---|---|---|
| Who defines the interface | The library | The framework |
| Who implements it | The library | **You** (the provider) |
| Who calls it | You | **The framework** |
| Direction of control | You → library | Framework → your code ("don't call us, we'll call you") |

This is the **Dependency Inversion Principle** applied at system scale: the Presto engine (high-level policy — SQL semantics, planning, scheduling) does not depend on any concrete data source (low-level detail). Both depend on the abstraction in the middle: the `presto-spi` module. Hive, Iceberg, MySQL, Kafka are all *providers* behind that abstraction, and the engine is written entirely against it.

The term comes from the JDK itself (JDBC drivers, charset providers, `java.util.ServiceLoader`), and Presto uses the JDK mechanism directly: every plugin jar declares its entry point in `META-INF/services/com.facebook.presto.spi.Plugin`, and Presto's `PluginManager` discovers it with `ServiceLoader`.

**Why Presto is architected this way** — three concrete payoffs:

1. **Federation.** The engine can join Hive ⋈ MySQL ⋈ Kafka in one query because all three look identical through the SPI.
2. **Independent shipping.** Connector authors compile against `presto-spi` + `presto-common` only — never engine internals — so plugins build and release outside the engine's tree and cadence.
3. **Stability boundary.** The SPI is the compatibility contract. Engine internals refactor freely (e.g., the `presto-main` → `presto-main-base` split, the C++ Prestissimo worker) without breaking the ~40 in-tree and countless private connectors.

---

## 2. The SPI Surface: `Plugin` and `CoordinatorPlugin`

A plugin is a jar (or directory of jars) whose entry class implements one of two interfaces. Both consist **entirely of `default` methods returning empty collections** — a deliberate evolution mechanism (see §6.7): a new extension point is a new default method, and every existing plugin keeps compiling.

### 2.1 `Plugin` — loaded on coordinator *and* workers

From `com.facebook.presto.spi.Plugin` (verified against source), grouped by concern:

| Extension point | Method(s) | What you provide |
|---|---|---|
| **Connectors** | `getConnectorFactories()` | Data sources (the flagship SPI, §4) |
| **Types & encodings** | `getTypes()`, `getParametricTypes()`, `getBlockEncodings()` | Custom SQL types + their columnar wire format |
| **Functions** | `getFunctions()`, `getSqlInvokedFunctions()` | Built-in-style UDFs (annotation-driven classes) |
| **Function namespaces** | `getFunctionNamespaceManagerFactories()` | Remote/managed UDF catalogs (Thrift/gRPC/function-server) |
| **Authentication** | `getPasswordAuthenticatorFactories()`, `getPrestoAuthenticatorFactories()` | LDAP, custom credential validation |
| **Authorization** | `getSystemAccessControlFactories()` | Ranger-style global access control |
| **Eventing** | `getEventListenerFactories()` | Query lifecycle events → audit/lineage (e.g. OpenLineage) |
| **Admission control** | `getResourceGroupConfigurationManagerFactories()`, `getQueryPrerequisitesFactories()` | Queue policies, pre-dispatch gates |
| **Session defaults** | `getSessionPropertyConfigurationManagerFactories()` | Per-user/source session property injection |
| **Spill/temp storage** | `getTempStorageFactories()` | Where exchange/spill temp data lands |
| **Elasticity** | `getNodeTtlFetcherFactories()`, `getClusterTtlProviderFactories()`, `getNodeStatusNotificationProviderFactory()` | Spot-instance TTL & node-death signals into the scheduler |
| **Optimizer history** | `getHistoryBasedPlanStatisticsProviders()` | HBO stats stores (e.g. `redis-hbo-provider`) |
| **Tracing / analysis** | `getTracerProviders()`, `getAnalyzerProviders()`, `getQueryPreparerProviders()` | OpenTelemetry etc.; even the analyzer itself is pluggable |
| **Protocol filters** | `getClientRequestFilterFactories()` | Client HTTP request interception |

### 2.2 `CoordinatorPlugin` — coordinator-only extension points

Newer, control-plane-only surface (verified): `getFunctionNamespaceManagerFactories()`, `getWorkerSessionPropertyProviderFactories()`, `getPlanCheckerProviderFactories()`, `getExpressionOptimizerFactories()`, `getTypeManagerFactories()`.

This interface exists largely **for the native (Prestissimo) migration**: a Velox sidecar registers function metadata, worker session properties, native plan checkers, and a native-semantics expression optimizer into the Java coordinator — planning stays consistent with C++ evaluation semantics without loading worker-irrelevant code everywhere.

There is also a `RouterPlugin` SPI for the `presto-router` gateway (pluggable cluster-selection schedulers, e.g. the plan-checker router plugin that routes queries by native-engine compatibility).

### 2.3 Deployment & activation model

```
plugin/                              etc/catalog/sales.properties
 ├─ hive-hadoop2/*.jar               ┌──────────────────────────────┐
 ├─ mysql/*.jar                      │ connector.name=mysql         │ ← matches ConnectorFactory.getName()
 └─ my-connector/*.jar               │ connection-url=jdbc:mysql:…  │ ← passed to factory as config map
                                     └──────────────────────────────┘
```

- **Plugin** = code, loaded once at server startup (`PluginManager`).
- **Catalog** = a *named instantiation* of a connector: each `etc/catalog/*.properties` file calls `ConnectorFactory.create(catalogName, configMap, context)`. One plugin → many catalogs (e.g., two `mysql` catalogs pointing at different databases). The catalog name becomes the first part of `catalog.schema.table` names and maps to a `ConnectorId`.

---

## 3. Classloader Isolation — the Load-Bearing Implementation Detail

Each plugin loads in its **own child-last `PluginClassLoader`**. Resolution order: for a whitelisted set of shared packages, delegate to the server's classloader (parent-first); for *everything else*, the plugin's own jars win.

The shared whitelist (`PluginManagerUtil.SPI_PACKAGES`, verified):

```java
com.facebook.presto.spi.          // the contract itself — must be THE same Class objects
com.facebook.presto.common        // Block/Page/Type — data crosses the boundary as these
com.fasterxml.jackson.annotation. // handle serialization annotations
io.airlift.slice.                 // Slice (binary data primitive)
com.facebook.airlift.units.       // DataSize/Duration config types
com.facebook.drift.annotations.   // Thrift serde annotations (+ TException types)
org.openjdk.jol.                  // object-size accounting
```

**Why this matters:** the engine ships its own Guava/Jackson/Netty versions; so does every connector (Hadoop's dependency tree is notoriously conflicting). Child-last isolation means a connector can use *any* version of *any* library without breaking the engine or sibling plugins. Only the seven package families above must be binary-identical across the boundary — and that is exactly the definition of the SPI compatibility surface: **the SPI isn't just "interfaces", it's the set of classes whose identity is shared between engine and plugins.**

Two supporting mechanisms complete the pattern:

- **`ThreadContextClassLoader`** (in `spi/classloader`): a try-with-resources guard that swaps the thread context classloader while engine threads execute plugin code — so plugin-internal frameworks that rely on TCCL (Hadoop config, JAXB, ServiceLoader inside the plugin) resolve inside the plugin's own world.
- **`ClassLoaderSafe*` decorators** (`spi/connector/classloader`): `ClassLoaderSafeConnectorMetadata`, `...SplitManager`, `...PageSourceProvider`, `...PageSinkProvider`, `...NodePartitioningProvider`, `...MergeSink`, `...PageSink` — thin wrappers that set the plugin's classloader around *every* method call. Connector factories wrap their services in these before returning them, so no engine call site can accidentally execute plugin code under the wrong classloader.

---

## 4. The Connector SPI in Depth

A `ConnectorFactory.create(...)` returns a `Connector`, which is a **service locator** for per-concern sub-services (all `default` methods — implement only what your source supports; verified against `Connector.java`):

```
Connector
 ├─ getMetadata(txn)                    → ConnectorMetadata            [coordinator]
 ├─ getSplitManager()                   → ConnectorSplitManager        [coordinator]
 ├─ getPageSourceProvider()             → ConnectorPageSourceProvider  [worker: reads]
 │    (or getRecordSetProvider()        → row-oriented fallback, engine adapts it)
 ├─ getPageSinkProvider()               → ConnectorPageSinkProvider    [worker: writes]
 ├─ getNodePartitioningProvider()       → expose storage bucketing to the engine
 ├─ getConnectorPlanOptimizerProvider() → connector-side plan rewrites (pushdown)
 ├─ getIndexProvider()                  → key-lookup joins
 ├─ getAccessControl()                  → per-catalog authorization
 ├─ getSystemTables(), getProcedures(), getDistributedProcedures(),
 │  getTableFunctions(), getSystemFunctions()
 ├─ get{Session,Table,Schema,Column,Analyze,MaterializedView}Properties()
 ├─ beginTransaction / commit / rollback  (+ ConnectorCommitHandle)
 ├─ getCapabilities()                   → declared feature set (ConnectorCapabilities)
 └─ getConnectorCodecProvider()         → custom (e.g. Thrift) handle serde for native workers
```

### 4.1 Query lifecycle → SPI call sequence

A `SELECT` against your connector triggers, in order:

| Phase | Where | SPI calls |
|---|---|---|
| Analysis | coordinator | `ConnectorMetadata.getTableHandle`, `getColumnHandles`, `getColumnMetadata`, view resolution |
| Optimization | coordinator | `getTableStatistics` (CBO input), `getTableLayouts(constraint)` (predicate pushdown negotiation), `ConnectorPlanOptimizer.optimize` (aggregate/limit/topN pushdown), `getCommonPartitioningHandle` (co-located join detection) |
| Scheduling | coordinator | `ConnectorSplitManager.getSplits(layout)` → async `ConnectorSplitSource.getNextBatch`; splits carry `getPreferredNodes` locality hints |
| Execution | workers | `ConnectorPageSourceProvider.createPageSource(split, columns)` → produce `Page`s (lazy blocks encouraged) |
| Writes | workers + coordinator | `beginInsert` → `ConnectorPageSinkProvider.createPageSink` per task → collected fragments → `finishInsert` (single-point metadata commit) |

### 4.2 The opaque-handle contract

`ConnectorTableHandle`, `ColumnHandle`, `ConnectorTableLayoutHandle`, `ConnectorSplit`, `ConnectorTransactionHandle`, `Connector{Insert,Output,Delete}TableHandle` are **marker interfaces with no methods**. The engine never inspects them — it only:

1. receives them from one SPI call,
2. serializes them (JSON via Jackson; optionally Thrift/custom via `ConnectorCodec` for C++ workers),
3. round-trips them across the cluster,
4. hands them back to the *same connector* in a later call.

This is the **opaque token / handle-body pattern**: all source-specific state (file paths, partition values, byte ranges, JDBC SQL, Kafka offsets) travels through the engine without coupling the engine to it. The only requirement is JSON-serializability and, for handle *resolution*, registration of concrete classes (`ConnectorHandleResolver`).

### 4.3 Pushdown as *negotiation*, not command

`getTableLayouts(session, table, Constraint<ColumnHandle>, desiredColumns)` embodies a principled pushdown protocol:

- Engine offers a `TupleDomain` predicate (sound over-approximation of the WHERE clause).
- Connector returns layouts with: the predicate portion it **enforces** (engine may drop it) vs the portion left **unenforced** (engine keeps a residual `FilterNode`).
- Correctness invariant: a connector may always return *more* rows than the predicate implies, never fewer. Pushdown is therefore an optimization that can be partially or wholly declined without ever changing results — which is what lets 40 heterogeneous connectors implement wildly different pushdown capabilities safely.

`ConnectorPlanOptimizer` generalizes this to whole plan subtrees (aggregation pushdown to Pinot/Druid, join pushdown in JDBC connectors): the connector receives the plan fragment above its scan and may rewrite it into its handle — same invariant, coarser granularity.

### 4.4 Capability declaration

`getCapabilities()` returns a set of `ConnectorCapabilities` enums (e.g. `NOT_NULL_COLUMN_CONSTRAINT`, dereference pushdown support...) that the analyzer/planner consults before generating plans requiring them — **capability negotiation** instead of runtime failure.

---

## 5. Toolkits: Don't Start From Zero

Two in-tree "abstract connector" layers implement the SPI once and expose a much smaller provider interface — the **Template Method pattern at module scale**:

- **`presto-base-jdbc`** — full connector implementation over a `JdbcClient` interface; MySQL/PostgreSQL/Oracle/SQL Server/ClickHouse/Redshift connectors are mostly a `JdbcClient` subclass (type mapping + SQL dialect + URL handling).
- **`presto-base-arrow-flight`** — same idea over Arrow Flight endpoints.
- **`presto-plugin-toolkit`**, `presto-record-decoder` (shared row decoders for Kafka/Redis), `presto-hive-common`/`presto-hive-metastore` (shared lake-connector machinery for Hive/Iceberg/Delta/Hudi) fill the same role for other families.

**Guidance:** implement raw SPI only for genuinely novel sources; for anything JDBC-ish or lake-ish, extend the toolkit — you inherit years of pushdown, type-mapping, and correctness fixes.

---

## 6. The Design Patterns Behind the SPI (named and mapped)

| # | Pattern | Where it appears in Presto | Why it's used |
|---|---|---|---|
| 6.1 | **Service Provider / ServiceLoader** (JDK idiom) | `META-INF/services/com.facebook.presto.spi.Plugin` + `PluginManager` | Zero-config discovery of provider implementations from a jar |
| 6.2 | **Dependency Inversion / Plugin Architecture** | Engine depends on `presto-spi`, never on connectors; connectors depend on `presto-spi`, never on engine | Independent evolution and shipping on both sides of the boundary |
| 6.3 | **Abstract Factory** | `ConnectorFactory`, `EventListenerFactory`, `PasswordAuthenticatorFactory`, `...Factory` everywhere; `Connector` itself is a factory-of-services | Defers construction to the provider; enables one plugin → many configured catalogs |
| 6.4 | **Opaque Token / Handle** | All `*Handle` and `ConnectorSplit` marker interfaces (§4.2) | Provider state flows through the framework without coupling; serialization is the only contract |
| 6.5 | **Decorator** | `ClassLoaderSafeConnectorMetadata` et al. (§3) | Cross-cutting concern (classloader switching) wrapped around every provider call without touching provider code |
| 6.6 | **Child-last classloading + shared-package whitelist** | `PluginClassLoader` + `SPI_PACKAGES` | Dependency isolation; the whitelist *is* the ABI definition. Same pattern at its extreme in `presto-spark-classloader-interface` (only one tiny module shared with Spark's classloader, everything else reflectively isolated) |
| 6.7 | **Interface evolution via `default` methods** | Every method on `Plugin`, `CoordinatorPlugin`, `Connector`, most of `ConnectorMetadata` | Additive-only SPI evolution: new extension points never break existing compiled plugins. The SPI discipline: *add defaults, deprecate, never remove hastily* |
| 6.8 | **Capability negotiation** | `Connector.getCapabilities()`, table-layout pushdown residuals, `ConnectorPlanOptimizer` | Framework asks what the provider can do instead of failing at runtime; degraded-but-correct fallback always exists |
| 6.9 | **Template Method / Toolkit** | `presto-base-jdbc`'s `JdbcClient`, record decoders, hive-common | Providers implement the small varying part; toolkit owns the invariant algorithm |
| 6.10 | **Service Locator** | `Connector.get*Provider()` accessors | One handle to a family of optional sub-services, resolved per concern |
| 6.11 | **Inversion of Control container** | Airlift/Guice `Bootstrap` inside plugins (most connectors build a private Guice injector in their factory) | Config binding (`@Config` setters on `*Config` classes), lifecycle (`@PostConstruct`/`@PreDestroy`) inside the isolated world |
| 6.12 | **Protocol-as-contract (Bridge at system level)** | Not SPI per se, but the sibling seam: coordinator↔worker HTTP/JSON+Thrift protocol lets C++ (Velox) workers replace Java workers | Same philosophy as the SPI — a stable declared boundary, two independent implementations |

The unifying idea across all twelve: **Presto defines seams, not implementations.** Each seam (SPI classes, shared-package whitelist, wire protocol) is small, versioned conservatively, and everything on either side is replaceable.

---

## 7. Writing a Minimal Connector (skeleton)

```java
// 1. Entry point — named in META-INF/services/com.facebook.presto.spi.Plugin
public class ExamplePlugin implements Plugin {
    @Override public Iterable<ConnectorFactory> getConnectorFactories() {
        return ImmutableList.of(new ExampleConnectorFactory());
    }
}

// 2. Factory — "connector.name=example" in a catalog file selects this
public class ExampleConnectorFactory implements ConnectorFactory {
    @Override public String getName() { return "example"; }
    @Override public ConnectorHandleResolver getHandleResolver() { return new ExampleHandleResolver(); }
    @Override public Connector create(String catalogName, Map<String,String> config, ConnectorContext ctx) {
        // typical: Guice Bootstrap with a config class, then wrap services ClassLoaderSafe
        return new ExampleConnector(...);
    }
}

// 3. The four essentials for a read-only connector:
//    ConnectorMetadata      — name → ExampleTableHandle, columns → ExampleColumnHandle
//    ConnectorSplitManager  — layout → List<ExampleSplit> (host hints, ranges)
//    ConnectorPageSourceProvider — split + columns → ConnectorPageSource producing Pages
//        (or implement RecordSet/RecordCursor and let the engine adapt rows → pages)
//    Handles                — plain JSON-serializable value classes (@JsonCreator/@JsonProperty)
```

Reference implementations by increasing sophistication: `presto-example-http` (minimal read-only), `presto-blackhole` (writes), `presto-tpch` (generated data + partitioning), `presto-base-jdbc` (pushdown), `presto-hive` (everything).

**Testing:** `presto-tests` offers `DistributedQueryRunner` to stand up an in-JVM multi-node cluster with your plugin installed — the standard way every in-tree connector is integration-tested.

---

## 8. Compatibility & Governance Rules of Thumb

1. **`presto-spi` + `presto-common` are the ABI.** Anything else a plugin reaches into (engine internals via reflection, provided-scope tricks) is unsupported and will break.
2. Changes to SPI interfaces should be **additive with defaults**; removals go through deprecation cycles. Review scrutiny on `presto-spi` PRs is intentionally higher (CODEOWNERS).
3. Handles must remain **JSON round-trippable across versions running simultaneously** (rolling upgrades: coordinator and workers may briefly differ) — prefer adding optional fields.
4. On **native (Prestissimo) clusters**, worker-side Java SPI implementations (`PageSourceProvider`, functions) do not run — plan for a Velox-side counterpart or coordinator-only functionality; `ConnectorCodecProvider`/Thrift serde exists to make handles consumable by C++ workers.
5. Everything a plugin does executes inside engine processes: **resource discipline applies** — respect `ConnectorPageSource.getSystemMemoryUsage()`, make split sources async, never block engine threads on unbounded IO without yielding futures.

---

*Generated 2026-07-15 from repository analysis (git HEAD, v0.299-SNAPSHOT). Interface listings verified against `Plugin.java`, `CoordinatorPlugin.java`, `Connector.java`, `PluginManagerUtil.java`.*
