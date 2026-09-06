# Ceph File Storage (CephFS): How a POSIX Filesystem Becomes RADOS Objects

Companion to `ceph-disk-io-internals.md` (what an OSD does with a request)
and `ceph-block-storage-rbd.md`. This document covers **CephFS**: how file
data and directory metadata are laid out in RADOS, how the Metadata Server
(MDS) journals and persists metadata, and how a client `write()`/`read()`
reaches the OSDs. Source references are to `C:\opensource\cpeh`
(commit `9438aabf3f7`).

---

## 1. The mental model

CephFS separates **data** from **metadata** at the pool level and at the
daemon level:

- **File data** is written by the *client* directly to OSDs in the data
  pool, striped into objects named `<inode>.<object-number>`. The MDS never
  touches file contents.
- **Metadata** (directory entries, inodes, layouts, xattrs) is owned by the
  *MDS*, which keeps it in memory, journals every change to a RADOS-backed
  log, and lazily writes directory fragments back as omap objects in the
  metadata pool.
- **Capabilities ("caps")** issued by the MDS tell each client which cached
  state it may trust and whether it may buffer writes. Caps are the
  coherence protocol between clients and the MDS.

```
 application
   │ open/read/write/mkdir/rename …
   ▼
 client (src/client/Client.cc — libcephfs, ceph-fuse)   ─ or ─  kernel client (fs/ceph, mounted via src/mount)
   │            │
   │ metadata   │ data: ObjectCacher → Filer → Striper → Objecter
   │ (MClientRequest)                     │
   ▼                                      ▼
 MDS (src/mds)                          OSDs, data pool:  <ino hex>.<objno %08x>
   │  MDCache / CInode / CDir / CDentry
   │  MDLog → Journaler → Filer → Objecter
   ▼
 OSDs, metadata pool:
   200.00000000, 200.00000001, …      MDS rank 0 journal (ino 0x200 = MDS_INO_LOG_OFFSET + rank)
   <dir ino>.<frag>                   directory fragment: omap key per dentry, header = fnode
   mds0_inotable, mds0_sessionmap,    tables
   mds0_openfiles.0, …
   500.00000000 …                     purge queue journal (ino 0x500)
```

---

## 2. Data layout: files are striped objects

### 2.1 Naming

`src/include/object.cc:46`

```cpp
const char *file_object_t::c_str() const {
  if (!buf[0])
    snprintf(buf, sizeof(buf), "%llx.%08llx", (long long unsigned)ino, (long long unsigned)bno);
  return buf;
}
```

Inode `0x10000000abc`, object 3 → `10000000abc.00000003` in the data pool
named by the file's layout. The same `file_layout_t` and `Striper` as RBD
(`ceph-block-storage-rbd.md` §3) are used; the defaults are 4 MiB objects
with no striping. Unlike RBD, the layout is a per-inode property inherited
from the parent directory and settable with the `ceph.file.layout` /
`ceph.dir.layout` virtual xattrs, so different directories can target
different pools (this is also how CephFS uses erasure-coded data pools).

### 2.2 The backtrace: making data objects self-describing

The first object of every file (`<ino>.00000000`) carries an xattr named
`parent` holding an `inode_backtrace_t`: the chain of (directory inode,
dentry name) pairs back to the root. The MDS writes it whenever the file is
created, renamed or has its layout changed (`src/mds/CInode.cc:66`):

```cpp
void CInodeCommitOperation::update(ObjectOperation &op, inode_backtrace_t &bt) {
  op.priority = priority;
  op.create(false);

  bufferlist parent_bl;
  encode(bt, parent_bl);
  op.setxattr("parent", parent_bl);

  // for the old pool there is no need to update the layout and symlink
  if (!update_layout_symlink)
    return;

  bufferlist layout_bl;
  encode(_layout, layout_bl, _features);
  op.setxattr("layout", layout_bl);
  ...
```

built from the in-memory tree (`:1416`):

```cpp
void CInode::build_backtrace(int64_t pool, inode_backtrace_t& bt)
{
  bt.ino = ino();
  bt.ancestors.clear();
  bt.pool = pool;

  CInode *in = this;
  CDentry *pdn = get_parent_dn();
  while (pdn) {
    CInode *diri = pdn->get_dir()->get_inode();
    bt.ancestors.push_back(inode_backpointer_t(diri->ino(), pdn->get_name(), in->get_inode()->version));
    in = diri;
    pdn = in->get_parent_dn();
  }
  ...
```

Its consumers are the "open by inode number" paths (NFS file handles, hard
links, `cephfs-data-scan` disaster recovery). `MDCache::fetch_backtrace`
(`src/mds/MDCache.cc:10508`) is a single `getxattr`:

```cpp
void MDCache::fetch_backtrace(inodeno_t ino, int64_t pool, bufferlist& bl, Context *fin)
{
  object_t oid = CInode::get_object_name(ino, frag_t(), "");
  mds->objecter->getxattr(oid, object_locator_t(pool), "parent", CEPH_NOSNAP, &bl, 0, fin);
```

Directories keep their backtrace on their dirfrag object in the metadata
pool (same object name, `.inode` suffix for the inode blob of a root-ish
directory, see `CInode::store` at `:1269`).

---

## 3. Metadata layout: directory fragments as omap

A directory is one or more **dirfrags** (`CDir`), each persisted as one
RADOS object in the metadata pool. The object is named after the directory
inode and fragment id (`src/mds/CInode.cc:1260`):

```cpp
object_t InodeStoreBase::get_object_name(inodeno_t ino, frag_t fg, std::string_view suffix)
{
  char n[60];
  snprintf(n, sizeof(n), "%llx.%08llx", (long long unsigned)ino, (long long unsigned)fg);
  ...
```

Inside, each dentry is an **omap key** whose value is the encoded dentry:
either the full inode (`CInode` for a primary link) or a remote-link
record. The `fnode_t` (fragstat, rstat, version) is the omap **header**.
Large directories are split into more fragments by the balancer
(`mds_bal_split_size = 10000` dentries; hard cap
`mds_bal_fragment_size_max = 100000`).

The commit is `CDir::_omap_commit_ops` (`src/mds/CDir.cc:2637`), batched
into ops of at most `mds_dir_max_commit_size` MiB (default 10):

```cpp
  SnapContext snapc;
  object_t oid = get_ondisk_object();
  object_locator_t oloc(metapool);

  map<string, bufferlist> _set;
  set<string> _rm;

  auto commit_one = [&](bool header=false) {
    ObjectOperation op;
    ceph_assert(header || !_set.empty() || !_rm.empty());

    // don't create new dirfrag blindly
    if (!_new)
      op.stat(nullptr, nullptr, nullptr);

    /*
     * save the header at the last moment.. If we were to send it off before
     * other updates, but die before sending them all, we'd think that the
     * on-disk state was fully committed even though it wasn't! However, since
     * the messages are strictly ordered between the MDS and the OSD, and
     * since messages to a given PG are strictly ordered, if we simply send
     * the message containing the header off last, we cannot get our header
     * into an incorrect state.
     */
    if (header) {
      bufferlist header;
      encode(*fnode, header);
      op.omap_set_header(header);
    }

    op.priority = op_prio;
    if (!_set.empty())
      op.omap_set(_set);
    if (!_rm.empty())
      op.omap_rm_keys(_rm);
    mdcache->mds->objecter->mutate(oid, oloc, op, snapc,
                                   ceph::real_clock::now(),
                                   0, gather.new_sub());
    ...
  };
```

On the OSD, omap keys land in RocksDB under BlueStore's `p`/`m` prefixes
(per-PG omap, see the disk-internals doc §4.2), which is why CephFS
metadata pools want fast devices: directory operations are RocksDB
operations, not data-device I/O.

This commit is **not** on the client's critical path. The MDS only commits
dirfrags when it wants to *expire* a journal segment (§4.3) or under memory
pressure. Between commits the authoritative state is the journal.

---

## 4. The MDS journal

### 4.1 What it is

The MDS journal is a byte stream written through `Journaler`
(`src/osdc/Journaler.cc`) into ordinary striped objects in the metadata
pool, inode `0x200 + rank` (`MDS_INO_LOG_OFFSET`, `src/mds/mdstypes.h:41`).
Rank 0's journal is objects `200.00000000`, `200.00000001`, … of the default
4 MiB size. A small header object (`200.00000000` is object 0; the header
lives in a separate `<ino>.00000000`-style object written with
`write_full`) records the trimmed/expire/write positions
(`src/osdc/Journaler.h:132`):

```cpp
  class Header {
    public:
    uint64_t trimmed_pos;
    uint64_t expire_pos;
    uint64_t unused_field;
    uint64_t write_pos;
    std::string magic;
    file_layout_t layout; //< The mapping from byte stream offsets
			     //  to RADOS objects
    stream_format_t stream_format; //< The encoding of LogEvents
				   //  within the journal byte stream
```

Every metadata mutation becomes a `LogEvent` (`EUpdate`, `EOpen`,
`ESession`, …) whose `EMetaBlob` carries the full new versions of every
dirty inode and dentry it touched. Replaying the journal after an MDS
failover therefore reconstructs the in-memory cache without reading any
dirfrag that a later event would overwrite.

### 4.2 Appending

`MDLog::_submit_thread` (`src/mds/MDLog.cc:491`) encodes events and hands
them to the journaler:

```cpp
    if (data.le) {
      LogEvent *le = data.le;
      auto&& ls = le->_segment;
      // encode it, with event type
      bufferlist bl;
      le->encode_with_header(bl, features);

      uint64_t write_pos = journaler->get_write_pos();
      le->set_start_off(write_pos);
      if (dynamic_cast<SegmentBoundary*>(le)) {
	ls->offset = write_pos;
      }
      ...
      // journal it.
      const uint64_t new_write_pos = journaler->append_entry(bl);  // bl is destroyed.
      ls->end = new_write_pos;

      MDSLogContextBase *fin;
      if (data.fin) {
	fin = dynamic_cast<MDSLogContextBase*>(data.fin);
	fin->set_write_pos(new_write_pos);
      } else {
	fin = new C_MDL_Flushed(this, new_write_pos);
      }

      journaler->wait_for_flush(fin);          // fin runs when write_pos is *safe* on OSDs

      if (data.flush)
	journaler->flush();
```

`Journaler::append_entry` (`src/osdc/Journaler.cc:609`) buffers in memory
and flushes whenever the write crosses an object boundary; `_do_flush`
(`:655`) turns the buffer into a striped write:

```cpp
  Context *onsafe = new C_Flush(this, flush_pos, now);  // on COMMIT
  pending_safe[flush_pos] = next_safe_pos;
  ...
  filer.write(ino, &layout, snapc,
	      flush_pos, len, write_bl, ceph::real_clock::now(),
	      0,
	      wrap_finisher(onsafe), write_iohint);

  flush_pos += len;
```

`Filer::write` (`src/osdc/Filer.cc:83`) is the striper plus a
scatter-gather write:

```cpp
void Filer::write(inodeno_t ino, const file_layout_t *layout, const SnapContext& snapc,
		  uint64_t offset, uint64_t len, ceph::buffer::list& bl,
		  ceph::real_time mtime, int flags, Context *oncommit, int op_flags) {
  std::vector<ObjectExtent> extents;
  Striper::file_to_extents(cct, ino, layout, offset, len, 0, extents);
  objecter->sg_write(extents, snapc, bl, mtime, flags, oncommit, op_flags);
}
```

Two details make the journal cheap:

- **Prezeroing** (`_issue_prezero`, `:830`): the journaler zeroes
  `journaler_prezero_periods` objects ahead of the write position so the
  object it is about to append into is guaranteed not to contain stale
  bytes from a previous incarnation of the journal, and so replay can stop
  at the first non-event bytes.
- **Safe position tracking** (`_finish_flush`, `:557`): completions arrive
  out of order, so `safe_pos` only advances to the smallest still-pending
  flush start. Waiters registered with `wait_for_flush(pos)` fire when
  `safe_pos >= pos`.

### 4.3 Segments, expiry and trimming

The journal is divided into `LogSegment`s (a new one every
`mds_log_events_per_segment = 1024` events or `mds_log_segment_size` bytes,
which defaults to the object size). `mds_log_max_segments = 128` bounds how
many can be un-expired. To expire a segment the MDS must make everything
that segment's events describe durable *elsewhere*
(`LogSegment::try_to_expire`, `src/mds/journal.cc:125`): commit dirty
dirfrags (`dir->commit(...)`, the omap write from §3), store dirty inodes
and backtraces (`in->store(...)`), flush tables and session maps. Once the
gather completes, `expire_pos` moves forward, the header is rewritten
(`Journaler::_write_head`, `:478`), and objects behind `trimmed_pos` are
deleted.

So the durable metadata write path is:

1. Event appended to journal → **client gets its reply** (§5.1).
2. Later, segment expiry writes the dirfrag omap keys and inode backtraces.
3. Later still, journal objects are trimmed.

### 4.4 Replying to clients: early vs safe

`Server::journal_and_reply` (`src/mds/Server.cc:2114`):

```cpp
  early_reply(mdr, in, dn);            // "unsafe" reply: result visible, not yet durable

  mdr->committing = true;
  submit_mdlog_entry(le, fin, mdr, __func__);

  if (mdr->is_queued_for_replay()) {
    ...
  } else if (mdr->did_early_reply)
    mds->locker->handle_locks_for_early_reply(mdr.get());
  else
    mdlog->flush();
```

With `mds_early_reply = true` (default) the client sees the result of a
`mkdir`/`rename`/`unlink` as soon as the MDS has applied it in memory. A
second "safe" reply follows when the journal write commits. The client
tracks these in `Inode::unsafe_ops`, and `fsync()` on a directory or file
waits for them (`Client::_fsync`, `src/client/Client.cc:13292`):

```cpp
  if (!syncdataonly && !in->unsafe_ops.empty()) {
    flush_mdlog_sync(in);                       // ask the MDS to flush its journal now

    MetaRequest *req = in->unsafe_ops.back();
    ldout(cct, 15) << "waiting on unsafe requests, last tid " << req->get_tid() <<  dendl;
    req->get();
    wait_on_list(req->waitfor_safe);
```

This is the same semantics as a local filesystem with a write-back
journal: `close()` does not imply durability, `fsync()` does.

---

## 5. Client data path

### 5.1 Caps decide the mode

Before touching data the client needs the right capability bits:
`CEPH_CAP_FILE_WR` to write at all, `CEPH_CAP_FILE_BUFFER` to buffer
writes locally, `CEPH_CAP_FILE_CACHE` to cache reads, `CEPH_CAP_FILE_LAZYIO`
to bypass coherence. The MDS grants `BUFFER`/`CACHE` only when a single
client has the file open for write (the "loner"), and revokes them when
another client opens it, forcing a flush.

`Client::_read` (`src/client/Client.cc:11552`):

```cpp
retry:
  if (f->mode & CEPH_FILE_MODE_LAZY)
    want = CEPH_CAP_FILE_CACHE | CEPH_CAP_FILE_LAZYIO;
  else
    want = CEPH_CAP_FILE_CACHE;
  {
    auto r = get_caps(f, CEPH_CAP_FILE_RD, want, &have, -1);
    ...
  }
  ...
  if (!conf->client_debug_force_sync_read &&
      conf->client_oc &&
      (have & (CEPH_CAP_FILE_CACHE | CEPH_CAP_FILE_LAZYIO))) {
    // CASE 1 - blocking or non-blocking caller with the client holding Fc caps
    if (f->flags & O_RSYNC) {
      _flush_range(in, offset, size);
    }
    rc = _read_async(f, offset, size, bl, iofinish.get());      // via ObjectCacher
    ...
  } else if (onfinish) {
    // CASE 2 - non-blocking caller with sync read from the OSD
    ...
  }
```

The sync read case is `Filer::read_trunc` (`:11449`) straight to the
Objecter, bypassing any cache.

Writes follow the same split (`src/client/Client.cc:12618`):

```cpp
int Client::WriteEncMgr_Buffered::do_write()
{
  // do buffered write
  if (!in->oset.dirty_or_tx)
    clnt->get_cap_ref(in, CEPH_CAP_FILE_CACHE | CEPH_CAP_FILE_BUFFER);

  clnt->get_cap_ref(in, CEPH_CAP_FILE_BUFFER);

  // async, caching, non-blocking.
  r = clnt->objectcacher->file_write(&in->oset, &in->layout,
                                     in->snaprealm->get_snap_context(),
                                     offset, size, *pbl, ceph::real_clock::now(),
                                     0, iofinish,
                                     !async
                                     ? clnt->objectcacher->CFG_block_writes_upfront()
                                     : false);
  return r;
}

int Client::WriteEncMgr_NotBuffered::do_write()
{
  clnt->get_cap_ref(in, CEPH_CAP_FILE_BUFFER);

  clnt->filer->write_trunc(in->ino, &in->layout, in->snaprealm->get_snap_context(),
                           offset, size, *pbl, ceph::real_clock::now(), 0,
                           in->truncate_size, in->truncate_seq,
                           iofinish);
  return 0;
}
```

`_write` (`:12664`) chooses between them based on `have & CEPH_CAP_FILE_BUFFER`,
`client_oc` (default `true`) and `O_SYNC`/`O_DIRECT`-style flags. Either way
the client also calls `mark_caps_dirty(in, CEPH_CAP_FILE_WR)` after
updating `in->size` and `in->mtime` locally; the new size reaches the MDS
later in a cap flush (§5.4).

### 5.2 Buffered path: ObjectCacher

`ObjectCacher::file_write` (`src/osdc/ObjectCacher.h:726`) stripes the
range and stores the bytes in per-object `BufferHead`s:

```cpp
  int file_write(ObjectSet *oset, file_layout_t *layout,
		 const SnapContext& snapc, loff_t offset, uint64_t len,
		 ceph::buffer::list& bl, ceph::real_time mtime, int flags,
		 Context *onfreespace, bool block_writes_upfront) {
    OSDWrite *wr = prepare_write(snapc, bl, mtime, flags, 0);
    Striper::file_to_extents(cct, oset->ino, layout, offset, len,
			     oset->truncate_size, wr->extents);
    return writex(wr, oset, onfreespace, nullptr, block_writes_upfront);
  }
```

`writex` (`ObjectCacher.cc:1749`) copies data into buffer heads, marks
them dirty and records the snap context each buffer was written under.
The `flusher_entry` thread (`:1969`) writes dirty buffers back when total
dirty exceeds `client_oc_target_dirty` (`client_oc_max_dirty = 100 MiB`
hard limit, `client_oc_size = 200 MiB` cache) or when a buffer is older
than `client_oc_max_dirty_age` (5 s):

```cpp
    loff_t actual = get_stat_dirty() + get_stat_dirty_waiting();
    if (actual > 0 && (uint64_t) actual > target_dirty) {
      // flush some dirty pages
      flush(&trace, actual - target_dirty);
    } else {
      // check tail of lru for old dirty items
      ceph::real_time cutoff = ceph::real_clock::now();
      cutoff -= max_dirty_age;
      BufferHead *bh = 0;
      while ((bh = static_cast<BufferHead*>(bh_lru_dirty.lru_get_next_expire())) != 0 &&
	     bh->last_write <= cutoff && max > 0) {
	  bh_write(bh, trace);
```

Each buffer head becomes one RADOS write (`bh_write`, `:1141`):

```cpp
  ceph_tid_t tid = writeback_handler.write(bh->ob->get_oid(),
					   bh->ob->get_oloc(),
					   bh->start(), bh->length(),
					   bh->snapc, bh->bl, bh->last_write,
					   bh->ob->truncate_size,
					   bh->ob->truncate_seq,
					   bh->journal_tid, trace, oncommit);
```

The client's `ObjecterWriteback` (`src/client/ObjecterWriteback.h`) turns
that into `Objecter::write_trunc`, which is a `CEPH_OSD_OP_WRITE` with
`truncate_seq`/`truncate_size` attached. Those two fields are how CephFS
keeps a concurrent truncate and a delayed write from resurrecting data:
the OSD checks them in `PrimaryLogPG::do_osd_ops` (see the disk-internals
doc, the `CEPH_OSD_OP_WRITE` case) and clips or drops a write whose
`truncate_seq` is stale.

### 5.3 Unbuffered path

Without `BUFFER` caps (shared writers, `O_SYNC`, `client_oc = false`), the
client goes `Filer::write_trunc` → `Objecter::sg_write_trunc` and the
`write()` syscall returns only when every OSD involved has committed. Reads
likewise go `Filer::read_trunc` → `Objecter` with no client cache at all.

### 5.4 Size and mtime: cap flushes

Data goes to the OSDs but the file *size* is metadata owned by the MDS. A
client that extends a file updates `in->size` locally (allowed up to the
`max_size` the MDS granted with the write cap) and later sends an
`MClientCaps` flush (`Client::send_cap`, `src/client/Client.cc:4047`):

```cpp
  m->size = in->size;
  m->max_size = in->max_size;
  m->truncate_seq = in->truncate_seq;
  m->truncate_size = in->truncate_size;
  m->mtime = in->mtime;
  m->atime = in->atime;
  m->ctime = in->ctime;
  ...
  m->change_attr = in->change_attr;
```

On the MDS, `Locker::_do_cap_update` (`src/mds/Locker.cc:4067`) applies
the new size and, if it changed or the client needs a bigger `max_size`,
journals an `EUpdate`/`EOpen` for the inode:

```cpp
  LogEvent *le;
  EMetaBlob *metablob;
  if (in->is_any_caps_wanted() && in->last == CEPH_NOSNAP) {
    EOpen *eo = new EOpen(mds->mdlog);
    eo->add_ino(in->ino());
    metablob = &eo->metablob;
    le = eo;
  } else {
    EUpdate *eu = new EUpdate(mds->mdlog, "check_inode_max_size");
    metablob = &eu->metablob;
    le = eu;
  }

  mdcache->predirty_journal_parents(mut, metablob, in, 0, PREDIRTY_PRIMARY);
  ...
  mds->mdlog->submit_entry(le, new C_Locker_FileUpdate_finish(this, in, mut,
      UPDATE_SHAREMAX, ref_t<MClientCaps>()));
```

`fsync()` (`Client::_fsync`) therefore does three things in order: flush
dirty buffers to OSDs and wait for commit (`_flush` → `objectcacher`), send
the cap flush, and wait for the MDS to ack it (`wait_sync_caps`) after the
MDS has journaled it. Only then are both the bytes and the size durable.

### 5.5 Snapshots

CephFS snapshots are per-directory (`mkdir .snap/name`). The MDS assigns a
snapid and the client threads the directory's `SnapContext` into every
data write (`in->snaprealm->get_snap_context()` in the excerpts above).
As with RBD, the OSD does the copy-on-write: a write whose snapc is newer
than the object's snapset clones the object first. Metadata snapshots are
handled by the MDS keeping old inode versions (`old_inodes`) in the
dirfrag.

---

## 6. Deleting a file

`unlink()` only moves the dentry into the MDS's **stray** directory
(`MDS_INO_STRAY(rank, i)`) and journals that. Data objects are removed
asynchronously by the **purge queue**, itself a `Journaler` on inode
`0x500 + rank` (`MDS_INO_PURGE_QUEUE`) so that a crash mid-purge resumes.
`StrayManager::purge` (`src/mds/StrayManager.cc:100`) enqueues a
`PurgeItem` with the inode, size, layout and snapc, and
`PurgeQueue::_execute_item` (`src/mds/PurgeQueue.cc:631`) computes the
object count and removes them through `Filer::purge_range`
(`src/osdc/Filer.cc:424`):

```cpp
int Filer::purge_range(inodeno_t ino, const file_layout_t *layout, const SnapContext& snapc,
		       uint64_t first_obj, uint64_t num_obj, ceph::real_time mtime,
		       int flags, Context *oncommit)
{
  // single object?  easy!
  if (num_obj == 1) {
    object_t oid = file_object_t(ino, first_obj);
    object_locator_t oloc = OSDMap::file_to_object_locator(*layout);
    objecter->remove(oid, oloc, snapc, mtime, flags, oncommit);
    return 0;
  }

  PurgeRange *pr = new PurgeRange(ino, *layout, snapc, first_obj,
				  num_obj, mtime, flags, oncommit);
  _do_purge_range(pr, 0, 0);
  return 0;
}
```

`_do_purge_range` keeps `filer_max_purge_ops` removes in flight. A 1 TiB
file is 262,144 `remove` ops, which is why large deletes show up as a slow
background drain in `ceph fs status` (`purge_queue` counters) rather than
as a slow `rm`.

---

## 7. What the OSD sees

| Client / MDS action                | RADOS op(s)                                                                    | Pool      |
|------------------------------------|--------------------------------------------------------------------------------|-----------|
| `write()` with buffer caps         | delayed `write` per 4 MiB object (`<ino>.<n>`) with snapc, truncate_seq/size  | data      |
| `write()` without buffer caps      | immediate `write` per object; syscall waits for commit                         | data      |
| `read()` with cache caps           | `read` on cache miss, readahead per `client_readahead_*`                        | data      |
| `fsync()`                          | outstanding data writes + cap flush → MDS journal append                       | data + metadata |
| `mkdir`/`rename`/`setattr`         | none on the client's path; MDS appends to `200.<n>` (rank 0)                   | metadata  |
| MDS journal segment expiry         | `omap_set`/`omap_rm_keys` + `omap_set_header` on `<dirino>.<frag>`; `setxattr parent` on `<ino>.00000000` | metadata + data |
| `unlink()`                         | MDS journal append; later `remove` per data object from the purge queue        | data      |
| Directory snapshot                 | MDS journal; subsequent data writes clone objects on the OSD                   | data      |

Everything in the data pool is plain object reads and writes and follows
the BlueStore path in `ceph-disk-io-internals.md`. Everything in the
metadata pool is either sequential appends (journals) or omap mutations
(dirfrags), which on the OSD are RocksDB writes with no data-device I/O.

---

## 8. File index

| Concern                                  | File                                                  |
|------------------------------------------|-------------------------------------------------------|
| Object naming                            | `src/include/object.cc`, `src/mds/CInode.cc:1260`     |
| Layouts and striping                     | `src/include/fs_types.h`, `src/osdc/Striper.cc`, `src/osdc/Filer.cc` |
| User-space client                        | `src/client/Client.cc`, `src/libcephfs.cc`, `src/ceph_fuse.cc`, `src/client/fuse_ll.cc` |
| Client write-back cache                  | `src/osdc/ObjectCacher.cc`, `src/client/ObjecterWriteback.h` |
| Capabilities (client side)               | `src/client/Client.cc` (`get_caps`, `send_cap`, `check_caps`) |
| Capabilities (MDS side)                  | `src/mds/Locker.cc`, `src/mds/Capability.h`           |
| Request handling, early reply            | `src/mds/Server.cc`                                   |
| In-memory metadata                       | `src/mds/MDCache.cc`, `CInode.cc`, `CDir.cc`, `CDentry.cc` |
| Dirfrag persistence (omap)               | `src/mds/CDir.cc` (`_omap_commit_ops`, `_omap_fetch`) |
| Backtraces                               | `src/mds/inode_backtrace.h`, `src/mds/CInode.cc` (`store_backtrace`) |
| Journal                                  | `src/mds/MDLog.cc`, `src/mds/journal.cc`, `src/mds/LogEvent.h`, `src/mds/events/`, `src/osdc/Journaler.cc` |
| Reserved inode ranges                    | `src/mds/mdstypes.h`                                  |
| Deletion                                 | `src/mds/StrayManager.cc`, `src/mds/PurgeQueue.cc`    |
| Snapshots                                | `src/mds/SnapRealm.cc`, `SnapServer.cc`, `src/client/ClientSnapRealm.cc` |
| Kernel mount helper / Windows client     | `src/mount/`, `src/dokan/`                            |
| Tunables                                 | `src/common/options/mds.yaml.in`, `mds-client.yaml.in` |
