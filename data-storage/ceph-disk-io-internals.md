# How Ceph Reads and Writes Data on Disk

A source-level walkthrough of the Ceph I/O path, from a client `write()` to the
`pwritev()`/`io_submit()` syscalls that hit a block device, and back again for
reads. Everything below is taken from the checked-out tree at
`C:\opensource\cpeh` (commit `9438aabf3f7`, release codename *tentacle*,
2025-12-30). Line numbers refer to that commit.

The document assumes you already know what RADOS, OSDs, PGs and CRUSH are. It
focuses on *mechanism*: what bytes land where, in what order, and what makes
the result durable.

---

## 1. The layer cake

Ceph does not talk to disks from the client. All disk I/O happens inside an OSD
daemon, and the OSD delegates it to an `ObjectStore` implementation. In this
tree the only production `ObjectStore` is **BlueStore**, which owns a raw block
device and manages it itself (no local filesystem in the data path).

```
 client (librados / librbd / CephFS / RGW)
   │  Objecter: object → PG → [OSD list] via CRUSH           src/osdc/Objecter.cc
   ▼  MOSDOp over the messenger
 OSD daemon (primary for that PG)                            src/osd/OSD.cc
   │  ms_fast_dispatch → enqueue_op → dequeue_op → PG::do_request
   ▼
 PrimaryLogPG                                                src/osd/PrimaryLogPG.cc
   │  do_op → execute_ctx → do_osd_ops   (builds a PGTransaction)
   │  issue_repop
   ▼
 PGBackend (ReplicatedBackend | ECBackend)                   src/osd/ReplicatedBackend.cc
   │  generate_transaction → ObjectStore::Transaction
   │  send MOSDRepOp to replicas, queue_transactions locally
   ▼
 ObjectStore = BlueStore                                     src/os/bluestore/BlueStore.cc
   │  queue_transactions → TransContext state machine
   │      data  ──► bdev->aio_write()      (direct to raw block device)
   │      meta  ──► RocksDB (onodes, extent maps, freelist, deferred log)
   │                   │
   │                   ▼
   │               BlueFS  (tiny purpose-built FS hosting RocksDB files)   src/os/bluestore/BlueFS.cc
   │                   │  file extents on block.wal / block.db / block
   ▼                   ▼
 BlockDevice = KernelDevice                                  src/blk/kernel/KernelDevice.cc
   │  O_DIRECT fd + libaio (io_submit / io_getevents) or io_uring
   │  fdatasync() for barriers, BLKDISCARD for trims
   ▼
 /dev/… (or a file) ── block, block.db, block.wal
```

Three ideas govern everything below:

1. **Data and metadata take different roads.** Object payload goes straight to
   the raw device via async direct I/O. Metadata (where the payload lives,
   checksums, xattrs, omap, free-space bitmap) goes into RocksDB, which lives
   on BlueFS, which itself writes to the raw device(s).
2. **Nothing is visible until the RocksDB commit lands.** BlueStore never
   overwrites live data in place at sub-allocation-unit granularity without
   journaling. A write either allocates fresh space (write data, then commit
   the new extent map) or, for small overwrites, journals the payload inside
   the RocksDB transaction (a *deferred* write) and applies it to the device
   later.
3. **The commit point is a single synchronous RocksDB write per batch**, issued
   by one thread (`_kv_sync_thread`), preceded by an `fdatasync()` of the data
   device so that any data the metadata points at is stable first.

---

## 2. Finding the disk: object → PG → OSD

Before any disk is touched, the client has to know which OSD is primary for
the object. That is a pure computation on the client using the OSDMap.

`src/osd/OSDMap.cc:2656`

```cpp
int OSDMap::object_locator_to_pg(
  const object_t& oid, const object_locator_t& loc, pg_t &pg) const
{
  if (loc.hash >= 0) {
    if (!get_pg_pool(loc.get_pool())) {
      return -ENOENT;
    }
    pg = pg_t(loc.hash, loc.get_pool());
    return 0;
  }
  return map_to_pg(loc.get_pool(), oid.name, loc.key, loc.nspace, &pg);
}
```

`map_to_pg` hashes the object name (or locator key) with `ceph_str_hash_rjenkins`
and stores the raw hash in `pg_t`. The PG is then run through CRUSH:

`src/osd/OSDMap.cc:2705`

```cpp
void OSDMap::_pg_to_raw_osds(
  const pg_pool_t& pool, pg_t pg,
  vector<int> *osds,
  ps_t *ppps) const
{
  // map to osds[]
  ps_t pps = pool.raw_pg_to_pps(pg);  // placement ps
  unsigned size = pool.get_size();

  // what crush rule?
  int ruleno = pool.get_crush_rule();
  if (ruleno >= 0)
    crush->do_rule(ruleno, pps, *osds, size, osd_weight, pg.pool());

  _remove_nonexistent_osds(pool, *osds);
  ...
}
```

The Objecter caches this mapping per PG and refreshes it on map changes
(`Objecter::_calc_target`, `src/osdc/Objecter.cc:2960-3090`):

```cpp
  pg_t pgid;
  if (t->precalc_pgid) {
    ...
    pgid = t->base_pgid;
  } else {
    int ret = osdmap->object_locator_to_pg(t->target_oid, t->target_oloc, pgid);
    ...
  }
  ...
  ps_t actual_ps = ceph_stable_mod(pgid.ps(), pg_num, pg_num_mask);
  pg_t actual_pgid(actual_ps, pgid.pool());
  if (!lookup_pg_mapping(actual_pgid, osdmap->get_epoch(), &up, &up_primary,
                         &acting, &acting_primary)) {
    osdmap->pg_to_up_acting_osds(actual_pgid, &up, &up_primary,
                                 &acting, &acting_primary);
    ...
  }
```

The `MOSDOp` message is sent to `acting_primary`. Reads go to the primary too
unless the client set `CEPH_OSD_FLAG_BALANCE_READS` / `LOCALIZE_READS`.

---

## 3. The OSD side: turning a client op into a storage transaction

### 3.1 Dispatch

`OSD::ms_fast_dispatch` (`src/osd/OSD.cc:7654`) puts the op on a sharded work
queue keyed by PG. A worker thread pulls it out in `OSD::dequeue_op`
(`src/osd/OSD.cc:9926`) and hands it to the PG:

```cpp
void OSD::dequeue_op(PGRef pg, OpRequestRef op, ThreadPool::TPHandle &handle)
{
  ...
  op->mark_reached_pg();
  op->osd_trace.event("dequeue_op");

  pg->do_request(op, handle);
  ...
}
```

Everything after this runs under the PG lock, which is why one PG's ops are
serialized and why BlueStore can rely on a per-collection sequencer (see §6.3).

### 3.2 Building the transaction

`PrimaryLogPG::do_op` → `execute_ctx` (`src/osd/PrimaryLogPG.cc:4209`) →
`do_osd_ops` (`:6048`). Each `CEPH_OSD_OP_*` in the message is applied to an
in-memory `PGTransaction`. The write case (`:6758`) validates the op against
the object's `object_info_t` (size, truncate sequence, alignment for EC pools)
and then records a `write` into `ctx->op_t`. No disk I/O has happened yet.

`execute_ctx` then either replies immediately for pure reads, or builds a
`RepGather` and calls `issue_repop` (`src/osd/PrimaryLogPG.cc:11516`):

```cpp
void PrimaryLogPG::issue_repop(RepGather *repop, OpContext *ctx)
{
  ...
  Context *on_all_commit = new C_OSD_RepopCommit(this, repop);
  ...
  recovery_state.pre_submit_op(soid, ctx->log, ctx->at_version);
  pgbackend->submit_transaction(
    soid,
    ctx->delta_stats,
    ctx->at_version,
    std::move(ctx->op_t),
    recovery_state.get_pg_trim_to(),
    recovery_state.get_pg_committed_to(),
    std::move(ctx->log),
    ctx->updated_hset_history,
    on_all_commit,
    repop->rep_tid,
    ctx->reqid,
    ctx->op);
}
```

`on_all_commit` fires only when *every* acting shard has committed to disk;
that is what ultimately sends the client reply flagged
`CEPH_OSD_FLAG_ACK | CEPH_OSD_FLAG_ONDISK` (`:4300-4400`).

### 3.3 Replication: one ObjectStore::Transaction, shipped to every replica

`ReplicatedBackend::submit_transaction` (`src/osd/ReplicatedBackend.cc:580`)
converts the logical `PGTransaction` into a concrete
`ObjectStore::Transaction`, sends it to replicas, and queues it locally:

```cpp
  vector<pg_log_entry_t> log_entries(_log_entries);
  ObjectStore::Transaction op_t;
  PGTransactionUPtr t(std::move(_t));
  set<hobject_t> added, removed;
  generate_transaction(t, coll, log_entries, &op_t, &added, &removed,
                       get_osdmap()->require_osd_release);
  ...
  op.waiting_for_commit.insert(
    parent->get_acting_recovery_backfill_shards().begin(),
    parent->get_acting_recovery_backfill_shards().end());

  issue_op(soid, at_version, tid, reqid, trim_to, pg_committed_to,
           added.size() ? *(added.begin()) : hobject_t(),
           removed.size() ? *(removed.begin()) : hobject_t(),
           log_entries, hset_history, &op, op_t);          // → MOSDRepOp to peers

  ...
  parent->log_operation(std::move(log_entries), hset_history, trim_to,
                        at_version, pg_committed_to, true, op_t);   // PG log into same txn

  op_t.register_on_commit(
    parent->bless_context(new C_OSD_OnOpCommit(this, &op)));

  vector<ObjectStore::Transaction> tls;
  tls.push_back(std::move(op_t));

  parent->queue_transactions(tls, op.op);                  // → BlueStore
```

Two things are worth noting for a storage engineer:

- **The PG log entry and the object mutation are in the same transaction.**
  `log_operation` appends `pg_log_entry_t` records as omap keys on the PG's
  meta object into `op_t`. BlueStore commits both atomically, which is what
  makes PG log replay after a crash consistent.
- **`parent->queue_transactions` is a one-liner into the store**
  (`src/osd/PrimaryLogPG.h:383`): `osd->store->queue_transactions(ch, tls, op, NULL)`.

On a replica, `ReplicatedBackend::do_repop` (`src/osd/ReplicatedBackend.cc:1255`)
decodes the shipped `ObjectStore::Transaction` from the `MOSDRepOp` payload,
adds its own PG-log transaction and queues both:

```cpp
  rm->opt.decode(m->get_middle().length() != 0 ?  p : d, d);
  ...
  rm->opt.set_fadvise_flag(CEPH_OSD_OP_FLAG_FADVISE_DONTNEED);
  ...
  rm->opt.register_on_commit(
    parent->bless_context(new C_OSD_RepModifyCommit(this, rm)));
  vector<ObjectStore::Transaction> tls;
  tls.reserve(2);
  tls.push_back(std::move(rm->localt));
  tls.push_back(std::move(rm->opt));
  parent->queue_transactions(tls, op);
```

The replica sets `FADVISE_DONTNEED` because replica copies are rarely read; it
keeps BlueStore's buffer cache from caching them (see `_choose_write_options`
in §6.5).

Erasure-coded pools take a different route (`ECBackend::submit_transaction`,
`src/osd/ECBackend.cc:949`): a `WritePlan` is computed, a read-modify-write
pipeline (`rmw_pipeline.start_rmw`) reads any partial stripes, encodes
parity, and each shard receives its own sub-transaction via `handle_sub_write`
(`:385`), which again ends in `queue_transactions`. From BlueStore's point of
view every shard write is an ordinary object write.

### 3.4 What an ObjectStore::Transaction looks like on the wire

`ObjectStore::Transaction` (`src/os/Transaction.h`) is a compact op-code
stream plus side buffers. Each op is a packed record:

`src/os/Transaction.h:163`

```cpp
  struct Op {
    ceph_le32 op;
    ceph_le32 cid;
    ceph_le32 oid;
    ceph_le64 off;
    ceph_le64 len;
    ceph_le32 dest_cid;
    ceph_le32 dest_oid;               //OP_CLONE, OP_CLONERANGE
    ceph_le64 dest_off;               //OP_CLONERANGE
    ceph_le32 hint;                   //OP_COLL_HINT,OP_SETALLOCHINT
    ceph_le64 expected_object_size;   //OP_SETALLOCHINT
    ceph_le64 expected_write_size;    //OP_SETALLOCHINT
    ceph_le32 split_bits;
    ceph_le32 split_rem;
  } __attribute__ ((packed)) ;
```

`write()` (`:870`) deliberately splits the payload into a page-aligned middle
and misaligned head/tail so that, after decode, the aligned part can be handed
to `O_DIRECT` I/O without a memcpy:

```cpp
  void write(const coll_t& cid, const ghobject_t& oid, uint64_t off, uint64_t len,
	       const ceph::buffer::list& write_data, uint32_t flags = 0) {
    Op* _op = _get_next_op();
    uint64_t alignstart = (0 - off) & ~CEPH_PAGE_MASK;
    _op->op = OP_WRITE;
    _op->cid = _get_coll_id(cid);
    _op->oid = _get_object_id(oid);
    _op->off = off;
    _op->len = len;
    ...
    if (len >= CEPH_PAGE_SIZE + alignstart) {
      uint64_t alignlen = (len - alignstart) & CEPH_PAGE_MASK;
      uint64_t suffixstart = alignstart + alignlen;
      if (alignstart != 0) {
        bufferlist prefix;
        prefix.substr_of(write_data, 0, alignstart);
        encode_nohead(prefix, data_misaligned_bl);
      }
      bufferlist aligned;
      aligned.substr_of(write_data, alignstart, alignlen);
      encode_nohead(aligned, data_aligned_bl);
      ...
```

---

## 4. BlueStore's on-disk model

### 4.1 Devices and the first 8 KiB

An OSD directory contains up to three block devices (symlinks or files):

| Path        | BlueFS role  | Contents                                                        |
|-------------|--------------|-----------------------------------------------------------------|
| `block`     | `BDEV_SLOW`  | Object data. Also hosts RocksDB SSTs/WAL if no db/wal device.   |
| `block.db`  | `BDEV_DB`    | Optional. RocksDB SSTs (and WAL if no `block.wal`).             |
| `block.wal` | `BDEV_WAL`   | Optional. RocksDB WAL and BlueFS journal only.                  |

`BlueStore::_minimal_open_bluefs` (`src/os/bluestore/BlueStore.cc:7628`) wires
them up; the resulting topology is persisted as `bluefs_layout_t`
(`src/os/bluestore/bluefs_types.h:294`):

```cpp
struct bluefs_layout_t {
  unsigned shared_bdev = 0;         ///< which bluefs bdev we are sharing
  bool dedicated_db = false;        ///< whether block.db is present
  bool dedicated_wal = false;       ///< whether block.wal is present

  bool single_shared_device() const {
    return !dedicated_db && !dedicated_wal;
  }
  ...
};
```

The first two 4 KiB blocks of each device are reserved
(`src/os/bluestore/bluestore_common.h:70-77`):

```cpp
static constexpr uint64_t BDEV_FIRST_LABEL_POSITION = 0;
static constexpr uint64_t BDEV_LABEL_BLOCK_SIZE = 4096;
...
static constexpr uint64_t SUPER_RESERVED = BDEV_LABEL_BLOCK_SIZE + BLUEFS_SUPER_BLOCK_SIZE;
```

- Offset 0: the **bdev label** (`bluestore_bdev_label_t`,
  `bluestore_types.h:38`): OSD uuid, device size, birth time, description and a
  `meta` map (the same key/values `read_meta()` exposes: fsid, whoami, etc.).
  Recent releases write redundant copies of this label at additional fixed
  positions on the main device so a damaged block 0 is recoverable.
- Offset 4096: the **BlueFS superblock** (`BlueFS::get_super_offset()` returns
  4096, `BlueFS.h:769`), CRC-protected, containing `log_fnode`: the extent list
  of the BlueFS journal. This is the root from which all RocksDB files are
  located.

Everything else on the device is a flat pool of allocation units managed by
the BlueStore allocator (for object data) and, on the shared device, also by
BlueFS (for RocksDB files).

### 4.2 RocksDB key namespaces

All BlueStore metadata is key/value data in RocksDB, partitioned by a
single-character prefix (`src/os/bluestore/BlueStore.cc:131`):

```cpp
const string PREFIX_SUPER = "S";       // field -> value
const string PREFIX_STAT = "T";        // field -> value(int64 array)
const string PREFIX_COLL = "C";        // collection name -> cnode_t
const string PREFIX_OBJ = "O";         // object name -> onode_t
const string PREFIX_OMAP = "M";        // u64 + keyname -> value
const string PREFIX_PGMETA_OMAP = "P"; // u64 + keyname -> value(for meta coll)
const string PREFIX_PERPOOL_OMAP = "m"; // s64 + u64 + keyname -> value
const string PREFIX_PERPG_OMAP = "p";   // u64(pool) + u32(hash) + u64(id) + keyname -> value
const string PREFIX_DEFERRED = "L";    // id -> deferred_transaction_t
const string PREFIX_ALLOC = "B";       // u64 offset -> u64 length (freelist)
const string PREFIX_ALLOC_BITMAP = "b";// (see BitmapFreelistManager)
const string PREFIX_SHARED_BLOB = "X"; // u64 SB id -> shared_blob_t
```

Object keys under `O` are built so that lexicographic order equals
`ghobject_t` order, which is what lets collection listing and PG splitting be
range scans (`src/os/bluestore/BlueStore.cc:449`):

```cpp
static void _get_object_key(const ghobject_t& oid, S *key)
{
  ...
  _key_encode_prefix(oid, key);          // shard, pool, reversed hash
  append_escaped(oid.hobj.nspace, key);
  if (oid.hobj.get_key().length()) {
    append_escaped(oid.hobj.get_key(), key);
    int r = oid.hobj.get_key().compare(oid.hobj.oid.name);
    if (r) {
      key->append(r > 0 ? ">" : "<");
      append_escaped(oid.hobj.oid.name, key);
    } else {
      key->append("=");
    }
  } else {
    append_escaped(oid.hobj.oid.name, key);
    key->append("=");
  }
  _key_encode_u64(oid.hobj.snap, key);
  _key_encode_u64(oid.generation, key);
  key->push_back(ONODE_KEY_SUFFIX);
}
```

Extent-map shards for large objects are stored under the *same* prefix, keyed
by onode key + u32 offset + `'x'` (`:511`), so an onode and its shards are
adjacent in the SST and a single iterator prefetches them.

### 4.3 The in-memory / on-disk object model

```
 Onode (one per object; key "O…")          bluestore_onode_t: nid, size, attrs, extent_map_shards[]
   └─ ExtentMap                            logical offset → Extent   (inline in onode or in 'x' shards)
        └─ Extent {logical_offset, blob_offset, length, BlobRef}
             └─ Blob                       bluestore_blob_t: PExtentVector, csum, flags, unused bitmap
                  └─ bluestore_pextent_t {offset, length}  ← physical bytes on `block`
                  └─ (optional) SharedBlob  key "X…", refcounted extents for clones
```

`bluestore_blob_t` (`src/os/bluestore/bluestore_types.h:491`) is the unit that
maps to physical disk:

```cpp
struct bluestore_blob_t {
private:
  PExtentVector extents;              ///< raw data position on device
  uint32_t logical_length = 0;        ///< original length of data stored in the blob
  uint32_t compressed_length = 0;     ///< compressed length if any
public:
  enum {
    LEGACY_FLAG_MUTABLE = 1,
    FLAG_COMPRESSED = 2,      ///< blob is compressed
    FLAG_CSUM = 4,            ///< blob has checksums
    FLAG_HAS_UNUSED = 8,      ///< blob has unused std::map
    FLAG_SHARED = 16,         ///< blob is shared; see external SharedBlob
  };
  uint32_t flags = 0;
  typedef uint16_t unused_t;
  unused_t unused = 0;     ///< portion that has never been written to (bitmap)
  uint8_t csum_type = Checksummer::CSUM_NONE;
  uint8_t csum_chunk_order = 0;       ///< csum block size is 1<<block_order bytes
  ceph::buffer::ptr csum_data;        ///< opaque std::vector of csum data
  ...
```

Key sizes (`src/common/options/global.yaml.in`):

| Option                              | Default (hdd / ssd) | Meaning                                                            |
|-------------------------------------|---------------------|--------------------------------------------------------------------|
| `bluestore_min_alloc_size_{hdd,ssd}`| 4 KiB / 4 KiB       | Allocation unit ("AU"). Smallest thing the allocator hands out.    |
| `bluestore_max_blob_size_{hdd,ssd}` | 64 KiB / 64 KiB     | Max blob; big writes are chopped into blobs of this size.          |
| `bluestore_prefer_deferred_size_{hdd,ssd}` | 64 KiB / 0   | Writes below this are journaled in RocksDB and applied later.      |
| `bluestore_csum_type`               | crc32c              | Per-blob checksum, chunked at `block_size` (4 KiB) by default.     |
| `bluestore_compression_mode`        | none                | none / passive / aggressive / force.                               |
| `bluestore_write_v2`                | false               | Selects the newer `Writer`-class write path (§6.6).                |

Two invariants fall out of this layout:

- **Disk is never written at finer than `block_size` (4 KiB) granularity**, and
  allocation is never finer than `min_alloc_size`. Byte-granular RADOS writes
  are padded/RMW'd (§6.6).
- **Checksums cover physical chunks, not logical ranges.** A read always
  fetches whole csum chunks and verifies before returning (§7).

---

## 5. Opening the device: KernelDevice

`BlueStore::_open_bdev` (`src/os/bluestore/BlueStore.cc:7136`) creates a
`BlockDevice` for `path + "/block"`, registers two callbacks (aio completion
and discard completion), and reads the label:

```cpp
int BlueStore::_open_bdev(bool create)
{
  string p = path + "/block";
  bdev = BlockDevice::create(cct, p, aio_cb, static_cast<void*>(this),
                             discard_cb, static_cast<void*>(this), "bluestore");
  int r = bdev->open(p);
  ...
  block_size = bdev->get_block_size();
  block_mask = ~(block_size - 1);
  block_size_order = std::countr_zero(block_size);
  ...
}
```

`KernelDevice::open` (`src/blk/kernel/KernelDevice.cc:161`) opens **two file
descriptors per write-lifetime hint**: one `O_DIRECT` for data, one buffered
for the few places that want page cache (BlueFS reads with
`bluefs_buffered_io`, label writes):

```cpp
  for (i = 0; i < WRITE_LIFE_MAX; i++) {
    int flags = 0;
    if (lock_exclusive && is_block && (i == 0)) {
      // If opening block device use O_EXCL flag. It gives us best protection,
      // as no other process can overwrite the data for as long as we are running.
      flags |= O_EXCL;
    }
    int fd = ::open(path.c_str(), O_RDWR | O_DIRECT | flags);
    ...
    fd_directs[i] = fd;

    fd  = ::open(path.c_str(), O_RDWR | O_CLOEXEC);
    ...
    fd_buffereds[i] = fd;
  }

#if defined(F_SET_FILE_RW_HINT)
  for (i = WRITE_LIFE_NONE; i < WRITE_LIFE_MAX; i++) {
    if (fcntl(fd_directs[i], F_SET_FILE_RW_HINT, &i) < 0) { ... }
    if (fcntl(fd_buffereds[i], F_SET_FILE_RW_HINT, &i) < 0) { ... }
  }
#endif

  dio = true;
  aio = cct->_conf->bdev_aio;
  if (!aio) {
    ceph_abort_msg("non-aio not supported");
  }

  // disable readahead as it will wreak havoc on our mix of
  // directio/aio and buffered io.
  r = posix_fadvise(fd_buffereds[WRITE_LIFE_NOT_SET], 0, 0, POSIX_FADV_RANDOM);
```

`choose_fd(buffered, write_hint)` (`:480`) picks the fd. The `WRITE_LIFE_*`
hints are passed down from RocksDB (WAL = short, SST levels = longer) so
multi-stream NVMe devices can separate them.

The async queue is libaio by default, io_uring if `bdev_ioring=true` and the
kernel supports it (`:86-100`):

```cpp
  bool use_ioring = cct->_conf.get_val<bool>("bdev_ioring");
  ...
  if (use_ioring && ioring_queue_t::supported()) {
    io_queue = std::make_unique<ioring_queue_t>(iodepth, use_ioring_hipri, use_ioring_sqthread_poll);
  } else {
    io_queue = std::make_unique<aio_queue_t>(iodepth);
  }
```

`iodepth` is `bdev_aio_max_queue_depth` (default 1024). A dedicated
`_aio_thread` per device reaps completions (§8.3).

---

## 6. The write path inside BlueStore

### 6.1 Entry point: `queue_transactions`

`src/os/bluestore/BlueStore.cc:15660`

```cpp
int BlueStore::queue_transactions(
  CollectionHandle& ch,
  vector<Transaction>& tls,
  TrackedOpRef op,
  ThreadPool::TPHandle *handle)
{
  list<Context *> on_applied, on_commit, on_applied_sync;
  ObjectStore::Transaction::collect_contexts(
    tls, &on_applied, &on_commit, &on_applied_sync);

  Collection *c = static_cast<Collection*>(ch.get());
  OpSequencer *osr = c->osr.get();

  // prepare
  TransContext *txc = _txc_create(static_cast<Collection*>(ch.get()), osr,
				  &on_commit, op);

  for (vector<Transaction>::iterator p = tls.begin(); p != tls.end(); ++p) {
    txc->bytes += (*p).get_num_bytes();
    _txc_add_transaction(txc, &(*p));          // decode ops, mutate in-memory onodes, queue aios
  }
  _txc_calc_cost(txc);

  _txc_write_nodes(txc, txc->t);               // encode dirty onodes/shared blobs into the kv txn

  // journal deferred items
  if (txc->deferred_txn) {
    txc->deferred_txn->seq = ++deferred_seq;
    bufferlist bl;
    encode(*txc->deferred_txn, bl);
    string key;
    get_deferred_key(txc->deferred_txn->seq, &key);
    txc->t->set(PREFIX_DEFERRED, key, bl);     // payload of small writes goes INTO RocksDB
  }

  _txc_finalize_kv(txc, txc->t);               // freelist allocate/release + statfs

  ...
  if (!throttle.try_start_transaction(*db, *txc, tstart)) {
    // ensure we do not block here because of deferred writes
    ++deferred_aggressive;
    deferred_try_submit();
    ...
    throttle.finish_start_transaction(*db, *txc, tstart);
    --deferred_aggressive;
  }
  ...
  // execute (start)
  _txc_state_proc(txc);

  // we're immediately readable (unlike FileStore)
  for (auto c : on_applied_sync) {
    c->complete(0);
  }
  ...
  return 0;
}
```

Observations:

- A `TransContext` (txc) carries **one RocksDB `WriteBatch`** (`txc->t`), **one
  `IOContext`** (`txc->ioc`) holding the pending aios for the data device, and
  optionally one `bluestore_deferred_transaction_t`.
- The "applied" callbacks fire immediately after `_txc_add_transaction`
  because the in-memory onode cache is already updated; readers that come
  through the same OSD see the new data before it is durable. Only
  `on_commit` waits for the kv commit.
- The throttle (`bluestore_throttle_bytes`, `..._deferred_bytes`) bounds the
  amount of un-committed data in flight; when it is exceeded, deferred
  batches are flushed aggressively to free it.

### 6.2 The TransContext state machine

`src/os/bluestore/BlueStore.h:1836`

```cpp
    typedef enum {
      STATE_PREPARE,
      STATE_AIO_WAIT,
      STATE_IO_DONE,
      STATE_KV_QUEUED,     // queued for kv_sync_thread submission
      STATE_KV_SUBMITTED,  // submitted to kv; not yet synced
      STATE_KV_DONE,
      STATE_DEFERRED_QUEUED,    // in deferred_queue (pending or running)
      STATE_DEFERRED_CLEANUP,   // remove deferred kv record
      STATE_DEFERRED_DONE,
      STATE_FINISHING,
      STATE_DONE,
    } state_t;
```

`_txc_state_proc` (`BlueStore.cc:14317`) drives it. The important transitions:

```cpp
    case TransContext::STATE_PREPARE:
      if (txc->ioc.has_pending_aios()) {
	txc->set_state(TransContext::STATE_AIO_WAIT);
	txc->had_ios = true;
	_txc_aio_submit(txc);                 // bdev->aio_submit(&txc->ioc)
	return;                               // resumes from the aio completion callback
      }
      // ** fall-thru **

    case TransContext::STATE_AIO_WAIT:
      ...
      _txc_finish_io(txc);  // may trigger blocked txc's too
      return;

    case TransContext::STATE_IO_DONE:
      ceph_assert(ceph_mutex_is_locked(txc->osr->qlock));  // see _txc_finish_io
      if (txc->had_ios) {
	++txc->osr->txc_with_unstable_io;
      }
      txc->set_state(TransContext::STATE_KV_QUEUED);
      if (cct->_conf->bluestore_sync_submit_transaction) {
	...
	} else {
	  _txc_apply_kv(txc, true);           // db->submit_transaction (async, no sync)
	}
      }
      {
	std::lock_guard l(kv_lock);
	kv_queue.push_back(txc);
	if (!kv_sync_in_progress) {
	  kv_sync_in_progress = true;
	  kv_cond.notify_one();               // wake _kv_sync_thread
	}
	if (txc->get_state() != TransContext::STATE_KV_SUBMITTED) {
	  kv_queue_unsubmitted.push_back(txc);
	  ++txc->osr->kv_committing_serially;
	}
	if (txc->had_ios)
	  kv_ios++;
	...
      }
      return;
    case TransContext::STATE_KV_SUBMITTED:
      _txc_committed_kv(txc);               // fires on_commit contexts → OSD replies to client
      // ** fall-thru **

    case TransContext::STATE_KV_DONE:
      if (txc->deferred_txn) {
	txc->set_state(TransContext::STATE_DEFERRED_QUEUED);
	_deferred_queue(txc);
	return;
      }
      txc->set_state(TransContext::STATE_FINISHING);
      break;

    case TransContext::STATE_DEFERRED_CLEANUP:
      txc->set_state(TransContext::STATE_FINISHING);
      // ** fall-thru **

    case TransContext::STATE_FINISHING:
      _txc_finish(txc);                     // release old extents to allocator, delete txc
      return;
```

In words:

1. **PREPARE**: data aios for freshly allocated space are queued into
   `txc->ioc`. If there are any, submit them and wait (AIO_WAIT).
2. **IO_DONE**: data is on the device (not necessarily stable). Submit the
   RocksDB batch asynchronously (`WriteOptions.sync=false`) and enqueue for
   the kv sync thread.
3. **KV_SUBMITTED → KV_DONE**: the kv sync thread has flushed the device and
   done a synchronous RocksDB write. The transaction is now durable and the
   OSD gets its commit callback.
4. **DEFERRED_QUEUED**: if the txc journaled small writes, they are now
   replayed to their final location asynchronously. The client already has
   its ack.
5. **FINISHING/DONE**: old extents freed, discards issued, txc deleted.

### 6.3 Ordering: the OpSequencer

Aios complete in any order, but RocksDB batches from one collection (PG) must
be submitted in order, otherwise a later txc's onode could be overwritten by
an earlier one's stale encoding. `_txc_finish_io` (`:14435`) enforces this:

```cpp
void BlueStore::_txc_finish_io(TransContext *txc)
{
  /*
   * we need to preserve the order of kv transactions,
   * even though aio will complete in any order.
   */
  OpSequencer *osr = txc->osr.get();
  std::lock_guard l(osr->qlock);
  txc->set_state(TransContext::STATE_IO_DONE);
  txc->ioc.release_running_aios();
  OpSequencer::q_list_t::iterator p = osr->q.iterator_to(*txc);
  while (p != osr->q.begin()) {
    --p;
    if (p->get_state() < TransContext::STATE_IO_DONE) {
      // blocked by an earlier txc whose aios are still in flight
      return;
    }
    if (p->get_state() > TransContext::STATE_IO_DONE) {
      ++p;
      break;
    }
  }
  do {
    _txc_state_proc(&*p++);
  } while (p != osr->q.end() &&
	   p->get_state() == TransContext::STATE_IO_DONE);
  ...
}
```

Each `Collection` has one `OpSequencer` (`BlueStore.h:2157`) holding an
intrusive list of its txcs in submission order. This is the only ordering
BlueStore guarantees, and it matches what the OSD needs since PG ops are
already serialized under the PG lock.

### 6.4 Decoding ops and dirtying metadata

`_txc_add_transaction` (`:15777`) walks the op stream. For `OP_WRITE` it
resolves the collection and onode (through the onode cache, faulting in the
`O…` key from RocksDB if needed) and calls `_write` → `_do_write` or
`_do_write_v2`. After all ops, `_txc_write_nodes` (`:14471`) serializes every
dirtied onode and shared blob into the batch:

```cpp
void BlueStore::_txc_write_nodes(TransContext *txc, KeyValueDB::Transaction t)
{
  // finalize onodes
  for (auto o : txc->onodes) {
    _record_onode(o, t);
    ...
    o->flushing_count++;
  }
  ...
  // finalize shared_blobs
  for (auto sb : txc->shared_blobs) {
    string key;
    auto sbid = sb->get_sbid();
    get_shared_blob_key(sbid, &key);
    if (sb->persistent->empty()) {
      t->rmkey(PREFIX_SHARED_BLOB, key);
    } else {
      bufferlist bl;
      encode(*(sb->persistent), bl);
      t->set(PREFIX_SHARED_BLOB, key, bl);
    }
  }
}
```

`_record_onode` (`:19212`) first updates/reshards the extent map (each shard
becomes its own `…x` key), then encodes `bluestore_onode_t` + spanning blobs +
the inline extent map into one value:

```cpp
void BlueStore::_record_onode(OnodeRef& o, KeyValueDB::Transaction &txn)
{
  // finalize extent_map shards
  o->extent_map.update(txn, false);
  if (o->extent_map.needs_reshard()) {
    o->extent_map.reshard(db, txn, o->onode.segment_size);
    o->extent_map.update(txn, true);
    ...
  }

  // bound encode
  size_t bound = 0;
  denc(o->onode, bound, flag);
  o->extent_map.bound_encode_spanning_blobs(bound);
  if (o->onode.extent_map_shards.empty()) {
    denc(o->extent_map.inline_bl, bound);
  }

  // encode
  bufferlist bl;
  {
    auto p = bl.get_contiguous_appender(bound, true);
    denc(o->onode, p, flag);
    o->extent_map.encode_spanning_blobs(p);
    if (o->onode.extent_map_shards.empty()) {
      denc(o->extent_map.inline_bl, p);
    }
  }
  ...
```

Shard sizing is tuned by `bluestore_extent_map_shard_target_size` (500 B) and
`..._max_size` (1200 B): the goal is that a random 4 KiB write to a large RBD
image only rewrites a few hundred bytes of metadata, not the whole extent map.

Finally `_txc_finalize_kv` (`:14534`) records allocation changes in the
freelist (§9) and updates per-pool statfs counters, all in the same batch.

### 6.5 Choosing how to write: `_choose_write_options`

`src/os/bluestore/BlueStore.cc:17374`

```cpp
void BlueStore::_choose_write_options(CollectionRef& c, OnodeRef& o,
   uint32_t fadvise_flags, WriteContext *wctx)
{
  if (fadvise_flags & CEPH_OSD_OP_FLAG_FADVISE_WILLNEED) {
    wctx->buffered = true;
  } else if (cct->_conf->bluestore_default_buffered_write &&
	     (fadvise_flags & (CEPH_OSD_OP_FLAG_FADVISE_DONTNEED |
			       CEPH_OSD_OP_FLAG_FADVISE_NOCACHE)) == 0) {
    wctx->buffered = true;
  }

  // apply basic csum block size
  wctx->csum_order = block_size_order;

  // checksum
  wctx->csum_type= c->csum_type.has_value() ? *(c->csum_type) : csum_type.load();

  // compression parameters
  unsigned alloc_hints = o->onode.alloc_hint_flags;
  auto cm = c->compression_mode.has_value() ? *(c->compression_mode) : comp_mode.load();
  wctx->compress = (cm != Compressor::COMP_NONE) &&
    ((cm == Compressor::COMP_FORCE) ||
     (cm == Compressor::COMP_AGGRESSIVE &&
      (alloc_hints & CEPH_OSD_ALLOC_HINT_FLAG_INCOMPRESSIBLE) == 0) ||
     (cm == Compressor::COMP_PASSIVE &&
      (alloc_hints & CEPH_OSD_ALLOC_HINT_FLAG_COMPRESSIBLE)));

  if ((alloc_hints & CEPH_OSD_ALLOC_HINT_FLAG_SEQUENTIAL_READ) &&
      (alloc_hints & CEPH_OSD_ALLOC_HINT_FLAG_RANDOM_READ) == 0 &&
      (alloc_hints & (CEPH_OSD_ALLOC_HINT_FLAG_IMMUTABLE |
                      CEPH_OSD_ALLOC_HINT_FLAG_APPEND_ONLY)) &&
      (alloc_hints & CEPH_OSD_ALLOC_HINT_FLAG_RANDOM_WRITE) == 0) {
    // will prefer large blob and csum sizes
    ...
  }
  ...
}
```

Pool properties (`compression_*`, `csum_type`) and per-object alloc hints
(set by RGW/CephFS via `CEPH_OSD_OP_SETALLOCHINT`) select checksum chunking,
target blob size and compression. `buffered` decides whether the written
bytes are kept in BlueStore's own buffer cache (they are never in the page
cache, since the fd is `O_DIRECT`).

### 6.6 Placing the bytes: allocate, deferred or direct

There are two write implementations in the tree, switched by
`bluestore_write_v2` (default `false` at this commit). Both end in the same
two primitives: `bdev->aio_write()` for direct writes and `_get_deferred_op()`
for journaled writes.

#### Legacy path: `_do_write` → `_do_write_data` → `_do_write_small` / `_do_write_big` → `_do_alloc_write`

`_do_write` (`:17523`):

```cpp
  WriteContext wctx;
  _choose_write_options(c, o, fadvise_flags, &wctx);
  o->extent_map.fault_range(db, offset, length);   // load the extent shards we touch
  _do_write_data(txc, c, o, offset, length, bl, &wctx);
  r = _do_alloc_write(txc, c, o, &wctx);
  ...
  // NB: _wctx_finish() will empty old_extents
  _wctx_finish(txc, c, o, &wctx);                  // release replaced extents into txc->released
  if (end > o->onode.size) {
    o->onode.size = end;
  }
  ...
  o->extent_map.compress_extent_map(dirty_start, dirty_end - dirty_start);
  o->extent_map.dirty_range(dirty_start, dirty_end - dirty_start);
```

`_do_write_data` splits the range at `min_alloc_size` boundaries: the aligned
middle goes to `_do_write_big`, unaligned head/tail to `_do_write_small`
(`:16238`). Small writes are where all the read-modify-write logic lives.
The relevant branch, writing into never-used blocks of an existing blob:

```cpp
	uint64_t head_pad, tail_pad;
	head_pad = p2phase(offset, chunk_size);
	tail_pad = p2nphase(end_offs, chunk_size);
	...
	uint64_t b_off = offset - head_pad - bstart;
	uint64_t b_len = length + head_pad + tail_pad;

        // direct write into unused blocks of an existing mutable blob?
        if ((b_off % chunk_size == 0 && b_len % chunk_size == 0) &&
            b->get_blob().get_ondisk_length() >= b_off + b_len &&
            b->get_blob().is_unused(b_off, b_len) &&
            b->get_blob().is_allocated(b_off, b_len)) {
          _apply_padding(head_pad, tail_pad, bl);
          ...
          if (b_len < prefer_deferred_size) {
              bluestore_deferred_op_t *op = _get_deferred_op(txc, bl.length());
              op->op = bluestore_deferred_op_t::OP_WRITE;
              b->get_blob().map(b_off, b_len, [&](uint64_t offset, uint64_t length) {
                  op->extents.emplace_back(bluestore_pextent_t(offset, length));
                  return 0;
                });
              op->data = bl;
          } else {
              b->get_blob().map_bl(b_off, bl, [&](uint64_t offset, bufferlist& t) {
                  bdev->aio_write(offset, t, &txc->ioc, wctx->buffered);
                });
          }
```

And when the target blocks already hold data, the classic RMW: read the
surrounding bytes with `_do_read`, splice, and journal the whole chunk as a
deferred write (`:16390-16450`, abbreviated):

```cpp
	    int r = _do_read(c.get(), o, offset - head_pad - head_read, head_read, head_bl, 0);
	    ...
	    int r = _do_read(c.get(), o, offset + length + tail_pad, tail_read, tail_bl, 0);
	    ...
          bluestore_deferred_op_t *op = _get_deferred_op(txc, bl.length());
          op->op = bluestore_deferred_op_t::OP_WRITE;
          ...
          op->data.claim_append(bl);
```

The read-then-journal step is what makes an in-place overwrite crash-safe:
the full chunk sits in RocksDB (WAL) before the device block is touched.

`_do_alloc_write` (`:16962`) is the allocation and I/O issue point for all
new blobs. Compression happens here, then a single allocator call for the
whole txc's need, then per-blob csum and either deferred or direct I/O:

```cpp
  PExtentVector prealloc;
  prealloc.reserve(2 * wctx->writes.size());
  int64_t prealloc_left = 0;
  prealloc_left = alloc->allocate(
    need, min_alloc_size, need,
    use_last_allocator_lookup_position ? -1 : 0,
    &prealloc);
  ...
  if (prealloc_left < 0 || prealloc_left < (int64_t)need) {
    ...
    return -ENOSPC;
  }

  for (auto& wi : wctx->writes) {
    bluestore_blob_t& dblob = wi.b->dirty_blob();
    ...
    PExtentVector extents;
    int64_t left = final_length;
    auto prefer_deferred_size_snapshot = prefer_deferred_size.load();
    while (left > 0) {
      ceph_assert(prealloc_left > 0);
      if (prealloc_pos->length <= left) {
	prealloc_left -= prealloc_pos->length;
	left -= prealloc_pos->length;
	txc->statfs_delta.allocated() += prealloc_pos->length;
	extents.push_back(*prealloc_pos);
	++prealloc_pos;
      } else {
	extents.emplace_back(prealloc_pos->offset, left);
	prealloc_pos->offset += left;
	prealloc_pos->length -= left;
	prealloc_left -= left;
	txc->statfs_delta.allocated() += left;
	left = 0;
	break;
      }
    }
    for (auto& p : extents) {
      txc->allocated.insert(p.offset, p.length);
    }
    dblob.allocated(p2align(b_off, min_alloc_size), final_length, extents);

    if (dblob.has_csum()) {
      dblob.calc_csum(b_off, *l);
    }
    ...
    Extent *le = o->extent_map.set_lextent(coll, wi.logical_offset,
                                           b_off + (wi.b_off0 - wi.b_off),
                                           wi.length0, wi.b, nullptr);
    wi.b->dirty_blob().mark_used(le->blob_offset, le->length);
    ...
    _buffer_cache_write(txc, o, wi.logical_offset, std::move(without_pad),
                        wctx->buffered ? 0 : Buffer::FLAG_NOCACHE);

    // queue io
    if (!g_conf()->bluestore_debug_omit_block_device_write) {
      if (data_size < prefer_deferred_size_snapshot) {
	bluestore_deferred_op_t *op = _get_deferred_op(txc, l->length());
	op->op = bluestore_deferred_op_t::OP_WRITE;
	int r = wi.b->get_blob().map(
	  b_off, l->length(),
	  [&](uint64_t offset, uint64_t length) {
	    op->extents.emplace_back(bluestore_pextent_t(offset, length));
	    return 0;
	  });
        op->data = *l;
      } else {
	wi.b->get_blob().map_bl(
	  b_off, *l,
	  [&](uint64_t offset, bufferlist& t) {
	    bdev->aio_write(offset, t, &txc->ioc, false);
	  });
	logger->inc(l_bluestore_write_new);
      }
    }
  }
```

So even a brand-new allocation can be deferred if it is small (on HDD, the
64 KiB default means most RBD 4K–32K writes are journaled: one sequential
RocksDB WAL append instead of a random seek, and the seek happens later in a
batch).

#### v2 path: `_do_write_v2` → `BlueStore::Writer`

`src/os/bluestore/Writer.cc` is a rewrite with the same on-disk result but a
simpler structure: punch a hole in the extent map, split the data into
blobs, decide deferred vs direct *once* for the whole write, then emit I/O.
The decision (`Writer.cc:1316`):

```cpp
void BlueStore::Writer::_defer_or_allocate(uint32_t need_size)
{
  // make a deferred decision
  uint32_t released_size = 0;
  for (const auto& r : released) {
    released_size += r.length;
  }
  uint32_t au_size = bstore->min_alloc_size;
  do_deferred = need_size <= released_size && released_size < bstore->prefer_deferred_size;
  ...
  if (do_deferred) {
    disk_allocs.it = released.begin();          // reuse the blocks we just freed, in place
    statfs_delta.allocated() += need_size;
    disk_allocs.pos = 0;
  } else {
    int64_t new_alloc_size = bstore->alloc->allocate(need_size, au_size, 0, 0, &allocated);
    ceph_assert(need_size == new_alloc_size);
    statfs_delta.allocated() += new_alloc_size;
    disk_allocs.it = allocated.begin();
    disk_allocs.pos = 0;
  }
}
```

Note the semantics: a deferred write in v2 *reuses the physical blocks that
the overwrite is releasing*. That is safe only because the payload is
journaled first and the old blocks are not handed back to the allocator until
the txc finishes (§6.9). Emission (`Writer.cc:764`):

```cpp
inline void BlueStore::Writer::_schedule_io(const PExtentVector& disk_extents, bufferlist data)
{
  if (test_write_divertor == nullptr) {
    if (do_deferred) {
      bluestore_deferred_op_t *op = bstore->_get_deferred_op(txc, data.length());
      op->op = bluestore_deferred_op_t::OP_WRITE;
      op->extents = disk_extents;
      op->data = data;
    } else {
      for (const auto& loc : disk_extents) {
        bufferlist data_chunk;
        data.splice(0, loc.length, &data_chunk);
        bstore->bdev->aio_write(loc.offset, data_chunk, &txc->ioc, false);
      }
    }
  }
  ...
}
```

Partial-block writes into a blob's *unused* bitmap can be direct even when
small, because there is no old data to corrupt (`_schedule_io_masked`,
`Writer.cc:713`; `_schedule_io` is at `:766`).

### 6.7 The commit point: `_kv_sync_thread`

One thread per BlueStore instance owns the durability barrier
(`BlueStore.cc:14970`). Per iteration it swaps out the queues, decides
whether the data device needs a flush, submits any not-yet-submitted batches,
then does one **synchronous** RocksDB write:

```cpp
      kv_committing.swap(kv_queue);
      kv_submitting.swap(kv_queue_unsubmitted);
      deferred_done.swap(deferred_done_queue);
      deferred_stable.swap(deferred_stable_queue);
      aios = kv_ios;
      ...
      l.unlock();

      bool force_flush = false;
      // if bluefs is sharing the same device as data (only), then we
      // can rely on the bluefs commit to flush the device and make
      // deferred aios stable.  that means that if we do have done deferred
      // txcs AND we are not on a single device, we need to force a flush.
      if (bluefs && bluefs_layout.single_shared_device()) {
	if (aios) {
	  force_flush = true;
	} else if (kv_committing.empty() && deferred_stable.empty()) {
	  force_flush = true;  // there's nothing else to commit!
	} else if (deferred_aggressive) {
	  force_flush = true;
	}
      } else {
      	if (aios || !deferred_done.empty()) {
	  force_flush = true;
      	}
      }

      if (force_flush) {
	// flush/barrier on block device
	bdev->flush();                                  // fdatasync(block)

        // if we flush then deferred done are now deferred stable
        if (deferred_stable.empty()) {
          deferred_stable.swap(deferred_done);
        } else {
          deferred_stable.insert(deferred_stable.end(), deferred_done.begin(),
                                 deferred_done.end());
          deferred_done.clear();
        }
      }

      // we will use one final transaction to force a sync
      KeyValueDB::Transaction synct = db->get_transaction();
      ...
      for (auto txc : kv_committing) {
	if (txc->get_state() == TransContext::STATE_KV_QUEUED) {
	  _txc_apply_kv(txc, false);                    // db->submit_transaction (async)
	  --txc->osr->kv_committing_serially;
	} else {
	  ceph_assert(txc->get_state() == TransContext::STATE_KV_SUBMITTED);
	}
	if (txc->had_ios) {
	  --txc->osr->txc_with_unstable_io;
	}
      }

      // release throttle *before* we commit.  this allows new ops
      // to be prepared and enter pipeline while we are waiting on
      // the kv commit sync/flush.
      throttle.release_kv_throttle(costs, txcs);

      // cleanup sync deferred keys
      for (auto b : deferred_stable) {
	for (auto& txc : b->txcs) {
	  bluestore_deferred_transaction_t& wt = *txc.deferred_txn;
	  string key;
	  get_deferred_key(wt.seq, &key);
	  synct->rm_single_key(PREFIX_DEFERRED, key);   // deferred payload no longer needed
	}
      }

      // submit synct synchronously (block and wait for it to commit)
      int r = db_was_opened_read_only || cct->_conf->bluestore_debug_omit_kv_commit ?
	0 : db->submit_transaction_sync(synct);
      ceph_assert(r == 0);
```

`RocksDBStore::submit_transaction` uses `WriteOptions.sync=false`
(`src/kv/RocksDBStore.cc:1627`); `submit_transaction_sync` uses `sync=true`
(`:1640`). Because RocksDB's WAL is a single sequential log, the one synced
write at the end makes all previously submitted (unsynced) batches durable
too. That is the trick that amortizes `fsync` across every txc in the batch.

The order of operations is the whole durability argument:

1. Data aios for new extents completed (`STATE_IO_DONE` reached).
2. `bdev->flush()` = `fdatasync()` on the data device: those extents are now
   stable, and so are any *deferred* replays that completed since the last
   flush.
3. RocksDB batches (new onodes pointing at the new extents, deferred payloads
   for small writes, freelist bits) are written to the WAL and synced.
4. Only now does `_txc_committed_kv` fire `on_commit`; the OSD replies.

If the OSD dies between 1 and 3, the new extents are unreferenced and the
allocator (rebuilt from the freelist at mount) treats them as free. If it
dies after 3 but before a deferred replay hits the device, `_deferred_replay`
(§6.8) redoes it from the `L` keys.

After the sync, txcs move to `_kv_finalize_thread` (`:15244`), which advances
them through `KV_SUBMITTED → KV_DONE`, triggers deferred submission when the
batch is full, and reaps collections. Splitting finalize into its own thread
keeps the sync thread's critical section as short as possible.

### 6.8 Deferred writes: journal in RocksDB, apply later

The journaled record (`bluestore_types.h:1302`):

```cpp
struct bluestore_deferred_op_t {
  typedef enum { OP_WRITE = 1 } type_t;
  __u8 op = 0;
  PExtentVector extents;         // physical destination(s)
  ceph::buffer::list data;       // the bytes
  ...
};

/// writeahead-logged transaction
struct bluestore_deferred_transaction_t {
  uint64_t seq = 0;
  std::list<bluestore_deferred_op_t> ops;
  interval_set<uint64_t> released;  ///< allocations to release after tx
  ...
};
```

Once a txc reaches `KV_DONE` with a `deferred_txn`, `_deferred_queue`
(`:15325`) merges its ops into the sequencer's pending `DeferredBatch`, whose
`iomap` is keyed by physical offset so adjacent writes coalesce:

```cpp
  tmp->txcs.push_back(*txc);
  bluestore_deferred_transaction_t& wt = *txc->deferred_txn;
  for (auto opi = wt.ops.begin(); opi != wt.ops.end(); ++opi) {
    const auto& op = *opi;
    ceph_assert(op.op == bluestore_deferred_op_t::OP_WRITE);
    bufferlist::const_iterator p = op.data.begin();
    for (auto e : op.extents) {
      tmp->prepare_write(cct, wt.seq, e.offset, e.length, p);
    }
  }
```

Submission (`_deferred_submit_unlock`, `:15406`) walks the iomap, glues
physically contiguous runs into one bufferlist, and issues one `aio_write`
per run:

```cpp
  uint64_t start = 0, pos = 0;
  bufferlist bl;
  auto i = b->iomap.begin();
  while (true) {
    if (i == b->iomap.end() || i->first != pos) {
      if (bl.length()) {
	  int r = bdev->aio_write(start, bl, &b->ioc, false);
	  ceph_assert(r == 0);
      }
      if (i == b->iomap.end()) {
	break;
      }
      start = 0;
      pos = i->first;
      bl.clear();
    }
    if (!bl.length()) {
      start = pos;
    }
    pos += i->second.bl.length();
    bl.claim_append(i->second.bl);
    ++i;
  }

  bdev->aio_submit(&b->ioc);
```

Batches are flushed when `bluestore_deferred_batch_ops` txcs accumulate
(`_kv_finalize_thread`), when the throttle is under pressure
(`deferred_aggressive`), or when a sequencer's queue exceeds
`bluestore_max_deferred_txc`. On completion, `_deferred_aio_finish` (`:15471`)
moves the batch to `deferred_done_queue`. The next `_kv_sync_thread` iteration
flushes the device (making it "stable") and deletes the `L` keys in `synct`.
Until that deletion is durable, the journal entry remains and would simply be
replayed again (replays are idempotent: same bytes, same offsets).

Replay at mount (`_deferred_replay`, `:15527`):

```cpp
  KeyValueDB::Iterator it = db->get_iterator(PREFIX_DEFERRED);
  for (it->lower_bound(string()); it->valid(); it->next(), ++count) {
    bluestore_deferred_transaction_t *deferred_txn = new bluestore_deferred_transaction_t;
    bufferlist bl = it->value();
    auto p = bl.cbegin();
    decode(*deferred_txn, p);
    bool has_some = _eliminate_outdated_deferred(deferred_txn, bluefs_extents);
    if (has_some) {
      TransContext *txc = _txc_create(ch.get(), osr,  nullptr);
      txc->deferred_txn = deferred_txn;
      txc->set_state(TransContext::STATE_KV_DONE);
      _txc_state_proc(txc);             // re-enters at DEFERRED_QUEUED
    }
    ...
  }
```

`_eliminate_outdated_deferred` drops any extents that BlueFS has since
claimed (possible when the device is shared and a stale deferred record
targets space that was freed and reallocated to RocksDB files).

### 6.9 Freeing old extents: `_wctx_finish` and `_txc_release_alloc`

Overwrites release the extents they replace, but not immediately:

```cpp
void BlueStore::_wctx_finish(TransContext *txc, CollectionRef& c, OnodeRef& o,
  WriteContext *wctx, set<SharedBlob*> *maybe_unshared_blobs)
{
  auto oep = wctx->old_extents.begin();
  while (oep != wctx->old_extents.end()) {
    ...
    if (blob.is_shared()) {
      // decrement refs in the SharedBlob; only fully unreferenced ranges are released
      ...
    }
    b->maybe_prune_tail();
    for (auto e : r) {
      txc->released.insert(e.offset, e.length);
      txc->statfs_delta.allocated() -= e.length;
      ...
    }
    ...
  }
}
```

`txc->released` is written to the freelist in the kv batch (so the space is
logically free after commit) but the in-memory allocator only learns about it
in `_txc_finish → _txc_release_alloc` (`:14751`), and only after every
*earlier* txc in the sequencer is also done:

```cpp
  while (!releasing_txc.empty()) {
    // release to allocator only after all preceding txc's have also
    // finished any deferred writes that potentially land in these
    // blocks
    auto txc = &releasing_txc.front();
    _txc_release_alloc(txc);
    ...
  }

void BlueStore::_txc_release_alloc(TransContext *txc)
{
  ...
  discard_queued = bdev->try_discard(txc->released);
  // if async discard succeeded, will do alloc->release when discard callback
  // else we should release here
  if (!discard_queued) {
      alloc->release(txc->released);
  }
  ...
}
```

This delay closes the race where a deferred write from txc N targets blocks
that txc N+1 released and a new txc N+2 re-allocated and wrote. With
`bdev_enable_discard=true`, TRIM is issued from dedicated discard threads
(`KernelDevice::_discard_thread`, `:790`) before the allocator sees the space.

---

## 7. The read path

### 7.1 OSD level

`PrimaryLogPG::do_read` (`src/osd/PrimaryLogPG.cc:5848`) clamps the range to
the object size and, for replicated pools, calls straight into the store on
the request thread:

```cpp
  } else {
    int r = pgbackend->objects_read_sync(
      soid, op.extent.offset, op.extent.length, op.flags, &osd_op.outdata);
```

`ReplicatedBackend::objects_read_sync` (`src/osd/ReplicatedBackend.cc:279`):

```cpp
int ReplicatedBackend::objects_read_sync(const hobject_t &hoid, uint64_t off,
  uint64_t len, uint32_t op_flags, bufferlist *bl)
{
  return store->read(ch, ghobject_t(hoid), off, len, *bl, op_flags);
}
```

Reads are synchronous inside the OSD worker thread; the async aio is awaited
with a condition variable. EC pools use `objects_read_async` and reconstruct
from k shards.

### 7.2 BlueStore::read → _do_read

`src/os/bluestore/BlueStore.cc:12560`

```cpp
int BlueStore::read(CollectionHandle &c_, const ghobject_t& oid,
  uint64_t offset, size_t length, bufferlist& bl, uint32_t op_flags)
{
  Collection *c = static_cast<Collection *>(c_.get());
  ...
  {
    std::shared_lock l(c->lock);
    OnodeRef o = c->get_onode(oid, false);        // onode cache or RocksDB "O…" get
    if (!o || !o->exists) {
      r = -ENOENT;
      goto out;
    }
    if (offset == length && offset == 0)
      length = o->onode.size;

    r = _do_read(c, o, offset, length, bl, op_flags);
    ...
  }
```

`_do_read` (`:12887`):

```cpp
  // generally, don't buffer anything, unless the client explicitly requests it.
  bool buffered = false;
  if (op_flags & CEPH_OSD_OP_FLAG_FADVISE_WILLNEED) {
    buffered = true;
  } else if (cct->_conf->bluestore_default_buffered_read &&
	     (op_flags & (CEPH_OSD_OP_FLAG_FADVISE_DONTNEED |
			  CEPH_OSD_OP_FLAG_FADVISE_NOCACHE)) == 0) {
    buffered = true;
  }

  if (offset + length > o->onode.size) {
    length = o->onode.size - offset;
  }

  o->extent_map.fault_range(db, offset, length);   // pull in the needed "…x" shards

  // for deep-scrub, we only read dirty cache and bypass clean cache in
  // order to read underlying block device in case there are silent disk errors.
  if (op_flags & CEPH_OSD_OP_FLAG_BYPASS_CLEAN_CACHE) {
    read_cache_policy = BufferSpace::BYPASS_CLEAN_CACHE;
  }

  // build blob-wise list to of stuff read (that isn't cached)
  ready_regions_t ready_regions;
  blobs2read_t blobs2read;
  _read_cache(o, offset, length, read_cache_policy, ready_regions, blobs2read);

  // read raw blob data.
  vector<bufferlist> compressed_blob_bls;
  IOContext ioc(cct, NULL, !cct->_conf->bluestore_fail_eio);
  r = _prepare_read_ioc(blobs2read, &compressed_blob_bls, &ioc);
  if (r < 0)
    return r;

  int64_t num_ios = blobs2read.size();
  if (ioc.has_pending_aios()) {
    num_ios = ioc.get_num_ios();
    bdev->aio_submit(&ioc);
    ioc.aio_wait();                                 // block this thread until io_getevents delivers
    r = ioc.get_return_value();
    if (r < 0) {
      ceph_assert(r == -EIO); // no other errors allowed
      return -EIO;
    }
  }

  bool csum_error = false;
  r = _generate_read_result_bl(o, offset, length, ready_regions,
                              compressed_blob_bls, blobs2read,
                              buffered && !ioc.skip_cache(),
                              &csum_error, bl);
  if (csum_error) {
    // Handles spurious read errors caused by a kernel bug.
    // We sometimes get all-zero pages as a result of the read under
    // high memory pressure. Retrying the failing read succeeds in most cases.
    if (retry_count >= cct->_conf->bluestore_retry_disk_reads) {
      return -EIO;
    }
    return _do_read(c, o, offset, length, bl, op_flags, retry_count + 1);
  }
```

Three things determine what hits the device:

1. **`_read_cache`** (`:12623`) walks the logical extents, checks the onode's
   `BufferSpace` (BlueStore's own cache; it caches decoded, verified data),
   and for cache misses builds `blobs2read`, rounding each region out to the
   blob's csum chunk size and merging adjacent regions within a blob:

   ```cpp
        // merge regions
        {
          uint64_t r_off = b_off;
          uint64_t r_len = l;
          uint64_t front = r_off % chunk_size;
          if (front) {
            r_off -= front;
            r_len += front;
          }
          unsigned tail = r_len % chunk_size;
          if (tail) {
            r_len += chunk_size - tail;
          }
          ...
   ```

2. **`_prepare_read_ioc`** (`:12720`) maps blob-relative ranges to physical
   extents and queues one `aio_read` per physical extent. Compressed blobs
   are always read whole:

   ```cpp
    if (bptr->get_blob().is_compressed()) {
      // read the whole thing
      ...
      auto r = bptr->get_blob().map(
        0, bptr->get_blob().get_ondisk_length(),
        [&](uint64_t offset, uint64_t length) {
          int r = bdev->aio_read(offset, length, &bl, ioc);
          ...
        });
    } else {
      // read the pieces
      for (auto& req : r2r) {
        auto r = bptr->get_blob().map(
          req.r_off, req.r_len,
          [&](uint64_t offset, uint64_t length) {
            int r = bdev->aio_read(offset, length, &req.bl, ioc);
            ...
          });
   ```

3. **`_generate_read_result_bl`** (`:12789`) verifies checksums *before*
   anything is returned or cached, decompresses, trims the csum-chunk padding,
   and zero-fills holes in the logical range:

   ```cpp
    if (bptr->get_blob().is_compressed()) {
      if (_verify_csum(o, &bptr->get_blob(), 0, compressed_bl, offset) < 0) {
        *csum_error = true;
        return -EIO;
      }
      bufferlist raw_bl;
      auto r = _decompress(compressed_bl, &raw_bl);
      ...
    } else {
      for (auto& req : r2r) {
        if (_verify_csum(o, &bptr->get_blob(), req.r_off, req.bl, offset) < 0) {
          *csum_error = true;
          return -EIO;
        }
        for (const auto& r : req.regs) {
          if (buffered) {
            o->bc.did_read(o->c->cache, r.logical_offset, std::move(region_buffer));
          }
          ready_regions[r.logical_offset].substr_of(req.bl, r.front, r.length);
        }
      }
    }
    ...
  // generate a resulting buffer
  while (pos < length) {
    if (pr != pr_end && pr->first == pos + offset) {
      bl.claim_append(pr->second);
      ...
    } else {
      bl.append_zero(l);       // sparse hole
      ...
    }
  }
   ```

`_verify_csum` (`:13023`) logs the exact physical location on mismatch, which
is what surfaces as `bad crc32c/0x1000 checksum at blob offset ...` in OSD
logs. A persistent mismatch is returned to the OSD as `-EIO`, which turns into
a scrub inconsistency or a read error to the client, never silently wrong
data.

---

## 8. The block device layer: how bytes actually reach the disk

### 8.1 Queuing a write

`KernelDevice::aio_write` (`src/blk/kernel/KernelDevice.cc:1125`) does not do
I/O. It appends an `aio_t` to the `IOContext`:

```cpp
int KernelDevice::aio_write(uint64_t off, bufferlist &bl, IOContext *ioc,
  bool buffered, int write_hint)
{
  uint64_t len = bl.length();
  ceph_assert(is_valid_io(off, len));           // off and len are block_size-aligned
  ...
  if ((!buffered || bl.get_num_buffers() >= IOV_MAX) &&
      bl.rebuild_aligned_size_and_memory(block_size, block_size, IOV_MAX)) {
    dout(20) << __func__ << " rebuilding buffer to be aligned" << dendl;
  }

  _aio_log_start(ioc, off, len);

#ifdef HAVE_LIBAIO
  if (aio && dio && !buffered) {
    ...
      if (bl.length() <= RW_IO_MAX) {
	// fast path (non-huge write)
	ioc->pending_aios.push_back(aio_t(ioc, choose_fd(false, write_hint)));
	++ioc->num_pending;
	auto& aio = ioc->pending_aios.back();
	bl.prepare_iov(&aio.iov);
	aio.bl.claim_append(bl);                  // keep the buffers alive until completion
	aio.pwritev(off, len);                    // io_prep_pwritev(&iocb, fd, iov, n, off)
      } else {
	// write in RW_IO_MAX-sized chunks
	...
      }
  } else
#endif
  {
    int r = _sync_write(off, bl, buffered, write_hint);   // pwritev + sync_file_range
    _aio_log_finish(ioc, off, len);
    if (r < 0)
      return r;
  }
  return 0;
}
```

`rebuild_aligned_size_and_memory` is the memcpy that `O_DIRECT` demands:
every iovec base and length must be a multiple of `block_size`. The
`ObjectStore::Transaction` encoding in §3.4 exists so this is usually a no-op.

`aio_t` (`src/blk/aio/aio.h`) wraps a libaio `iocb` (which must be the first
member so the pointer can be cast back on completion):

```cpp
struct aio_t {
  struct iocb iocb{};  // must be first element; see shenanigans in aio_queue_t
  void *priv;
  int fd;
  boost::container::small_vector<iovec,4> iov;
  uint64_t offset, length;
  long rval;
  ceph::buffer::list bl;  ///< write payload (so that it remains stable for duration)
  ...
  void pwritev(uint64_t _offset, uint64_t len) {
    offset = _offset;
    length = len;
    io_prep_pwritev(&iocb, fd, &iov[0], iov.size(), offset);
  }
  void preadv(uint64_t _offset, uint64_t len) {
    offset = _offset;
    length = len;
    io_prep_preadv(&iocb, fd, &iov[0], iov.size(), offset);
  }
};
```

### 8.2 Submitting

`KernelDevice::aio_submit` (`:983`) moves the pending list to running and
calls the queue backend:

```cpp
void KernelDevice::aio_submit(IOContext *ioc)
{
  if (ioc->num_pending.load() == 0) {
    return;
  }

  // move these aside, and get our end iterator position now, as the
  // aios might complete as soon as they are submitted and queue more
  // wal aio's.
  list<aio_t>::iterator e = ioc->running_aios.begin();
  ioc->running_aios.splice(e, ioc->pending_aios);

  int pending = ioc->num_pending.load();
  ioc->num_running += pending;
  ioc->num_pending -= pending;
  ...
  void *priv = static_cast<void*>(ioc);
  int retry_max = cct->_conf->bdev_aio_submit_retry_max;
  int initial_delay_us = cct->_conf->bdev_aio_submit_retry_initial_delay_us;
  int r, retries = 0;
  r = io_queue->submit_batch(ioc->running_aios.begin(), e,
			     priv, &retries, retry_max, initial_delay_us);
  ...
}
```

`aio_queue_t::submit_batch` (`src/blk/aio/aio.cc:19`) is a thin loop over
`io_submit(2)` with exponential backoff on `EAGAIN` (queue full):

```cpp
  while (cur != end || pushed < pulled) {
    while (cur != end && pulled < max_iodepth) {
      cur->priv = priv;
      piocb[pulled] = &(*cur);
      ++pulled;
      ++cur;
    }
    int toSubmit = pulled - pushed;
    r = io_submit(ctx, toSubmit, (struct iocb**)(piocb + pushed));
    if (r >= 0 && r < toSubmit) {
      pushed += r;
      done += r;
      r = -EAGAIN;
    }
    if (r < 0) {
      if (r == -EAGAIN && attempts-- > 0) {
	usleep(delay);
	delay *= 2;
	(*retries)++;
	continue;
      }
      return r;
    }
    ...
  }
```

The io_uring backend (`src/blk/kernel/io_uring.cc`) is structurally the same:
`init_sqe` calls `io_uring_prep_writev`/`io_uring_prep_readv` against
registered (fixed) fds, `submit_batch` calls `io_uring_submit`, and
completions are drained with `io_uring_for_each_cqe`. `bdev_ioring_hipri`
enables `IORING_SETUP_IOPOLL` and `bdev_ioring_sqthread_poll` enables
`IORING_SETUP_SQPOLL`.

### 8.3 Completion: the aio thread

`KernelDevice::_aio_thread` (`:663`) polls `io_getevents` with a
`bdev_aio_poll_ms` (250 ms) timeout, reaping up to `bdev_aio_reap_max` (16)
events per call:

```cpp
    int r = io_queue->get_next_completed(cct->_conf->bdev_aio_poll_ms, aio, max);
    ...
    if (r > 0) {
      for (int i = 0; i < r; ++i) {
	IOContext *ioc = static_cast<IOContext*>(aio[i]->priv);
	_aio_log_finish(ioc, aio[i]->offset, aio[i]->length);
	...
	// set flag indicating new ios have completed.  we do this *before*
	// any completion or notifications so that any user flush() that
	// follows the observed io completion will include this io.
	io_since_flush.store(true);

	long r = aio[i]->get_return_value();
        if (r < 0) {
          if (ioc->allow_eio && is_expected_ioerr(r)) {
            ioc->set_return_value(-EIO);
          } else {
	    ...
	    ceph_abort_msg("Unexpected IO error. This may suggest a hardware issue. Please check your kernel log!");
          }
        } else if (aio[i]->length != (uint64_t)r) {
          ceph_abort_msg("unexpected aio return value: does not match length");
        }

	// NOTE: once num_running and we either call the callback or
	// call aio_wake we cannot touch ioc or aio[] as the caller
	// may free it.
	if (ioc->priv) {
	  if (--ioc->num_running == 0) {
	    aio_callback(aio_callback_priv, ioc->priv);   // → BlueStore::aio_cb → txc->aio_finish
	  }
	} else {
          ioc->try_aio_wake();                            // → wakes _do_read's ioc.aio_wait()
	}
      }
    }
```

The policy on write errors is deliberate: a failed write aio **aborts the
OSD**. A short write or `EIO` on the data device means the on-disk state no
longer matches what BlueStore believes, and the safe response is to crash and
let peering/recovery re-replicate from healthy OSDs. Reads may return `EIO`
(when `allow_eio` is set, which `_do_read` does unless
`bluestore_fail_eio=true`) because the checksum layer above can cope.

For write txcs, `ioc->priv` is the `TransContext`; `aio_cb`
(`BlueStore.cc:5722`) calls `txc->aio_finish(store)` which re-enters
`_txc_state_proc` in `STATE_AIO_WAIT`. For deferred batches, it is the
`DeferredBatch` and the path is `_deferred_aio_finish`.

### 8.4 Barriers: `flush()`

`KernelDevice::flush` (`:494`) is the only place BlueStore forces the device's
volatile cache to disk:

```cpp
int KernelDevice::flush()
{
  // protect flush with a mutex.  note that we are not really protecting
  // data here.  instead, we're ensuring that if any flush() caller
  // sees that io_since_flush is true, they block any racing callers
  // until the flush is observed.  that allows racing threads to be
  // calling flush while still ensuring that *any* of them that got an
  // aio completion notification will not return before that aio is
  // stable on disk: whichever thread sees the flag first will block
  // followers until the aio is stable.
  std::lock_guard l(flush_mutex);

  bool expect = true;
  if (!io_since_flush.compare_exchange_strong(expect, false)) {
    dout(10) << __func__ << " no-op (no ios since last flush), flag is "
	     << (int)io_since_flush.load() << dendl;
    return 0;
  }
  ...
  int r = ::fdatasync(fd_directs[WRITE_LIFE_NOT_SET]);
  ...
  if (r < 0) {
    r = -errno;
    derr << __func__ << " fdatasync got: " << cpp_strerror(r) << dendl;
    ceph_abort();
  }
  return r;
}
```

`fdatasync()` on an `O_DIRECT` block device fd translates to a cache flush
(`REQ_PREFLUSH`) to the drive. The `io_since_flush` CAS means back-to-back
`flush()` calls with no intervening completion are free, which matters when
BlueFS and the kv sync thread both flush the same shared device.

### 8.5 Synchronous writes

A handful of writers (bdev label, BlueFS superblock, `bluefs_sync_write=true`)
use `KernelDevice::write` → `_sync_write` (`:1034`): a `pwritev` loop that
handles short writes, followed by `sync_file_range` when buffered:

```cpp
  do {
    auto r = ::pwritev(choose_fd(buffered, write_hint), &iov[idx], iov.size() - idx, o);
    if (r < 0) { ... }
    o += r;
    left -= r;
    if (left) {
      // skip fully processed IOVs
      while (idx < iov.size() && (size_t)r >= iov[idx].iov_len) {
        r -= iov[idx++].iov_len;
      }
      // update partially processed one if any
      if (r) {
        iov[idx].iov_base = static_cast<char*>(iov[idx].iov_base) + r;
        iov[idx].iov_len -= r;
        r = 0;
      }
    }
  } while (left);

#ifdef HAVE_SYNC_FILE_RANGE
  if (buffered) {
    // initiate IO and wait till it completes
    auto r = ::sync_file_range(fd_buffereds[WRITE_LIFE_NOT_SET], off, len,
      SYNC_FILE_RANGE_WRITE|SYNC_FILE_RANGE_WAIT_AFTER|SYNC_FILE_RANGE_WAIT_BEFORE);
    ...
  }
#endif
  io_since_flush.store(true);
```

### 8.6 Reads at the device

`KernelDevice::read` (`:1410`) is a blocking `pread` into a freshly allocated
page-aligned buffer; `aio_read` (`:1458`) queues a `preadv` iocb the same way
`aio_write` does. `read_random` (`:1530`) serves BlueFS/RocksDB point reads
and falls back to `direct_read_unaligned` (`:1492`), which reads the enclosing
aligned range into a bounce buffer and memcpys out, whenever the caller's
offset, length or buffer address is not block/page aligned. This is the path
every RocksDB SST block read takes.

---

## 9. Free-space tracking

Two structures exist for the same information, one fast and volatile, one
durable:

- **Allocator** (`src/os/bluestore/Allocator.h`), default `hybrid` (AVL tree
  for the bulk of free space, bitmap for fragmented ranges). Pure in-memory.
  Rebuilt on mount from the freelist, or (with the `NCB`/null freelist
  manager) by scanning all onodes.
- **FreelistManager**, `BitmapFreelistManager` (`BitmapFreelistManager.cc`),
  persisted under the `b` prefix as one bitmap value per
  `bluestore_freelist_blocks_per_key` blocks. Allocation and release are the
  **same operation**, an XOR, submitted as a RocksDB *merge* operand so the
  batch never has to read the current value:

```cpp
void BitmapFreelistManager::_xor(uint64_t offset, uint64_t length,
  KeyValueDB::Transaction txn)
{
  // must be block aligned
  ceph_assert((offset & block_mask) == offset);
  ceph_assert((length & block_mask) == length);

  uint64_t first_key = offset & key_mask;
  uint64_t last_key = (offset + length - 1) & key_mask;

  if (first_key == last_key) {
    bufferptr p(blocks_per_key >> 3);
    p.zero();
    unsigned s = (offset & ~key_mask) / bytes_per_block;
    unsigned e = ((offset + length - 1) & ~key_mask) / bytes_per_block;
    for (unsigned i = s; i <= e; ++i) {
      p[i >> 3] ^= 1ull << (i & 7);
    }
    string k;
    make_offset_key(first_key, &k);
    bufferlist bl;
    bl.append(p);
    txn->merge(bitmap_prefix, k, bl);
  } else {
    // first key ... middle keys (all_set_bl) ... last key
    ...
  }
}
```

Because `_txc_finalize_kv` puts these merges in the same batch as the onode
update, the freelist and the extent map can never disagree after a crash.

BlueFS has its own allocator per device (`BlueFS::_allocate`, `BlueFS.cc:4402`)
and, on a shared device, draws from the *same* BlueStore allocator instance
(`shared_alloc`) with a larger allocation unit, tracking its extents in the
BlueFS journal rather than in the freelist.

---

## 10. Metadata I/O: RocksDB on BlueFS

RocksDB never opens a file. `BlueRocksEnv` (`src/os/bluestore/BlueRocksEnv.cc`)
implements `rocksdb::Env` on top of BlueFS.

### 10.1 Writes (WAL appends, SST flushes, compactions)

```cpp
class BlueRocksWritableFile : public rocksdb::WritableFile {
  BlueFS *fs;
  BlueFS::FileWriter *h;
  ...
  rocksdb::Status Append(const rocksdb::Slice& data) override {
    fs->append_try_flush(h, data.data(), data.size());
    return rocksdb::Status::OK();
  }
  rocksdb::Status Flush() override {
    fs->flush(h);
    return rocksdb::Status::OK();
  }
  rocksdb::Status Sync() override { // sync data
    fs->fsync(h);
    return rocksdb::Status::OK();
  }
  rocksdb::Status RangeSync(uint64_t offset, uint64_t nbytes) override {
    // round down to page boundaries
    ...
    if (nbytes)
      fs->flush_range(h, offset, nbytes);
    return rocksdb::Status::OK();
  }
```

`append_try_flush` (`BlueFS.cc:4096`) buffers in the `FileWriter` until
`bluefs_min_flush_size` accumulates, then `_flush_F` → `_flush_range_F`
(`:3905`) allocates extents on demand and writes:

```cpp
int BlueFS::_flush_range_F(FileWriter *h, uint64_t offset, uint64_t length)
{
  ...
  uint64_t allocated = h->file->fnode.get_allocated();
  // do not bother to dirty the file if we are overwriting
  // previously allocated extents.
  if (allocated < end) {
    int r = _allocate(vselector->select_prefer_bdev(h->file->vselector_hint),
		      end - allocated, 0, &h->file->fnode, ...);
    ...
    h->file->is_dirty = true;
  }
  if (h->file->fnode.size < end) {
    h->file->fnode.size = end;
    if (!h->file->envelope_mode()) {
      h->file->is_dirty = true;
    }
  }
  int res = _flush_data(h, offset, length, buffered);
  ...
}
```

`vselector` decides which device gets the extents: WAL files prefer
`BDEV_WAL`, SSTs prefer `BDEV_DB`, everything spills to `BDEV_SLOW` when the
faster device is full (this is the "spillover" you see in `ceph osd df`).

`_flush_data` (`:3977`) handles the partial-block tail (BlueFS rewrites the
last partial block by keeping its tail in memory), then issues one aio per
physical extent and submits:

```cpp
  unsigned partial = x_off & ~super.block_mask();
  if (partial) {
    x_off -= partial;
    offset -= partial;
    length += partial;
    // waiting for previous aio to complete
    for (auto p : h->iocv) {
      if (p) {
	p->aio_wait();
      }
    }
  }

  auto bl = h->flush_buffer(cct, partial, length, super);   // pad to block_size
  ...
  while (length > 0) {
    uint64_t x_len = std::min(p->length - x_off, length);
    bufferlist t;
    t.substr_of(bl, bloff, x_len);
    if (cct->_conf->bluefs_sync_write) {
      bdev[p->bdev]->write(p->offset + x_off, t, buffered, h->write_hint);
    } else {
      bdev[p->bdev]->aio_write(p->offset + x_off, t, h->iocv[p->bdev], buffered, h->write_hint);
    }
    h->dirty_devs[p->bdev] = true;
    ...
    ++p;
    x_off = 0;
  }
  for (unsigned i = 0; i < MAX_BDEV; ++i) {
    if (bdev[i]) {
      if (h->iocv[i] && h->iocv[i]->has_pending_aios()) {
        bdev[i]->aio_submit(h->iocv[i]);
      }
    }
  }
```

### 10.2 fsync: data, then the BlueFS journal

`BlueFS::_fsync` (`:4301`) is what RocksDB's `sync=true` write ultimately
becomes:

```cpp
int BlueFS::_fsync(FileWriter *h, bool force_dirty)
{
  ...
    int r = _flush_F(h, true);            // push remaining buffer to the device
    if (r < 0)
      return r;
    _flush_bdev(h);                       // wait for this writer's aios, then fdatasync each dirty device
    if (h->file->is_dirty || force_dirty) {
      _signal_dirty_to_log_D(h);          // queue fnode (extent list / size) change to the BlueFS log
      h->file->is_dirty = false;
    }
    {
      std::lock_guard dl(dirty.lock);
      if (dirty.seq_stable < h->file->dirty_seq) {
	old_dirty_seq = h->file->dirty_seq;
      }
    }
  if (old_dirty_seq) {
    _flush_and_sync_log_LD(old_dirty_seq);  // write + fdatasync the BlueFS journal
  }
  _maybe_compact_log_LNF_NF_LD_D();
  return 0;
}
```

So a RocksDB sync costs at most two device flushes: one for the file data,
one for the BlueFS journal (only when the file's extent list changed, i.e.
when it grew into new allocations; pure overwrites of preallocated WAL space
skip the second). `_flush_and_sync_log_core` (`:3643`) encodes the pending
`bluefs_transaction_t`, pads to a block, appends to the journal file
(`ino 1`, whose extents are in the superblock) and flushes:

```cpp
  bufferlist bl;
  bl.reserve(super.block_size);
  encode(log.t, bl);
  // pad to block boundary
  size_t realign = super.block_size - (bl.length() % super.block_size);
  if (realign && realign != super.block_size)
    bl.append_zero(realign);
  ...
  log.writer->append(bl);
  // prepare log for new transactions
  log.t.clear();
  log.t.seq = log.seq_live;
  uint64_t new_data = _flush_special(log.writer);
```

The journal is periodically compacted (rewritten as a snapshot of all fnodes)
and the new journal head is published by rewriting the superblock
(`_write_super`, `:1281`: a synchronous 4 KiB write at offset 4096 with a
CRC).

### 10.3 Reads

`BlueRocksRandomAccessFile::Read` → `BlueFS::read_random` → `_read_random`
(`:2391`) → `_bdev_read_random` → `KernelDevice::read_random` (`pread`). There
is a small per-reader prefetch buffer (`bluefs_max_prefetch`) for sequential
scans, but no BlueFS-level page cache; RocksDB's block cache is the cache.
With `bluefs_buffered_io=true` (the default on most releases) these reads go
through the buffered fd so the kernel page cache can help with SST metadata.

---

## 11. Two end-to-end timelines

### 11.1 A 4 KiB RBD write on an HDD-backed replicated pool (size 3)

1. Client Objecter maps the object to PG `1.2f`, primary osd.7, sends `MOSDOp`.
2. osd.7: `dequeue_op → PrimaryLogPG::do_op → execute_ctx → do_osd_ops`
   records `write(off, 4096)` in a `PGTransaction`.
3. `issue_repop → ReplicatedBackend::submit_transaction`: builds one
   `ObjectStore::Transaction` (object write + object_info xattr + PG log omap
   entries), ships it in `MOSDRepOp` to osd.3 and osd.12, and calls
   `BlueStore::queue_transactions` locally.
4. BlueStore `_txc_add_transaction`: onode faulted from cache/RocksDB;
   `_do_write_small` finds the target 4 KiB block is already allocated and
   used → reads nothing (chunk-aligned), sizes 4 KiB `< prefer_deferred_size
   (64 KiB)` → `_get_deferred_op`. No data aio is queued.
5. `_txc_write_nodes` encodes the onode; the deferred payload is put under
   `L<seq>`; freelist untouched (no allocation). `_txc_state_proc`: no aios →
   falls to `IO_DONE` → `KV_QUEUED`, `db->submit_transaction` (unsynced),
   wake kv sync thread.
6. `_kv_sync_thread`: `bdev->flush()` is skipped because this batch has no
   data aios and no completed deferred batches; `submit_transaction_sync(synct)`
   → RocksDB WAL append + `BlueFS::_fsync` → `fdatasync()` on the WAL device
   (`block.wal` if present, otherwise `block.db` or `block`). On a single
   shared device that BlueFS fsync is also what makes any earlier deferred
   replays stable, which is why the sync thread can skip its own flush there.
7. `_kv_finalize_thread`: `KV_SUBMITTED → KV_DONE` → `on_commit` → OSD
   `op_commit`; when osd.3 and osd.12 reply `MOSDRepOpReply`, the client gets
   `ONDISK`.
8. Later (batch full or throttle): `_deferred_submit_unlock` → one
   `aio_write(4096 bytes)` at the block's physical offset → `io_submit`.
9. Next kv sync iteration: `fdatasync(block)` → `L<seq>` deleted in `synct`.

Random 4 KiB writes therefore cost the HDD **one sequential WAL append per
commit batch** plus **one random write per deferred replay**, but the client
latency includes only the first.

### 11.2 A 4 MiB RGW object write on an SSD pool

1–3. As above.
4. `_do_write` (or `Writer::do_write`) splits 4 MiB into 64 blobs of
   `max_blob_size` (64 KiB). `_do_alloc_write`: `alloc->allocate(4 MiB)`
   returns a few large extents; per blob `calc_csum` (16 crc32c per blob);
   `prefer_deferred_size_ssd = 0` → every blob is `bdev->aio_write`'d into
   `txc->ioc` (one or more `iocb`s per blob, merged by `map_bl` per physical
   extent).
5. `_txc_finalize_kv`: `fm->allocate()` → XOR merges under `b…` for the new
   ranges. `_txc_state_proc`: `PREPARE → AIO_WAIT`, `io_submit` of ~64 iocbs.
6. `_aio_thread` reaps completions; when `num_running` hits 0, `aio_cb` →
   `_txc_finish_io` → `IO_DONE → KV_QUEUED`.
7. `_kv_sync_thread`: `aios > 0` → `fdatasync(block)`; then
   `submit_transaction_sync` (onode with 64 blobs and their extents, checksums,
   freelist). Commit.
8. On a later overwrite/delete, the 4 MiB is released into `txc->released`,
   XOR'd back into the freelist in that txc's batch, and after that txc
   finishes, optionally `BLKDISCARD`ed and returned to the allocator.

---

## 12. Reference: the files that matter

| Concern                              | File                                        |
|--------------------------------------|---------------------------------------------|
| Client placement (CRUSH)             | `src/osdc/Objecter.cc`, `src/osd/OSDMap.cc`, `src/crush/` |
| OSD op dispatch                      | `src/osd/OSD.cc`                            |
| Op execution, replication            | `src/osd/PrimaryLogPG.cc`, `src/osd/ReplicatedBackend.cc`, `src/osd/ECBackend.cc`, `src/osd/ECCommon.cc` |
| Transaction wire format              | `src/os/Transaction.h`, `src/os/ObjectStore.h` |
| BlueStore core, txc state machine    | `src/os/bluestore/BlueStore.cc`, `BlueStore.h` |
| v2 write path                        | `src/os/bluestore/Writer.cc`                |
| On-disk types (onode, blob, deferred)| `src/os/bluestore/bluestore_types.h`        |
| Allocators                           | `src/os/bluestore/{Hybrid,Avl,Bitmap,Btree2}Allocator.cc` |
| Persistent freelist                  | `src/os/bluestore/BitmapFreelistManager.cc` |
| BlueFS                               | `src/os/bluestore/BlueFS.cc`, `bluefs_types.h` |
| RocksDB glue                         | `src/os/bluestore/BlueRocksEnv.cc`, `src/kv/RocksDBStore.cc` |
| Block device abstraction             | `src/blk/BlockDevice.h`, `src/blk/kernel/KernelDevice.cc` |
| libaio / io_uring queues             | `src/blk/aio/aio.cc`, `src/blk/kernel/io_uring.cc` |
| Tunables                             | `src/common/options/global.yaml.in`         |

Useful runtime knobs for observing the path: `ceph daemon osd.N perf dump`
exposes `bluestore.*` (`kv_sync_lat`, `kv_flush_lat`, `deferred_write_ops`,
`write_small`, `write_big`, `read_wait_aio_lat`, `csum_lat`) and `bluefs.*`
(`bytes_written_wal`, `bytes_written_sst`, `slow_used_bytes` for spillover);
`ceph daemon osd.N dump_ops_in_flight` shows an op's position in the pipeline
(`reached_pg`, `sub_op_commit_rec`, `commit_sent`).
