# RocksDB Internal Architecture — A Deep Technical Walkthrough

> Target audience: staff/principal engineers who need to reason about RocksDB's
> correctness, concurrency, and performance characteristics at the source level.
> Based on the open-source tree at `facebook/rocksdb` (v11.6 era). File references
> are relative to the repository root.

---

## Table of Contents

1. [Big Picture: The LSM Engine](#1-big-picture-the-lsm-engine)
2. [Core Data Model: Internal Keys, Sequence Numbers, Value Types](#2-core-data-model)
3. [The Write Path](#3-the-write-path)
4. [Write-Ahead Log (WAL)](#4-write-ahead-log-wal)
5. [Memtables](#5-memtables)
6. [MVCC: Sequence Numbers, Snapshots, and SuperVersion](#6-mvcc-snapshots-and-superversion)
7. [Version Management and the MANIFEST](#7-version-management-and-the-manifest)
8. [Flush](#8-flush)
9. [The Read Path: Get, MultiGet, Iterators](#9-the-read-path)
10. [SST File Format (BlockBasedTable)](#10-sst-file-format-blockbasedtable)
11. [Block Cache](#11-block-cache)
12. [Compaction](#12-compaction)
13. [Merge Operator Internals](#13-merge-operator-internals)
14. [Range Deletions](#14-range-deletions)
15. [Integrated BlobDB and Wide Columns](#15-integrated-blobdb-and-wide-columns)
16. [Transactions](#16-transactions)
17. [Crash Recovery and DB Open](#17-crash-recovery-and-db-open)
18. [Error Handling and Fault Tolerance](#18-error-handling-and-fault-tolerance)
19. [Concurrency Model Summary](#19-concurrency-model-summary)
20. [Performance Engineering Machinery](#20-performance-engineering-machinery)
21. [Notable Modern Features](#21-notable-modern-features)

---

## 1. Big Picture: The LSM Engine

RocksDB is a log-structured merge-tree (LSM) storage engine forked from LevelDB
and heavily rewritten for server workloads: multi-threaded compaction, column
families, transactions, tiered storage, and pluggable everything (memtable
representation, table format, filesystem, compression, cache).

The write lifecycle of a key-value pair:

```
Put(k,v)
  └─> WriteBatch (in-memory serialized record group)
       └─> Write thread group commit
            ├─> WAL append (durability)          db/log_writer.cc
            └─> MemTable insert (visibility)     db/memtable.cc
                 └─> Flush → L0 SST file          db/flush_job.cc
                      └─> Compaction → L1..Lmax   db/compaction/
```

The read lifecycle consults the same structures newest-first:

```
Get(k): mutable memtable → immutable memtable(s) → L0 files (newest→oldest)
        → L1..Lmax (one candidate file per level, binary search)
```

Key architectural properties that drive everything else:

- **Append-only storage.** SST files and WAL/MANIFEST logs are immutable once
  written. All mutation is by writing new files and atomically switching
  metadata (the `Version` mechanism, §7). This is what makes snapshots,
  backups, checkpoints, and crash recovery tractable.
- **A single global sequence number** (56 bits) totally orders all writes across
  all column families in a DB. MVCC falls out of `(user_key, seqno)` ordering.
- **One big mutex (`DBImpl::mutex_`) for metadata, lock-free hot paths for
  data.** The DB mutex protects version installation, flush/compaction
  scheduling, and column family metadata — never per-key operations. Reads are
  lock-free after acquiring a `SuperVersion` reference (§6); writes serialize
  through a purpose-built lock-free write queue (§3), not the mutex.
- **Background work is decoupled.** Flush and compaction run on shared thread
  pools (`Env::Priority::HIGH` for flush, `LOW` for compaction), scheduled by
  `DBImpl::MaybeScheduleFlushOrCompaction()` in
  `db/db_impl/db_impl_compaction_flush.cc`.

### Source tree map

| Directory | Contents |
|---|---|
| `db/` | Engine core: DBImpl, memtable, WAL, versions, flush, recovery |
| `db/db_impl/` | `DBImpl` split by concern: `db_impl_write.cc`, `db_impl_open.cc`, `db_impl_compaction_flush.cc`, `db_impl_files.cc`, plus read-only/secondary/follower variants |
| `db/compaction/` | Compaction pickers (leveled/universal/FIFO), `CompactionIterator`, `CompactionJob`, remote compaction service |
| `db/blob/` | Integrated BlobDB (value separation) |
| `db/wide/` | Wide-column entities and attribute groups |
| `memtable/` | Memtable representations: `InlineSkipList`, hash-skiplist, hash-linklist, vector; `WriteBufferManager` |
| `table/` | Table reader/writer abstraction; `block_based/` is the production SST format; `plain/`, `cuckoo/` are specialized formats |
| `cache/` | `LRUCache`, `HyperClockCache` (clock_cache.cc), secondary/tiered caches |
| `options/` | Options structs, OPTIONS file serialization, `Configurable`/`Customizable` reflection framework |
| `env/`, `file/` | `Env`/`FileSystem` abstraction, prefetching, direct and async I/O plumbing |
| `util/` | Coding, CRC32c/XXH3, Bloom/Ribbon filter cores, arena, rate limiter, threading |
| `utilities/` | Transactions, backup, checkpoint, TTL, WriteBatchWithIndex, etc. |
| `monitoring/` | Statistics, PerfContext, IOStats, instrumented mutex |
| `include/rocksdb/` | The public API surface |

Component-level design notes maintained in-tree live under
`docs/components/` (read flow, write flow, stress testing) — worth reading
alongside this document.

---

## 2. Core Data Model

### 2.1 Internal keys

Everything in the LSM (memtable entries, SST data blocks) stores **internal
keys**, defined in `db/dbformat.h`:

```
InternalKey := user_key ⧺ (sequence << 8 | value_type)   // 8-byte little-endian trailer
```

- `sequence` is a 56-bit `SequenceNumber` (`kMaxSequenceNumber = 2^56 - 1`).
  Packing seq+type into one uint64 keeps the trailer at 8 bytes
  (`kNumInternalBytes = 8`).
- `value_type` (`enum ValueType`) distinguishes the operation. The on-disk
  values are frozen forever ("DO NOT CHANGE THESE ENUM VALUES"). The
  interesting ones:
  - `kTypeValue (0x1)` — a Put.
  - `kTypeDeletion (0x0)` — a tombstone.
  - `kTypeSingleDeletion (0x7)` — a single-delete tombstone with weaker
    semantics but cheaper compaction (§12.5).
  - `kTypeMerge (0x2)` — a merge operand (§13).
  - `kTypeRangeDeletion (0xF)` — range tombstone, stored in a dedicated meta
    block, not in data blocks (§14).
  - `kTypeBlobIndex (0x11)` — pointer into a blob file (§15).
  - `kTypeWideColumnEntity (0x16)` — wide-column value (§15).
  - `kTypeValuePreferredSeqno (0x18)` — value carrying a unix write time, used
    by tiered storage to place data by age (§21).
  - A family of `WAL only` types (column-family-tagged variants, 2PC markers
    `kTypeBeginPrepareXID`/`kTypeEndPrepareXID`/`kTypeCommitXID`/`kTypeRollbackXID`)
    that appear in WriteBatches/WAL but never in the LSM.

### 2.2 Ordering

`InternalKeyComparator` (`db/dbformat.h`) orders by:

1. `user_key` ascending (per the user's `Comparator`), then
2. `sequence` **descending**, then
3. `type` descending.

Consequence: for a given user key, the *newest* version sorts *first*. A point
lookup for key `k` at snapshot `s` seeks the internal key `(k, s,
kValueTypeForSeek)` and the first entry with matching user key is the visible
version. This single design decision shapes every iterator in the system.

### 2.3 User-defined timestamps

When a comparator declares `timestamp_size() > 0`, every user key carries a
trailing application timestamp that participates in comparison (user key asc,
timestamp desc). This gives applications (e.g. MyRocks-style HLC usage)
a second versioning dimension independent of seqnos. Plumbing for this touches
nearly every layer; the WAL records timestamp sizes per column family via
`kUserDefinedTimestampSizeType` records (`db/log_format.h`) so recovery can
interpret keys correctly even if the comparator changes timestamp size.

### 2.4 WriteBatch encoding

`WriteBatch` (`db/write_batch.cc`, internals in `db/write_batch_internal.h`) is
the unit of atomicity. Its representation is a single string:

```
+--------------+-------------+----------------------------------+
| seqno (8B)   | count (4B)  | records...                       |
+--------------+-------------+----------------------------------+
record := value_type (1B)
          [cf_id varint32]              // only for ColumnFamily* types
          key_len varint32, key bytes
          [val_len varint32, val bytes] // for types that carry a value
```

The 12-byte header's seqno is filled in *late*, by the write thread, once the
batch's position in the global order is known. `count` is the number of
operations; the batch consumes `count` consecutive sequence numbers starting at
`seqno`. Atomicity across column families is inherent: one batch, one WAL
record, one contiguous seqno range.

Optional **per-key protection info** (`protection_bytes_per_key`) attaches
integrity-check bytes to each op (`db/write_batch.cc`,
`db/kv_checksum.h`), carried from the API boundary through the WAL and into
the memtable, defending against in-memory corruption between layers.

---

## 3. The Write Path

Entry point: `DBImpl::WriteImpl()` in `db/db_impl/db_impl_write.cc`. This is
one of the most carefully engineered concurrency hot paths in the codebase.

### 3.1 The write thread and group commit

`WriteThread` (`db/write_thread.h/.cc`) implements leader-based group commit
without a conventional lock:

1. Each writing thread wraps its batch in a `WriteThread::Writer` (a stack
   object!) and calls `JoinBatchGroup(w)`, which CAS-pushes the writer onto a
   lock-free Treiber stack (`newest_writer_`).
2. The writer whose push found an empty stack becomes **leader**; everyone else
   blocks in `AwaitState()` — first spinning, then adaptive yielding
   (`max_yield_usec` tuned by feedback), then a real mutex+condvar sleep. The
   three-stage wait is a deliberate latency/CPU tradeoff; the yield credit
   logic reduces context switches on loaded hosts.
3. The leader calls `EnterAsBatchGroupLeader()`, which snapshots the current
   stack, links the intrusive list oldest→newest, and closes the group
   (bounded by `max_write_batch_group_size_bytes`, default 1MB, and by
   compatibility constraints — e.g. a batch requiring `sync` can't join a
   non-sync group).
4. The leader concatenates all batches (or writes them as separate WAL records
   under one seqno assignment), performs **one** WAL append and (optionally)
   one fsync for the entire group, assigns sequence numbers to each member,
   then either:
   - inserts all batches into memtables itself, or
   - calls `LaunchParallelMemTableWriters()` so every member inserts its own
     batch concurrently (`allow_concurrent_memtable_write`, default on, valid
     for skiplist memtables). Completion is detected by an atomic countdown in
     `WriteGroup`; the last finisher runs `ExitAsBatchGroupFollower/Leader`.
5. The leader hands leadership to the next queued writer and unblocks
   followers, propagating a shared `Status`.

The state machine transitions (`STATE_INIT → STATE_GROUP_LEADER →
STATE_PARALLEL_MEMTABLE_WRITER → STATE_COMPLETED`) are documented at the top of
`db/write_thread.h`; `SetState` fan-out uses a tree pattern so waking N
followers costs O(√N) per thread rather than O(N) serial stores.

### 3.2 Write modes

Three orthogonal option-driven variants restructure this pipeline:

- **`enable_pipelined_write`**: splits the queue in two — a WAL queue and a
  memtable queue (`newest_memtable_writer_`). A group finishes its WAL append,
  then moves to the memtable queue while the *next* group starts its WAL
  append. Increases throughput by overlapping WAL I/O with memtable insertion;
  costs some per-write latency and code complexity.
- **`two_write_queues`**: a second write queue (`nonmem_write_thread_`) that is
  allowed to write the WAL *without* touching memtables. Used by
  WritePrepared/WriteUnprepared transactions (§16) so prepare records commit to
  the WAL concurrently with normal writes. Introduces the split between
  `VersionSet::LastSequence()` (visible for reads) and
  `LastAllocatedSequence()` / `LastPublishedSequence()` — a subtle but critical
  distinction: seqnos are *allocated* at WAL time but *published* (made visible)
  possibly later and possibly out of order, coordinated in
  `WriteImpl`/`WriteImplWALOnly`.
- **`unordered_write`**: relaxes the invariant that memtable insertion order
  matches seqno order. Writers get seqnos in the WAL queue, then insert into
  memtables fully concurrently with no leader; visibility is gated by a
  "pending writes" barrier (`WaitForPendingWrites`) so snapshots still see a
  prefix-consistent state. Buys big throughput on write-heavy multi-CF
  workloads; only offers snapshot consistency when combined with
  WritePrepared transactions (which recover ordering at commit time).

### 3.3 Preprocessing, stalls, and flow control

Before writing, the leader runs `PreprocessWrite()`:

- **WAL rollover** (`SwitchWAL`) when `max_total_wal_size` is exceeded — forces
  flush of the CFs holding the oldest live WAL so it can be released.
- **Write buffer limits**: `WriteBufferManager` (`memtable/write_buffer_manager.cc`)
  enforces a global cap across DBs/CFs; may trigger flushes
  (`HandleWriteBufferManagerFlush`) or block writers outright (stall queue in
  the WBM, `allow_stall`).
- **Write stalls/slowdowns**: `WriteController` applies delay
  (`delayed_write_rate`) or full stop based on per-CF conditions computed in
  `ColumnFamilyData::RecalculateWriteStallConditions()`
  (`db/column_family.cc`): L0 file count (`level0_slowdown_writes_trigger` /
  `level0_stop_writes_trigger`), pending compaction bytes
  (`soft_pending_compaction_bytes_limit` / hard limit), and memtable count
  (`max_write_buffer_number`). The delay token mechanism shapes ingest to
  compaction throughput — this is *the* backpressure mechanism of the engine.

Stall decisions are recalculated on every SuperVersion install and folded into
`SuperVersion::write_stall_condition` so the write path can check them cheaply.

### 3.4 Memtable insertion

`WriteBatchInternal::InsertInto()` walks batch records through a
`MemTableInserter` handler (`db/write_batch.cc`) which dispatches per record
type: finds the CF's current memtable via `ColumnFamilyMemTablesImpl`,
executes merges inline if possible (`moptions->max_successive_merges`), handles
in-place updates (`inplace_update_support`), and honors 2PC markers by
buffering into `rebuilding_trx_` during recovery. Concurrent insertion relies
on the memtable rep's `InsertKeyConcurrently` path (§5).

---

## 4. Write-Ahead Log (WAL)

Files: `db/log_format.h`, `db/log_writer.cc`, `db/log_reader.cc`,
`db/wal_manager.cc`, `db/wal_edit.h`.

### 4.1 Physical format

The WAL is a sequence of 32KB blocks (`kBlockSize = 32768`). Records never
span block boundaries implicitly; instead they are **fragmented**:

```
record_header := crc32c (4B) | length (2B) | type (1B)        // 7 bytes
type ∈ {FULL, FIRST, MIDDLE, LAST}                            // fragmentation
recyclable variants add: log_number (4B)                      // 11-byte header
```

- CRC32c is computed over type+payload and *masked* (`util/crc32c.h`) so that
  CRCs of CRC-bearing data don't collide.
- If < 7 bytes remain in a block, the tail is zero-padded (`kZeroType` is
  reserved so preallocated/zeroed regions parse as "no record").
- **Recycled WALs** (`recycle_log_file_num`): old WAL files are reused to avoid
  allocation/metadata cost on fsync-heavy workloads. Since stale bytes from the
  previous incarnation follow the valid tail, recyclable records embed the log
  number; a mismatch marks end-of-log. This changes the "where does the log
  end" question from "first corruption" to a provable boundary.
- Additional record types: `kSetCompressionType` (WAL compression, e.g. ZSTD),
  `kUserDefinedTimestampSizeType` (§2.3), and `kPredecessorWALInfoType`
  (records the previous WAL's number/size/last-seqno so recovery can detect a
  *missing entire WAL file* — a hole that per-record CRCs cannot catch).
  Types with the high bit set (`kRecordTypeSafeIgnoreMask`) are safely
  ignorable by old readers — forward compatibility for new record types.

### 4.2 Logical content and lifecycle

Each WAL record's payload is one serialized WriteBatch. One WAL serves *all*
column families of a DB. A WAL can be deleted only when every CF whose data it
contains has flushed past it: `ColumnFamilyData::log_number_` tracks the
minimum WAL still needed per CF; `FindObsoleteFiles`
(`db/db_impl/db_impl_files.cc`) computes the minimum over live CFs.
`track_and_verify_wals_in_manifest` additionally records WAL births/deaths in
the MANIFEST (`db/wal_edit.h`) so recovery can verify that no WAL is silently
missing or truncated.

### 4.3 Durability knobs

- `WriteOptions::disableWAL` — memtable-only writes; crash loses them.
- `WriteOptions::sync` — fsync (or fdatasync/`range_sync`) before ack.
- `DBOptions::manual_wal_flush` — WAL appends buffer in memory until
  `FlushWAL()`; pairs with a dedicated `log_write_mutex_`.
- `wal_bytes_per_sync` — background range-sync to smooth I/O.
- WAL fsync failures are treated as fatal-by-default (§18) because a lost
  acknowledged write violates durability; there is no safe retry once the page
  cache state is unknown ("fsyncgate" semantics).

---

## 5. Memtables

Files: `db/memtable.h/.cc`, `db/memtable_list.h/.cc`, `memtable/`,
`include/rocksdb/memtablerep.h`.

### 5.1 Entry encoding

A memtable entry is a single arena-allocated buffer:

```
varint32 internal_key_size | user_key | seq<<8|type (8B) | varint32 value_size | value
```

The `MemTableRep` stores only the `char*` handle; comparison decodes in place.
Memory comes from a `ConcurrentArena` (`memory/concurrent_arena.h`) — per-core
shards over a shared arena to make concurrent allocation nearly contention-free;
an `AllocTracker` charges the `WriteBufferManager`.

### 5.2 InlineSkipList

The default rep (`memtable/skiplistrep.cc` wrapping
`memtable/inlineskiplist.h`) is a highly tuned skip list:

- **Inline tower**: a node's next-pointer array is allocated *before* the key
  bytes in the same allocation (`next_[-(level)]` indexing), so node and key
  share a cache line. Height is geometric with branching factor 4.
- **Lock-free concurrent insert** (`InsertConcurrently`): per-level CAS splice
  with recomputation on failure. This is what `allow_concurrent_memtable_write`
  relies on.
- **Splice hints**: sequential inserts reuse a `Splice` (the remembered search
  path), dropping insert cost from O(log n) to nearly O(1) for keys inserted in
  order — huge for time-ordered keys. Exposed via
  `memtable_insert_with_hint_prefix_extractor` keyed by prefix.
- Reads are lock-free with acquire/release publication; no node is ever freed
  while the memtable is alive (arena lifetime), sidestepping reclamation
  entirely — memtables are freed wholesale after flush once unreferenced.

Alternative reps trade generality for speed: `HashSkipListRep` /
`HashLinkListRep` (per-prefix buckets; no cross-prefix ordered iteration),
`VectorRep` (bulk-load: append then sort). All non-skiplist reps forbid
concurrent insert.

### 5.3 Filters and lookup acceleration

- **Memtable prefix bloom** (`memtable_prefix_bloom_size_ratio`): a dynamic
  bloom (`util/dynamic_bloom.h`) over prefixes (and optionally whole keys),
  consulted by `MemTable::Get` before touching the skip list.
- **Batch lookup optimization** (`memtable_batch_lookup_optimization`): for
  MultiGet, caches the skiplist search path between consecutive sorted keys,
  reducing per-key cost from O(log N) to O(log d) for key distance d.
- **Per-key checksum** (`memtable_protection_bytes_per_key`): verifies entry
  integrity on read/flush.

### 5.4 Immutable memtables and MemTableList

When a memtable fills (`write_buffer_size`, checked via `ShouldScheduleFlush`),
`DBImpl::SwitchMemtable()` (called with the DB mutex under the write thread)
creates a fresh memtable+WAL, moves the old one into the CF's immutable list.
`MemTableList` (`db/memtable_list.h`) manages the flush pipeline with an
internal COW `MemTableListVersion` (so readers hold a stable list without
locks). `min_write_buffer_number_to_merge` batches multiple immutables into a
single flush. **Mempurge** (`experimental_mempurge_threshold`) can instead
compact immutable memtables back into a new memtable, avoiding an L0 write for
short-lived data.

---

## 6. MVCC: Snapshots and SuperVersion

### 6.1 Sequence-number MVCC

Every read is executed "as of" a sequence number: either
`versions_->LastSequence()` at read start or an explicit `Snapshot`.
`SnapshotImpl` (`db/snapshot_impl.h`) is a node in a doubly-linked list guarded
by the DB mutex; the list supplies compaction with the set of seqnos that must
be preserved (§12.4). Snapshots are cheap (no data copy, one list node) but
*not free*: long-lived snapshots pin obsolete versions of keys through
compaction (write-amp and space-amp grow), which is why `SnapshotList` tracks
`oldest` and the compaction code computes visibility "stripes".

### 6.2 SuperVersion: the read-side unit of consistency

`SuperVersion` (`db/column_family.h:207`) bundles, per column family:

```cpp
struct SuperVersion {
  ColumnFamilyData* cfd;
  ReadOnlyMemTable* mem;        // current mutable memtable
  MemTableListVersion* imm;     // immutable memtable list (COW)
  Version* current;             // SST metadata snapshot (§7)
  MutableCFOptions mutable_cf_options;
  uint64_t version_number;
  WriteStallCondition write_stall_condition;
  ...
};
```

A reader that holds one SuperVersion reference sees an immutable, consistent
view of the entire CF state. Acquisition is the read path's only
synchronization: `DBImpl::GetAndRefSuperVersion()` first tries a **thread-local
cached reference** (`ColumnFamilyData::local_sv_`, using a sentinel-swap
protocol `ThreadLocalPtr::Swap` so installers can invalidate cached refs
without blocking readers) and only falls back to `mutex_` + ref-count on cache
miss. This makes Get() mutex-free in steady state.

Installation (`InstallSuperVersion*`, in `db/column_family.cc` and
`db/db_impl/db_impl.cc`) happens under the DB mutex on every memtable switch,
flush result, compaction result, or mutable-option change; old SuperVersions
are unref'd and their cleanup (dropping memtable/Version refs) deferred out of
the mutex where possible.

---

## 7. Version Management and the MANIFEST

Files: `db/version_set.h/.cc`, `db/version_edit.h/.cc`,
`db/version_builder.cc`.

### 7.1 Version / VersionStorageInfo

A `Version` is an immutable snapshot of a column family's SST file metadata:
per-level sorted vectors of `FileMetaData*` (smallest/largest internal key,
file size, seqno range, blob linkage, unique id, ...), held in a
`VersionStorageInfo`. Versions form a linked list per CF (`VersionSet` →
`ColumnFamilyData::current_`); iterators and compactions ref-count the exact
Version they started with, so file deletion is safe only when no Version
references a file (`FindObsoleteFiles` + `PurgeObsoleteFiles`).

`VersionStorageInfo` also precomputes read/compaction acceleration state:
`file_indexer_` (§9.1), per-level `level_files_brief_` (flattened arrays for
cache-friendly binary search), compaction scores, `bottommost_files_`, and
overlap statistics.

### 7.2 VersionEdit and LogAndApply

All metadata mutation is expressed as a `VersionEdit`: a delta record ("add
file F to level L", "delete file", "new WAL", "set last sequence", CF
add/drop, blob file add/garbage, ...). `VersionSet::LogAndApply()` is the
single choke point that:

1. Queues the caller in `manifest_writers_` (group commit for metadata,
   mirroring the write thread pattern — one fsync covers many edits).
2. Applies edits to a `VersionBuilder` to produce the new `Version`
   (recently optimized to avoid quadratic behavior during point-in-time
   MANIFEST recovery).
3. Appends the serialized edits to the **MANIFEST** file — which is physically
   a WAL-format log (§4.1) of VersionEdit payloads — and fsyncs.
4. Installs the new Version and a fresh SuperVersion under the DB mutex.

Atomic multi-CF operations (atomic flush, `IngestExternalFiles`) write all
their edits as **one MANIFEST write group**, which is the recovery-time
atomicity boundary.

### 7.3 CURRENT and manifest rollover

`CURRENT` is a one-line file naming the live `MANIFEST-<number>`. When the
MANIFEST exceeds `max_manifest_file_size`, a new one is written from scratch
(a full snapshot VersionEdit followed by deltas) and CURRENT is atomically
updated (write temp + rename + dir fsync). Recovery = read CURRENT, replay the
named MANIFEST end-to-end through a `VersionBuilder` per CF (§17).

---

## 8. Flush

Files: `db/flush_job.cc`, `db/builder.cc`, `db/db_impl/db_impl_compaction_flush.cc`,
`db/flush_scheduler.cc`.

A flush converts one or more immutable memtables of a CF into an L0 SST:

1. **Scheduling.** Flush requests enter `flush_queue_`; the HIGH-priority
   thread pool runs `DBImpl::BackgroundFlush` -> `FlushMemTableToOutputFile`.
   `max_background_jobs` splits threads between flush/compaction; flushes get
   priority because a stalled flush stalls *all* writes.
2. **Picking.** `FlushJob::PickMemTable` grabs the earliest immutables
   (respecting `min_write_buffer_number_to_merge`); their entries are merged
   through a `MergingIterator` + `CompactionIterator` so that flush already
   performs snapshot-aware garbage collection (e.g. dropping overwritten
   versions invisible to any snapshot).
3. **Building.** `BuildTable()` (`db/builder.cc`) drives the
   `TableBuilder` (section 10), separates large values into blob files if
   enabled (section 15), collects range tombstones into their meta block,
   computes table properties, and optionally records write-time metadata for
   tiered placement.
4. **Install.** The result is a `VersionEdit` (add L0 file, advance CF
   `log_number_`) committed via `LogAndApply` (7.2). Only then can the
   memtables and the WAL segments they covered be released.

**Atomic flush** (`atomic_flush=true`): all CFs' picked memtables are flushed
and their edits committed in a single MANIFEST group, guaranteeing cross-CF
point-in-time consistency after crash even with `disableWAL` writes.

Flush failures do not lose data (memtables are retained); the error handler
(section 18) decides between retry, background recovery, or read-only mode.

---

## 9. The Read Path

### 9.1 Point lookup: `DBImpl::GetImpl` (`db/db_impl/db_impl.cc`)

```
GetAndRefSuperVersion (thread-local, lock-free)
  -> determine read seqno (snapshot or LastSequence / LastPublishedSequence)
  -> LookupKey lk(user_key, seqno)
  -> sv->mem->Get()          // memtable: bloom -> skiplist seek
  -> sv->imm->Get()          // each immutable memtable, newest first
  -> sv->current->Get()      // SST levels via FilePicker
  -> ReturnAndCleanupSuperVersion
```

`Version::Get` (`db/version_set.cc`) uses `FilePicker`:

- **L0**: every file may contain the key, so files are probed newest-to-oldest
  (files are sorted by largest seqno). This is why L0 file count is a direct
  read-amp multiplier and drives write stalls.
- **L1+**: files are disjoint, so binary search finds the single candidate
  file. `FileIndexer` (`db/file_indexer.h`) precomputes, for each file at
  level L, the bracketing index range in level L+1, so the per-level binary
  search narrows as the lookup descends instead of restarting -- an important
  constant factor on deep LSMs.

Each file probe goes `TableCache::Get` (`db/table_cache.cc`; caches open
`TableReader`s keyed by file number, `max_open_files` controls its capacity) ->
`BlockBasedTable::Get` (10.4) with a `GetContext` (`table/get_context.cc`) --
a small state machine (`kNotFound -> kFound / kDeleted / kMerge / ...`)
that accumulates merge operands across levels and stops the descent the moment
a definitive Put/Delete is seen. `PinnableSlice` lets the result reference
cached block memory directly (zero-copy) by pinning the block cache handle.

### 9.2 MultiGet

`DBImpl::MultiGetImpl` + `MultiGetContext` (`table/multiget_context.h`) batch
up to `MAX_BATCH_SIZE = 32` keys into a bitmask-driven pipeline:

- Keys are sorted by comparator, then processed level-by-level *together*:
  one superversion acquisition, shared file/filter/index probes for keys
  falling into the same file, batched bloom checks (with explicit cache-line
  prefetching in the filter code), and the memtable batch-lookup optimization
  (5.3).
- With `USE_COROUTINES` (folly coroutines) and
  `ReadOptions::optimize_multiget_for_io`, index/filter/data block reads for
  *different levels/files* are issued as parallel async I/Os
  (`table/block_based/block_based_table_reader_sync_and_async.h` compiles the
  reader twice, sync and coroutine variants, from one source file).

### 9.3 Iterators

The iterator stack, outermost first:

| Layer | File | Role |
|---|---|---|
| `ArenaWrappedDBIter` | `db/arena_wrapped_db_iter.cc` | Allocates the whole stack in one arena; supports `Refresh()` (re-seat on a new SuperVersion, optionally keeping position) |
| `DBIter` | `db/db_iter.cc` | Internal-to-user key translation: collapses versions, applies tombstones, executes merges, direction reversal, `iterate_upper_bound`, prefix mode |
| `MergingIterator` | `table/merging_iterator.cc` | k-way merge over children with a binary min-heap (forward) / max-heap (reverse); integrates truncated range-tombstone iterators so covered keys are skipped *inside* the heap |
| children | | memtable iter, per-immutable iters, one iter per L0 file, one `LevelIterator` per L1+ level (lazily opens files, two-level pattern) |

Key subtleties worth knowing:

- **Reverse iteration is materially more expensive**: for `Prev`, `DBIter`
  must find the *newest visible* version of the *previous* user key, which
  requires scanning back over all versions of a key; the merging heap must be
  rebuilt on direction change.
- **Tombstone shadowing** is why iterating over a heavily-deleted range is
  slow: `DBIter` silently skips millions of tombstoned internal keys.
  Mitigations: `ReadOptions::iterate_upper_bound` (lets `LevelIterator` stop
  early), compaction filters, `DeleteRange` (section 14) instead of
  point-delete storms, and `max_sequential_skip_in_iterations` (after 8
  sequential skips of the same user key, DBIter switches from Next() to a
  reseek).
- **Prefix mode** (`prefix_extractor` + `ReadOptions::prefix_same_as_start` /
  `total_order_seek=false`): enables bloom-based file/block skipping during
  Seek, at the cost of undefined ordering guarantees outside the prefix.
- **Pinning**: `ReadOptions::pin_data` and the `PinnedIteratorsManager` let
  merge/user layers hold Slices into blocks without copies across `Next()`
  boundaries.
- **Tailing iterators** (`db/forward_iterator.cc`) and the newer `Refresh()`
  path serve change-data-capture style consumers without rebuilding state.

### 9.4 Prefetching and async I/O

`FilePrefetchBuffer` (`file/file_prefetch_buffer.cc`) implements adaptive
readahead for iterators (`ReadOptions::readahead_size`, auto-ramping from 8KB
doubling up to `max_auto_readahead_size` on detected sequential access), with
`async_io=true` overlapping prefetch of block N+1 with consumption of block N
via `FSRandomAccessFile::ReadAsync` (io_uring on Linux). Compaction inputs use
a fixed `compaction_readahead_size`. `MultiScan` (section 21) generalizes this
to caller-declared scan ranges.

---

## 10. SST File Format (BlockBasedTable)

Files: `table/format.h/.cc`, `table/block_based/block_based_table_builder.cc`,
`block_based_table_reader.cc`, `block.cc`, `block_builder.cc`,
`filter_policy.cc` (cores in `util/bloom_impl.h`, `util/ribbon_impl.h`).

### 10.1 Physical layout

```
[data block 1] ... [data block N]
[meta block: filter (full or partitioned)]
[meta block: filter partition index]            (if partitioned)
[meta block: compression dictionary]            (optional)
[meta block: range tombstones]                  (optional)
[meta block: table properties]
[metaindex block]                               -> names/handles of meta blocks
[index block (possibly partitioned)]
[footer]                                        // fixed-size tail
```

The **footer** (`table/format.h`, class `Footer`) is the bootstrap: a magic
number identifying the table format, a format_version, a checksum-type byte,
and BlockHandles (varint64 offset+size pairs) for the metaindex and index
blocks, padded to a fixed length (`Footer::kNewVersionsEncodedLength`).
`BlockBasedTableOptions::format_version` (current default **7**,
`include/rocksdb/table.h`) gates encoding features; readers support all older
versions -- SST files are forward-immutable.

### 10.2 Data blocks

`BlockBuilder` emits ~`block_size` (default 4KB) chunks with **prefix
compression**: each entry stores `shared_len | non_shared_len | value_len`
varints, then the unshared key bytes and the value. Every
`block_restart_interval` (default 16) entries, a **restart point** stores the
full key; the block tail holds the uint32 restart-offset array + count. Binary
search happens over restart points, then linear decode within an interval --
the classic space/CPU tradeoff. An optional **hash index inside the block**
(`data_block_index_type = kDataBlockBinaryAndHash`) maps key -> restart index
for point lookups, skipping the binary search.

Blocks are individually compressed (per-CF `compression`, default now LZ4;
`bottommost_compression` commonly ZSTD with dictionary compression) and
followed by a 5-byte trailer: compression-type byte + 4-byte checksum
(CRC32c/xxHash/XXH3) covering the compressed payload plus the type byte.

### 10.3 Index and filters

- **Index block**: one entry per data block -- a *separator* key (shortened
  via the comparator's `FindShortestSeparator`) -> BlockHandle. Variants:
  `kBinarySearch`, `kHashSearch` (prefix -> block), `kTwoLevelIndexSearch`
  (**partitioned index**: top-level index in memory, partitions demand-loaded
  through the block cache -- bounds memory for huge files), and
  `kBinarySearchWithFirstKey` (stores each block's first key so iterators can
  defer data-block loads on Seek).
- **Filter**: modern full filters are built over the whole file's keys (or
  prefixes) and probed *before* the index on Get:
  - **Bloom** (`util/bloom_impl.h`, `FastLocalBloomImpl`): cache-local -- all
    of a key's probes land in one 64-byte cache line selected by hash;
    SIMD/prefetch friendly; ~10 bits/key gives ~1% false positives.
  - **Ribbon** (`util/ribbon_impl.h`, `Standard128Ribbon`): ~30% smaller than
    Bloom at equal FP rate, higher construction CPU -- the intended choice for
    cold levels. `NewRibbonFilterPolicy(bpk, bloom_before_level)` mixes both:
    Bloom for high (hot, rewritten-often) levels, Ribbon below.
  - **Partitioned filters** mirror partitioned indexes.
  - `whole_key_filtering` and prefix filtering can coexist in one filter.
- `cache_index_and_filter_blocks` moves index/filter residency decisions into
  the block cache (with `pin_l0_filter_and_index_blocks_in_cache` and
  `MetadataCacheOptions` for pinning policy); otherwise they live in
  table-reader heap memory bounded by `max_open_files`.

### 10.4 Read algorithm (`BlockBasedTable::Get`)

```
FullFilterKeyMayMatch?  -- no --> return NotFound   (bloom/ribbon, ~1 cache miss)
  \-- yes -> index Seek -> data BlockHandle
       -> block cache lookup (section 11); on miss: ReadBlockContents
          (file I/O, verify checksum, decompress with dict) -> insert to cache
       -> Block::Seek (restart-point binary search) -> iterate entries,
          feed GetContext until user_key mismatch
```

### 10.5 Identity: cache keys and unique IDs

Every SST gets a globally-unique identity derived from the DB session id plus
file number (`cache/cache_key.cc`, `table/unique_id.cc`). `OffsetableCacheKey`
gives each block a stable cache key = file identity combined with offset,
valid across DB reopens -- this is what makes persistent/secondary caches and
caches shared across DB instances sound. The same mechanism backs the
128/192-bit `GetUniqueIdFromTableProperties` used to detect file mixups.

---

## 11. Block Cache

Files: `cache/sharded_cache.h`, `cache/lru_cache.cc`, `cache/clock_cache.cc`,
`cache/secondary_cache_adapter.cc`, `cache/compressed_secondary_cache.cc`.

The block cache stores **uncompressed** blocks (data, index, filter,
dictionary), keyed as in 10.5, with per-entry `CacheEntryRole` accounting.
All implementations are sharded (`ShardedCache`; shard chosen by high hash
bits) to reduce lock contention.

### 11.1 LRUCache

Classic chained hash table + doubly-linked LRU list per shard, one mutex per
shard. Notable engineering:

- **Midpoint insertion / priority pools**: the LRU list is segmented into
  high-priority (index/filter), low-priority, and bottom-priority (blob,
  secondary-cache-promoted) regions (`high_pri_pool_ratio`,
  `low_pri_pool_ratio`) so scans of cold data cannot flush hot metadata.
- Handles are ref-counted; `strict_capacity_limit` decides whether inserts may
  exceed capacity (pinned entries) or fail with `Status::MemoryLimit`.
- Erase-on-zero-ref semantics let iterators pin blocks (9.3) safely.

### 11.2 HyperClockCache

`clock_cache.cc` implements a mostly **lock-free CLOCK** replacement cache --
the recommended choice for high-concurrency read workloads. Two variants
behind `HyperClockCacheOptions`:

- **FixedHyperClockCache**: open-addressed table sized from
  `estimated_entry_charge`; no per-shard mutex on lookup/release -- atomics
  encode ref-count and clock state per slot.
- **AutoHyperClockCache** (used when no charge estimate is given): grows the
  table dynamically (linear-hashing style) while preserving lock freedom on
  the read path.

Eviction sweeps a clock hand decrementing usage counters; insertion tolerates
transient over-occupancy. The design deliberately trades exact LRU ordering
for removal of the shard mutex -- the LRU mutex is the classic scalability
wall at high QPS.

### 11.3 Secondary and tiered caches

`SecondaryCache` chains a second tier under the primary block cache:
`CompressedSecondaryCache` keeps *compressed* blocks in memory (a mid-tier
between the uncompressed cache and disk), and NVM/SSD-backed implementations
(e.g. CacheLib in Meta production) sit behind the same interface. On
primary-cache eviction, eligible entries demote (with an admission policy and
"standalone handle" probation so one-hit-wonders don't churn the tier); on
miss, lookups promote asynchronously. `TieredSecondaryCache` +
`CacheWithSecondaryAdapter` manage split capacity accounting, including
dynamic adjustment of the compressed-tier ratio.

### 11.4 Memory accounting via the cache

`CacheReservationManager` (`cache/cache_reservation_manager.h`) "charges"
non-block memory (memtables via `WriteBufferManager(cache)`, filter
construction scratch, file metadata, blob cache,
`CacheEntryRoleOptions::charged`) into the block cache by inserting
placeholder entries -- giving operators one knob that actually bounds total
memory, with `MemoryAllocator` / `jemalloc_nodump_allocator` handling the real
allocations.

---

## 12. Compaction

Files: `db/compaction/compaction_picker*.cc`, `compaction_job.cc`,
`compaction_iterator.cc`, `compaction_outputs.cc`, `subcompaction_state.cc`,
`compaction_service_job.cc`.

Compaction is where the LSM's deferred work happens: merging sorted runs,
dropping shadowed versions and tombstones, applying compaction filters,
re-compressing, and moving data across storage tiers. It is the primary
determinant of write amplification, space amplification, and read
amplification -- pick any two.

### 12.1 Leveled compaction (`compaction_picker_level.cc`)

The default style. Invariants: L0 files may overlap each other; L1+ each form
a single sorted run of disjoint files.

- **Scoring** (`VersionStorageInfo::ComputeCompactionScore`): L0 score =
  max(file count / `level0_file_num_compaction_trigger`, L0 bytes /
  `max_bytes_for_level_base` considerations); Ln score = level bytes (minus
  bytes already being compacted) / target. The highest score >= 1 wins.
  *Compensated file size* inflates files full of tombstones/deletions so
  delete-heavy files get compacted sooner (this is the mechanism behind
  tombstone GC responsiveness).
- **Dynamic level sizing** (`level_compaction_dynamic_level_bytes`, default
  on): target sizes are derived *upward from the last level's actual size*
  (each upper level = lower / `max_bytes_for_level_multiplier`), rather than
  downward from L1. This bounds space-amp at ~1.11x for multiplier 10 and
  makes level targets self-adjusting as the DB grows/shrinks. Small DBs keep
  data in the last levels only; L0 can compact directly into the first
  non-empty target level.
- **File selection**: within the winning level, files are prioritized by
  `kMinOverlappingRatio` (default; file size / overlapping bytes in the next
  level -- cheapest write-amp first), with marked-for-compaction files (TTL,
  periodic, bottommost seqno-zeroing) taking precedence. The chosen file plus
  its next-level overlap defines the compaction; a hygiene expansion may pull
  in more same-level files if it does not grow next-level overlap.
- **Intra-L0 compaction**: when L0 has many small files but L0->Lbase would be
  too wide, L0 files are compacted among themselves to cut read-amp.
- **Trivial move**: if an input file overlaps nothing in the output level, it
  is "moved" by a metadata-only VersionEdit -- no I/O at all.

### 12.2 Universal compaction (`compaction_picker_universal.cc`)

Tiered-style: the DB is a sequence of **sorted runs** (each L0 file is a run;
each populated level is a run), newest first. Triggers, in priority order:

1. **Space amplification** (`max_size_amplification_percent`): if
   size(all runs except last) / size(last run) exceeds the limit, compact
   everything into the last level (full compaction).
2. **Size ratio** (`size_ratio`): merge a prefix of runs whose sizes are
   within the ratio of their accumulated total (keeps runs geometrically
   spaced -- this is the tiered write-amp/read-amp dial).
3. **Run count** (`level0_file_num_compaction_trigger`): merge the newest runs
   to keep total run count bounded.

Universal trades write-amp (lower: each byte is rewritten O(log N) times
without the leveled fan-out multiplier) for space-amp (transiently up to 2x
during full compaction) and read-amp (more runs to consult). Modern universal
supports subcompactions and incremental last-level compaction to soften the
full-compaction cliffs.

### 12.3 FIFO compaction (`compaction_picker_fifo.cc`)

For time-series/cache data: no merging at all -- oldest files are dropped when
`max_table_files_size` is exceeded or TTL expires; optional lightweight
intra-L0 merging and file-temperature-based tiering
(`file_temperature_age_thresholds`). O(1) write-amp, no ordering across
files, deletion is the only GC.

### 12.4 CompactionIterator: the semantic core (`compaction_iterator.cc`)

Everything above decides *which files*; `CompactionIterator` decides *which
internal keys survive*. It consumes a merged input stream (via
`CompactionMergingIterator`, which also interleaves range tombstones) and for
each user key applies, in order:

- **Snapshot stripes**: the live snapshot list partitions seqnos into stripes;
  within a stripe only the newest version of a key is needed. A Put shadowed
  by a newer Put *in the same stripe* is dropped; versions straddling a
  snapshot boundary are all preserved. `earliest_snapshot_` +
  `snapshot_checker` (WritePrepared) drive visibility decisions.
- **Tombstone GC**: a Delete can be dropped only at the **bottommost level**
  for its key range (`Compaction::KeyNotExistsBeyondOutputLevel` checks lower
  levels' key ranges), and only if no live snapshot can still see the deleted
  key. Until then it must be preserved to keep shadowing older versions.
- **SingleDelete pairing**: `kTypeSingleDeletion` annihilates together with
  the single Put it meets (both removed mid-LSM, without needing to reach the
  bottom) -- valid only under the API contract that a key is never
  overwritten or double-single-deleted; violations surface as corruption
  in `force_consistency_checks` / stress tests.
- **Merge folding**: consecutive merge operands are combined via
  `MergeHelper::MergeUntil` (section 13), producing a Put if a base value or
  tombstone is reached within visibility constraints.
- **Compaction filter**: user callback may drop, keep, change value, or
  convert entries (`CompactionFilter::FilterV3` supports wide columns and
  blob values); filter decisions interact with snapshots
  (`ignore_snapshots` semantics were a historical footgun -- filters run
  regardless of snapshots now, by contract).
- **Seqno zeroing**: at the bottommost level, if a key's version is visible to
  all snapshots, its seqno is rewritten to 0. This is a real optimization:
  seqno-0 keys prefix-compress better and enable the "no seqno" fast paths;
  it is also load-bearing for ingestion (global seqno) semantics.
- **Preferred-seqno / write-time handling** (`kTypeValuePreferredSeqno`) for
  tiered placement decisions (section 21).

### 12.5 CompactionJob mechanics

`CompactionJob::Run` splits the key range into **subcompactions**
(`max_subcompactions`, boundary picking uses file-boundary anchors sampled
across inputs) executed on the thread pool; each writes its own output files
via `CompactionOutputs` (cutting files at `target_file_size_base`, at
grandparent-overlap limits (`max_compaction_bytes`) to keep *future*
compactions bounded, and at tiering boundaries). Outputs are fsynced, table
properties verified, then all subcompaction edits are installed atomically via
`LogAndApply`. Compaction reads bypass the block cache for data blocks
(`fill_cache=false`) and may use their own direct-I/O settings
(`use_direct_io_for_flush_and_compaction`,
`use_direct_io_for_compaction_reads`) to avoid polluting the page cache;
writes go through a `WritableFileWriter` with rate limiting
(`rate_limiter`, priority LOW) and `bytes_per_sync` smoothing.

**Remote compaction** (`compaction_service_job.cc`, `CompactionService`
interface): the primary serializes a `CompactionServiceInput` (input files,
options, snapshot context), ships it to a stateless worker which runs
`DBImplSecondary::CompactWithoutInstallation` on shared storage, and installs
the returned `CompactionServiceResult` locally, falling back to local
compaction on deserialization failure. This is the substrate for
disaggregated-storage deployments.

**Manual compaction** (`CompactRange`) coordinates with automatic compactions
through `manual_compaction_dequeue_` and can force rewriting the bottommost
level (`BottommostLevelCompaction`), used for space reclamation and
tombstone purging.

### 12.6 TTL, periodic, and marked compactions

`ttl` and `periodic_compaction_seconds` mark files whose data age exceeds
thresholds (age from table properties' creation time / oldest key time);
`VersionStorageInfo::ComputeFilesMarkedForCompaction` feeds these into the
picker ahead of score-based work. Bottommost files with seqnos > 0 blocked
only by an old snapshot get re-marked when the snapshot releases
(`bottommost_files_mark_threshold_`).

---

## 13. Merge Operator Internals

Files: `db/merge_helper.cc`, `db/merge_context.h`,
`include/rocksdb/merge_operator.h`.

`Merge(k, operand)` writes `kTypeMerge` entries -- deferred read-modify-write.
The cost model is asymmetric by design: writes stay O(1); reads must collect
*all* merge operands newer than the newest Put/Delete for the key:

- **Get**: `GetContext` accumulates operands in a `MergeContext`
  (autovector of pinned slices) while descending memtable -> levels; on
  hitting a base value/tombstone (or exhausting the LSM), calls
  `MergeOperator::FullMergeV2` with `MergeOperationInput{key, existing_value,
  operand_list}`. `GetMergeOperands()` exposes the raw operands without
  merging.
- **Iterators**: `DBIter` performs the same resolution per key during scans.
- **Compaction**: `MergeHelper::MergeUntil` opportunistically folds operand
  chains: full merge if a base value is reached within the same snapshot
  stripe, else `PartialMergeMulti` to at least shorten the chain. Operands
  can never be reordered across snapshot boundaries or unflushed levels --
  correctness requires the operand *sequence* be preserved exactly.
- `max_successive_merges` bounds in-memtable chains by eagerly merging at
  write time when the memtable already holds >= N operands for the key
  (trading write CPU for read latency).

The API contract that bites people: `FullMergeV2` must be deterministic and
associative-with-respect-to-partial-merge; a failed merge
(`Status::MergeOperatorFailed`) poisons reads of that key and, during
compaction, fails the compaction.

---

## 14. Range Deletions

Files: `db/range_del_aggregator.cc`, `db/range_tombstone_fragmenter.cc`,
`table/block_based` range tombstone meta block, `db/memtable.cc` range-del
handling.

`DeleteRange(begin, end)` writes a `kTypeRangeDeletion` entry
(key=begin, value=end) into a **dedicated range-del memtable rep** alongside
the point memtable; on flush these become the SST's range tombstone meta
block (unfragmented, as written).

The read-side machinery:

1. **Fragmentation** (`FragmentedRangeTombstoneList`): overlapping tombstones
   from one source are split at all endpoints into non-overlapping fragments,
   each carrying the set of seqnos covering it, sorted so lookups are a
   binary search. Memtable fragmentation is done lazily/cached
   (`FragmentedRangeTombstoneListCache`) since the memtable keeps changing.
2. **Aggregation** (`RangeDelAggregator`): a point lookup consults per-source
   fragmented lists top-down -- a key is dead if any tombstone fragment with
   seqno > key's seqno (and <= read seqno) covers it. `ShouldDelete` is O(log
   fragments) per source with skew-friendly caching of the current fragment
   for iteration.
3. **Iterators**: `MergingIterator` integrates `TruncatedRangeDelIterator`s
   (tombstones clipped to their file's boundaries -- truncation is essential
   because a tombstone in file F must not delete keys in a *newer* file that
   sorts inside F's range but was excluded from F's compaction) so covered
   point keys are skipped inside the heap rather than surfaced to DBIter.
4. **Compaction**: `CompactionRangeDelAggregator` drops covered point keys,
   writes surviving tombstone fragments into outputs (clipped per output
   file), and drops tombstones entirely at the bottommost level when no
   snapshot needs them. Files whose range is fully covered by a tombstone can
   be deleted wholesale (`DeleteFilesInRange` for the manual variant).

Historical note: pre-fragmentation implementations of DeleteRange had both
correctness and O(n^2) performance issues; the fragmenter design (2018) is
what made DeleteRange production-safe. It remains cheaper to write than N
point tombstones but adds per-read overhead proportional to live tombstone
count -- the guidance is "use for bulk erase, then compact".

---

## 15. Integrated BlobDB and Wide Columns

### 15.1 BlobDB (`db/blob/`)

Value separation for large values (WiscKey-style), integrated into the main
engine (the old `utilities/blob_db` is legacy):

- With `enable_blob_files=true`, flush/compaction route values >=
  `min_blob_size` into `.blob` files via `BlobFileBuilder`; the LSM stores a
  `kTypeBlobIndex` entry whose value is a `BlobIndex` {blob file number,
  offset, size, compression}.
- Blob files are Version citizens: `BlobFileMetaData` in
  `VersionStorageInfo`, added/garbage-tracked through VersionEdits
  (`blob_file_addition.h`, `blob_file_garbage.h`), linked to the SSTs that
  reference them (`linked_ssts`).
- Reads resolve through `BlobSource` (`blob_file_cache.h` for open readers,
  `blob_cache` in the block cache with `prepopulate_blob_cache` warm-on-write)
  -- one extra I/O per cold blob read, in exchange for compaction not
  rewriting big values.
- **GC is piggybacked on compaction**: `enable_blob_garbage_collection`
  forces compactions to rewrite blob references older than
  `blob_garbage_collection_age_cutoff` (fraction of blob-file number space),
  relocating live blobs into new files so old files' garbage ratio reaches
  100% and they get deleted; `blob_garbage_collection_force_threshold`
  triggers targeted compactions of SSTs pointing at high-garbage blob files
  (`ComputeFilesMarkedForForcedBlobGC`).
- Newest addition (11.6): experimental **embedded blob SSTs**
  (`SstFileWriter::OpenWithEmbeddedBlobs`, `table/embedded_blob_sst.h`) --
  blob records co-located in the same SST file for ingestion workflows.

### 15.2 Wide columns and attribute groups (`db/wide/`)

`PutEntity(key, {col->val,...})` stores a `kTypeWideColumnEntity` whose value
is a serialized column map (`wide_column_serialization.cc`); the anonymous
default column bridges to plain `Get` (returns the default column's value).
`AttributeGroup` APIs generalize entities across column families (one logical
entity's attributes partitioned across CFs, read via
`GetEntity`/`MultiGetEntity` and the `AttributeGroupIterator`
(`db/attribute_group_iterator_impl.cc`)). `CoalescingIterator` merges
same-key entities across CFs. Merge and compaction filters have
entity-aware hooks (`FullMergeV3`, `FilterV3`).

---

## 16. Transactions

Files: `utilities/transactions/`, `utilities/write_batch_with_index/`,
`db/snapshot_checker.h`.

### 16.1 Building block: WriteBatchWithIndex

`WriteBatchWithIndex` (WBWI) wraps a WriteBatch with a skip-list index over
its entries, enabling read-your-own-writes (`GetFromBatchAndDB`,
`NewIteratorWithBase` overlays batch entries on a DB iterator). All
transaction variants stage uncommitted writes in a WBWI. A recent
optimization (`memtable/wbwi_memtable.cc`) lets a committed WBWI be **ingested
as an immutable memtable** instead of being re-inserted key-by-key -- large
transaction commit becomes O(1).

### 16.2 Optimistic transactions

`OptimisticTransactionDB`: no locks; each written key records its expected
last seqno. Commit enters the write thread and validates via
`TransactionUtil::CheckKeysForConflicts` -- each key's latest seqno in the DB
(checked against memtables; fails with `Status::TryAgain` if memtable history
is insufficient) must not exceed the snapshot seqno. Cheap for low contention,
aborts under contention, validation cost O(keys written).

### 16.3 Pessimistic transactions: lock management

`PessimisticTransactionDB` acquires locks at write time (and `GetForUpdate`):

- **PointLockManager** (`utilities/transactions/lock/point/`): 16 (default
  `num_stripes`) stripe-mutexed hash maps per CF; lock info tracks
  shared/exclusive owners and expiration. Deadlock detection is an on-demand
  BFS over the wait-for graph (`deadlock_detect`, depth-bounded, with a
  captured `DeadlockPath` buffer for debugging); otherwise timeouts
  (`lock_timeout`) break cycles.
- **RangeTreeLockManager** (`lock/range/range_tree/`, derived from PerconaFT's
  locktree): interval tree of locked ranges with escalation
  (`max_num_locks`), supporting `GetRangeLock` for phantom-free range
  guarding. Wait notifications are interruptible via kill callbacks.

### 16.4 Write policies

The commit protocol is pluggable (`TxnDBWritePolicy`):

- **WriteCommitted** (default): data enters memtables only at Commit; Prepare
  writes only WAL markers (`kTypeBeginPrepareXID` ... `kTypeEndPrepareXID`).
  Simple visibility (everything in the LSM is committed); long transactions
  buffer their whole write set in memory.
- **WritePrepared**: data enters memtables at *Prepare* with prepare-seqnos;
  Commit writes a commit marker and publishes a (prepare_seq -> commit_seq)
  mapping. Visibility ("is seq visible to snapshot?") is answered by
  `WritePreparedTxnDB::IsInSnapshot` using a lock-free **CommitCache** (a
  fixed 8M-slot (2^23) ring of recent commits), `PreparedHeap` for
  in-flight minimums, `old_commit_map_` for evicted entries still needed by
  live snapshots, and `max_evicted_seq_` advancing as the cache wraps. A
  `SnapshotChecker` threads this logic into compaction and reads. Requires
  `two_write_queues` for concurrent prepare/commit WAL writes.
- **WriteUnprepared**: extends WritePrepared by streaming large transactions
  into memtables *before* Prepare (unprep-seqnos tracked per txn), bounding
  transaction memory; rollback becomes a write of compensating records.

The seqno bookkeeping distinctions (allocated vs published, section 3.2) exist
almost entirely to serve these policies.

### 16.5 Two-phase commit and recovery

XA-style 2PC: `Prepare()` hardens the transaction in the WAL; on recovery,
prepared-but-uncommitted transactions are reconstructed
(`rebuilding_trx_` in the WriteBatch handler) and offered via
`GetAllPreparedTransactions` for the coordinator to commit/rollback. WAL
retention respects the oldest prepared transaction's log
(`FindMinLogContainingOutstandingPrep`).

---

## 17. Crash Recovery and DB Open

Files: `db/db_impl/db_impl_open.cc`, `db/version_set.cc` (`VersionSet::Recover`),
`db/log_reader.cc`, `db/repair.cc`.

`DB::Open` sequence:

1. **Lock and identify**: acquire the `LOCK` file, read `IDENTITY`, read
   `CURRENT` -> live MANIFEST name.
2. **MANIFEST replay** (`VersionSet::Recover`): stream VersionEdit records
   through per-CF `VersionBuilder`s, reconstructing the exact file topology,
   `next_file_number_`, `last_sequence_`, per-CF `log_number_`, and (if
   tracked) the expected set of live WALs. Column families are instantiated
   against their recorded comparator/options (mismatches fail Open).
3. **WAL discovery and ordering**: list `*.log` files with number >= the
   minimum CF `log_number_`, sort by number, replay in order.
4. **WAL replay** (`DBImpl::RecoverLogFiles`): each record's WriteBatch is
   re-inserted into memtables (through the same `MemTableInserter` as live
   writes -- one code path), honoring per-CF `log_number_` cutoffs so already
   flushed data is not double-applied. Memtables that fill during replay are
   flushed mid-recovery. Sequence handling accounts for the
   two_write_queues/unordered cases (seqno holes from unpublished writes).
5. **WAL consistency policy** (`wal_recovery_mode`):
   - `kPointInTimeRecovery` (default): stop at the first corruption; recover
     the longest clean prefix. Combined with
     `track_and_verify_wals_in_manifest` and `kPredecessorWALInfoType`
     records, silent truncation/hole detection is strong.
   - `kAbsoluteConsistency`: any corruption fails Open (for apps that cannot
     tolerate losing acknowledged-but-unsynced writes).
   - `kTolerateCorruptedTailRecords` / `kSkipAnyCorruptedRecords`: looser
     modes for best-effort salvage.
6. **Post-replay**: flush recovered memtables if `avoid_flush_during_recovery`
   is off; write a fresh MANIFEST snapshot if needed; delete obsolete files;
   start background work. `DBImpl::DeleteObsoleteFiles` is versioned-state
   driven -- a file is deletable only if it is in no live Version, not
   pending output of a running job (`pending_outputs_`/file-number barriers),
   and not held by `DisableFileDeletions`/checkpoints.

**Atomic-flush recovery** must additionally discard *suffixes* of CF data
that would break cross-CF consistency (only whole atomic groups are kept).
`DBOptions::best_efforts_recovery` relaxes MANIFEST-vs-storage mismatches by
rolling back to the newest point-in-time Version whose files all exist --
the same machinery powering the **secondary/follower** instances
(`db_impl_secondary.cc`, `db_impl_follower.cc`) that continuously tail the
primary's MANIFEST/WALs on shared or replicated storage.

`db/repair.cc` (`RepairDB`) is the last resort: scans all SSTs/WALs, rebuilds
a MANIFEST from file contents, sacrificing exact history for availability.

---

## 18. Error Handling and Fault Tolerance

Files: `db/error_handler.cc`, `include/rocksdb/status.h`, `util/status.cc`.

### 18.1 Status discipline

Every fallible operation returns `Status` (payload: code + subcode + severity
+ message, stored in a single heap string only when non-OK -- the OK path is
allocation-free). Builds with `ASSERT_STATUS_CHECKED` make destroying an
unchecked non-OK Status abort: error handling is enforced, not advisory.
`IOStatus` extends this with retryability and data-loss hints from the
FileSystem layer.

### 18.2 The ErrorHandler state machine

Background errors (flush/compaction/WAL/manifest I/O failures) route through
`ErrorHandler::SetBGError`, which maps (code, subcode, retryable flag,
operation context) to a severity:

- `kSoftError`: e.g. retryable flush I/O error -- writes may continue
  (bounded), background retry scheduled.
- `kHardError`: e.g. `Status::NoSpace` -- writes rejected until resolved;
  compactions may proceed after space recovery (SstFileManager tracks free
  space and can clear the condition when reclaimable space suffices).
- `kFatalError` / `kUnrecoverableError`: e.g. WAL sync failure, MANIFEST
  write failure with unsynced data at risk -- DB becomes read-only; only
  reopen (or `DBImpl::Resume`) can clear it.

**Automatic recovery** (`ErrorHandler::RecoverFromBGError`,
`max_bgerror_resume_count`, `bgerror_resume_retry_interval`): for retryable
errors, a dedicated recovery thread re-runs the failed operation (e.g. retry
flush) and, on success, restores normal operation -- the mechanism behind
surviving transient remote-storage blips (recent work extends this tolerance
into db_stress/db_bench). `EventListener::OnErrorRecovery*` callbacks expose
the lifecycle.

### 18.3 Data integrity layers

Defense in depth against corruption:

- Block checksums (CRC32c/XXH3) on every SST block; whole-file checksum
  (`file_checksum_gen_factory`, verified by backup/ingestion).
- WAL per-record CRC + WAL-set verification in MANIFEST.
- Key-value protection (`protection_bytes_per_key`) covering
  WriteBatch -> WAL -> memtable -> flush handoffs
  (`db/kv_checksum.h`) -- catches in-memory bit flips.
- `paranoid_file_checks`, `verify_checksums_in_compaction`,
  `force_consistency_checks` (LSM invariant assertions on every Version
  build), `VerifyChecksum()` / `VerifyFileChecksums()` APIs.
- `best_efforts_recovery` + `RepairDB` for salvage.

---

## 19. Concurrency Model Summary

A mental model of "who holds what":

| Mechanism | Protects | Notes |
|---|---|---|
| `DBImpl::mutex_` (one per DB) | Version/SuperVersion install, CF set, flush/compaction scheduling state, snapshot list, memtable switch | An `InstrumentedMutex`; long operations must release it (pattern: gather -> unlock -> I/O -> relock -> install) |
| Write thread (lock-free stack + per-writer futex-style waits) | Total order of writes, WAL append, seqno allocation | Section 3; second instance for `two_write_queues` |
| SuperVersion refcount + thread-local cache | Entire read-side state per CF | Readers never take the DB mutex in steady state |
| `InlineSkipList` CAS splices | Concurrent memtable insert | Publication via release stores; readers acquire |
| Block/table cache shard locks or lock-free clock | Cache state | HyperClock removes even shard locks from reads |
| `manifest_writers_` queue | MANIFEST append ordering | Group commit, mirrors write thread |
| Background job bookkeeping (`bg_compaction_scheduled_`, `pending_outputs_`, ...) | File lifetime vs. deletion | All under DB mutex |
| Port primitives (`port/`) | cross-platform mutex/condvar/CPU hints | plus `folly` synchronization when built with folly |

Testing machinery for concurrency: **SyncPoint** (`test_util/sync_point.h`)
lets tests impose cross-thread orderings and inject callbacks at named source
locations (`TEST_SYNC_POINT`), the foundation of most race-regression tests;
`db_stress` (`db_stress_tool/`) runs randomized concurrent workloads with
fault injection (`utilities/fault_injection_fs.h`) and verifies against an
in-memory model plus WritePrepared/timestamp/txn-specific invariants; the
crash test kills processes mid-write and verifies recovery.

---

## 20. Performance Engineering Machinery

Cross-cutting infrastructure a performance investigation will touch:

- **Arena allocation** (`memory/arena.h`, `concurrent_arena.h`): memtables and
  iterator stacks allocate from arenas (bump-pointer, per-core sharded for
  memtables; optional hugepage backing) -- allocation cost and fragmentation
  are engineered out of the hot path.
- **Statistics** (`monitoring/statistics.cc`): global tickers + histograms,
  implemented with core-local counters (`util/core_local.h`) to avoid
  cache-line ping-pong; `stats_level` trades detail for overhead.
- **PerfContext / IOStatsContext** (`monitoring/perf_context_imp.h`):
  thread-local per-operation counters (block read counts, bloom hits, mutex
  wait micros...) -- the first tool to reach for on "why is this Get slow".
- **Rate limiting** (`util/rate_limiter.cc`): token-bucket
  `GenericRateLimiter` with priority (compaction LOW vs flush HIGH), optional
  auto-tuning; `WriteAmpBasedRateLimiter` variants exist downstream.
- **Thread pools** (`util/threadpool_imp.cc`, `env/env_posix.cc`): HIGH/LOW
  pools sized by `max_background_jobs`; `IncreaseParallelism()` is the
  one-liner most deployments need.
- **File abstraction** (`include/rocksdb/file_system.h`, `file/`):
  `FSSequentialFile`/`FSRandomAccessFile`/`FSWritableFile` +
  `WritableFileWriter`/`RandomAccessFileReader` wrappers centralize
  buffering, checksum, rate limit, sync policy (`bytes_per_sync`),
  direct I/O alignment, and IO instrumentation. `Env` composes a FileSystem
  + SystemClock + thread facilities; everything (mock, encrypted, remote,
  fault-injecting filesystems) plugs in here.
- **Direct I/O + readahead**: `use_direct_reads`,
  `use_direct_io_for_flush_and_compaction`, and (new)
  `use_direct_io_for_compaction_reads` separate user-read caching from
  compaction streaming; `FilePrefetchBuffer` handles alignment and overlap.
- **db_bench + microbenchmarks**: `tools/db_bench_tool.cc` is the canonical
  macro-benchmark (every perf-relevant feature must be exercisable there);
  `microbench/` has google-benchmark units. Perf claims in PRs are expected
  to come with db_bench numbers (see CLAUDE.md guidance in-tree).
- **Tracing**: `trace_replay/` records and replays workloads
  (`DB::StartTrace`), including block-cache access traces for cache research.

---

## 21. Notable Modern Features

A quick index of newer subsystems a current-tree reader will encounter:

- **Tiered storage** (`last_level_temperature`,
  `preclude_last_level_data_seconds`, `preserve_internal_time_seconds`):
  data's write time is tracked (seqno-to-time mapping maintained in a
  dedicated table property / `SeqnoToTimeMapping`), and compaction places
  recent data in the "proximal" level and cold data in the last level, which
  can live on a different storage temperature (`Temperature::kCold` handled
  by the FileSystem). `kTypeValuePreferredSeqno` entries carry write time.
- **MultiScan** (`ReadOptions` + iterator `Prepare(scan_ranges)`): declares
  multiple scan ranges up front so the table reader can batch/async-prefetch
  their blocks (io_uring), plus the read-scoped block buffer provider API to
  bypass the block cache for scan-only data.
- **Follower/secondary instances** (`db_impl_follower.cc`,
  `db_impl_secondary.cc`): read-only replicas tailing MANIFEST/WAL on shared
  storage with catch-up (`TryCatchUpWithPrimary`).
- **External file ingestion** (`db/external_sst_file_ingestion_job.cc`):
  `SstFileWriter`-produced files ingested with a **global seqno** (either
  assigned into a level below all overlapping data or forced to L0);
  two-phase `PrepareFileIngestion`/`CommitFileIngestionHandle` (11.5) moves
  the heavy work off the write-blocking critical section. Ingestion behind
  live writes interacts subtly with seqno zeroing and file boundaries --
  a historically bug-rich area.
- **Remote compaction / disaggregated storage**: section 12.5; plus
  `OffpeakTimeOption`, `CompactionService` scheduling, and remote-IO error
  tolerance in stress tooling.
- **User-defined timestamps** (section 2.3) now supported across most APIs
  including read-only DBs and transactions
  (`WriteCommittedTransaction` timestamped commits).
- **Per-key placement / attribute groups / wide columns** (section 15.2).
- **WBWI-as-memtable ingestion** (section 16.1) for large-transaction commit.
- **Embedded blob SSTs** (11.6, experimental; section 15.1).
- **Auto-tuned index search** (`BlockSearchType::kAuto` +
  `uniform_cv_threshold`): per-block uniformity metadata enables
  interpolation-style search over binary search when key spacing is uniform.

---

## Appendix A: File-format compatibility rules of thumb

- SST `format_version` and the `ValueType` enum are append-only contracts;
  readers must handle all historical encodings. New block features gate on
  format_version, new record semantics gate on new ValueTypes.
- WAL/MANIFEST record types with the high bit set are ignorable by old
  readers (forward compatibility); others are not -- adding a required record
  type is a downgrade barrier.
- Options are stored in the OPTIONS-<n> file via the `Configurable` string
  serialization; unknown options are tolerated on open (`ignore_unknown_options`)
  to permit binary downgrades.

## Appendix B: Where to start reading code, in order

1. `db/dbformat.h` -- the data model in 400 lines.
2. `db/db_impl/db_impl_write.cc` + `db/write_thread.h` -- the write path.
3. `db/column_family.h` -- CFD/SuperVersion relationships (the ASCII diagram
   at the top is the best single picture of object ownership in the engine).
4. `db/version_set.h` -- Version/VersionSet/LogAndApply.
5. `db/compaction/compaction_iterator.cc` -- where LSM semantics actually live.
6. `table/block_based/block_based_table_reader.cc` -- the read hot path.
7. `docs/components/read_flow/` and `docs/components/write_flow/` -- in-tree
   deep dives that complement this document.
