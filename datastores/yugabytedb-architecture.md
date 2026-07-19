# YugabyteDB — Architecture and Technical Implementation Deep Dive

> Target audience: staff/principal engineers. Based on direct analysis of the source tree at
> `C:\workspace\opensource\yugabyte-db` (master branch, mid-2026). File paths below are relative to
> the repo root and refer to actual source files.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Process & Deployment Topology](#2-process--deployment-topology)
3. [The Query Layer (YQL)](#3-the-query-layer-yql)
4. [YSQL: The PostgreSQL Fork and PgGate](#4-ysql-the-postgresql-fork-and-pggate)
5. [DocDB: The Distributed Document Store](#5-docdb-the-distributed-document-store)
6. [Key/Value Encoding (dockv)](#6-keyvalue-encoding-dockv)
7. [The RocksDB Fork and Storage Internals](#7-the-rocksdb-fork-and-storage-internals)
8. [Time: Hybrid Logical Clocks and MVCC](#8-time-hybrid-logical-clocks-and-mvcc)
9. [Replication: Raft Consensus](#9-replication-raft-consensus)
10. [Distributed Transactions](#10-distributed-transactions)
11. [Concurrency Control: Locking, Wait Queues, Deadlock Detection](#11-concurrency-control-locking-wait-queues-deadlock-detection)
12. [The Master and the Catalog](#12-the-master-and-the-catalog)
13. [Tablet Lifecycle: Bootstrap, Splitting, Remote Bootstrap](#13-tablet-lifecycle-bootstrap-splitting-remote-bootstrap)
14. [YSQL Advanced Topics: DDL, Backfill, Colocation, Pushdown](#14-ysql-advanced-topics)
15. [Cross-Cluster: xCluster and CDC](#15-cross-cluster-xcluster-and-cdc)
16. [Backup, Snapshots, and PITR](#16-backup-snapshots-and-pitr)
17. [RPC Framework and Threading Model](#17-rpc-framework-and-threading-model)
18. [Vector Indexes](#18-vector-indexes)
19. [Observability: ASH, Metrics](#19-observability)
20. [Repo Map and Build System](#20-repo-map-and-build-system)

---

## 1. System Overview

YugabyteDB is a distributed SQL database built as **two loosely coupled halves**:

1. **A query layer (YQL)** — most notably **YSQL**, a genuine fork of PostgreSQL (currently based on
   PG 15, in `src/postgres/`) whose storage layer (heap, buffer manager, WAL) is replaced by calls
   into the distributed store. A second API, **YCQL**, is a Cassandra-compatible layer implemented
   natively in C++ (`src/yb/yql/cql/`).

2. **A distributed document store (DocDB)** — a sharded, Raft-replicated, MVCC key-value store built
   on a heavily modified RocksDB fork (`src/yb/rocksdb/`), with hybrid logical clock (HLC)
   timestamps, distributed ACID transactions, and automatic sharding/rebalancing.

The core architectural bet: **reuse the PostgreSQL upper half unmodified in spirit** (parser,
analyzer, planner, executor, PL/pgSQL, extensions) while replacing everything below the executor's
table-access boundary with RPCs to DocDB. This gives runtime-compatible SQL semantics (not just
wire-compatibility) at the cost of a fork that must be rebased onto new PG majors.

```
                 ┌─────────────────────────────────────────────┐
   psql/JDBC ───►│ YSQL: postgres backend (src/postgres)       │
                 │   parser → planner → executor               │
                 │       └── PgGate (src/yb/yql/pggate)        │
                 │             └── PgClient RPC/shared-mem     │
   cqlsh ───────►│ YCQL: CQLServer (src/yb/yql/cql)            │
                 └───────────────┬─────────────────────────────┘
                                 ▼
                 ┌─────────────────────────────────────────────┐
                 │ yb-tserver process                          │
                 │  PgClientService / TabletServerService      │
                 │  ┌───────────────────────────────────────┐  │
                 │  │ Tablet (src/yb/tablet)                │  │
                 │  │  Raft (src/yb/consensus) + WAL        │  │
                 │  │  DocDB (src/yb/docdb, src/yb/dockv)   │  │
                 │  │  RegularDB + IntentsDB (RocksDB fork) │  │
                 │  └───────────────────────────────────────┘  │
                 └─────────────────────────────────────────────┘
                                 ▲ heartbeats / tablet reports
                 ┌───────────────┴─────────────────────────────┐
                 │ yb-master (src/yb/master)                   │
                 │  CatalogManager, sys.catalog tablet,        │
                 │  load balancer, DDL orchestration           │
                 └─────────────────────────────────────────────┘
```

---

## 2. Process & Deployment Topology

### 2.1 yb-master (`src/yb/master/`)

A small (typically 3-node) Raft group that owns cluster metadata:

- **`sys.catalog`** — a single special tablet, itself Raft-replicated across the masters, storing
  all catalog entities (tables, tablets, namespaces, users, snapshots, replication configs) as
  protobuf values in DocDB format. Loaded into in-memory maps at leader election by
  `catalog_loaders.cc`.
- **`CatalogManager`** (`catalog_manager.cc`, ~20K lines) — the brain: table/tablet lifecycle, DDL
  orchestration, tablet placement, schema changes.
- **Load balancer** (`cluster_balance.cc`) — moves tablet replicas and leaders to satisfy placement
  policy (cloud/region/zone aware), balance load, and honor preferred-leader zones.
- Masters do **not** sit on the data path. Clients cache tablet locations (`client/meta_cache.cc`)
  and talk directly to tservers.

### 2.2 yb-tserver (`src/yb/tserver/`)

The data-plane process on every node. Hosts:

- A set of **tablets** (shards), each an independent Raft group participant.
- **`TabletServerService`** — write/read RPCs for YCQL and internal ops.
- **`PgClientService`** (`pg_client_service.cc`) — the endpoint YSQL backends talk to (§4.3).
- **Heartbeater** (`heartbeater.cc`) — periodic heartbeats to the master leader carrying tablet
  reports, metrics, and lease renewals.
- The **YSQL postmaster is spawned and supervised by the tserver** (`yql/pgwrapper/`): every
  tserver runs a co-located PostgreSQL process tree; backends connect to the *local* tserver.
- Optionally **YSQL Connection Manager** (`yql/ysql_conn_mgr_wrapper/`), a fork of the Odyssey
  connection pooler (`src/odyssey/`) providing server-side connection multiplexing.

### 2.3 Tablets

A table is split into **tablets** by hash and/or range of the primary key (§5.2). Each tablet =
one Raft group (default RF=3) = one WAL + two RocksDB instances (§5.4). Tablet peers on a tserver
are managed by `TSTabletManager` (`tserver/ts_tablet_manager.cc`).

---

## 3. The Query Layer (YQL)

`src/yb/yql/` contains all API front-ends:

| Component | Path | Notes |
|---|---|---|
| YSQL glue (PgGate) | `yql/pggate/` | C++ library linked into the postgres backend |
| YCQL server | `yql/cql/` | Native C++ CQL wire protocol + parser/analyzer/executor |
| YEDIS (Redis, legacy) | `yql/redis/` | Deprecated Redis-compatible API |
| PG process supervisor | `yql/pgwrapper/` | Spawns/monitors postmaster from tserver |
| Connection manager | `yql/ysql_conn_mgr_wrapper/` | Odyssey-based pooler |

**YCQL** is worth a note even if YSQL dominates: it's a from-scratch C++ implementation
(parser in `yql/cql/ql/ptree/`, executor in `yql/cql/ql/exec/`) that maps CQL rows onto DocDB
documents directly, uses the `client/` async batcher (`batcher.cc`, `async_rpc.cc`) rather than
PgGate, and supports only single-row or explicitly-declared transactional consistency. YCQL
expressions evaluate through builtin function tables in `src/yb/bfql/` / `bfcommon/`
("builtin functions for QL"), while YSQL pushdown uses `bfpg/` and `ybgate/`.

---

## 4. YSQL: The PostgreSQL Fork and PgGate

### 4.1 The fork strategy

`src/postgres/` is a full PostgreSQL source tree with YB modifications inline, conventionally
marked and largely concentrated in `yb_*` files:

- New executor nodes: `src/postgres/src/backend/executor/nodeYbSeqscan.c`,
  `nodeYbBatchedNestloop.c` (batched nested loop join — batches outer-row keys into one RPC),
  `nodeYbBitmapIndexscan.c`, `nodeYbBitmapTablescan.c`.
- Planner extensions: `optimizer/path/yb_uniqkeys.c` (distinct pushdown),
  cost model changes for LSM indexes, and a newer cost-based optimizer mode (CBO) with
  DocDB-aware costing.
- Commands: `commands/yb_cmds.c` (DDL bridging into DocDB), `yb_tablegroup.c`, `yb_profile.c`.
- `src/postgres/src/backend/utils/misc/pg_yb_utils.c` — the grab-bag of session-level YB glue.
- `ybgate/` — an embeddable subset of the PG expression evaluator that DocDB links **server-side**
  so tservers can evaluate pushed-down PG expressions (§14.4) without a postgres process.

Key invariant: **PostgreSQL system catalogs (`pg_class`, `pg_attribute`, …) are themselves stored
in DocDB**, in the `sys.catalog` tablet on the masters (for the "template" metadata) and mirrored
per-database. There is no local heap storage; `initdb` runs once at cluster creation to populate
the catalog into DocDB.

### 4.2 PgGate (`src/yb/yql/pggate/`)

PgGate is the C++ API boundary between the postgres executor and DocDB. The postgres side calls a
flat C API (`ybc_pggate.h` / `pg_callbacks.cc`); internally:

- **`PgApiImpl`** — the top-level API object; owns per-statement objects.
- **Statement objects** — `PgSelect`/`PgDmlRead` (`pg_dml_read.cc`), `PgInsert`/`PgUpdate`/
  `PgDelete` (`pg_dml_write.cc`), `PgDdl` (`pg_ddl.cc`). These build **protobuf operation
  requests** (`PgsqlReadRequestPB` / `PgsqlWriteRequestPB`, defined in `src/yb/common/pgsql_protocol.proto`).
- **`PgDocOp`** (`pg_doc_op.cc`) — the async operation engine: splits requests across tablets,
  manages parallelism, prefetching, and response streaming (`pg_doc_op_fetch_stream.cc`).
- **`PgOperationBuffer`** (`pg_operation_buffer.cc`) — write buffering. Non-conflicting writes in a
  statement/transaction are buffered and flushed in batches (bounded by `ysql_session_max_batch_size`),
  which is the main mechanism that makes multi-row DML latency acceptable. Buffering must flush
  before any read that could observe the buffered writes (read-your-writes hazard tracking).
- **`PgFkReferenceCache`** (`pg_fk_reference_cache.cc`) — batches FK existence checks.
- **`PgTxnManager`** — maps PG transaction/isolation state onto DocDB transaction semantics.

### 4.3 PgClient: backend ↔ tserver transport

PgGate does **not** talk to remote tablet servers directly. It talks to the **local tserver's
`PgClientService`** (`tserver/pg_client_service.cc`, protocol in `tserver/pg_client.proto`), which
hosts a **`PgClientSession`** per backend (`pg_client_session.cc`). The session owns the actual
`client::YBClient` machinery, transaction objects, and per-session state. Rationale:

- Postgres backends are processes; keeping meta-cache, transaction state, and RPC infrastructure in
  the (single, threaded) tserver amortizes it across backends.
- Large data transfer uses **shared memory** between backend and tserver
  (`tserver/pg_shared_mem_pool.cc`, `tserver_shared_mem.cc`) to avoid serializing row data through
  the RPC stack; small control RPCs go over the local socket.
- Caches live tserver-side and are shared: `pg_table_cache.cc` (tables/schemas),
  `pg_response_cache.cc` (read result caching for catalog prefetch), `pg_sequence_cache.cc`
  (sequence range allocation — sequences are rows in a special `sequences_data` table, cached in
  ranges to avoid a distributed RTT per `nextval`).

### 4.4 Read/write path from SQL to DocDB (summary)

Write: executor → PgGate `PgInsert` → buffered in `PgOperationBuffer` → flush → `PgClientSession`
→ `YBSession`/`Batcher` groups ops per tablet → `WriteRpc` to tablet leaders → tablet
`WriteQuery` → DocDB (§5, §10). Read: executor → `PgSelect` with pushed-down conditions/targets →
`PgDocOp` fans out `PgsqlReadRequestPB` per tablet (or a single request for point lookups) →
tablet `read_query.cc` → `DocRowwiseIterator` (§5.5) → rows stream back (paged by
`ysql_prefetch_limit`, sidecar-encoded, optionally via shared memory).

---

## 5. DocDB: The Distributed Document Store

DocDB spans four directories:

- `src/yb/dockv/` — key/value **encoding** (no storage dependencies): `DocKey`, `SubDocKey`,
  `PrimitiveValue`, packed rows, intents, TTL, partitioning.
- `src/yb/docdb/` — the **storage engine logic**: write batches, readers/iterators, conflict
  resolution, compaction hooks, lock managers, wait queues.
- `src/yb/tablet/` — the **tablet**: hosts the DBs, MVCC manager, operation pipeline, transaction
  participant/coordinator, bootstrap, snapshots.
- `src/yb/rocksdb/` — the forked LSM engine.

### 5.1 Document model

Every row is a *document*: a `DocKey` (primary key) mapping to column values addressed by
`SubDocKey = DocKey + subkeys`. For YSQL, subkeys are column IDs (`kColumnId`); for YCQL
collections, subkeys can be map keys / list indexes. Each versioned fact is one RocksDB entry
whose key ends in a `DocHybridTime` — this *is* the MVCC representation: no separate undo/redo,
history is in the LSM itself and is garbage-collected by compactions past the *history cutoff*
(`tablet/operations/history_cutoff_operation.cc`, retention controlled by
`timestamp_history_retention_interval_sec`).

### 5.2 Sharding

`dockv/partition.cc` defines the partition schema. Two schemes:

- **Hash sharding** (default for YSQL hash-partitioned tables, all YCQL): a 16-bit hash
  (`YBPartition::HashColumnCompoundValue`, essentially a truncated hash of the hashed PK columns)
  prefixes the encoded key as `kUInt16Hash + 2 bytes`. The hash space [0x0000, 0xFFFF] is divided
  into contiguous tablet ranges — this makes *range splits of the hash space* possible while
  keeping key→tablet routing a simple binary search over partition bounds.
- **Range sharding**: tablets own ranges of the raw encoded key; supports dynamic splitting at
  observed split points (§13.2).

Client-side routing: `client/meta_cache.cc` caches `tablet_id → partition bounds + replica
locations + leader`, refreshed lazily on `NOT_THE_LEADER`/`TABLET_NOT_FOUND` errors via master
lookups (batched in `GetTableLocations`).

### 5.3 Tablet = WAL + RegularDB + IntentsDB

`tablet/tablet.h` (see members around line 1287):

```cpp
std::unique_ptr<rocksdb::DB> regular_db_;   // committed data, MVCC-timestamped
std::unique_ptr<rocksdb::DB> intents_db_;   // provisional records of in-flight txns
```

- **The Raft WAL is the only WAL.** Both RocksDB instances run with their own write-ahead logging
  disabled; durability comes from Raft log entries (`consensus/log.cc`). Flushed-frontier metadata
  (§7.2) records the max OpId persisted in each SSTable so bootstrap knows where to replay from.
- **RegularDB** holds committed data keyed by `SubDocKey + DocHybridTime`.
- **IntentsDB** holds provisional (uncommitted) records plus a reverse index from transaction ID to
  its intents (§10.2). Separating them keeps aborted garbage out of the main LSM and lets reads
  skip intent processing entirely for non-transactional workloads.

### 5.4 Write path inside a tablet

1. RPC arrives at the leader; `WriteQuery` (tablet/write_query.cc) acquires **in-memory row locks**
   via `SharedLockManager` (`docdb/shared_lock_manager.cc`) for the keys involved (this serializes
   *concurrent RPCs* touching the same keys on this node; it is not the transaction-level lock).
2. Reads needed by the write (e.g., YSQL `UPDATE` read-modify-write is done in the executor, but
   YCQL/upsert paths may read) execute; conflict resolution runs for transactional writes (§10.4).
3. A `WriteOperation` (`tablet/operations/write_operation.cc`) is submitted to Raft via the
   `OperationDriver`/`Preparer` pipeline (`operation_driver.cc`, `preparer.cc`) — the Preparer
   batches multiple operations into one Raft entry submission.
4. On majority replication, `MvccManager` (§8.3) assigns/confirms the commit hybrid time, and the
   write batch is applied to RegularDB or IntentsDB.
5. The response returns to the client. Followers apply the same batch at the same hybrid time when
   they replicate the entry.

### 5.5 Read path inside a tablet

`tserver/read_query.cc` → `Tablet::HandlePgsqlReadRequest` → `docdb/pgsql_operation.cc` →
`DocRowwiseIterator` (`docdb/doc_rowwise_iterator.cc`) over an `IntentAwareIterator`
(`docdb/intent_aware_iterator.cc`), which merges:

- RegularDB entries visible at the read time (`ReadHybridTime`), and
- IntentsDB entries: intents of *this* transaction (own writes are visible), and **committed but
  not-yet-applied** intents of other transactions (resolved via the transaction status cache,
  `docdb/transaction_status_cache.cc`).

Reads are served at a chosen `ReadHybridTime` after waiting for the tablet's **safe time** (§8.4)
to reach it. Filters, projections, and (for YSQL) expression evaluation are applied tablet-side
where pushed down. `doc_read_context.h` carries schema + packing info to the iterator.

---

## 6. Key/Value Encoding (dockv)

### 6.1 Key encoding

All keys are order-preserving byte strings built from typed components. Each component starts with
a one-byte `KeyEntryType` tag (`dockv/value_type.h` — the definitive enum; tags are chosen so that
byte-wise `memcmp` order equals semantic order). Selected tags:

```
kGroupEnd        '!'   end of hashed/range group; sorts below all values so prefixes sort first
kHybridTime      '#'   introduces the DocHybridTime suffix
kNullLow         '$'   NULL (ascending)
kUInt16Hash      'G'   2-byte partition hash prefix
kInt32/kInt64    'H'/'I'  big-endian with sign bit flipped (order-preserving)
kString          'S'   zero-encoded (0x00 → 0x00 0x01, terminator 0x00 0x00)
kColumnId        'K'   column ID subkey
kIntentTypeSet   13    prefix for intent keys in IntentsDB (sorts below all key bytes)
kTransactionApplyState 7   apply-progress markers for large txn apply
Descending variants (kStringDescending, kInt64Descending, …) store complemented bytes.
```

A YSQL row key looks like:

```
DocKey    = [kUInt16Hash][hash][hashed cols...][kGroupEnd][range cols...][kGroupEnd]
SubDocKey = DocKey [kColumnId][column_id]
RocksDB key = SubDocKey [kHybridTime][^DocHybridTime]   (^ = complemented so newer sorts first)
```

`DocHybridTime` = `HybridTime` (§8.1) + `write_id` (intra-batch sequence number) — this makes
every version key unique and totally ordered even within one Raft batch.

### 6.2 Value encoding and packed rows

Values are `ValueEntryType`-tagged primitives, optionally preceded by *control fields* (TTL,
user-set timestamp — `dockv/value.h`, `expiration.h`).

**Packed rows** (`dockv/packed_row.h`) collapse an entire row insert into a *single* KV pair
instead of one pair per column:

- **V1** (`kPackedRowV1`, tag `'z'`): `varint schema_version`, then a `uint32` end-offset per
  variable-length column, then concatenated column values — O(1) column extraction by offset
  arithmetic. NULLs are zero-length values (distinguishable because non-null values always embed a
  `ValueEntryType`).
- **V2** (`kPackedRowV2`, tag `'|'`): a newer, more compact layout (null bitmap-based).
- `SchemaPacking` (`dockv/schema_packing.cc`) is versioned per schema-version and stored in tablet
  metadata; readers map packed values through the packing for the recorded schema version, so
  **schema changes don't rewrite data** — updates write per-column entries on top of the packed
  base row, and compactions can repack (merge packed row + overlying column updates into a new
  packed row, `docdb/docdb_compaction_context.cc`).

This is the single most important storage optimization for YSQL write throughput: without it, an
N-column insert costs N RocksDB entries.

### 6.3 Intent encoding

(`dockv/intent.h`) IntentsDB has two families of entries:

1. **Intent records**: key = `SubDocKey(no HT) [kIntentTypeSet][type_set] [kHybridTime][DocHybridTime]`,
   value = `transaction_id [subtxn_id] write_id  actual_value`. `IntentTypeSet` is a bitset of
   {weak/strong} × {read/write} — weak intents are taken on key *prefixes* (ancestors) so that a
   strong intent on a row conflicts with a strong intent on the whole-table/prefix level cheaply.
2. **Reverse index**: key = `transaction_id + doc_ht`, value = the intent key. Used to find all of
   a transaction's intents at apply/abort time without scanning.

---

## 7. The RocksDB Fork and Storage Internals

`src/yb/rocksdb/` diverged from upstream years ago. Notable YB-specific changes:

### 7.1 Removed/replaced subsystems
- **No RocksDB WAL** (Raft log replaces it); no column families in the upstream sense (two whole
  DB instances instead); merge operators unused.

### 7.2 Frontiers
`docdb/consensus_frontier.h` — each memtable/SSTable carries a `ConsensusFrontier` (min/max) with:
`OpId` (Raft index persisted), hybrid time bounds, history cutoff, max schema version. Used for:
- **Bootstrap dedup**: replay WAL only from the flushed OpId.
- **Compaction-time GC** decisions (history cutoff propagation).
- **File-level TTL expiry**: `docdb/compaction_file_filter.cc` drops whole SSTables whose max
  expiration is past — O(1) expiry for time-series workloads.

### 7.3 Read/scan optimizations
- **`BoundedRocksDBIterator`** (`docdb/bounded_rocksdb_iterator.cc`) constrains iterators to the
  tablet's key range (important post-split, when an SSTable is shared by two tablets).
- **Key buffering & prefix seeks** in `IntentAwareIterator` minimize comparisons; `SeekForward`
  uses the LSM's ordering guarantees aggressively.
- **Scan-forward optimization** and max-seek heuristics tuned for the "many versions per key"
  shape MVCC produces.

### 7.4 Compaction integration
`docdb/docdb_compaction_context.cc` implements DocDB-aware compaction: drops overwritten versions
older than history cutoff, applies TTL, resolves packed-row + column-update merges, removes
tombstoned rows, and (in full compactions scheduled by `tserver/full_compaction_manager.cc`)
reclaims space post-split. Deletes in DocDB are *tombstones* (`kTombstone`) subject to the same
MVCC/history rules.

---

## 8. Time: Hybrid Logical Clocks and MVCC

### 8.1 HybridTime

`src/yb/common/hybrid_time.h`: a 64-bit value = `(physical_microseconds << 12) | logical`.
12 bits of logical counter disambiguate events within the same microsecond. The HLC algorithm
(`server/hybrid_clock.cc`) guarantees: (a) monotonicity per node, (b) causality propagation —
every RPC carries the sender's HT and the receiver ratchets its clock forward
(Lamport-clock style), so any two causally related events have ordered HTs even across nodes.

### 8.2 Clock skew and the read-restart window

Physical clocks are assumed synchronized within `max_clock_skew_usec` (default 500 ms; deployments
with NTP/PTP/AWS Time Sync can lower it). A read at time `ht_read` may miss a commit that happened
"before" it in real time but got a higher HT from a fast clock. DocDB handles this with the
**read-time triple** (`common/read_hybrid_time.h`): `read`, `local_limit`, `global_limit =
read + max_clock_skew`. If a scan encounters a committed record with `read < ht ≤ global_limit`,
the record is *possibly-in-the-past* → the operation fails with **`ReadRestart`**, and the query
layer transparently retries at a higher read time (restarts are cheap: for Read Committed they are
handled per-statement; `local_limit` is ratcheted per-tablet to shrink the ambiguity window on
retry). This is how YugabyteDB avoids both TrueTime hardware and commit-wait latency.

### 8.3 MvccManager

`tablet/mvcc.cc` — per-tablet tracker of in-flight hybrid times. The leader assigns HTs to Raft
entries in increasing order; `MvccManager` maintains the queue of assigned-but-not-yet-replicated
HTs and computes **safe time** = a time `t` such that no future commit can get HT ≤ t
(`SafeTimeSource` enum: `kNow`, `kNextInQueue`, `kHybridTimeLease`, `kPropagated`,
`kLastReplicated`). Reads at `ht_read` block until `safe_time ≥ ht_read`.

### 8.4 Leader leases

`consensus/leader_lease.h`: majority-granted, *monotonic-clock-based* leases let the leader serve
reads locally without a Raft round-trip while guaranteeing no other peer can commit writes (an old
leader's lease must expire before a new leader can serve). **Hybrid time leases** additionally
bound the HT a new leader may use, making follower-computed safe time valid. Follower reads
(`yb_read_from_followers`) and read replicas serve bounded-staleness reads at
`safe_time - staleness`.

---

## 9. Replication: Raft Consensus

`src/yb/consensus/` is a production Raft (lineage: Apache Kudu, heavily evolved):

- **`RaftConsensus`** (`raft_consensus.cc`) — state machine: elections (`leader_election.cc`, with
  pre-elections to avoid term inflation), config changes (joint-free, one-server-at-a-time
  add/remove via `ChangeConfig`), leader stepdown with targeted successor.
- **`PeerMessageQueue`** (`consensus_queue.cc`) — per-peer replication pipeline: tracks each
  peer's `last_received`, computes majority watermark (commit index), majority-replicated lease
  bounds, and handles flow control. **`LogCache`** (`log_cache.cc`) keeps recent entries in memory
  so followers are served without disk reads.
- **`Log`** (`log.cc`) — the WAL: segmented, asynchronously allocated, with an index
  (`log_index.cc`) mapping OpId→file offset. Supports `log_min_seconds_to_retain` +
  durable-watermark GC coordinated with CDC/xCluster consumers, and min-replicated anchoring
  (`log_anchor_registry.cc`) so remote bootstrap sources keep needed segments.
- **YB deltas from textbook Raft**:
  - **Leader leases** (§8.4) for lease-based consistent reads.
  - **Hybrid time propagation** in AppendEntries.
  - **Operation-type awareness**: entries are `ReplicateMsg`s wrapping typed operations
    (`tablet/operations/*` — write, change-metadata, split, snapshot, truncate, update-txn,
    history-cutoff, clone), applied through the shared `OperationDriver` pipeline.
  - **Retryable requests / exactly-once semantics** (`consensus/retryable_requests.cc`): client
    writes carry (client_id, request_id, min_running_request_id); the tablet deduplicates retries
    across leader changes — this is what makes automatic client retries safe for non-idempotent
    writes. State is persisted/restored via `tablet_bootstrap_state_manager.cc`.
  - **Remote bootstrap** (§13.3) instead of snapshot-install RPCs.

Replication factor is per-table/per-cluster placement policy; the master's load balancer drives
config changes to converge actual placement to policy.

---

## 10. Distributed Transactions

The design descends from Percolator/Spanner lineage but with distributed status tablets instead of
a single timestamp oracle. Core pieces:

| Piece | Path |
|---|---|
| Transaction status tablets | system table `transactions`, coordinator per status tablet |
| `TransactionCoordinator` | `tablet/transaction_coordinator.cc` |
| `TransactionParticipant` | `tablet/transaction_participant.cc` |
| Client txn object | `client/transaction.cc` (`YBTransaction`), `client/transaction_manager.cc` |
| Conflict resolution | `docdb/conflict_resolution.cc` |
| Status cache for reads | `docdb/transaction_status_cache.cc` |

### 10.1 Lifecycle

1. **Begin**: `YBTransaction` picks a status tablet (locality-aware — prefers a status tablet
   whose leader is local; `transaction_manager.cc`) and generates a UUID `TransactionId`. The
   priority (random, band-based for optimistic vs explicit-locking txns) is assigned for conflict
   arbitration.
2. **Write intents**: each participating tablet's leader, on a transactional write, runs conflict
   resolution (§10.4), then replicates a Raft entry that writes **intents** into IntentsDB
   (§6.3). The first write also registers the txn with the status tablet (heartbeated thereafter;
   an expired-heartbeat txn is presumed aborted).
3. **Commit**: client sends `UpdateTransaction(COMMITTING)` to the status tablet. The coordinator
   replicates a `COMMITTED` record with commit HT = its current HT. **Commit point = Raft
   replication of that status record.** The coordinator then asynchronously sends `APPLYING` to
   all participant tablets.
4. **Apply**: each participant reads the txn's intents via the reverse index and rewrites them
   into RegularDB at the commit HT (`ApplyIntentsTask`, `tablet/apply_intents_task.cc`; large
   transactions apply incrementally with `kTransactionApplyState` progress markers so apply is
   restartable), then intents are cleaned (`remove_intents_task.cc`, `cleanup_intents_task.cc`).
5. **Read visibility before apply**: a reader encountering an intent consults the status tablet
   (`GetTransactionStatus`, results cached per-read in `transaction_status_cache.cc`): if the txn
   committed with `commit_ht ≤ read_time`, the intent's value is *treated as committed data* even
   though not yet applied. This removes apply latency from the visibility critical path.

`TransactionParticipant` also runs recovery duties: aborting expired transactions
(`cleanup_aborts_task.cc`), responding to status queries, and tracking min-running-txn HT (which
bounds what compactions may GC in IntentsDB).

**Subtransactions / savepoints**: intents carry a `SubTransactionId`; rollback-to-savepoint marks
the aborted subtxn range in the txn metadata, and apply/read paths skip aborted subtxn intents
(`common/transaction.h`, `docdb/conflict_data.h`). This is how PG savepoints and PL/pgSQL
exception blocks work without physical undo.

**Promotion**: a transaction that starts as a single-tablet fast-path op and then touches a second
tablet is *promoted* to a distributed txn; likewise geo-local txns can promote from a
tablespace-local status tablet to a global one (`client/transaction.cc` promotion logic).

### 10.2 Single-shard fast path

Writes confined to one tablet at one instant skip the coordinator entirely: the write is
replicated with its commit HT directly into RegularDB (no intents). This is why single-row
autocommit DML latency ≈ one Raft round.

### 10.3 Isolation levels

DocDB implements three (`common/transaction.h` → `IsolationLevel`):

- **`SNAPSHOT_ISOLATION`** (maps to PG `REPEATABLE READ`): reads at a fixed `read_time`; writes
  take write intents; first-committer-wins on write-write conflicts.
- **`SERIALIZABLE_ISOLATION`**: additionally takes **read intents** (strong read locks) on read
  rows/predicates so read-write conflicts are detected. Uniquely, `UPDATE`-style
  read-modify-writes take `strong read + strong write` in one op.
- **`READ_COMMITTED`**: implemented in the query layer (`src/postgres` + `pg_client_session.cc`)
  as per-statement snapshots with transparent per-statement retries on conflict/read-restart
  (statement restart uses a new read time; requires buffering the statement or falls back to
  errors when output already streamed).

### 10.4 Conflict resolution

`docdb/conflict_resolution.cc`: for each write (or serializable read), scan IntentsDB for
conflicting intents (strong-vs-strong on same key, strong-vs-weak on prefixes) and RegularDB for
committed writes newer than the read time (write-write conflict → `TransactionErrorCode::kConflict`
→ PG serialization error 40001 unless retried). For pending conflicting txns, the outcome depends
on concurrency-control mode:

- **Fail-on-Conflict** (classic): compare priorities — abort the lower-priority txn.
- **Wait-on-Conflict** (`enable_wait_queues=true`, the PG-compatible pessimistic mode): enqueue in
  the tablet's **wait queue** (§11.2) until conflicting txns finish.

---

## 11. Concurrency Control: Locking, Wait Queues, Deadlock Detection

### 11.1 Lock managers

- **`SharedLockManager`** (`docdb/shared_lock_manager.cc`) — per-tablet, in-memory, short-lived
  latches held for the duration of a single write RPC's prepare/apply; keyed by encoded key
  prefixes with the same weak/strong intent-type semantics.
- **`ObjectLockManager`** (`docdb/object_lock_manager.cc`, `object_lock_shared_state.cc`) — newer
  table/object-level DDL lock manager, with a **shared-memory fastpath** so PG backends can
  acquire uncontended object locks without an RPC
  (`PgClient::TryAcquireObjectLockInSharedMemory`, `yql/pggate/pg_client.h:302`). This underpins
  PG-compatible DDL↔DML interlocking (replacing the older best-effort catalog-version-based
  approach).
- **Advisory locks**: `tserver/ysql_advisory_lock_table.cc` — PG advisory locks mapped onto a
  dedicated DocDB table with intent semantics.
- **Row locks** (`SELECT ... FOR UPDATE/SHARE/KEY SHARE`): mapped to intent type sets
  (e.g. FOR UPDATE → strong read; buffered/batched via `pg_explicit_row_lock_buffer.cc`).

### 11.2 Wait queues

`docdb/wait_queue.cc` — per-tablet queues of waiter transactions blocked on blockers, with:
resumption on blocker commit/abort, fairness via serial numbers + txn start time, and integration
with the **`LocalWaitingTxnRegistry`** (`local_waiting_txn_registry.cc`) which publishes
waiting-for edges to the deadlock detector.

### 11.3 Deadlock detection

`docdb/deadlock_detector.cc` — runs at the **transaction coordinator** of each status tablet.
Waiting-for graph edges stream in from tablet wait queues; detection uses distributed **probes**
forwarded along edges; a probe returning to its origin proves a cycle, and a victim (by start
time/priority) is aborted with a deadlock error. This gives PG-equivalent semantics
(pessimistic blocking + deadlock kill) instead of the older abort-on-conflict behavior.

---

## 12. The Master and the Catalog

### 12.1 sys.catalog

One tablet, RF = number of masters, schema = generic (entry_type, entity_id) → protobuf
(`master/catalog_entity_info.proto`). All state mutations go through Raft on this tablet, so
master failover only requires replaying the tablet + rebuilding in-memory maps
(`catalog_loaders.cc`). Entities: namespaces, tables, tablets, users/roles (YCQL), UDTs,
snapshots/schedules, CDC streams, xCluster/universe replication configs, tablespaces, auto-flags
config.

### 12.2 CatalogManager responsibilities

- **Table creation**: pick tablet count (`ysql_num_shards_per_tserver` etc. or explicit
  `SPLIT INTO`), create tablet protos, place replicas per placement policy
  (`catalog_manager_util.cc`), issue async `CreateTablet` RPCs (`async_rpc_tasks.cc`), track until
  a quorum reports running.
- **Alter/DDL fan-out**: schema changes replicate as `ChangeMetadataOperation` to every tablet;
  the master tracks per-tablet schema-version convergence.
- **Tablet health**: replaces dead replicas (after `follower_unavailable_considered_failed_sec`),
  triggered from tablet reports in heartbeats.
- **Load balancing** (`cluster_balance.cc`): bounded-concurrency moves — add replica → wait for
  remote bootstrap → remove old; leader-only moves via leader stepdown, respecting per-table and
  global limits, zone-aware.
- **YSQL catalog version** management: a per-database `pg_yb_catalog_version` counter bumped on
  breaking DDL; heartbeats propagate it to tservers; backends compare against their cached version
  to invalidate PG relcache/syscache (fine-grained per-DB invalidation, plus incremental
  invalidation messages in newer versions).
- **Tablespaces → placement**: `catalog_manager` maps PG tablespaces with placement JSON to
  per-table Raft placement (geo-partitioning).

### 12.3 Master–TServer protocol

Heartbeats (`master/master_heartbeat.proto`) carry: tablet reports (deltas, full on
re-registration), storage/CPU metrics, catalog version, auto-flags config version, xCluster
safe-time info. The master returns: tablets to delete (orphans), config/flag updates, DDL
operations to execute. TServers never talk to each other for control-plane state.

---

## 13. Tablet Lifecycle: Bootstrap, Splitting, Remote Bootstrap

### 13.1 Local bootstrap

`tablet/tablet_bootstrap.cc`: on tablet start, open RocksDB instances, read flushed frontiers
(max OpId per DB), then replay WAL segments — only entries newer than the flushed frontier
actually re-apply; committed-but-unapplied transactions are reconstructed into
`TransactionParticipant` state. Retryable-request dedup state is restored from the periodically
flushed bootstrap state (`tablet_bootstrap_state_manager.cc`).

### 13.2 Tablet splitting

Automatic, online, range-based (works for hash tablets by splitting the hash range):

1. TServer-side heuristics/master policy (`master/tablet_split_manager.cc`; low/high phase size
   thresholds scale with tablets-per-node) pick a tablet; the leader computes the **split key**
   (approximate SSTable midpoint, `docdb_boundary_values_extractor.cc` / RocksDB
   `GetMiddleKey`).
2. A **`SplitOperation`** (`tablet/operations/split_operation.cc`) replicates *through the
   tablet's own Raft log*, so all replicas split deterministically at the same OpId.
3. Children are created on the same nodes, initially **sharing the parent's SSTables** via
   RocksDB hard links, each with a key-bounds restriction (`BoundedRocksDBIterator` enforces
   bounds; full compaction rewrites each child to its own data, after which the parent is
   deleted).
4. Clients discover the split lazily: writes to the parent get a "tablet split" error containing
   child partition info; `meta_cache.cc` refines its map and re-routes. The parent is retained
   until children are durable and CDC/xCluster consumers no longer need it.

### 13.3 Remote bootstrap

`tserver/remote_bootstrap_*.cc`: when the master (or a Raft leader adding a peer) needs a new
replica, the new peer downloads a consistent checkpoint from the leader: RocksDB SSTable files
(hard-link-based checkpoint), WAL segment tail, and consensus metadata, then joins as a
`PRE_VOTER` and catches up through normal replication before promotion to `VOTER`. Anchors
(`log_anchor_registry`) prevent the source from GC-ing needed WAL during the copy.

---

## 14. YSQL Advanced Topics

### 14.1 DDL atomicity and online schema change

DDL touches two worlds: the PG catalog (rows in sys.catalog via the same transactional write path)
and DocDB schema (tablet metadata). YB makes DDL atomic via **DDL transactions with rollback**:
the DocDB schema change is marked pending and the master verifies the outcome of the PG catalog
transaction (`master/ysql_ddl_handler.cc`, `ysql_ddl_verification_task.cc`), rolling
forward/backward the DocDB side accordingly. Online, multi-stage schema changes follow the
classic F1 approach: e.g. ADD CONSTRAINT/index build go through DELETE_ONLY → WRITE_AND_DELETE →
BACKFILL → READ_WRITE_DELETE stages, each a schema version fanned out to tablets, with concurrent
DML always seeing a compatible version.

### 14.2 Online index backfill

`master/backfill_index.cc` + tserver `backfill_service`: after the index reaches
write-and-delete visibility, the master coordinates per-tablet backfill tasks that scan the
indexed table **as of a fixed read time** and bulk-write index entries (bypassing the
transaction path but with proper HT stamping), then flips the index to readable. Failed/paused
backfills are resumable per-tablet.

### 14.3 Colocation and tablegroups

Small tables can share a single tablet (colocated database or `commands/yb_tablegroup.c`
tablegroups): each table gets a `kColocationId` prefix in its keys within the shared tablet.
This collapses per-table Raft/WAL/rocksdb overhead for many-small-tables schemas (the common
microservices pattern) and makes cross-table transactions within the colocation single-shard
(fast path). The `sys.catalog` tablet itself uses the colocation mechanics (`kColocationId` key
prefixes for PG catalog tables).

### 14.4 Expression and aggregate pushdown

`PgsqlReadRequestPB` carries: WHERE clause remainders (serialized PG expressions evaluated
tablet-side by `ybgate/` via `docdb/doc_pg_expr.cc`), target expressions, aggregate partials
(`COUNT/SUM/MIN/MAX` computed per-tablet, combined in the executor), LIMIT/OFFSET,
row-mark (locking) info, sampling (for ANALYZE), and **index request nesting** (an index scan +
base-table fetch in one RPC round for covered→base lookups). Batched nested loop join
(`nodeYbBatchedNestloop.c`) rewrites the inner index condition into a batched `IN`-style key list
— turning O(rows) RPCs into O(rows / batch_size).

### 14.5 Catalog caching & connection startup

Backend startup would naively read hundreds of catalog rows over RPC. Mitigations: aggressive
**catalog prefetch** into the tserver-side `PgResponseCache`, shared relcache init files, and the
connection manager (Odyssey) reusing warmed physical backends across logical connections.

---

## 15. Cross-Cluster: xCluster and CDC

### 15.1 xCluster (async universe replication)

Tablet-WAL-shipping between clusters (`tserver/xcluster_consumer.cc`, `xcluster_poller.cc`,
producer side in `src/yb/cdc/`):

- Consumers poll producer tablet leaders (`GetChanges`) for WAL records; applied via a special
  write path (`xcluster_write_implementations.cc`) that **preserves original commit hybrid
  times** (external HT) so MVCC ordering is retained.
- Modes: unidirectional, bidirectional (last-writer-wins by HT), and **transactional
  consistency mode**, where the consumer maintains an **xCluster safe time**
  (`xcluster_safe_time_map.cc`) — reads on the target run at the safe time so cross-tablet
  atomicity of source transactions is preserved.
- **Automatic DDL replication** (newer mode): DDLs are captured into a `ddl_queue` table and
  re-executed on the target (`xcluster_ddl_queue_handler.cc`).
- Checkpointing per (stream, tablet) lives in the producer's `cdc_state` table; log retention
  honors consumer progress.

### 15.2 CDCSDK (change data capture for external consumers)

`src/yb/cdc/` (`cdcsdk_producer.cc`) exposes ordered change streams per tablet, consumed by the
Debezium-based connector (`java/`). Supports before-images, colocated tables, tablet-split
handling (stream continuity onto children), and a **PG logical-replication-compatible** facade
(replication slots, `pgoutput`-style protocol via `src/postgres` walsender emulation +
`cdc_state` virtual WAL — `docdb_pgapi` / `cdcsdk_virtual_wal.cc`), letting stock PG logical
replication clients consume YB changes.

---

## 16. Backup, Snapshots, and PITR

- **Distributed snapshots** (`tablet/operations/snapshot_operation.cc`,
  `master/master_snapshot_coordinator.cc`): a snapshot is a Raft-replicated operation per tablet
  creating a RocksDB checkpoint (hard links) at a cluster-chosen HT — consistent because reads of
  the snapshot are performed *as of* that HT, not because tablets checkpoint simultaneously.
- **Backups**: snapshot + copy SSTables out (orchestrated externally by yb_backup /
  YBA; `tserver/backup_service.cc`), plus a YSQL dump of the catalog. Restore = import files +
  remap IDs.
- **PITR** (`snapshot schedules`, `tablet/restore_util.cc`, `master/clone/`): periodic snapshots +
  history retention let the cluster restore *in place* to any time within the retention window —
  implemented as a rewind: data past the restore time is deleted/overwritten via restoration
  operations, catalog state rolled back on the master; also powers **database cloning**
  (`master/clone/`, instant thin clones from a snapshot + WAL replay to target time).

---

## 17. RPC Framework and Threading Model

`src/yb/rpc/` — a custom asynchronous RPC stack (no gRPC):

- **Reactor threads** (libev-based, `rpc/reactor.cc`): N event-loop threads own sockets;
  serialization/deserialization happens on reactor threads; handlers dispatch to service thread
  pools with per-service queues (`rpc/service_pool.cc`) and queue-limit backpressure
  (`ServiceUnavailable` on overflow).
- **`Sidecars`** (`rpc/sidecars.cc`): zero-copy payload attachment — row data avoids protobuf
  encoding; this is how read results stream efficiently.
- **Protobuf-defined services** compiled by `gen_yrpc` (`src/yb/gen_yrpc/`) from `.proto` service
  definitions into service/proxy stubs.
- Deadline propagation, per-call tracing (`util/trace.cc`), TLS (`rpc/secure_stream.cc`),
  compression, and connection multiplexing are built in.
- **Thread pools** (`util/threadpool.cc`, `rpc/thread_pool.cc`, `rpc/strand.cc`,
  `rpc/scheduler.cc`): strands give per-entity serial execution atop shared pools — e.g. per-txn
  and per-tablet work ordering without dedicated threads. The tablet apply pipeline
  (`tablet/preparer.cc`) and Raft use dedicated pools with token-based submission
  (`ThreadPoolToken`) for per-tablet ordering.

General principle: **everything on the data path is asynchronous**; blocking a reactor thread is a
bug. The codebase uses `Status`/`Result<T>` (`util/status.h`, `util/result.h`) everywhere instead
of exceptions.

---

## 18. Vector Indexes

Recent, actively developed (`src/yb/vector_index/`, `src/yb/hnsw/`, `src/yb/ann_methods/`,
`docdb/doc_vector_index.cc`, `src/postgres/.../ann_methods`):

- pgvector-compatible YSQL syntax; the index is a DocDB-managed structure per tablet: vectors are
  assigned `VectorId`s (`dockv/doc_vector_id.cc`, key tag `kVectorId`), with an HNSW
  implementation (`src/yb/hnsw/` — custom, plus usearch integration in third-party) stored in
  tablet-local chunks, flushed with the tablet (see recent commit "Flush vector indexes on
  graceful tablet shutdown") and compacted/merged in the background.
- Reads perform per-tablet ANN search merged by the query layer; MVCC visibility is respected by
  filtering results against the read time.

---

## 19. Observability

- **ASH (Active Session History)** — `src/yb/ash/`: continuous sampling of active
  sessions/RPCs across PG backends and tserver RPCs into a queryable in-memory ring
  (`yb_active_session_history` view). Wait states are instrumented via `WaitStateInfo`
  (macros `SET_WAIT_STATUS`, `SCOPED_WAIT_STATUS`, ASH metadata propagated across RPC hops —
  `wait_state.h`), keyed by `root_request_id` so a SQL statement can be traced across
  backend → pggate → tserver → tablet → consensus.
- **Metrics** (`util/metrics.h`): per-tablet/table/server Prometheus-exported metric registry;
  histograms are HdrHistogram-based.
- Built-in web UIs on every process (`server/webserver.cc`, master `master-path-handlers.cc`):
  tablet states, Raft status, RocksDB stats, `/rpcz`, `/threadz`, `/memz` (tcmalloc), and a
  pprof-compatible profiling endpoint.

---

## 20. Repo Map and Build System

| Path | Contents |
|---|---|
| `src/yb/util/` | Foundation: Status/Result, threading, memory (arena, mem-tracker hierarchy), net, encryption primitives, flags (gFlags + **AutoFlags** — flags auto-enabled only when the whole universe reaches a version, `previous_auto_flags.json`) |
| `src/yb/gutil/` | Google base library vendored (from Kudu lineage) |
| `src/yb/common/` | Schema, types, `HybridTime`, `ReadHybridTime`, transaction metadata, `pgsql_protocol.proto`, `ql_protocol.proto` |
| `src/yb/fs/` | FS manager: multi-drive data dir layout, WAL/data placement |
| `src/yb/encryption/` | Encryption at rest (universe keys, encrypted file abstraction) |
| `src/yb/server/` | Shared server base: clock, webserver, RPC server harness |
| `src/yb/integration-tests/` | Mini-cluster & external mini-cluster test harnesses |
| `src/yb/tools/` | `yb-admin`, `yb-ts-cli`, `ysql_dump`, sst dump, bulk load tools |
| `java/` | Java client, CDC connector, YB test framework (`yb-client`, JUnit-driven mini-cluster tests) |
| `managed/` | YugabyteDB Anywhere (control plane product; Play/Scala + React) |
| `build-support/` | `yb_build.sh` entrypoints, lint, thirdparty download orchestration |

**Build**: CMake + Ninja via `./yb_build.sh` (see `src/AGENTS.md`); compilers pinned via
`build-support/`; third-party deps prebuilt and downloaded (yugabyte-db-thirdparty releases).
PostgreSQL builds via its autoconf inside the CMake orchestration; `pggate` links into the
backend; DocDB links `ybgate` for pushdown evaluation. Tests: gtest C++ tests colocated with
sources (`*-test.cc`), Java integration tests under `java/`, PG regress tests under
`src/postgres/src/test/regress` with YB schedules.

---

## Appendix A: Life of a Distributed YSQL Transaction (end-to-end)

```
BEGIN;                             -- PgTxnManager: lazy, nothing distributed yet
UPDATE accounts SET b=b-100 WHERE id=1;
  postgres executor → PgUpdate → PgOperationBuffer (buffered)
  (first write) YBTransaction created; status tablet chosen; txn registered
  flush → PgClientSession → Batcher → WriteRpc(tablet A leader)
    tablet A: SharedLockManager latch → conflict_resolution.cc scan
      → (wait_queue if conflict & wait-on-conflict) → Raft replicate intents
      → intents_db_: [row1/col b][IntentTypeSet: strong write][HT] = txnid, value
UPDATE accounts SET b=b+100 WHERE id=2;   -- same, tablet B; txn now multi-shard
COMMIT;
  UpdateTransaction(COMMITTING) → status tablet leader
    coordinator replicates COMMITTED @ commit_ht  ← durability/commit point
  ← success to client
  async: APPLYING → tablets A,B
    participants read reverse index, write rows into regular_db_ @ commit_ht,
    delete intents
Concurrent reader @ read_time:
  IntentAwareIterator hits intent → TransactionStatusCache → status tablet:
    COMMITTED @ commit_ht ≤ read_time ? visible : skip
    PENDING & possibly-committed-in-skew-window? → ReadRestart retry
```

## Appendix B: Key Design Trade-offs (staff-level judgment calls)

1. **Fork PG, don't reimplement**: runtime compatibility (PL/pgSQL, extensions, exact SQL
   semantics) for the price of major-version rebases (a multi-quarter effort each time; the repo
   history shows the PG11→15 rebase machinery).
2. **Raft-per-tablet, no per-node consensus**: fine-grained placement/balancing and per-tablet
   leader locality vs. per-tablet WAL/heartbeat overhead — mitigated by colocation, multi-Raft
   heartbeat batching (`consensus_peers.cc`), and tablet splitting instead of pre-sharding.
3. **HLC + read restarts instead of TrueTime/commit-wait**: no special hardware and no added
   commit latency; cost is a bounded-uncertainty retry path and clock-skew configuration
   sensitivity.
4. **Intents in a separate RocksDB instead of a lock table + buffered writes**: provisional data
   is durable and replicated (survives leader failover — locks *are* the data), avoids
   memory-bound lock tables; cost is double-write amplification for distributed txns
   (intent write + apply into regular DB).
5. **Two-instance LSM per tablet, no RocksDB WAL/column-families**: Raft log is the single
   source of durability truth; frontiers stitch LSM flush state to Raft indexes.
6. **Packed rows + schema-versioned unpacking**: O(1) column access and near-heap write
   amplification for wide inserts without data rewrites on DDL.
7. **Local-tserver proxy for PG backends (PgClientService)**: process-per-connection PG model
   made viable by centralizing distributed state per node and shipping rows via shared memory.
```
