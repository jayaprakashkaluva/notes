# Apache Spark — Catalyst & SQL Execution Internals

> Based on analysis of the Spark source tree at `C:\workspace\opensource\spark`, version
> **5.0.0-SNAPSHOT**, HEAD `ee11a92a9f1`.
> Companions: [spark-architecture.md](spark-architecture.md) ·
> [spark-core-internals.md](spark-core-internals.md)
>
> Code referenced here lives in `sql/api`, `sql/catalyst`, and `sql/core`.
> `sql/catalyst` is ~440k lines — larger than `core` — and is where most of Spark's
> engineering investment of the last decade landed.

---

## 1. The Query Pipeline

`QueryExecution` (`sql/core/.../execution/QueryExecution.scala`) is the state machine.
Each phase is a `lazy val`, so phases are computed on demand and memoized, and each is
wrapped in `QueryPlanningTracker` instrumentation (the numbers behind `df.explain("cost")`
and the SQL tab's planning breakdown):

```
 SQL text ──ANTLR──▶ Unresolved LogicalPlan
                        │
                        ├─ analyzed          (Analyzer / Resolver: catalog, types, ids)
                        ├─ commandExecuted   (eager execution of DDL/commands)
                        ├─ normalized        (plan normalization for cache matching)
                        ├─ withCachedData    (CacheManager substitution)
                        ├─ optimizedPlan     (Optimizer rule batches)
                        ├─ sparkPlan         (SparkPlanner strategies → physical)
                        ├─ executedPlan      (preparations: exchanges, sorts, codegen)
                        └─ toRdd             (RDD[InternalRow])
```

DataFrame API calls skip the parser and construct the unresolved logical plan directly —
`df.filter(...)` builds a `Filter(UnresolvedAttribute(...), child)`. This is why DataFrame
and SQL have identical performance: they converge at the same unresolved tree.

---

## 2. The Tree/Rule Substrate

### 2.1 `TreeNode` (`catalyst/trees/TreeNode.scala`)

Everything in Catalyst — expressions, logical plans, physical plans — is a `TreeNode`:
an **immutable**, `Product`-based (Scala case class) node with a `children: Seq[BaseType]`.
Immutability is what makes rules composable: a rule returns a new tree, never mutates.

Key operations:
- `transformDown` / `transformUp` / `transformWithPruning` — apply a partial function to
  matching nodes, rebuilding only the changed spine (`mapChildren` returns `this` when no
  child changed, so unchanged subtrees are shared by reference).
- `makeCopy` — reflective reconstruction via the case-class constructor. Its cost is the
  reason `withNewChildren` short-circuits on reference equality.
- `canonicalized` — a normalized form (expression IDs zeroed, commutative operands sorted,
  aliases stripped) used for plan equality: exchange reuse, subquery reuse, and cache
  lookup all key on it.
- `treeString` / `simpleString` — the `explain` output; `ExplainUtils` handles the
  `FORMATTED` mode with per-operator sections.

### 2.2 Tree-pattern bitmasks — the scaling fix

The original design applied every rule to every node, giving O(rules × nodes) work per
batch iteration. `TreePatternBits` (`catalyst/trees/TreePatternBits.scala`) fixed this:

```scala
trait TreePatternBits {
  protected def treePatternBits: BitSet
  @inline final def containsPattern(t: TreePattern): Boolean = treePatternBits.get(t.id)
  final def containsAllPatterns(patterns: TreePattern*): Boolean
  final def containsAnyPattern(patterns: TreePattern*): Boolean
}
```

Each node computes a `BitSet` that is the union of its own `TreePattern` markers (e.g.
`INNER_LIKE_JOIN`, `LITERAL`, `AGGREGATE`, `UNRESOLVED_ATTRIBUTE` — the enum in
`TreePatterns.scala`) and its children's bits. A rule declares the patterns it cares about
and uses `transformWithPruning(_.containsPattern(FILTER))`, so entire subtrees are skipped
with one bit test instead of a full walk.

Complementing this, `RuleIdCollection` assigns each rule an ID, and nodes track which rules
have already run on them and made no change — so a rule that reached a fixed point on a
subtree isn't reapplied. Together these turned analysis+optimization of large plans from
quadratic into something closer to linear.

### 2.3 `RuleExecutor` (`catalyst/rules/RuleExecutor.scala`)

```scala
abstract class Strategy { def maxIterations: Int }
case object Once extends Strategy { val maxIterations = 1 }
case class FixedPoint(maxIterations, errorOnExceed, maxIterationsSetting) extends Strategy
```

`execute` loops each `Batch` until the plan stops changing or `maxIterations`
(`spark.sql.optimizer.maxIterations`, default 100) is hit. On exceeding, it either logs or
throws depending on `errorOnExceed` — a non-converging rule set is a bug, and Spark treats
it as one in tests.

Two safety mechanisms worth knowing:
- **Idempotence checking** for `Once` batches (line ~304): after running a `Once` batch, run
  it again and assert the plan is unchanged. Rules excluded from this live in
  `excludedOnceBatches`. This catches the classic "rule keeps adding an alias every time it
  runs" bug that only manifests under AQE re-planning.
- **`isPlanIntegral`** — a per-executor invariant check (e.g. "all expression IDs are
  unique", "no unresolved nodes after analysis") run between rules when
  `spark.sql.planChangeLog.level` / testing flags are set.

`QueryExecutionMetering` accumulates per-rule time and effectiveness, exposed via
`RuleExecutor.dumpTimeSpent()` — the tool for diagnosing "why does planning take 30 seconds".

---

## 3. Parser (`sql/api/src/main/antlr4/.../SqlBaseLexer.g4`, `SqlBaseParser.g4`)

ANTLR4, split into a lexer and parser grammar. `AstBuilder`
(`catalyst/parser/AstBuilder.scala`) is the visitor that turns the parse tree into
unresolved Catalyst nodes; `SparkSqlParser` (`sql/core`) extends it with the
Spark-specific commands.

Notable properties:
- The grammar lives in `sql/api`, not `sql/catalyst`, so the Connect client can parse
  identifiers without Catalyst.
- Reserved vs. non-reserved keywords are switchable (`spark.sql.ansi.enforceReservedKeywords`)
  — ANSI mode reserves more.
- `ParserRuleContext.origin` propagates line/column into every node, which is what produces
  the caret-annotated error messages (`== SQL ==` blocks with `^^^`).
- SQL scripting (`SqlScriptingLogicalPlans.scala`, `SqlScriptingContextManager.scala`) adds
  BEGIN/END blocks, variables, control flow, and cursors — a procedural layer on top of the
  declarative one.

---

## 4. Analysis

### 4.1 The fixed-point `Analyzer` (`catalyst/analysis/Analyzer.scala`)

A `RuleExecutor[LogicalPlan]` with batches roughly in this order:

1. **Substitution** — `CTESubstitution`, `WindowsSubstitution`, `EliminateUnions`,
   `SubstituteUnresolvedOrdinals` (turns `GROUP BY 1` into a real expression).
2. **Resolution** (fixed point, the big one) — `ResolveRelations`, `ResolveReferences`,
   `ResolveFunctions`, `ResolveAliases`, `ResolveSubquery`, `ResolveAggregateFunctions`,
   `ResolveGroupingAnalytics`, `ResolveWindowFrame`, `ResolveNaturalAndUsingJoin`,
   `ResolveOutputRelation`, plus the entire type-coercion suite.
3. **Post-hoc** — `ApplyCharTypePadding`, `UpdateAttributeNullability`,
   `HandleSpecialCommand`.
4. **CheckAnalysis** — not a rule but a validator: walks the plan bottom-up and throws
   `AnalysisException` with an error class for every unresolved attribute, type mismatch,
   invalid aggregate, ambiguous reference, unsupported correlated subquery, etc.

**Expression IDs.** `NamedExpression` carries an `ExprId(id: Long, jvmId: UUID)`. Attribute
resolution is by ID after analysis, not by name — this is what makes self-joins work.
`DeduplicateRelations` rewrites one side's IDs when the same relation appears twice.
Nearly every subtle analyzer bug in Spark's history has been an expression-ID collision or
a failure to deduplicate.

**Type coercion** (`TypeCoercion.scala` / `AnsiTypeCoercion.scala`) is a set of rules —
`ImplicitTypeCasts`, `PromoteStrings`, `DecimalPrecision`, `FunctionArgumentConversion`,
`InConversion`, `WidenSetOperationTypes`, `DateTimeOperations`, `CollationTypeCoercion` —
run to fixed point. ANSI mode swaps in a stricter rule set that refuses string→numeric
implicit promotion. `DecimalPrecision` implements the Hive/SQL Server precision-and-scale
derivation rules for arithmetic, including the "allow precision loss" behavior behind
`spark.sql.decimalOperations.allowPrecisionLoss`.

### 4.2 The single-pass `Resolver` (in-flight, `analysis/resolver/`, 102 files)

The fixed-point analyzer's cost model is poor: a rule that could resolve a node in one visit
instead gets re-run across the whole tree until nothing changes. The `Resolver` replaces
this with an explicit **bottom-up recursive descent**:

> "The Resolver implements a single-pass bottom-up analysis algorithm in the Catalyst. …
> The Resolver is a one-shot object per each SQL/DataFrame logical plan, the calling code
> must re-create it for every new analysis run." — `Resolver.scala:60`

The structure mirrors the language rather than the rule set: `AggregateResolver`,
`JoinResolver`, `FilterResolver`, `HavingResolver`, `WindowResolver`,
`LateralColumnAliasResolver`, `ExpressionResolver`, etc., with explicit state objects for
things the fixed-point analyzer keeps implicit — `NameScope`/`AttributeScopeStack` for name
resolution, `ExpressionIdAssigner` for ID allocation, `CteScope` for CTE visibility.

Migration is handled by `HybridAnalyzer`, which is unusually well-designed for a risky
swap-out:

| Mode | Config | Behavior |
|---|---|---|
| Legacy | default | fixed-point analyzer only |
| Single-pass | `spark.sql.analyzer.singlePassResolver.enabled` | new resolver only (dev) |
| Tentative | `...enabledTentatively` | new resolver, fall back to legacy on unsupported features |
| Dual-run | `...dualRunEnabled` + `ANALYZER_DUAL_RUN_SAMPLE_RATE` | run both, **compare plans and schemas**, return the legacy result |

`ResolverGuard` decides whether a given plan uses only features the new resolver supports,
and `ExplicitlyUnsupportedResolverFeature` is the escape. In dual-run, disagreement between
the two analyzers is a hard error with a dedicated error class
(`QueryCompilationErrors.fixedPointFailedSinglePassSucceeded`), so divergence is caught in
CI rather than in production. `LogicalPlanDifference` produces the diff.

This is the correct pattern for replacing a correctness-critical component: shadow-run the
new implementation against the old on real traffic, and make disagreement loud.

### 4.3 Catalogs

Two generations coexist:
- **V1** — `SessionCatalog` (`catalyst/catalog/SessionCatalog.scala`) over an
  `ExternalCatalog` (`InMemoryCatalog` or `HiveExternalCatalog`). Handles temp views,
  global temp views, function registry, and the legacy Hive metastore path.
- **V2** — `CatalogPlugin` / `TableCatalog` / `FunctionCatalog` / `ProcedureCatalog`
  (`sql/catalyst/src/main/java/.../connector/catalog/`), multi-catalog by design.
  `CatalogManager` resolves a multipart identifier `cat.ns.tbl` against registered
  plugins, falling back to the session catalog.
  `DelegatingCatalogExtension` lets a plugin wrap the built-in session catalog — the hook
  Delta Lake and Iceberg use.

`ResolveCatalogs`, `RelationResolution`, and `V2TableReference` mediate between them. This
tree also carries `TransactionalCatalogPlugin`, `SupportsSchemaEvolution`, `Changelog`/
`ChangelogRange`, and `Relation`/`RelationCatalog` — evidence of ongoing DSv2 expansion
toward transactions and CDC as first-class catalog concepts.

---

## 5. Optimizer (`catalyst/optimizer/Optimizer.scala`)

`defaultBatches` is worth reading in full; the essential shape:

```
 Convert python UDFs to Catalyst        Once
 Finish Analysis                        FixedPoint(1)   ← RuntimeReplaceable expansion,
                                                          ComputeCurrentTime, ReplaceExpressions
 Rewrite With expression                fixedPoint
 Eliminate Distinct                     Once
 Inline CTE                             Once
 Union                                  fixedPoint
 LocalRelation early                    fixedPoint
 Pullup Correlated Expressions          Once
 Subquery                               FixedPoint(1)
 Replace Operators                      fixedPoint      ← Intersect/Except → Join/Aggregate
 Aggregate                              fixedPoint
 ── operatorOptimizationBatch ──                        ← see below
 Clean Up Temporary CTE Info            Once
 Pre CBO Rules                          Once
 Early Filter and Projection Push-Down  Once            ← DSv2 pushdown
 Update CTE Relation Stats              Once
 Join Reorder                           FixedPoint(1)   ← CostBasedJoinReorder
 Push Down Join Through Union           Once
 Eliminate Sorts                        Once
 Decimal Optimizations                  fixedPoint
 Distinct Aggregate Rewrite             Once
 Object Expressions Optimization        fixedPoint
 LocalRelation                          fixedPoint
 Optimize One Row Plan                  fixedPoint
 Check Cartesian Products               Once
 RewriteSubquery                        Once
 NormalizeFloatingNumbers               Once
 ReplaceUpdateFieldsExpression          Once
```

The **operator optimization batch** is itself three passes:

```
 Batch("Operator Optimization before Inferring Filters", fixedPoint, <ruleSet>)
 Batch("Infer Filters", Once, InferFiltersFromGenerate, InferFiltersFromConstraints)
 Batch("Operator Optimization after Inferring Filters", fixedPoint, <same ruleSet>)
 Batch("Push extra predicate through join", fixedPoint,
       PushExtraPredicateThroughJoin, PushDownPredicates)
```

Running the same ~60-rule set twice around filter inference is a pragmatic admission that
constraint propagation creates new optimization opportunities that the earlier pass can't
see. The rule set groups into: **push down** (`PushDownPredicates`, `ColumnPruning`,
`LimitPushDown`, `PushProjectionThroughUnion`, `PushDownLeftSemiAntiJoin`), **combine**
(`CollapseProject`, `CollapseRepartition`, `CollapseWindow`, `CombineUnions`), and
**constant folding / strength reduction** (`ConstantFolding`, `ConstantPropagation`,
`BooleanSimplification`, `NullPropagation`, `LikeSimplification`,
`UnwrapCastInBinaryComparison`, `SimplifyConditionals`, `PushFoldableIntoBranches`).

### 5.1 Rules worth calling out

- **`InferFiltersFromConstraints`** — `QueryPlanConstraints` derives a constraint set
  (`isNotNull(a)`, `a = 5`, …) per operator and pushes inferred predicates to the other side
  of a join. This is what makes `a JOIN b ON a.k = b.k WHERE a.k = 5` filter *both* sides.
  It is also the classic planning-time blowup on wide plans, hence
  `spark.sql.constraintPropagation.enabled`.
- **`ColumnPruning` + `NestedColumnAliasing` + `SchemaPruning`** — prune to the leaf, and
  push *nested field* access into Parquet/ORC readers so `SELECT s.a` doesn't materialize
  all of `s`.
- **`RewriteDistinctAggregates`** — multiple `DISTINCT` aggregates in one query are rewritten
  into an `Expand` that duplicates rows with a grouping ID, then two aggregation levels.
  This is why `COUNT(DISTINCT a), COUNT(DISTINCT b)` costs a row multiplication.
- **`DecorrelateInnerQuery`** + **`RewriteCorrelatedScalarSubquery`** — correlated subqueries
  are decorrelated into joins. Domain joins are introduced where the correlation can't be
  expressed as an equi-join. This subsystem is the source of most subquery-related
  `AnalysisException`s.
- **`InjectRuntimeFilter`** — inserts Bloom-filter (`BloomFilterMightContain`) or semi-join
  runtime filters computed from the build side of a join, evaluated on the probe side scan.
  Distinct from dynamic partition pruning; complementary to it.
- **`CostBasedJoinReorder`** — dynamic-programming join enumeration over
  `spark.sql.cbo.joinReorder.dp.threshold` (12) relations, using `Statistics` from
  `statsEstimation/`. Requires `ANALYZE TABLE ... COMPUTE STATISTICS FOR COLUMNS` to be
  meaningful; without column stats the estimates are crude and AQE is the better lever.
- **`PropagateEmptyRelation` / `OptimizeOneRowPlan`** — collapse provably-empty or
  provably-single-row subtrees, which is more valuable than it sounds under AQE (a stage
  that produces zero rows lets the rest of the plan be simplified).

### 5.2 Statistics (`plans/logical/statsEstimation/`)

Two modes:
- **Size-only** (default): `sizeInBytes` propagated with crude heuristics; sufficient for
  the broadcast-join threshold decision.
- **CBO** (`spark.sql.cbo.enabled`): `BasicStatsPlanVisitor` +
  `FilterEstimation`/`JoinEstimation`/`AggregateEstimation`/`ProjectEstimation` propagate
  row counts and per-column histograms/NDVs through the plan.

Honest assessment: CBO in Spark is under-used in practice because it needs manually
maintained table statistics that most deployments don't collect. AQE largely subsumed it by
measuring instead of estimating.

---

## 6. Physical Planning

### 6.1 `SparkPlanner` strategies (`execution/SparkStrategies.scala`)

`QueryPlanner.plan` applies `Strategy` objects (`GenericStrategy[SparkPlan]`), each mapping
a logical operator to one or more physical alternatives, and returns an `Iterator[SparkPlan]`
— nominally supporting cost-based selection, but in practice Spark takes `.next()`, i.e.
the first alternative. Cost-based physical selection remains vestigial.

Strategies in the file: `SpecialLimits`, `JoinSelection`, `AsOfJoinSelection`,
`Aggregation`, `Window`, `WindowGroupLimit`, `InMemoryScans`, `PythonEvals`,
`BasicOperators`, plus the streaming ones (`StatefulAggregationStrategy`,
`StreamingJoinStrategy`, `StreamingDeduplicationStrategy`,
`FlatMapGroupsWithStateStrategy`, `StreamingTransformWithStateStrategy`).
`DataSourceStrategy` and `DataSourceV2Strategy` live in `execution/datasources/`.

### 6.2 `JoinSelection`

Preference order for an equi-join:

1. **Broadcast hash join** — if one side is smaller than
   `spark.sql.autoBroadcastJoinThreshold` (10 MB) or a `BROADCAST` hint is present, and the
   join type allows broadcasting that side (can't broadcast the left side of a left outer
   join). Build side is materialized into a `HashedRelation` on the driver and broadcast.
2. **Shuffled hash join** — if `spark.sql.join.preferSortMergeJoin` is false, or one side is
   much smaller and would build a hash map that fits, or a `SHUFFLE_HASH` hint.
3. **Sort-merge join** — the general case, requires sortable keys.
4. **Broadcast nested loop join** / **Cartesian product** — for non-equi joins. Cartesian
   requires `spark.sql.crossJoin.enabled` semantics or an explicit `CROSS JOIN`;
   `CheckCartesianProducts` errors otherwise.

Hints (`hints.scala`, `ResolveHints`) — `BROADCAST`, `MERGE`, `SHUFFLE_HASH`,
`SHUFFLE_REPLICATE_NL` — are honored strictly and are the standard escape hatch when
statistics mislead the planner.

### 6.3 `HashedRelation` (`execution/joins/HashedRelation.scala`)

Two implementations, and the split matters for performance:

**`UnsafeHashedRelation`** — general keys, backed by `BytesToBytesMap` (off-heap open
addressing over `TaskMemoryManager` pages). Serialization format:
`[numKeys][numFields] then [keySize][valueSize][keyBytes][valueBytes]*`.

**`LongHashedRelation` / `LongToUnsafeRowMap`** — specialization for a single long key
(after `HashJoin` rewrites integral keys into a packed long). Values are packed into one
`page: Array[Long]`:

```
 page:  [row1 bytes][address1][row2 bytes][address2] ...
        address (8 bytes) = offset+size of the NEXT value for the same key; 0 terminates
 keys, two modes:
   sparse: array = [key1][address1][key2][address2]... , slot = key % cap,
           quadratic probing with triangular numbers
   dense:  array = [address1][address2]...            , slot = key - minKey
```

The map starts sparse; `optimize()` converts to dense when the key range is tight enough,
turning a probe into a single array index with no hashing and no probing. This is the
fastest join path Spark has, and it's why casting a join key to `long` sometimes produces
a step-change in performance.

`EmptyHashedRelation` and `HashedRelationWithAllNullKeys` are singletons that let the join
short-circuit entirely — and let AQE convert a join to an empty relation.

### 6.4 Aggregation (`execution/aggregate/`)

Three physical operators, chosen by `AggUtils.planAggregateWithoutDistinct` etc.:

- **`HashAggregateExec`** — the default. Uses `TungstenAggregationIterator` over a
  `BytesToBytesMap` of `UnsafeRow` buffers. When the map can't grow, it **switches to
  sort-based aggregation**: spill the map's contents sorted by key, then merge-aggregate the
  sorted streams. This graceful degradation (rather than OOM) is one of Tungsten's better
  properties. Requires all aggregate buffer fields to be mutable fixed-length types.
- **`ObjectHashAggregateExec`** — for aggregates with JVM-object buffers (`collect_list`,
  `percentile`, typed UDAFs). Falls back to sorting after
  `spark.sql.objectHashAggregate.sortBased.fallbackThreshold` keys (128).
- **`SortAggregateExec`** — requires sorted input; used when hashing isn't possible or when
  `ReplaceHashWithSortAgg` proves the child already provides the ordering.

**Partial/final split.** A distributive aggregate becomes `Partial` (map side) →
`Exchange` → `Final` (reduce side). `PartialMerge` and `Complete` cover the distinct
rewrite and the AQE-combined cases. `CombineAdjacentAggregation` (in this tree) fuses an
adjacent partial/final pair into a single `Complete` aggregate when no shuffle separates
them.

**Vectorized fast path.** `VectorizedHashMapGenerator` / `RowBasedHashMapGenerator` emit a
small, fixed-capacity, cache-resident hash map generated as Java source, checked before the
main `BytesToBytesMap`. For low-cardinality group-bys this keeps the hot loop entirely in L1.

### 6.5 `EnsureRequirements` and the partitioning algebra

Each `SparkPlan` declares `requiredChildDistribution: Seq[Distribution]` and
`requiredChildOrdering: Seq[Seq[SortOrder]]`, and exposes `outputPartitioning: Partitioning`
and `outputOrdering`. `EnsureRequirements` inserts `ShuffleExchangeExec` and `SortExec`
wherever a child doesn't already satisfy its parent's requirement.

```
 Distribution                    satisfied by Partitioning
 ─────────────────────────────   ────────────────────────────────────────
 UnspecifiedDistribution         anything
 AllTuples                       SinglePartition
 ClusteredDistribution(exprs)    HashPartitioning(subset of exprs)
                                 RangePartitioning / DataSourcePartitioning (co-partitioned)
 OrderedDistribution(ordering)   RangePartitioning matching a prefix
 BroadcastDistribution(mode)     BroadcastPartitioning(same mode)
```

`PartitioningCollection` lets a join advertise both sides' partitionings, so a subsequent
join on either key set avoids a shuffle. This algebra is the entire basis of shuffle
elimination, and it is also why `repartition(n, col)` followed by a `groupBy(col)` is free
while `repartition(n)` followed by `groupBy(col)` is not.

`spark.sql.requireAllClusterKeysForCoPartition` and the `SupportsReportPartitioning` DSv2
hook (Storage-Partitioned Join) extend this to sources that already know their layout —
a bucketed or Iceberg-partitioned table can join without any shuffle at all.

---

## 7. Whole-Stage Code Generation

### 7.1 The produce/consume protocol (`execution/WholeStageCodegenExec.scala`)

`CollapseCodegenStages` fuses a maximal subtree of `CodegenSupport` operators into one
`WholeStageCodegenExec`, which generates a single Java class implementing a tight loop.
The protocol, from the class comment:

```
   WholeStageCodegen       Plan A               FakeInput        Plan B
 =========================================================================
 -> execute()
     |
  doExecute() --------->   inputRDDs() -------> inputRDDs() ------> execute()
     |
     +----------------->   produce()
                             |
                          doProduce()  -------> produce()
                                                   |
                                                doProduce()
                                                   |
                         doConsume() <--------- consume()
     |
  doConsume()  <--------  consume()
```

`produce()` walks *down* to the source, which emits the driving loop in `doProduce()`; then
`consume()` walks *back up*, each operator pasting its per-row logic into the loop body via
`doConsume()`. The result is one method with no virtual calls and no `Iterator.next()`
per operator — replacing the "volcano" iterator model with a fused loop.

`InputAdapter` is the boundary node that hides a non-codegen subtree and feeds it in as an
`Iterator[InternalRow]`. `BlockingOperatorWithCodegen` (sort, aggregate) is the variant that
must consume all input before producing — it terminates one loop and starts another.

The `*(n)` markers in `explain` output are codegen stage IDs, assigned depth-first
post-order by `CollapseCodegenStages`; ID `0` marks a temporary/fallback object.

### 7.2 Compilation and its limits (`codegen/CodeGenerator.scala`)

Generated source is compiled by Janino (`CodeCompiler`, with a JDK-compiler backend also
present in this tree) and cached in a `NonFateSharingCache` keyed by the code string —
"non-fate-sharing" meaning a compilation failure for one query doesn't abort other queries
waiting on the same cache entry. Size: `spark.sql.codegen.cache.maxEntries`.

The JVM constraints Catalyst has to design around are encoded as constants:

| Constant | Value | Why |
|---|---|---|
| `DEFAULT_JVM_HUGE_METHOD_LIMIT` | 8000 | HotSpot refuses to JIT methods over 8 KB of bytecode |
| `MAX_JVM_METHOD_PARAMS_LENGTH` | 255 | JVM method parameter limit |
| `MAX_JVM_CONSTANT_POOL_SIZE` | 65535 | class constant pool limit |
| `GENERATED_CLASS_SIZE_THRESHOLD` | 1,000,000 | above this, split methods into a private inner class |
| `MERGE_SPLIT_METHODS_THRESHOLD` | 3 | group tiny split methods to avoid call overhead |

`splitExpressions` breaks wide projections into helper methods; `addNewFunction` may place
them in nested inner classes when the outer class gets too large. If generated code exceeds
`spark.sql.codegen.hugeMethodLimit` (8000), the stage **falls back to interpreted
execution** with a warning — the counter-intuitive case where a very wide `SELECT` is
*slower* with codegen enabled.

`CodegenFallback` is the per-expression escape: an expression that can't generate code emits
a call back into its `eval`. `CodeGeneratorWithInterpretedFallback` handles the
projection-level fallback (`spark.sql.codegen.factoryMode`).

**Subexpression elimination** (`EquivalentExpressions`, `SubExprEvaluationRuntime`) finds
common subtrees within a projection and evaluates each once into a local variable.

### 7.3 `UnsafeRow` (`sql/catalyst/src/main/java/.../expressions/UnsafeRow.java`)

The binary row format everything above operates on:

```
 [ null-tracking bit set ][ fixed-length values, 8 bytes/field ][ variable-length region ]
   ceil(numFields/64)*8 bytes      one word per field             strings, arrays, maps,
                                   primitives inline;             structs
                                   var-length = (offset<<32)|len
```

Properties that make the rest of the design work:
- **Fixed field offset** — reading field *i* is `base + bitSetWidth + i*8`, no schema walk.
- **Relocatable** — a row is a contiguous byte range with only *relative* offsets, so it can
  be memcpy'd, sorted by pointer, spilled, and shipped without deserialization. This is
  precisely the `supportsRelocationOfSerializedObjects` property that unlocks the
  Tungsten shuffle writer (see [core internals §6.1](spark-core-internals.md#61-writer-selection-shufflesortsortshufflemanagerscala)).
- **8-byte alignment** — `Platform.getLong`-friendly, and comparison of two rows can be done
  word-at-a-time (`UnsafeRow.equals` is a `memcmp`).

`UnsafeArrayData`, `UnsafeMapData`, and `ColumnarBatch`/`ColumnVector` (for the vectorized
Parquet/ORC readers) round out the format family. `GenerateUnsafeProjection` emits the code
that writes into this layout; `GenerateUnsafeRowJoiner` concatenates two `UnsafeRow`s
without going through fields — used on the join hot path.

**Sorting.** `UnsafeExternalSorter` sorts `(pointer, prefix)` pairs where the prefix is a
sortable 8-byte encoding of the leading sort key (`PrefixComparators`). Most comparisons
resolve on the prefix alone, never touching the row. `RadixSort` handles the
prefix-only-and-no-nulls case in O(n) passes. This is the single biggest reason Spark's
sort throughput jumped in the Tungsten era.

---

## 8. Adaptive Query Execution (`execution/adaptive/`)

### 8.1 The loop (`AdaptiveSparkPlanExec.scala`)

`InsertAdaptiveSparkPlan` wraps the physical plan in an `AdaptiveSparkPlanExec`, which is a
**leaf node** from the outside — so all downstream preparation rules become no-ops and AQE
owns the plan entirely.

```
 loop:
   createQueryStages(currentPhysicalPlan)
     └─ bottom-up: at each Exchange whose children are all materialized stages,
        create a ShuffleQueryStageExec / BroadcastQueryStageExec, apply
        queryStageOptimizerRules + postStageCreationRules, and materialize() it async
   await any stage completion
   on completion:
     update the logical plan: replace the materialized subtree with a LogicalQueryStage
       carrying real MapOutputStatistics
     reOptimize(logicalPlan):
        logicalPlan.invalidateStatsCache()
        optimizer.execute(logicalPlan)              ← AQEOptimizer, logical re-optimization
        planner.plan(ReturnAnswer(optimized))       ← full re-planning with real stats
        applyPhysicalRules(preprocessingRules ++ queryStagePreparationRules)
     if the new plan is better (SimpleCostEvaluator: fewer shuffles), adopt it
 until no more stages to create
 then execute the final stage
```

`reOptimize` catches `InvalidAQEPlanException` and keeps the old plan — a re-plan that
can't be validated is discarded rather than risked.

### 8.2 The three optimizations that matter

**Coalesce shuffle partitions** (`CoalesceShufflePartitions`, `ShufflePartitionsUtil`) —
with real `MapOutputStatistics`, merge adjacent reduce partitions until each is about
`spark.sql.adaptive.advisoryPartitionSizeInBytes` (64 MB). This is why
`spark.sql.shuffle.partitions` stopped mattering: set it high and let AQE coalesce down.
Implemented as an `AQEShuffleReadExec` with `CoalescedPartitionSpec`s — no data movement,
just a different reader partitioning over the same shuffle files.

**Skew join** (`OptimizeSkewedJoin`) — a partition larger than
`skewedPartitionFactor` × median and larger than `skewedPartitionThresholdInBytes`
(256 MB) is split into `PartialReducerPartitionSpec`s, and the matching partition on the
other side is *replicated* to each split. Turns one 100 GB straggler task into 40 balanced
ones. Only applies to sort-merge and shuffled-hash joins.

**Join strategy switch** (`LogicalQueryStageStrategy`, `DemoteBroadcastHashJoin`,
`ConvertSortMergeJoinToShuffledHashJoin`) — once a side's true size is known, a sort-merge
join can become a broadcast or shuffled-hash join, and a broadcast can be demoted if the
build side turned out large or contains too many empty partitions. `AQEPropagateEmptyRelation`
prunes subtrees that materialized zero rows.

Supporting rules: `OptimizeShuffleWithLocalRead` (after a broadcast conversion, read shuffle
blocks locally instead of re-shuffling), `OptimizeSkewInRebalancePartitions` (for
`REBALANCE`), `ReuseAdaptiveSubquery`, `PlanAdaptiveDynamicPruningFilters`.

Note in `optimizeQueryStage`: for the **final** stage, `AQEShuffleReadRule`s are filtered
out unless `spark.sql.adaptive.applyFinalStageShuffleOptimizations` — coalescing the final
stage changes the number of output files, which surprises writers.

### 8.3 Dynamic partition pruning (`execution/dynamicpruning/`)

Distinct from AQE but related. `PartitionPruning` (a logical rule) detects
`fact JOIN dim ON fact.p = dim.k WHERE dim.filter`, and inserts a
`DynamicPruningSubquery` on the fact table's partition column. At execution,
`PlanDynamicPruningFilters` either reuses the broadcast built for the join (free) or plans a
separate subquery (costed, gated on an estimated benefit). The fact-table scan then prunes
partitions using the actual dimension keys. Under AQE,
`PlanAdaptiveDynamicPruningFilters` does the same against materialized stages.

---

## 9. Data Sources

### 9.1 V1 (`execution/datasources/`)

`FileSourceScanExec` over a `FileIndex` (`InMemoryFileIndex` /
`CatalogFileIndex` / `MetadataLogFileIndex`), `PartitioningAwareFileIndex` handling
Hive-style partition discovery and schema inference. `FileFormat` implementations:
Parquet, ORC, Avro, JSON, CSV, text, binary.

Partition planning: files are grouped into `FilePartition`s targeting
`spark.sql.files.maxPartitionBytes` (128 MB) with `openCostInBytes` (4 MB) charged per file
so that thousands of tiny files don't become thousands of tasks. Splittability depends on
the codec (gzip is not splittable; Parquet/ORC split at row-group/stripe boundaries).

Writes go through `FileFormatWriter` + `FileCommitProtocol`
(`HadoopMapReduceCommitProtocol`, or `SQLHadoopMapReduceCommitProtocol`), coordinated by the
`OutputCommitCoordinator` (see [core internals §8](spark-core-internals.md#8-outputcommitcoordinator)).
`DynamicPartitionDataWriter` handles `partitionBy` with the sort-then-write strategy.

### 9.2 Vectorized readers

`VectorizedParquetRecordReader` decodes directly into `ColumnarBatch`/`OnHeapColumnVector`
(or `OffHeapColumnVector`), 4096 rows at a time, skipping `UnsafeRow` materialization
entirely for scans that stay columnar. `ColumnarToRowExec` inserts the transition where a
row-based operator needs it; `ApplyColumnarRulesAndInsertTransitions` manages the boundary
and is the extension point for native/GPU engines (this is how the RAPIDS and Comet plugins
attach).

### 9.3 V2 (`sql/catalyst/src/main/java/.../connector/`)

```
 TableProvider / TableCatalog
   └─ Table  (capabilities: BATCH_READ, BATCH_WRITE, MICRO_BATCH_READ,
              CONTINUOUS_READ, STREAMING_WRITE, TRUNCATE, OVERWRITE_BY_FILTER, …)
        ├─ SupportsRead  → ScanBuilder → Scan → Batch → InputPartition[] +
        │                                        PartitionReaderFactory → PartitionReader
        └─ SupportsWrite → WriteBuilder → Write → BatchWrite → DataWriterFactory → DataWriter
```

Pushdown is opt-in via mixin interfaces, and the list is long:
`SupportsPushDownFilters` / `SupportsPushDownV2Filters`, `SupportsPushDownRequiredColumns`,
`SupportsPushDownAggregates`, `SupportsPushDownLimit` / `Offset` / `TopN`,
`SupportsPushDownTableSample`, `SupportsPushDownJoin`,
`SupportsPushDownVariantExtractions`, `SupportsRuntimeFiltering` /
`SupportsRuntimeV2Filtering` (the DPP/runtime-filter hook).
Reporting flows the other way: `SupportsReportStatistics`, `SupportsReportPartitioning`
(storage-partitioned join), `SupportsReportOrdering`, `HasPartitionStatistics`.

Row-level operations (`RowLevelOperation`, `DeltaWrite`, `SupportsRowLevelOperations`) give
MERGE/UPDATE/DELETE a proper contract — the analyzer rules `RewriteMergeIntoTable`,
`RewriteUpdateTable`, `RewriteDeleteFromTable` compile SQL DML into either a
copy-on-write rewrite or a merge-on-read delta write, depending on what the source declares.

The pushdown rules themselves are `V2ScanRelationPushDown` and friends in the
"Early Filter and Projection Push-Down" optimizer batch.

---

## 10. Design Assessment (Staff+ Lens)

**What is genuinely excellent**

- **`TreeNode` + `Rule` + `RuleExecutor`.** A few hundred lines of core abstraction that has
  absorbed CBO, DSv2, AQE, streaming incrementalization, and a full procedural SQL dialect
  without structural change. That is a remarkable amount of load for that little machinery.
- **AQE.** The correct architectural response to the fact that cardinality estimation is
  undecidable in general. Re-planning against measured statistics between materialization
  boundaries turns the hardest problem in query optimization into an engineering problem.
- **The `UnsafeRow` relocatability property.** One format decision that simultaneously
  enables pointer-based sorting, serializer-free shuffle, off-heap hash maps, and spilling.
  Rarely does one representation choice pay off in that many subsystems.
- **The `HybridAnalyzer` migration strategy.** Dual-run with plan comparison and a sample
  rate is exactly how you replace a component where a silent behavior change is a data
  corruption incident.
- **Error classes.** Moving from message-string assertions to a typed condition registry
  made error messages refactorable and localizable, and made Connect's error propagation
  possible.

**Real tensions**

- **Rule-order fragility.** The preparation and AQE rule lists are dense with comments
  explaining that rule X must run after rule Y *because* Y drops an ordering that X relies
  on. These are load-bearing comments, not documentation — the ordering constraints are not
  expressible in the framework, only in prose. That is a latent source of regressions.
- **Two analyzers, 102 extra files.** Correct migration strategy, but a large and long-lived
  duplication where every new analyzer feature must be implemented twice or explicitly
  declared unsupported.
- **`QueryPlanner` pretends to be cost-based and isn't.** `plan()` returns an `Iterator` of
  alternatives and the caller takes `.next()`. Physical plan choice is a hard-coded
  preference order plus AQE fixups. Honest, but the abstraction implies something it doesn't
  deliver.
- **CBO is largely vestigial.** It requires statistics almost nobody collects, and AQE
  measures what CBO estimates. Keeping both means maintaining an estimation framework whose
  main consumer is join reordering on a threshold of 12 relations.
- **Codegen's cliff edges.** Silent fallback to interpretation past 8000 bytecodes, and
  constant-pool/method-size splitting heuristics tuned by magic numbers, mean performance is
  discontinuous in ways that are invisible from the query text. A 200-column `SELECT` can be
  slower than a 100-column one for reasons no user can be expected to predict.
- **Configuration surface.** `SQLConf` is well past a thousand entries. The defaults are
  mostly right; the problem is that when they're wrong, nothing in the failure points at the
  knob that would fix it.
