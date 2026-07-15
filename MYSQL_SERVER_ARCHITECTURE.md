# MySQL Server — Architecture & Implementation Deep Dive

Source tree: `C:\workspace\opensource\mysql-server` (trunk, **MySQL 9.7.0 LTS** per `MYSQL_VERSION`).
All paths are relative to the repo root; `file:line` references were verified against this tree.
Line numbers drift as trunk moves — treat them as anchors, not gospel.

---

## 1. Executive summary

MySQL is a **two-layer database server**:

1. **SQL layer** (`sql/`) — connection handling, wire protocol, parser, name
   resolution, optimizer, iterator executor, replication, and one `THD` object per
   connection. It knows nothing about how rows are physically stored.
2. **Storage engine layer** — pluggable engines behind the virtual `handler`
   interface (`sql/handler.h:4752`). **InnoDB** (`storage/innobase/`, ~500k LOC) is
   the only fully transactional first-class engine and is itself a complete storage
   manager: buffer pool, ARIES-style redo WAL, undo-log-based MVCC, row-level
   next-key locking, B+tree indexes, its own background thread pool.

The single most load-bearing design fact: **both layers keep independent durable
logs** — the binary log above the API, InnoDB's redo log below it. Every
transaction commit is therefore an **internal two-phase commit** coordinated by
`ha_commit_trans()` (`sql/handler.cc:1686`) with the binlog acting as the
transaction-coordinator (XA) log. Group commit, GTID assignment, crash recovery,
and replication consistency all hang off this protocol (§12).

Key modernizations that define the 8.0/9.x codebase versus the folklore version
of MySQL internals:

- **Transactional data dictionary** — no `.frm` files; all metadata lives in
  InnoDB tables in `mysql.ibd`, cached by `dd::cache::Dictionary_client` (§9).
- **Atomic DDL** — DDL writes DD changes, engine changes, and binlog in one
  transaction, with the InnoDB `DDL_LOG` doing redo/rollback of file operations.
- **Iterator executor** — execution is a Volcano-style tree of `RowIterator`s
  built from a physical-plan IR (`AccessPath`, `sql/join_optimizer/access_path.h:238`);
  the old nested-loop-only `sub_select()` interpreter is gone.
- **Two optimizers** — the traditional heuristic/greedy one (`sql/sql_optimizer.cc`)
  and a research-grade **hypergraph** join optimizer (`sql/join_optimizer/`,
  DPhyp enumeration) sharing the `AccessPath` IR (§6).
- **Lock-free redo log** (8.0) with dedicated writer/flusher/checkpointer threads
  in InnoDB (§11.6).
- **Component/service infrastructure** (`components/`) gradually replacing the
  legacy plugin API (§14.4).

Removed along the way (do not look for them): query cache (8.0),
`mysql_native_password` (9.0), `.frm`/`db.opt`/`par` metadata files, the
pre-iterator executor, `innodb_file_format` knobs.

---

## 2. Source tree map

```
mysql-server/
├── sql/                  SQL layer core (~1.5M LOC): mysqld.cc, sql_parse.cc, THD,
│   │                     items, optimizer, executor, replication (rpl_*), binlog
│   ├── conn_handler/     Accept loop + per-thread connection handler
│   ├── dd/               Transactional data dictionary (entities, cache, upgrade)
│   ├── join_optimizer/   Hypergraph optimizer + AccessPath IR (shared by both optimizers)
│   ├── iterators/        Executor: RowIterator hierarchy, hash join, sorting, windows
│   ├── range_optimizer/  Range/index-merge/skip-scan access method analysis
│   ├── histograms/       Equi-height/singleton histograms, value maps
│   ├── binlog/           Binlog subcomponents (services, monitoring)
│   ├── auth/             ACL, roles, authentication plugins interface
│   ├── locks/            Shared spin locks etc.
│   └── resourcegroups/   Resource group (CPU affinity) support
├── sql-common/           Client/server shared code (my_time, protocol pieces)
├── storage/
│   ├── innobase/         InnoDB (see §11 map)
│   ├── temptable/        In-memory engine for internal temp tables (default since 8.0)
│   ├── heap/             Legacy MEMORY engine
│   ├── myisam/, myisammrg/  Legacy non-transactional engine + MERGE
│   ├── archive/, blackhole/, csv/, federated/, example/
│   ├── ndb/              NDB Cluster engine (large, separate ecosystem)
│   ├── perfschema/       performance_schema implementation (an engine!)
│   └── secondary_engine_mock/  Mock for HeatWave-style secondary engines
├── plugin/               Legacy-API plugins: group_replication, semisync, clone,
│                         keyring, audit_log, fulltext parsers, connection_control…
├── components/           New-style components: keyrings, validate_password, logging…
├── libs/mysql/           Modern reusable libraries (binlog event reading, gtid, serialization…)
├── libbinlogevents/      Binlog event (de)serialization library (shim over libs/)
├── libchangestreams/     Change-stream (CDC) client library
├── mysys/                Portability runtime: my_alloc (MEM_ROOT), my_file, charsets glue
├── strings/              Character sets/collations, dtoa, decimal
├── vio/                  Network I/O abstraction (TCP, TLS, named pipes, shared memory)
├── include/              Public + internal headers (my_*.h, mysql/, field_types.h…)
├── client/               mysql, mysqldump, mysqlbinlog, mysql_secure_installation…
├── router/               MySQL Router (standalone routing proxy, own CMake subtree)
├── unittest/gunit/       Google Test unit tests
├── mysql-test/           MTR integration test suites (t/*.test + r/*.result)
├── cmake/                Build-system modules (compiler flags, SSL, sanitizers…)
└── scripts/              mysqld_safe, system tables bootstrap SQL generation
```

Rule of thumb for navigation: `sql/sql_<verb>.cc` implements a statement family
(`sql_select.cc`, `sql_insert.cc`, `sql_table.cc` = DDL, `sql_show.cc`);
`sql/item*.{h,cc}` is the expression tree; `sql/rpl_*` is replication;
InnoDB files are `<module><digit><name>.cc` (`buf0buf.cc`, `trx0trx.cc`) with
headers under `storage/innobase/include/`.

---

## 3. Layered architecture

```
            ┌───────────────────────────────────────────────────────────┐
 clients →  │  vio (TCP/TLS/pipe)  →  Protocol_classic (wire protocol)  │
            ├───────────────────────────────────────────────────────────┤
            │  Connection mgmt: Connection_acceptor → one THD/connection │
            │  do_command() → dispatch_command()      (sql/sql_parse.cc) │
            ├───────────────────────────────────────────────────────────┤
            │  Parser (sql_yacc.yy) → Parse Tree → LEX/Query_block AST   │
            │  Resolver (sql_resolver.cc)  — names, types, transforms    │
            │  Optimizer — traditional (sql_optimizer.cc)                │
            │            — hypergraph  (join_optimizer/)  → AccessPath   │
            │  Executor  — RowIterator tree (sql/iterators/)             │
            ├───────────────────────────────────────────────────────────┤
            │  Cross-layer services: MDL (mdl.cc) · Data Dictionary (dd/)│
            │  Binary log + internal 2PC (binlog.cc, handler.cc)         │
            │  Replication (rpl_*) · ACL (auth/) · performance_schema    │
            ├──────────────────── handler API (handler.h) ───────────────┤
            │  InnoDB │ temptable │ MyISAM │ perfschema │ NDB │ ...      │
            │  (each engine owns its files, caching, logging, recovery)  │
            └───────────────────────────────────────────────────────────┘
```

Contrast with Postgres (see `POSTGRESS_ARCHITECTURE_ANALYSIS.md`): MySQL is
**thread-per-connection in one process** (no fork; all state shared in one
address space), storage is **pluggable** rather than integrated, MVCC is
**undo-log based** (old versions reconstructed backwards from rollback segments)
rather than heap-tuple versioning with VACUUM, and logical replication via the
binlog is the primary replication mechanism rather than physical WAL shipping.

---

## 4. Process model & connection lifecycle

### 4.1 Startup

`main()` (`sql/main.cc`) → `mysqld_main()` (`sql/mysqld.cc:10451`). The sequence
that matters:

1. **Early init**: `my_init()`, load `--defaults-file` options, initialize error
   log, PSI (performance schema instrumentation) bootstraps *before* almost
   everything so later allocations are attributed.
2. `init_common_variables()` — parse all system variables, charsets, time zones.
3. `init_server_components()` — the heart of boot:
   - initialize MDL, table caches, query plugins;
   - `ha_initialize_handlerton()` for each builtin engine — **InnoDB starts here**,
     which means redo recovery, doublewrite restore, undo rollback of
     uncommitted transactions all happen inside plugin init;
   - data dictionary bootstrap (`dd::init`), upgrade if the DD version changed;
   - binary log open + **XA recovery**: scan the last binlog, collect XIDs of
     prepared-but-not-committed transactions, tell engines to commit/rollback
     (`ha_recover`) — this is the 2PC resolution pass (§12.4).
4. Network init (`network_init()`): listen sockets, named pipes/shared memory on
   Windows, admin interface.
5. `mysqld_socket_acceptor->connection_event_loop()` — the accept loop; the main
   thread becomes the acceptor.

Shutdown reverses it: signal handler thread sets abort flag → close listeners →
kick all THDs (`Global_THD_manager`, `sql/mysqld_thd_manager.cc`) → shut down
engines (InnoDB does a sharp/fuzzy checkpoint depending on
`innodb_fast_shutdown`).

### 4.2 Threading model

**One OS thread per client connection** (the default `Per_thread_connection_handler`,
`sql/conn_handler/connection_handler_per_thread.cc:246 handle_connection`). A small
**thread cache** (`thread_cache_size`) parks finished threads to be reused by new
connections, so accept → dispatch does not always pay `pthread_create`. An
enterprise-only thread-pool plugin exists; the community server has no built-in
thread pool for user connections.

Everything else is background threads: signal handler, error-log flusher, main
acceptor, event scheduler (`sql/event_scheduler.cc`), replication receiver/applier
threads (§13), and InnoDB's own fleet (§11.10). `Global_THD_manager` tracks every
`THD` for `SHOW PROCESSLIST`/`KILL`.

### 4.3 THD — the per-connection kitchen sink

`class THD` (`sql/sql_class.h:953`) is *the* context object: security context,
transaction state (`Transaction_ctx`: per-statement and per-session engine
registrations), MDL context, `Diagnostics_area` (errors/warnings), open-table
list, `LEX` (current statement AST), MEM_ROOTs, binlog caches, GTID ownership,
replication state, PSI hooks, kill flag. It inherits `MDL_context_owner` and
`Query_arena`. Any function deep in the server can reach it through
`current_thd` (thread-local, `sql/current_thd.cc`) — a pervasive, deliberate
global that makes THD lifetime bugs a classic failure mode. Locking rule:
`thd->LOCK_thd_data` protects fields other threads may read (KILL, processlist).

### 4.4 Wire protocol

Classic MySQL protocol implemented in `sql/protocol_classic.cc` +
`sql-common/`. Length-prefixed packets, sequence ids; commands are one byte
(`COM_QUERY`, `COM_STMT_PREPARE`, `COM_STMT_EXECUTE`, `COM_PING`, `COM_BINLOG_DUMP_GTID`
— the last one is how replicas subscribe, §13). Handshake/auth in
`sql/auth/sql_authentication.cc`; default plugin `caching_sha2_password`
(RSA-protected password exchange or TLS). X Protocol (protobuf, port 33060) is
`plugin/x/`. Resultsets stream row-by-row; there is no server-side cursor by
default (prepared statements can request one).

### 4.5 Command loop

```
handle_connection (conn_handler)            per-thread loop
  └─ thd_prepare_connection → login/ACL
  └─ while (!aborted) do_command(thd)       sql/sql_parse.cc:1347
       └─ read one packet (blocking read; idle connections sit here)
       └─ dispatch_command(thd, com_data)   sql/sql_parse.cc:1752
            ├─ COM_QUERY → dispatch_sql_command → parse → execute  (§5)
            ├─ COM_STMT_* → prepared-statement path (sql/sql_prepare.cc)
            └─ ... 30-odd other commands
```

`dispatch_command` also drives: statement timers, PSI statement instrumentation,
slow-query logging, per-statement DA reset — read it once end-to-end; it is the
best single map of statement lifecycle.

---

## 5. Journey of a SELECT — parse and resolve

### 5.1 Parse

`dispatch_sql_command()` → `parse_sql()` (`sql/sql_parse.cc:7208`) → Bison
grammar `sql/sql_yacc.yy` (~19k lines) with a hand-written lexer
(`sql/sql_lex.cc`). The grammar builds **Parse Tree (PT) nodes**
(`sql/parse_tree_nodes.h`, `parse_tree_items.h`) — dumb syntax objects — and a
subsequent **contextualization** pass (`PT_*::contextualize`) converts them into
the semantic AST:

- `LEX` — top-level statement descriptor (one per statement, owned by THD).
- `Query_expression` (`sql/sql_lex.h:643`) — a query expression (UNION/EXCEPT/
  INTERSECT of query blocks, plus ORDER/LIMIT); since 8.0.31 set operations form
  a proper `Query_term` tree (`sql/query_term.h`).
- `Query_block` (`sql/sql_lex.h:1179`) — one SELECT: item list, FROM tables
  (`Table_ref` chain), WHERE/HAVING conditions, GROUP/ORDER, windows. Nested
  subqueries hang off items (`Item_subselect`) forming a Query_expression tree.
- Expressions are `Item` trees (`sql/item.h:929`) — see §5.3.

Digest computation (normalized statement hash for performance_schema) happens in
the lexer. Keyword list is generated (`gen_lex_token.cc`).

### 5.2 Prepare/resolve

Every DML statement is a `Sql_cmd` subclass (`Sql_cmd_select`, `Sql_cmd_update`…);
`mysql_execute_command()` (`sql/sql_parse.cc`, giant switch) calls
`Sql_cmd_dml::execute()` → `prepare()` → `Query_block::prepare()`
(`sql/sql_resolver.cc:184`). Resolution does, in order:

1. **Table opening & locking**: `open_tables_for_query()` → `open_table()`
   (`sql/sql_base.cc`) — MDL acquisition (§10), TABLE_SHARE lookup (§8.3),
   `handler::ha_open()`. Views are expanded here (parsed from DD, merged or
   materialized).
2. **Name resolution**: bind every `Item_field` to a `Field` in an opened
   `TABLE`, resolve aliases, check privileges column-by-column.
3. **Type aggregation** and implicit casts; since 8.0 comparisons inject explicit
   `Item_typecast_*` nodes.
4. **Permanent transformations** (`apply_query_transformations`):
   - derived tables / views: **merge** into outer block or mark for
     materialization (`sql/sql_derived.cc`, plus `sql/join_optimizer/derived_keys.cc`
     adds keys to materialized derived tables);
   - IN/EXISTS subquery → **semijoin** conversion, or subquery-to-derived
     rewrite (`transform_grouped_to_derived`);
   - outer-join simplification (nullability analysis → inner join);
   - ORDER BY removal in subqueries, aggregate checks
     (`sql/aggregate_check.h` — functional-dependency analysis for
     `ONLY_FULL_GROUP_BY`, worth reading, it is a small formal system).

Prepared statements: same path, but the resolved tree must be **re-executable**
— hence `Query_arena` swapping, `Item::cleanup()` discipline, and the
rollback-of-transformations machinery for re-prepare on metadata change
(`sql/sql_prepare.cc`, `Reprepare_observer`).

### 5.3 Item & Field — the expression layer

- `Item` (`sql/item.h:929`): ~200 subclasses; the contract every one implements
  is `val_int()/val_real()/val_str()/val_decimal()/val_json()` + `is_null()`,
  `fix_fields()` (resolution), `used_tables()` bitmap (drives optimizer
  placement), `const_item()`, `walk()/transform()` (tree rewriting), and
  `print()` (used to regenerate view definitions — Items must round-trip!).
- `Field` (`sql/field.h`) is the storage-facing column: owns the byte layout in
  the record buffer (`TABLE::record[0]`), NULL bitmap handling, charset,
  `store()/val_*()` conversion matrix. `Item_field` bridges the two worlds.
- Rows move through execution **materialized in record buffers** (engine format,
  `uchar*`), not as abstract tuples: `TABLE::record[0]` is "current row"; items
  read fields lazily. Hash join/temp tables pack rows via `sql/pack_rows.h`.

`TABLE_SHARE` (`sql/table.h:731`) is the immutable per-table metadata (one per
table, refcounted, in the **table definition cache**); `TABLE` (`sql/table.h:1456`)
is a per-use instance (one per table reference per connection, pooled in the
**table cache**, `table_open_cache` instances sharded by
`table_open_cache_instances`). Each `TABLE` owns a `handler*` — the engine cursor.

---

## 6. Optimizer

MySQL ships **two optimizers** sharing one physical-plan IR:

### 6.1 The AccessPath IR

`struct AccessPath` (`sql/join_optimizer/access_path.h:238`) — a tagged union of
~40 physical operators (TABLE_SCAN, INDEX_RANGE_SCAN, REF, EQ_REF, NESTED_LOOP_JOIN,
HASH_JOIN, SORT, AGGREGATE, MATERIALIZE, WINDOW, LIMIT_OFFSET…), each with
`cost`, `num_output_rows`, and operator-specific payload. Both optimizers emit an
AccessPath tree; `CreateIteratorFromAccessPath()`
(`sql/join_optimizer/access_path.cc`) lowers it 1:1 to executable iterators.
EXPLAIN (`sql/join_optimizer/explain_access_path.cc`) renders the same tree —
`EXPLAIN FORMAT=TREE` is literally the plan, not a reconstruction.

### 6.2 Traditional optimizer (`sql/sql_optimizer.cc`)

`JOIN::optimize()` (`sql/sql_optimizer.cc:344`) — per query block:

1. **Constant/const-table detection**: tables guaranteed ≤1 row (empty, or
   eq_ref on unique key from constants) are read immediately and folded into
   constants.
2. **Condition processing**: equality propagation (`COND_EQUAL` multiple-equality
   sets), constant folding, trivial-condition removal
   (`optimize_cond`, `sql/sql_optimizer.cc`).
3. **Range analysis** (`sql/range_optimizer/`): per table, build a `SEL_TREE`
   from the condition (an interval algebra over index prefixes —
   `range_opt_param`, `get_mm_tree`), cost candidate range scans, index merge
   (union/intersect/sort-union), **skip scan** and group-min-max (loose index
   scan for `GROUP BY`/`DISTINCT`). Output: cheapest single-table access methods.
4. **Join order search** (`Optimize_table_order`, `sql/sql_planner.cc`):
   greedy/exhaustive DP over permutations, bounded by `optimizer_search_depth`
   and pruned by `prune_level` heuristic; `best_access_path()` picks per-table
   access (ref/eq_ref/range/scan/dynamic-range) given the prefix. Cost model in
   `sql/opt_costmodel.cc` + server cost constants stored in
   `mysql.server_cost`/`engine_cost` tables (`sql/opt_costconstants.cc`);
   row estimates from engine statistics (`handler::records_in_range`,
   `rec_per_key`) and **histograms** (`sql/histograms/`, equi-height/singleton,
   used for non-indexed predicate selectivity).
5. **Post-join-order**: semijoin strategy choice (FirstMatch, MaterializeScan/
   Lookup, LooseScan, DuplicateWeedout — `sql/sql_optimizer.cc` +
   `opt_hints`), attach conditions to tables, ORDER/GROUP optimization (skip
   sort if index provides order), window setup, temp-table decisions.
6. `create_access_paths()` converts the chosen `JOIN_TAB`/`QEP_TAB` plan into an
   AccessPath tree; hash join replaces block-nested-loop automatically when no
   usable join index exists (join condition extracted into equi-join predicates).

Weaknesses to know: join-order search is per-query-block (no global cross-block
optimization), selectivity model is crude (independence assumed, default guesses
like 10% for range), and decisions interleave with rewrites, making it hard to
extend — which is exactly why the hypergraph optimizer exists.

### 6.3 Hypergraph optimizer (`sql/join_optimizer/`)

Entry: `FindBestQueryPlan()` (`sql/join_optimizer/join_optimizer.cc:9889`,
inner: `:9262`). Selected via `optimizer_switch=hypergraph_optimizer=on`
(gated: available in debug builds and used in production by HeatWave; the
codepath is fully present in this tree). Design — a textbook cost-based
optimizer, cleanly separated from rewrites:

1. `MakeJoinHypergraph()` (`make_join_hypergraph.cc`): relational algebra tree →
   **hypergraph** (nodes = tables, hyperedges = join predicates; hyperedges
   capture non-inner-join reordering constraints per the Moerkotte/Neumann
   conflict-detection papers — see `RelationalExpression` and the CD-C conflict
   rules).
2. **DPhyp** (`hypergraph.cc`, `subgraph_enumeration.h`): dynamic programming
   over connected subgraphs, enumerating csg-cmp pairs; `NodeMap` bitmaps
   (`node_map.h`, 64-bit; `OverflowBitset` beyond 61 tables).
3. **Graph simplification** (`graph_simplification.cc`): when the DP space
   exceeds `optimizer_max_subgraph_pairs`, iteratively constrain the graph
   (forcing cheap-looking join orders) until enumeration is tractable —
   a principled replacement for greedy search.
4. **Interesting orders** (`interesting_orders.cc`): orders/groupings tracked
   through a precomputed DFSM (logical→physical ordering states), enabling
   sort-ahead, sort elimination via functional dependencies — based on
   Neumann/Moerkotte's ordering framework; the file header is a good read.
5. Per-subplan candidates kept in a **Pareto set** (cost × output order ×
   parameterized-ness, `compare_access_paths.h`); costs from a new, more
   uniform cost model (`cost_model.cc`, `cost_constants.h`) and selectivity
   estimation (`estimate_selectivity.cc`) that actually uses histograms and
   index dives uniformly.
6. `FinalizePlanForQueryBlock()` (`finalize_plan.cc`) — late materialization of
   temp tables, moving expressions, attaching filters.

Read `join_optimizer.cc`'s 200-line header comment — it is the design doc.

### 6.4 EXPLAIN & optimizer trace

- `EXPLAIN [FORMAT=TRADITIONAL|JSON|TREE]` (`sql/opt_explain*.cc`);
  `EXPLAIN ANALYZE` wraps iterators in `TimingIterator` (§7) — actual rows/loops/
  time per operator.
- Optimizer trace (`SET optimizer_trace='enabled=on'`, `sql/opt_trace.cc`) dumps
  the full decision log as JSON; hypergraph has its own trace
  (`join_optimizer/optimizer_trace.cc`).

---

## 7. Executor

### 7.1 Iterator model

Execution = pull-based Volcano tree of `RowIterator`s
(`sql/iterators/row_iterator.h`): `Init()` once, `Read()` per row (0 = row in
`table->record[0]` / item buffers, -1 = EOF, 1 = error). Driven by
`Query_expression::ExecuteIteratorQuery()` (`sql/sql_union.cc:1036`):
`for (;;) { if (iterator->Read()) break; query_result->send_data(...); }`.

Key iterators (`sql/iterators/`):

- **Scans** (`basic_row_iterators.h`): `TableScanIterator`, `IndexScanIterator`,
  `IndexRangeScanIterator` (wraps range-optimizer quick selects),
  `FollowTailIterator` (recursive CTEs).
- **Lookups** (`ref_row_iterators.h`): `RefIterator`, `EQRefIterator` (with
  1-row cache), `RefOrNullIterator`, `DynamicRangeIterator` (re-plans range per
  outer row), `FullTextSearchIterator`.
- **Joins**: `NestedLoopIterator` (inner/outer/anti/semi via `JoinType`),
  `HashJoinIterator` (`hash_join_iterator.cc`) — build side hashed into memory
  (`hash_join_buffer.cc`, rows packed via `pack_rows.h`); on overflow spills to
  **chunk files** (`hash_join_chunk.cc`, Grace hash join with per-chunk
  probe matching), degrades gracefully for LEFT/anti joins; `BKAIterator`
  (`bka_iterator.cc`) — batched key access feeding MRR (§8.2).
- **Composites** (`composite_iterators.cc`): `FilterIterator`,
  `AggregateIterator` (relies on input grouped order or feeds from temp table),
  `MaterializeIterator` (UNION/derived/CTE materialization, with de-dup by
  hash index on the temp table; shared CTE scan via `CacheInvalidatorIterator`),
  `TemptableAggregateIterator`, `StreamingIterator`, `LimitOffsetIterator`,
  `NestedLoopSemiJoinWithDuplicateRemovalIterator`, `WeedoutIterator`.
- **Sorting** (`sorting_iterator.cc` + `sql/filesort.cc`): filesort with
  fixed/variable-length sort keys (`Sort_param`), in-memory quicksort or
  merge-sort over disk runs (`merge_many_buff.h`), optional packed addon fields
  vs row-id sorting (row-id → requires re-fetch via `SortFileIndirectIterator`);
  PQ optimization for `ORDER BY … LIMIT n` (`bounded_queue.h`).
- **Windows** (`window_iterators.cc`): buffering vs streaming window evaluation,
  frame navigation in `sql/window.cc`.
- `TimingIterator` (`timing_iterator.h`) wraps any iterator for
  `EXPLAIN ANALYZE` (template, near-zero cost when unused).

DML goes through the same machinery: `UpdateRowsIterator`,
`DeleteRowsIterator` sit at the plan root (multi-table update/delete included).

### 7.2 Internal temp tables

`sql/sql_tmp_table.cc` — created for materialization, GROUP BY without index,
UNION/DISTINCT, window buffering. Engine: **TempTable** (`storage/temptable/`,
default) — row format optimized in-memory engine with mmap spill
(`temptable_max_ram`, `temptable_max_mmap`); falls back to InnoDB on-disk
(`internal_tmp_disk_storage_engine` is gone — always InnoDB on disk now; BLOBs
handled by TempTable since 8.0.23). Overflow path: if a temp table exceeds
memory limits mid-write, it is **converted in place** to the disk engine.

---

## 8. Storage-engine (handler) API

### 8.1 handlerton & handler

- `handlerton` (`sql/handler.h` ~2700) — one per engine, the **static** vtable:
  `commit`, `rollback`, `prepare`, `recover`, `create`, `ddse_*` (DD storage
  hooks), `clone_*`, `redo_log_set_state`, notify_exclusive_mdl, cost
  constants… Engines register it via the plugin init (`ha_innodb.cc:
  innodb_init`).
- `handler` (`sql/handler.h:4752`) — one instance per open `TABLE` per
  connection, the **cursor/vtable** for row access. Core virtuals every engine
  implements:
  - table: `open/close`, `rnd_init/rnd_next/rnd_pos/position` (full scan +
    row-id refetch), `info()` (statistics into `ha_statistics`);
  - index: `index_init/index_read_map/index_next[_same]/index_prev/
    index_first/index_last`, `records_in_range()` (range cost input);
  - write: `write_row/update_row/delete_row`, bulk hints
    (`start_bulk_insert`);
  - DDL: `create`, `delete_table`, `rename_table`, in-place ALTER family
    (`check_if_supported_inplace_alter`, `prepare/inplace_alter_table`,
    `commit_inplace_alter_table`) — this is how online DDL is negotiated;
  - transactions: engines don't see BEGIN; they see
    `external_lock(F_RDLCK/F_WRLCK/F_UNLCK)` per statement +
    `trans_register_ha()` self-registration; commit/rollback arrive via
    handlerton (§12).
  - `extra(HA_EXTRA_*)` — a grab-bag of ~80 hint enums (keyread, batched
    ops, semi-consistent read…); grep `ha_extra_function`.
- The `ha_` prefixed non-virtual wrappers (`handler::ha_write_row` etc.) add PSI
  instrumentation and invariant checks around the engine virtuals — call those,
  never the raw virtuals.

### 8.2 Optimizer↔engine contracts

- Statistics: `info(HA_STATUS_*)` fills `stats.records`, `rec_per_key` per index
  (InnoDB: from `innodb_stats_persistent` sampled dives, stored in
  `mysql.innodb_table_stats`/`innodb_index_stats`).
- `records_in_range()` — live B-tree dives per candidate range (bounded by
  `eq_range_index_dive_limit`).
- **MRR** (multi-range read): `multi_range_read_init/next` — engine may
  reorder key lookups; InnoDB's DS-MRR sorts row ids for sequential heap
  access; BKA feeds it whole join batches.
- **ICP** (index condition pushdown): `idx_cond_push()` — engine evaluates a
  pushed `Item` on index entries before fetching the row.
- **Engine condition pushdown** and aggregates pushdown exist mainly for
  NDB/secondary engines (`engine_push`).
- Secondary-engine hooks (`secondary_engine` flags, `prepare_secondary_engine`)
  — the HeatWave integration surface; `storage/secondary_engine_mock/` shows
  the contract.

### 8.3 Table caches

Open path (`sql/sql_base.cc: open_table`): MDL lock → **table definition
cache** lookup (`get_table_share`, shares keyed by db+name, LRU-evicted,
`table_definition_cache`) → share constructed from the **data dictionary**
(`sql/dd_table_share.cc` builds TABLE_SHARE from `dd::Table`) → **table cache**
(per-instance sharded `Table_cache`, `sql/table_cache.cc`) yields a free
`TABLE`, else `open_table_from_share()` + `handler::ha_open()`. FLUSH TABLES /
DDL invalidate via share versioning + MDL.

---

## 9. Transactional data dictionary (`sql/dd/`)

Since 8.0 there are **no `.frm` files**. All catalog metadata is rows in ~30
InnoDB tables living in the hidden `mysql` tablespace (`mysql.ibd`), defined in
`sql/dd/impl/tables/*.cc` (tables, columns, indexes, schemata, tablespaces,
triggers, events, routines, character_sets…). You cannot query them directly —
`INFORMATION_SCHEMA` is implemented as **views over DD tables**
(`sql/dd/impl/system_views/`), which made I_S fast (it used to open every table).

- Object model: `dd::Table`, `dd::Index`, `dd::Column`… (`sql/dd/types/`),
  POD-ish entities serialized to/from the DD tables via a small ORM
  (`sql/dd/impl/raw/`, `object_key`s).
- **Dictionary_client** (`sql/dd/cache/dictionary_client.h`) — per-THD client
  with an `Auto_releaser` scope; backed by a shared cache
  (`Shared_dictionary_cache`) with per-object-type maps. Uncommitted-object
  registry supports DDL reading its own uncommitted changes.
- **SDI** (serialized dictionary information): every tablespace also stores a
  JSON copy of its tables' metadata (`sql/dd/impl/sdi.cc`, InnoDB stores it in
  internal B-trees; `ibd2sdi` utility reads it) — makes `.ibd` files
  self-describing for IMPORT TABLESPACE and disaster recovery.
- **Atomic DDL**: a DDL statement = one DD transaction + engine file ops made
  crash-safe by InnoDB's `DDL_LOG` (`storage/innobase/log/log0ddl.cc`) —
  a persistent log of file create/delete/rename ops replayed or rolled back at
  recovery. Post-DDL hooks run after commit. The binlog entry and DD commit are
  atomic via the same 2PC used for DML (§12) — this is what killed the old
  "half-executed ALTER" class of bugs.
- Versioning: DD has its own version + upgrade machinery
  (`sql/dd/upgrade/`, `dd::info_schema::*` versions) that runs during startup
  when a newer server opens an older data directory.

---

## 10. Metadata locking (MDL) (`sql/mdl.cc`, `sql/mdl.h`)

`MDL_context` (`sql/mdl.h:1415`) per THD; keys are (namespace, schema, name)
where namespace ∈ {GLOBAL, SCHEMA, TABLE, FUNCTION, TRIGGER, EVENT, TABLESPACE,
BACKUP_LOCK, ACL_CACHE…}. Lock types form a lattice from `MDL_INTENTION_EXCLUSIVE`
through `MDL_SHARED_READ`/`WRITE`, `MDL_SHARED_UPGRADABLE` (DDL's starting
point), `MDL_SHARED_NO_WRITE`, up to `MDL_EXCLUSIVE`; durations
STATEMENT/TRANSACTION/EXPLICIT (LOCK TABLES, FLUSH TABLES WITH READ LOCK).

Implementation points:

- Lock table sharded (`mdl_locks_hash_partitions`); each `MDL_lock` keeps
  granted/waiting bitmaps + a **fast path** for common DML types (lock-free
  atomic counters packed into a 64-bit word, "unobtrusive" locks) so concurrent
  SELECT/DML on the same table never serialize on the MDL mutex.
- **Deadlock detection**: on wait, walk the wait-for graph across MDL *and*
  table-level locks (`Deadlock_detection_visitor`), victim chosen by weight
  (DDL outweighs DML). This is separate from InnoDB's row-lock deadlock
  detector (§11.8) — the two graphs are bridged only by timeouts and by
  `thd_report_lock_wait` notifications.
- Statement-level savepoints of the MDL context enable rollback-to-savepoint.
- `LOCK INSTANCE FOR BACKUP` = BACKUP_LOCK namespace — blocks file-changing
  ops while allowing DML; how hot backup tools coordinate.

---

## 11. InnoDB deep dive (`storage/innobase/`)

### 11.1 Module map

```
api/     embedded-style cursor API      lock/  row & table locks, deadlock det.
arch/    page/log archiver (for clone)  log/   redo log write/flush/checkpoint/recovery + DDL_LOG
btr/     B+tree ops, cursors, AHI(btr0sea)  mach/  little-endian on-page encodings
buf/     buffer pool, LRU, flushing, dblwr  mem/   memory heaps (arena)
clone/   physical clone (donor/recipient)   mtr/   mini-transactions
data/    in-memory tuple types (dtuple)     os/    async I/O, files, events
ddl/     parallel index build (8.0.27+)     page/  page format, page cursor
dict/    in-memory dictionary + dd bridge   pars/  internal SQL parser (legacy, used by fts)
eval/    interpreter for pars/              que/   query graph runtime (legacy internal)
fil/     tablespace/file layer              read/  read views (MVCC snapshots)
fsp/     space/extent allocation            rem/   record formats + comparators
fts/     full-text index engine             row/   row operations: search/ins/upd/del/undo/vers/log
fut/     file-based utilities/lists         srv/   server glue, background threads, monitor
gis/     R-tree (spatial)                   sync/  latches: rw-locks, mutexes, sharded counters
ha/      hash tables                        trx/   transactions, undo, rollback segs, purge, read views
handler/ ha_innodb.cc (5MB!) + inplace DDL  usr/   sessions
ibuf/    change buffer (deprecated)         ut/    utilities (rnd, lists, crc32…)
include/ ALL headers (foo0bar.h + .ic inline files)
```

Entry point for everything SQL-side: `storage/innobase/handler/ha_innodb.cc`
(`ha_innobase : public handler`, `innodb_init()` wires the handlerton), online
DDL in `handler0alter.cc`.

### 11.2 On-disk layout: spaces → extents → pages → records

- **Tablespaces** (`fil/fil0fil.cc`): system (`ibdata1` — dblwr legacy,
  change buffer), **file-per-table `.ibd`** (default), general tablespaces,
  `mysql.ibd` (DD), undo tablespaces (`undo_001…`, dynamic since 8.0.14),
  temp (`ibtmp1` + session temp `#innodb_temp/`). `fil_space_t`/`fil_node_t`;
  space id → file mapping; a sharded `Fil_shard` system manages open file
  handles (`innodb_open_files`).
- **Pages**: `innodb_page_size` default **16KB**. Extent = 1MB (64×16K pages).
  `fsp/fsp0fsp.cc` allocates: space header (page 0) → segment inodes (`fseg`) →
  extents (XDES descriptors); small segments start with 32 fragment pages
  before switching to whole extents. Every index gets two segments (non-leaf +
  leaf).
- **Page format** (`page/page0page.h`): 38-byte FIL header (checksum, page no,
  prev/next — leaf pages form a doubly-linked list per level, LSN, page type,
  space id) · page header (dir slots, heap top, n_recs, PAGE_LEVEL,
  PAGE_INDEX_ID…) · infimum/supremum system records · record heap growing up ·
  sparse **page directory** growing down (slots own 4–8 records; search =
  binary search slots, then linear) · 8-byte FIL trailer (checksum + LSN low
  bits; torn-page detection).
- **Record format** (`rem/rem0rec.cc`, COMPACT/DYNAMIC): variable-length
  header: nullable bitmap + var-len length bytes (reversed, before the record
  origin), 5-byte fixed header (record type, heap no, n_owned, delete-mark bit,
  next-record offset — records within a page are a singly linked list in key
  order). Clustered index leaf row = PK cols + **DB_TRX_ID** (6B) +
  **DB_ROLL_PTR** (7B) + remaining cols; secondary leaf = key cols + PK cols
  (no trx id → the MVCC dance in §11.7). Long columns overflow to off-page
  LOB pages (`lob/`, with 8.0 partial-update-aware LOB indexes: zlob/lob
  first/data/index pages).

### 11.3 Indexes: everything is a B+tree

Table = **clustered index** on PK (or first unique not-null key, or hidden
6-byte `DB_ROW_ID`). Secondary indexes store PK as the row pointer. `btr/`:

- `btr_cur_search_to_nth_level()` (`btr/btr0cur.cc`) — root-to-leaf descent;
  latching follows "latch coupling" with the **index latch** (`dict_index_t::lock`)
  + per-page rw-latches; optimistic descent latches only the leaf,
  pessimistic (structure-modifying) latches the whole subtree via mtr.
- SMOs: page split (90-10 or middle, `btr_page_split_and_insert`), merge on
  underflow (<50% with neighbor), tree height grows at the root (root page id
  never changes — it's in the DD/dict).
- **Persistent cursors** (`btr0pcur`): store position, restore after latch
  release — how row_search resumes between handler calls.
- **AHI** (adaptive hash index, `btr/btr0sea.cc`): hash on frequently-used
  key prefixes → leaf record pointer, built per-index heuristically, sharded
  by `innodb_adaptive_hash_index_parts`; **default OFF since 8.4** (scalability
  regressions under contention beat its lookup wins).
- R-tree for spatial (`gis/`), same page infrastructure with MBR keys.

### 11.4 Buffer pool (`buf/buf0buf.cc`)

`buf_pool_t` (`include/buf0buf.h:2293`) × `innodb_buffer_pool_instances`
(page id hashes to instance). Per instance:

- **page_hash**: (space,page) → `buf_page_t`, sharded rw-locked hash.
- **LRU list** with **midpoint insertion** (`innodb_old_blocks_pct`, default
  3/8 old): new reads land at the old/young boundary; a page moves to the
  young half only if touched again after `innodb_old_blocks_time` (25ms) —
  scan resistance. `buf_page_make_young`, young/old counters in
  `SHOW ENGINE INNODB STATUS`.
- **free list** + LRU eviction (single-page flush by user threads is a last
  resort; normally page cleaners keep free lists fed).
- **flush list**: dirty pages ordered by **oldest_modification LSN** — the
  structural link between buffer pool and checkpointing (§11.6): checkpoint
  LSN = min(oldest_modification) across instances.
- Reads: `buf_page_get_gen()` (`buf/buf0buf.cc:4439`) — hash lookup, pin
  (`buf_fix_count`), possible sync read via async I/O completion; **linear
  read-ahead** (`buf0rea.cc`, `innodb_read_ahead_threshold`) and random
  read-ahead (off by default).
- Writes are **never in-place at read time**: dirty pages flushed by **page
  cleaner coordinator + workers** (`buf/buf0flu.cc`), adaptive flushing driven
  by redo-generation rate and checkpoint age (`page_cleaner` loop, target:
  keep checkpoint age < async limits derived from `innodb_redo_log_capacity`).
- **Doublewrite buffer** (`buf/buf0dblwr.cc`): since 8.0.20 in standalone
  `#ib_16384_*.dblwr` files; batched: pages first written+fsynced to dblwr,
  then to their real locations — torn-page protection matching the 16K page vs
  4K sector mismatch. Recovery checks dblwr copies for torn pages before redo.
- Resizable online (`innodb_buffer_pool_size`, chunked); dump/restore of page
  ids across restart (`buf0lru` + `ib_buffer_pool` file).

### 11.5 Mini-transactions (`mtr/mtr0mtr.cc`, `include/mtr0mtr.h:177`)

The atomicity quantum of the storage layer. An `mtr_t`:

1. accumulates **latches** (page/index/file latches held until commit) and
2. accumulates **redo records** (m_log, a growable buffer) for every page
   modification made through `mach_write_*`/`mlog_*`.

`mtr_commit()` → `log_buffer_reserve()` (atomically reserve LSN range in the
**lock-free log buffer**) → copy redo → release latches → add dirtied pages to
flush list stamped with end-LSN. Rules that make recovery correct: a page's
newest_modification = mtr end LSN; page can't be flushed while an mtr holds it;
WAL: redo must reach disk before the dirty page does (enforced via
`buf_flush_note_modification` + log_flusher ordering). Multi-page operations
(page split!) are one mtr → crash-atomic.

### 11.6 Redo log (`log/log0log.cc`, `log0write.cc`, `log0chkp.cc`)

8.0 redesigned this to be **lock-free for writers**:

- Writers reserve LSN intervals via atomic fetch-add on `log.sn` (sn = LSN
  minus header bytes), copy records into the ring **log buffer** concurrently,
  and mark completion in the **recent_written** link buffer (`Link_buf`) —
  a sliding-window structure that lets the writer thread discover the maximal
  contiguous LSN without locks. Same trick (`recent_closed`) orders
  flush-list insertion.
- Dedicated threads (`log0write.cc`): **log_writer** (log buffer → OS page
  cache, tracks `write_lsn`), **log_flusher** (fsync, `flushed_to_disk_lsn`),
  **log_write_notifier**/**log_flush_notifier** (wake user threads waiting on
  LSN milestones), **log_checkpointer** (`log0chkp.cc`, writes checkpoint
  records; checkpoint LSN = min oldest_modification − margin).
- `innodb_flush_log_at_trx_commit`: 1 = wait for flushed_to_disk_lsn ≥ commit
  LSN (durable), 2 = wait for write only, 0 = neither (background once/sec).
- Files: 32 pre-allocated `#innodb_redo/#ib_redoN` files consumed round-robin;
  total capacity governed **dynamically** by `innodb_redo_log_capacity`
  (8.0.30+; resizer thread grows/shrinks live).
- Record format: `mlog_id_t` typed records (MLOG_REC_INSERT,
  MLOG_COMP_PAGE_CREATE…), space+page id, body; **physiological logging**
  (physical to a page, logical within it).
- **Recovery** (`log/log0recv.cc`): find last checkpoint → parse records into
  a per-page hash (`recv_sys`) → apply in page-no order with read-ahead
  batches (pages may be read via dblwr-repaired copies) → redo complete, then
  undo phase: resurrect prepared/active trxs from undo (`trx0roll.cc`
  background-rolls back non-prepared ones). InnoDB always runs this at
  startup — "crash recovery" is just the non-empty case.

### 11.7 Undo & MVCC (`trx/`, `read/`, `row/row0vers.cc`)

- **trx_t** (`include/trx0trx.h:675`). Lifecycle
  (`trx_start_low` → `trx_commit()` at `trx/trx0trx.cc:2229`): transactions are
  lazily started; read-only transactions with no locks are **non-registered**
  (auto-commit non-locking SELECT never allocates a trx id — huge scalability
  win; `trx->id = 0` until first write).
- **Undo logs**: per-write-trx undo segments in undo tablespaces' **rollback
  segments** (`trx0rseg`, 128 rsegs × `innodb_rollback_segments`); undo
  records (`trx0rec`) store the before-image: insert-undo (only for rollback,
  purged at commit) vs update-undo (needed by MVCC, kept until purge).
  `DB_ROLL_PTR` in each clustered row points at the undo record that recreates
  the previous version → **version chains live in undo, not in the table**.
- **ReadView** (`include/read0types.h:48`, `read/read0read.cc`): snapshot =
  {low_limit_id (next trx id), up_limit_id (min active), m_ids (active list),
  creator id}. Visibility: trx_id < up_limit → visible; ≥ low_limit or in
  m_ids → invisible → follow roll_ptr chain in `row_vers_build_for_consistent_read`
  reconstructing older versions until visible. REPEATABLE READ: one view per
  trx; READ COMMITTED: view per statement. Views live in an MVCC list
  (`read0read.cc`) so purge knows the oldest.
- **row_search_mvcc()** (`row/row0sel.cc:4431`) — the ~1500-line function
  every SELECT row passes through: persistent-cursor positioning, ICP,
  visibility check + version building, locking-read vs consistent-read split,
  **secondary-index trick**: secondary records have no trx id, only a per-page
  max_trx_id — if it's too new, fetch the clustered record and version-check
  there; delete-marked handling; prefetch cache (`prebuilt->fetch_cache`,
  8-row batches into the handler).
- **Purge** (`trx/trx0purge.cc`): coordinator + `innodb_purge_threads` workers
  consume update-undo of committed trxs older than the oldest ReadView:
  physically remove delete-marked records, truncate undo segments, truncate
  undo tablespaces (`innodb_undo_log_truncate`). Long-running snapshots ⇒
  purge lag ⇒ history list length growth ⇒ version-chain walks slow everything
  — the classic InnoDB pathology.

### 11.8 Locking (`lock/lock0lock.cc`)

- Row locks attach to (space,page) in a global sharded **lock_sys hash**;
  one `lock_t` per (trx, page, mode) with a **bitmap over heap numbers** —
  a lock on 100 rows of a page is one object. Modes: S/X; types:
  `LOCK_REC_NOT_GAP` (record only), `LOCK_GAP` (gap before record),
  **next-key** (both) — gap/next-key locks exist purely to block phantoms in
  REPEATABLE READ and for binlog-safe statement replication;
  READ COMMITTED uses only record locks + semi-consistent reads.
- Table locks: intention modes IS/IX + S/X + AUTO-INC (special table lock;
  `innodb_autoinc_lock_mode=2` interleaved is default with row binlog).
- Acquisition: `lock_rec_lock()` (`lock/lock0lock.cc:1864`) — fast path: no
  conflict → set bit; conflict → enqueue waiting lock, `lock_wait_suspend_thread`
  with `innodb_lock_wait_timeout`.
- **Implicit locks**: a newly written row is *not* explicitly locked — writers
  detect conflict via trx id activity check (`lock_rec_convert_impl_to_expl`)
  and materialize the lock lazily. Saves enormous lock memory on bulk insert.
- **Deadlock detection**: on each wait, DFS over wait-for graph
  (`Deadlock_checker` / since 8.0.18 a dedicated background thread computes
  and picks victims by weight — smallest undo+locks); `innodb_deadlock_detect=off`
  falls back to timeouts for extreme-contention workloads.
- CATS (contention-aware lock scheduling, 8.0.20+): grant order by trx age
  under contention instead of pure FIFO.

### 11.9 Change buffer & other caches

- **ibuf** (`ibuf/ibuf0ibuf.cc`): buffers secondary-index modifications for
  pages not in the buffer pool into a system-tablespace B-tree, merged on later
  page read or by background merge. **Deprecated; default
  `innodb_change_buffering=none` since 8.4** — modern SSDs + large pools made
  its bookkeeping and corruption surface a bad trade. Know it exists for
  reading old data dirs.
- Per-index **persistent statistics** (`dict0stats.cc`) sampled by dives,
  recalced by `dict_stats_thread` after 10% churn.

### 11.10 Background thread fleet (srv/)

`srv_master_thread` (`srv/srv0srv.cc`, once/sec housekeeping: ibuf merge, dict
stats, DDL log cleanup), page cleaners + LRU managers, log_writer/flusher/
checkpointer/notifiers (§11.6), purge coordinator+workers, `srv_monitor_thread`
(SHOW ENGINE INNODB STATUS deadlock/monitor output), `buf_resize_thread`,
`fts_optimize_thread`, GTID persister flush (`clone/arch` threads when active),
async I/O: `innodb_read/write_io_threads` completing `os0aio` requests
(Windows: native async I/O; Linux: libaio/io_uring where enabled).

### 11.11 Online / parallel DDL

- In-place ALTER (`handler/handler0alter.cc`): negotiates
  INSTANT (metadata-only: add/drop column instant since 8.0.29 with row
  version counters in records) → INPLACE (rebuild or not; concurrent DML
  captured in **row log** (`row0log.cc`) and applied at the end under brief
  X-MDL) → COPY fallback.
- Parallel index build (`ddl/`, 8.0.27+): parallel scan (partition B-tree by
  ranges) → per-thread sort runs → parallel merge + B-tree bulk build
  (`innodb_ddl_threads`, `innodb_ddl_buffer_size`).
- Parallel read infrastructure (`row0pread.cc`) also serves
  `CHECK TABLE`/COUNT(*) (`innodb_parallel_read_threads`).

---

## 12. Commit pipeline: binlog + internal 2PC + group commit

The crown jewels. Two durable logs must agree: the **binary log** (source of
truth for replication and point-in-time recovery) and **InnoDB redo**.

### 12.1 The protocol

`ha_commit_trans()` (`sql/handler.cc:1686`), for a transaction touching a
transactional engine with binlog enabled:

```
1. PREPARE phase
   ha_prepare_low() → for each registered engine: hton->prepare()
     InnoDB: trx_prepare() — writes PREPARE state + XID into the undo segment,
     flushes redo per innodb_flush_log_at_trx_commit
2. COMMIT phase — owned by the binlog as transaction coordinator (TC_LOG):
   MYSQL_BIN_LOG::commit() (sql/binlog.cc:7107)
     → ordered_commit()      (sql/binlog.cc:7886)
```

### 12.2 Group commit — 3 pipelined stages

`ordered_commit()` implements **binlog group commit** via
`Commit_stage_manager` (`sql/rpl_commit_stage_manager.cc`). Threads queue into a
stage; the **first becomes leader**, processes everyone's work, followers just
wait — classic leader/follower batching, pipelined so three groups can be in
flight:

- **FLUSH stage**: leader asks InnoDB to flush redo up to the group's max
  prepare-LSN **once for the whole batch** (`ha_flush_logs`), then flushes each
  THD's **binlog cache** (per-THD `binlog_cache_mngr`: stmt + trx caches,
  spill to temp file above `binlog_cache_size`) into the binlog file buffer.
  **GTIDs are assigned here**, in queue order — this is why commit order ==
  binlog order == GTID order.
- **SYNC stage**: fsync the binlog per `sync_binlog` (1 = every group —
  durable; N>1 = every N groups). `binlog_group_commit_sync_delay` can
  deliberately widen groups.
- **COMMIT stage**: leader calls `ha_commit_low()` for each member **in queue
  order** (when `binlog_order_commits=1`) → InnoDB `trx_commit()`
  (`trx/trx0trx.cc:2229`): mark committed in undo, release locks, register
  redo — but here the redo flush can be skipped (already durable via binlog +
  recovery rule below). Semi-sync ACK wait hooks into after-sync/after-commit
  (per `rpl_semi_sync_source_wait_point`).

Crash-safety invariant: **a transaction whose XID event made it into a
synced binlog is committed; anything prepared but not in the binlog rolls
back.**

### 12.3 Engine-only / binlog-less cases

Binlog disabled → InnoDB commits directly (still 2PC-free single phase, redo
flush at commit). Multiple transactional engines without binlog → `TC_LOG_MMAP`
(a small mmap'd XID log) coordinates. `--log-replica-updates=0` appliers:
binlog off for the applier thread → engine-direct commit path.

### 12.4 Recovery as the other half of 2PC

Startup (`MYSQL_BIN_LOG::open` → `binlog_recover`): scan the last binlog for
XIDs whose commit was recorded; `ha_recover()` collects each engine's prepared
XIDs (InnoDB: scan undo for PREPARED trxs) → commit those found in binlog,
roll back the rest. Explicit `XA PREPARE` transactions survive as PREPARED for
the user to resolve (state also carried via GTID-aware XA since 8.0.29+
one-phase applier improvements).

---

## 13. Replication

### 13.1 Binary log & events

`sql/binlog.cc` (~13k lines) + event definitions shared with client tools in
`libs/mysql/binlog/event/` (a.k.a. libbinlogevents). File = magic + sequence of
events (each: timestamp, type, server_id, size, next-pos, flags). Formats: ROW
(default; before/after row images per `binlog_row_image`), STATEMENT
(unsafe-statement detection machinery in `sql/rpl_*`), MIXED.
Typical trx in ROW: `GTID_EVENT` → `QUERY(BEGIN)` → `TABLE_MAP`* →
`WRITE/UPDATE/DELETE_ROWS`* → `XID_EVENT`. Compression: `binlog_transaction_compression`
(ZSTD payload events). Encryption: `binlog_encryption` with keyring.
Index file (`binlog_index.cc`) tracks segments; purge by
`binlog_expire_logs_seconds`.

### 13.2 GTID machinery (`sql/rpl_gtid_*.cc`)

`Gtid` = TSID (source UUID [+ tag since 8.4]) : sequence-no.
`Gtid_set` — interval-compressed per-TSID sets (the workhorse type;
`rpl_gtid_set.cc`); `Gtid_state` — global `gtids_executed` / `gtids_owned`.
Assignment at flush stage (§12.2); persisted in `mysql.gtid_executed` table
(written by InnoDB **during commit** for binlog-less appliers, compressed by a
background thread `rpl_gtid_persist.cc`). `GTID_MODE` state machine supports
online enablement. Replica start: `MASTER_AUTO_POSITION` sends its
`gtid_executed` set in `COM_BINLOG_DUMP_GTID`; source skips what's already
there.

### 13.3 Source side

Dump thread per replica: `rpl_binlog_sender.cc` — reads binlog files
(`binlog_reader.cc` abstraction) and streams events, waiting on the binlog's
update condition at EOF. Semi-sync (`plugin/semisync/`): source waits for N
replica ACKs (`rpl_semi_sync_source_wait_for_replica_count`) at after-sync
(lossless) before making the trx visible.

### 13.4 Replica side

Per-channel (`Multisource_info`, `rpl_msr.cc`) pair of services:

- **Receiver (I/O) thread** (`handle_slave_io`, `rpl_replica.cc` ~11k lines):
  connects (`rpl_mysql_connect.cc`), pulls events, writes **relay log** (same
  format as binlog), updates `mysql.slave_relay_log_info`-adjacent metadata
  (`Master_info`, crash-safe in `mysql.slave_master_info` per
  `SOURCE_INFO_REPOSITORY`, table-only since 8.3). Async connection failover
  (`rpl_async_conn_failover*.cc`) auto-reconnects to alternate sources.
- **Applier (SQL) thread** + **MTS workers** (`Relay_log_info`, `rpl_rli.cc`;
  workers `rpl_rli_pdb.cc`): coordinator reads relay log
  (`rpl_applier_reader.cc`), assigns transactions to
  `replica_parallel_workers` by **LOGICAL_CLOCK** (source-side group-commit
  intervals: events carry `last_committed`/`sequence_number` — trxs whose
  intervals overlap ran concurrently on source ⇒ safe in parallel) enhanced by
  **WRITESET** dependency tracking (source hashes each row's keys,
  `binlog_transaction_dependency_tracking=WRITESET` collapses dependencies to
  actual row overlap — massive parallelism gain). `rpl_mta_submode.cc` has the
  scheduling; gaps tracked for crash-safe `relay_log_recovery`;
  `replica_preserve_commit_order` uses the **Commit_order_manager**
  (`rpl_replica_commit_order_manager.cc`) so workers commit in source order.
  Applier events apply rows via the same handler API
  (`rpl_record.cc` unpacks row images; `Rows_log_event::do_apply_event` in
  `log_event.cc`, hash/index scans to find target rows).

### 13.5 Group Replication (`plugin/group_replication/`)

Paxos-based virtually-synchronous group membership: **XCom** (Mencius-family
consensus, `libmysqlgcs/src/bindings/xcom/`) totally orders trx write-set
certification messages. On commit, a trx broadcasts (write set hashes +
gtid_executed snapshot version); every member runs deterministic
**certification** (first-committer-wins on write-set conflicts) → commit or
rollback. Single-primary (default) or multi-primary. Flow control paces
writers to the slowest certifier/applier. It plugs into the server via
replication channels (`group_replication_applier`) and the same MTS applier.
MySQL InnoDB Cluster = GR + MySQL Router (§14.6) + Shell.

### 13.6 Clone (`plugin/clone/` + `storage/innobase/clone/`)

Physical snapshot transfer for provisioning replicas/GR members:
page-archiving + redo-archiving (`arch/`) let a donor stream a consistent
snapshot while active; recipient applies + performs recovery; finishes with
GTID position for CHANGE REPLICATION SOURCE. `CLONE INSTANCE FROM …`.

---

## 14. Cross-cutting infrastructure

### 14.1 Memory: MEM_ROOT everywhere

`MEM_ROOT` (`mysys/my_alloc.cc`, `include/my_alloc.h`) — arena/region
allocator: geometric block growth, pointer-bump alloc, **no per-object free**,
destroyed wholesale (`Clear()`). THD has statement and connection arenas;
TABLE_SHARE, each optimizer phase, hash-join buffers use their own. Idiom:
`new (thd->mem_root) Item_func_eq(...)` — most parse/optimize objects have
trivial or never-run destructors. This is why MySQL rarely leaks per-statement
memory and why object lifetime bugs = use-after-free of a whole arena.
Instrumented allocation: `my_malloc(key, …)` attributes to PSI memory keys
(`sql/psi_memory_key.cc`) → `memory_summary_*` P_S tables.

### 14.2 Error handling & diagnostics

No exceptions (`-fno-exceptions` mindset; InnoDB uses `dberr_t` returns, SQL
layer `bool`/`int` + THD state). `my_error(ER_X, MYF(0), …)` → pushes a
condition into `Diagnostics_area` (`sql/sql_error.cc`); handlers
(`sql/error_handler.h` `Internal_error_handler` stack) let callers intercept
(e.g., ignore missing table during DROP). Error definitions:
`share/messages_to_clients.txt` / `messages_to_error_log.txt` (comp_err
generates headers). Structured error log with filter/sink components
(`sql/server_component/log_builtins*`, JSON/syslog sinks).

### 14.3 performance_schema (`storage/perfschema/`)

Implemented **as a storage engine** over in-memory ring buffers/HASH tables.
Instrumentation interface = **PSI** ABI (`include/mysql/psi/`): every mutex,
rwlock, cond, file, thread, stage, statement, transaction, memory alloc, socket
and MDL object is wrapped with `PSI_*_CALL` macros compiled to registered
callbacks. Aggregation pyramid: events_waits → stages → statements →
transactions, each `_current/_history/_history_long` + summaries by
thread/account/digest/program. Fixed-size allocation at startup (autoscaled),
lock-free circular buffers; "container" scaling in `pfs_buffer_container.cc`.
sys schema (`scripts/sys_schema/`) = curated views on top.

### 14.4 Plugins vs components

- **Legacy plugin API** (`include/mysql/plugin.h`, `sql/sql_plugin.cc`):
  flat vtables per plugin type (storage engine, ft parser, audit, auth,
  keyring…), linked to server internals (can call THD functions). Still how
  engines, semisync, GR load.
- **Component infrastructure** (`components/`, `mysql/components/`): 8.0+
  **service registry** — components declare provided/required *services*
  (versioned pure vtables) resolved via `mysql_service_registry`; the "minimal
  chassis" boots the registry without a server. Everything new (keyring_*,
  validate_password, error-log sinks, telemetry) is a component.
  `mysql.component` table = persisted install list. Direction of travel:
  server functionality itself exposed as ~200 services
  (`sql/server_component/`).

### 14.5 mysys, vio, strings

- `mysys/` — the 1995-vintage portability layer that still carries the server:
  `my_file` wrappers, `IO_CACHE` (buffered sequential file I/O — binlog,
  filesort use it), thread keys, options parser (`my_getopt`).
- `vio/` — network abstraction: TCP, TLS (OpenSSL, `viossl*.c`), Unix socket,
  Windows named pipe/shared memory; non-blocking variants for the client lib.
- `strings/` — charset/collation library (`CHARSET_INFO`), all collations
  data-driven from `share/charsets/` + built-ins; default
  `utf8mb4_0900_ai_ci` (UCA 9.0.0 weights); also decimal (`decimal.cc`) and
  dtoa. Collation = tailored weight strings + contractions — this library is
  why `WEIGHT_STRING()` exists.

### 14.6 Router (`router/`)

Separate binary (own CMake subtree): lightweight L7 proxy speaking the MySQL
protocol; with InnoDB Cluster metadata it does R/W splitting to primary,
RO load-balancing, and fast failover routing. Plugin architecture (harness +
routing/metadata_cache plugins).

---

## 15. Build & test

- **CMake** superbuild: `cmake -B build -DWITH_DEBUG=1 -DWITH_BOOST=...`
  (Boost needed for GIS); `WITH_ASAN/UBSAN/TSAN`, `WITH_HYPERGRAPH_OPTIMIZER`
  (debug), unity builds for InnoDB. Generated code: lexer tables
  (`gen_lex_hash`, `gen_lex_token`), error messages (`comp_err`), Bison
  outputs.
- **MTR** (`mysql-test/`): `./mtr main.select` — spins up server(s), runs
  `.test` files (a DSL: SQL + `--let/--source/--replace_result`, sync points
  via **DEBUG_SYNC** (`sql/debug_sync.cc`) — deterministic concurrency tests
  by pausing threads at named points). Result files `.result` are golden
  outputs. Suites: main, innodb, rpl, gr, perfschema… `--big-test`,
  `--valgrind`, `--repeat`, sharded CI via `--parallel`.
- **Unit tests**: `unittest/gunit/` (Google Test; link server pieces —
  fast iteration on optimizer/containers/InnoDB utils).
- Debug instrumentation: `DBUG` package (`--debug=d,info:t:o,/tmp/trace` —
  function-level trace from the `DBUG_TRACE`/`DBUG_PRINT` macros littered
  everywhere), `DBUG_EXECUTE_IF` fault injection driven by
  `SET debug='+d,simulate_xxx'` — grep a symbol in tests to find how a code
  path is exercised.

---

## 16. Reading guide — where to start

1. `sql/sql_parse.cc:1347` `do_command` → `dispatch_command` — statement
   lifecycle end-to-end.
2. `sql/join_optimizer/join_optimizer.cc` header comment — modern optimizer
   design doc.
3. `sql/iterators/composite_iterators.cc` — how execution composes.
4. `sql/handler.h:4752` — the SE contract; then
   `storage/innobase/handler/ha_innodb.cc` `ha_innobase::index_read` →
   `row_search_mvcc` (`storage/innobase/row/row0sel.cc:4431`) — SQL-to-storage
   in one call chain.
5. `sql/binlog.cc:7886` `ordered_commit` + `sql/handler.cc:1686`
   `ha_commit_trans` — the 2PC/group-commit heart.
6. `storage/innobase/log/log0write.cc` top-of-file comment — the lock-free
   redo design, with ASCII diagrams, one of the best-documented parts of the
   tree.
7. `storage/innobase/trx/trx0trx.cc` + `read/read0read.cc` — MVCC in ~2 files.

### Architectural through-lines worth internalizing

- **Two logs ⇒ 2PC everywhere**: binlog-as-TC-log explains group commit
  stages, GTID ordering, crash recovery, semi-sync hooks, and why
  `sync_binlog=1` + `innodb_flush_log_at_trx_commit=1` is "the durable
  config".
- **The handler API is the real schema of the system**: everything the
  optimizer can know (statistics, range estimates) and everything execution
  can do (ICP/MRR/bulk hints) is negotiated through it; features live or die
  by whether they can be expressed there.
- **Arena memory + no exceptions** shape all SQL-layer code style.
- **THD is a god object by design** — cheap context passing beats purity at
  this scale; the cost is discipline around cross-thread access
  (LOCK_thd_data) and re-executability (arenas).
- **InnoDB is ARIES with undo-based MVCC**: redo for durability, undo for
  atomicity *and* versioning, purge as the garbage collector whose lag is the
  system's main aging failure mode.

