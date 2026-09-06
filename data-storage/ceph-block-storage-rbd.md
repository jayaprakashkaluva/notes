# Ceph Block Storage (RBD): How a Block Device Becomes RADOS Objects

Companion to `ceph-disk-io-internals.md`. That document covers what happens
once a request reaches an OSD. This one covers the layer above it for **RBD
(RADOS Block Device)**: how `librbd` turns a virtual disk into RADOS objects,
what metadata it keeps, and how a block read or write travels from the guest
down to `librados`. Source references are to the tree at `C:\opensource\cpeh`
(commit `9438aabf3f7`).

---

## 1. The mental model

An RBD image is a **sparse array of fixed-size RADOS objects**, plus a handful
of metadata objects. There is no block-map, no allocation table and no
journal in the data path by default. Object *N* holds image bytes
`[N * object_size, (N+1) * object_size)`. If that region has never been
written, the object simply does not exist and reads of it return zeros.

```
 guest / application
   │  read(off, len) / write(off, len)
   ▼
 librbd  (src/librbd)                     ─ or ─  kernel rbd driver (src/krbd.cc maps it)
   │  ImageDispatcher  (QoS, exclusive lock, journal, writeback cache…)
   │  ImageRequest: image extents → object extents  (Striper)
   │  ObjectDispatcher (cache, crypto, journal, parent-cache, core)
   │  ObjectRequest:  one neorados WriteOp/ReadOp per object
   ▼
 librados / neorados  →  Objecter → OSD                      (see ceph-disk-io-internals.md §2 onward)

 pool "rbd":
   rbd_directory                       omap: name ↔ id for every image in the pool
   rbd_id.<name>                       tiny object: the image id
   rbd_header.<id>                     omap: size, order, features, snapshots, parent, lock…
   rbd_object_map.<id>[.<snapid>]      bit vector: 2 bits per data object
   rbd_data.<id>.0000000000000000      object 0 of the image (object_size bytes, default 4 MiB)
   rbd_data.<id>.0000000000000001
   …
   journal.<id> / journal_data.…        only if the journaling feature is on
```

`src/include/rbd_types.h:15`

```cpp
/* New-style rbd image 'foo' consists of objects
 *   rbd_id.foo              - id of image
 *   rbd_header.<id>         - image metadata
 *   rbd_object_map.<id>     - optional image object map
 *   rbd_data.<id>.00000000
 *   rbd_data.<id>.00000001
 *   ...                     - data
 */

#define RBD_HEADER_PREFIX      "rbd_header."
#define RBD_OBJECT_MAP_PREFIX  "rbd_object_map."
#define RBD_DATA_PREFIX        "rbd_data."
#define RBD_ID_PREFIX          "rbd_id."
```

---

## 2. Image metadata: the header object

The header is an omap-only object manipulated exclusively through the `rbd`
object class (`src/cls/rbd/cls_rbd.cc`), which runs *inside the OSD*. That
gives every header mutation (resize, snapshot create, lock acquire) the
atomicity of a single RADOS op without the client having to read-modify-write.

Creation writes the initial keys (`cls/rbd/cls_rbd.cc:769`, `create`):

```cpp
  map<string, bufferlist> omap_vals;
  omap_vals["size"] = sizebl;
  omap_vals["order"] = orderbl;
  omap_vals["features"] = featuresbl;
  omap_vals["object_prefix"] = object_prefixbl;
  omap_vals["snap_seq"] = snap_seqbl;
  omap_vals["create_timestamp"] = timestampbl;
```

`object_prefix` is the string every data object name starts with; `order` is
`log2(object_size)` (default `rbd_default_order = 22`, so 4 MiB objects).
Snapshots are further keys under `snapshot_<id>`
(`RBD_SNAP_KEY_PREFIX`, `cls_rbd.cc:79`), parent linkage under `parent`,
image metadata under `metadata_*`, and the exclusive lock lives in the
object's lock state.

`librbd::image::CreateRequest` (`src/librbd/image/CreateRequest.cc`) walks
the creation sequence as a chain of async RADOS ops: reserve the id in
`rbd_directory` (`add_image_to_directory`), write `rbd_id.<name>`
(`create_id_object`), then create the header with an exclusive-create guard:

`src/librbd/image/CreateRequest.cc:430`

```cpp
void CreateRequest<I>::create_image() {
  std::ostringstream oss;
  oss << RBD_DATA_PREFIX;
  if (m_data_pool_id != -1) {
    oss << stringify(m_io_ctx.get_id()) << ".";     // separate data pool: prefix carries pool id
  }
  oss << m_image_id;
  ...
  librados::ObjectWriteOperation op;
  op.create(true);
  cls_client::create_image(&op, m_size, m_order, m_features, oss.str(),
                           m_data_pool_id);
  ...
  int r = m_io_ctx.aio_operate(m_header_obj, comp, &op);
```

Note the **data pool** option: metadata stays in the pool the image was
created in, data objects can live in a different (typically erasure-coded)
pool. The `format_string` used to name data objects is derived when the
image is opened (`src/librbd/ImageCtx.cc:236`):

```cpp
    layout.stripe_unit = stripe_unit;
    layout.stripe_count = stripe_count;
    layout.object_size = 1ull << order;
    layout.pool_id = pool_id;  // FIXME: pool id overflow?

    delete[] format_string;
    size_t len = object_prefix.length() + 16;
    format_string = new char[len];
    if (old_format) {
      snprintf(format_string, len, "%s.%%012llx", object_prefix.c_str());
    } else {
      snprintf(format_string, len, "%s.%%016llx", object_prefix.c_str());
    }
```

and applied per object (`src/librbd/Utils.h:156`):

```cpp
std::string data_object_name(I* image_ctx, uint64_t object_no) {
  char buf[RBD_MAX_OBJ_NAME_SIZE];
  size_t length = snprintf(buf, RBD_MAX_OBJ_NAME_SIZE,
                           image_ctx->format_string, object_no);
  ...
```

So object 5 of an image with id `1a2b3c` is `rbd_data.1a2b3c.0000000000000005`.
Because the name embeds the object number, no lookup is needed to go from
an image offset to an object name.

---

## 3. Mapping bytes to objects: the Striper

RBD shares the striping code with CephFS. `file_layout_t`
(`src/include/fs_types.h:107`) describes it:

```cpp
struct file_layout_t {
  uint32_t stripe_unit;   ///< stripe unit, in bytes,
  uint32_t stripe_count;  ///< over this many objects
  uint32_t object_size;   ///< until objects are this big
  int64_t pool_id;        ///< rados pool id
  std::string pool_ns;
  ...
  static file_layout_t get_default() {
    return file_layout_t(1<<22, 1, 1<<22);    // 4 MiB objects, no striping
  }
```

With the default `stripe_count = 1`, the mapping is trivial:
`object_no = offset / object_size`. With the `STRIPINGV2` feature an image can
spread consecutive `stripe_unit` chunks across `stripe_count` objects to
parallelize sequential I/O. The general algorithm is
`Striper::file_to_extents` (`src/osdc/Striper.cc:182`):

```cpp
  __u32 object_size = layout->object_size;
  __u32 su = layout->stripe_unit;
  __u32 stripe_count = layout->stripe_count;
  if (stripe_count == 1) {
    su = object_size;
  }
  uint64_t stripes_per_object = object_size / su;

  uint64_t cur = offset;
  uint64_t left = len;
  while (left > 0) {
    // layout into objects
    uint64_t blockno = cur / su; // which block
    // which horizontal stripe (Y)
    uint64_t stripeno = blockno / stripe_count;
    // which object in the object set (X)
    uint64_t stripepos = blockno % stripe_count;
    // which object set
    uint64_t objectsetno = stripeno / stripes_per_object;
    // object id
    uint64_t objectno = objectsetno * stripe_count + stripepos;

    // map range into object
    uint64_t block_start = (stripeno % stripes_per_object) * su;
    uint64_t block_off = cur % su;
    uint64_t max = su - block_off;

    uint64_t x_offset = block_start + block_off;
    uint64_t x_len = (left > max) ? max : left;
    ...
    // one ObjectExtent per object, with buffer_extents telling which parts
    // of the caller's buffer map to it
```

librbd wraps it in `area_to_object_extents` (`src/librbd/io/Utils.cc:204`),
which also shifts the offset when an encrypted image reserves a header area
at the front of the image:

```cpp
void area_to_object_extents(I* image_ctx, uint64_t offset, uint64_t length,
                            ImageArea area, uint64_t buffer_offset,
                            striper::LightweightObjectExtents* object_extents) {
  offset = area_to_raw_offset(*image_ctx, offset, area);
  Striper::file_to_extents(image_ctx->cct, &image_ctx->layout, offset, length,
                           0, buffer_offset, object_extents);
}
```

---

## 4. The dispatch pipeline

Every I/O passes through two ordered chains of pluggable layers. Each layer
may consume the request, transform it, or pass it on.

`src/librbd/io/Types.h:58`

```cpp
  IMAGE_DISPATCH_LAYER_QUEUE,            // async hand-off from the API thread
  IMAGE_DISPATCH_LAYER_QOS,              // rbd_qos_* token buckets (IOPS / bandwidth)
  IMAGE_DISPATCH_LAYER_EXCLUSIVE_LOCK,   // block writes until this client owns the lock
  IMAGE_DISPATCH_LAYER_REFRESH,          // re-read header if it changed (resize, snap)
  IMAGE_DISPATCH_LAYER_MIGRATION,        // live migration redirection
  IMAGE_DISPATCH_LAYER_JOURNAL,          // journaling feature
  IMAGE_DISPATCH_LAYER_WRITE_BLOCK,      // quiesce for snapshots
  IMAGE_DISPATCH_LAYER_WRITEBACK_CACHE,  // persistent write-log (pwl) cache
  IMAGE_DISPATCH_LAYER_CORE,             // ImageRequest: split into object requests
  ...
  OBJECT_DISPATCH_LAYER_CACHE,           // in-memory ObjectCacher (rbd_cache)
  OBJECT_DISPATCH_LAYER_CRYPTO,          // LUKS encryption
  OBJECT_DISPATCH_LAYER_JOURNAL,         // wait for journal commit before object write
  OBJECT_DISPATCH_LAYER_PARENT_CACHE,    // immutable-object cache daemon for clones
  OBJECT_DISPATCH_LAYER_SCHEDULER,       // merge adjacent small writes
  OBJECT_DISPATCH_LAYER_CORE,            // ObjectRequest: talk to RADOS
```

The two layers with the biggest effect on what reaches the OSD are the
**exclusive lock** (only one writer, so object-map and journal updates are
safe) and the **client-side cache**.

### 4.1 Client-side caching

`rbd_cache = true` and `rbd_cache_policy = writearound` by default
(`src/common/options/rbd.yaml.in:100-113`). The cache is the same
`ObjectCacher` used by the CephFS client, plugged in as an object-dispatch
layer (`src/librbd/cache/ObjectCacherObjectDispatch.cc`). Two behaviours
matter operationally:

- **Writethrough until flush** (`rbd_cache_writethrough_until_flush = true`):
  the cache stays writethrough until the guest issues its first flush,
  proving the guest honours barriers. Only then does `max_dirty` become
  non-zero (`ObjectCacherObjectDispatch.cc:445`):

  ```cpp
    if (m_writethrough_until_flush && m_max_dirty > 0) {
      m_object_cacher->set_max_dirty(m_max_dirty);
  ```

- **Writeback** buffers up to `rbd_cache_max_dirty` (24 MiB) and a flusher
  thread writes dirty buffers out after `rbd_cache_max_dirty_age` (1 s) or
  under pressure. A dirty buffer is written by
  `ObjectCacherWriteback::write` (`src/librbd/cache/ObjectCacherWriteback.cc:185`),
  which creates an ordinary `ObjectDispatchSpec::create_write` so it re-enters
  the chain *below* the cache layer.

`writearound` (the default) caches reads but sends writes straight through
and completes them only on RADOS commit. A guest fsync therefore always
means "on OSD disks", never "in librbd memory".

The optional **persistent write-log cache** (`rbd_persistent_cache_mode`,
default `disabled`) is a different beast: an image-level layer backed by
local SSD or PMEM that acks writes once they are durable locally and
replays them to RADOS in order.

---

## 5. Write path

### 5.1 Image level

`AbstractImageWriteRequest::send_request` (`src/librbd/io/ImageRequest.cc:426`)
is where a block write becomes per-object work:

```cpp
  uint64_t clip_len = 0;
  LightweightObjectExtents object_extents;
  for (auto &extent : this->m_image_extents) {
    // map to object extents
    io::util::area_to_object_extents(&image_ctx, extent.first, extent.second,
                                     this->m_image_area, clip_len,
                                     &object_extents);
    clip_len += extent.second;
  }

  int ret = prune_object_extents(&object_extents);     // discards: align to granularity
  ...
  aio_comp->set_request_count(object_extents.size());
  if (!object_extents.empty()) {
    uint64_t journal_tid = 0;
    if (journaling) {
      journal_tid = append_journal_event();            // journaling feature only
    }

    // it's very important that IOContext is captured here instead of
    // e.g. at the API layer so that an up-to-date snap context is used
    // when owning the exclusive lock
    send_object_requests(object_extents, image_ctx.get_data_io_context(),
                         journal_tid);
  }
```

The `IOContext` carries the **snap context** (the image's current snapshot
sequence). It is what makes RBD snapshots work: the OSD compares the
write's `snapc` against the object's on-disk snapset and clones the object
if a snapshot was taken since it was last written (the OSD-side
copy-on-write described in `ceph-disk-io-internals.md` §3). librbd never
copies anything for a snapshot; it only bumps `snap_seq` in the header.

### 5.2 Object level

`AbstractObjectWriteRequest` (`src/librbd/io/ObjectRequest.cc:412`) runs a
short state machine per object: consult the object map, maybe copy-up from
the parent, write, update the object map.

```cpp
void AbstractObjectWriteRequest<I>::send() {
  {
    std::shared_lock image_lock{image_ctx->image_lock};
    if (image_ctx->object_map == nullptr) {
      m_object_may_exist = true;
    } else {
      // should have been flushed prior to releasing lock
      ceph_assert(image_ctx->exclusive_lock->is_lock_owner());
      m_object_may_exist = image_ctx->object_map->object_may_exist(
        this->m_object_no);
    }
  }

  if (!m_object_may_exist && is_no_op_for_nonexistent_object()) {
    // e.g. a discard of an object that does not exist
    this->async_finish(0);
    return;
  }

  pre_write_object_map_update();
}
```

`pre_write_object_map_update` (`:440`) flips the object's bits to
`OBJECT_EXISTS` *before* the data write if they are not already set, so a
crash between the two leaves the map conservative (says "exists" for an
object that may not). The states (`src/include/rbd/object_map_types.h`):

```cpp
static const uint8_t OBJECT_NONEXISTENT  = 0;
static const uint8_t OBJECT_EXISTS       = 1;
static const uint8_t OBJECT_PENDING      = 2;   // discard in progress
static const uint8_t OBJECT_EXISTS_CLEAN = 3;   // exists, unchanged since last snapshot
```

The map itself is a `BitVector<2>` stored in `rbd_object_map.<id>`
(`rbd_object_map.<id>.<snapid>` for snapshots) and updated through
`cls_client::object_map_update`, which the OSD applies with `cls_cxx_read2` /
`cls_cxx_write` of the affected bytes only (`cls/rbd/cls_rbd.cc:3477`).
Its purpose is to skip RADOS round-trips: reads of non-existent objects
return zeros locally, discards of non-existent objects are no-ops, and
`rbd du` / `rbd diff` need no object listing.

The write itself (`:489`):

```cpp
void AbstractObjectWriteRequest<I>::write_object() {
  neorados::WriteOp write_op;
  if (m_copyup_enabled) {
    if (m_guarding_migration_write) {
      cls_client::assert_snapc_seq(
        &write_op, snap_seq, cls::rbd::ASSERT_SNAPC_SEQ_LE_SNAPSET_SEQ);
    } else {
      write_op.assert_exists();        // clone: fail with ENOENT if object absent → copyup
    }
  }

  add_write_hint(&write_op);
  add_write_ops(&write_op);
  ceph_assert(write_op.size() != 0);

  image_ctx->rados_api.execute(
    {data_object_name(this->m_ictx, this->m_object_no)},
    *this->m_io_context, std::move(write_op),
    librbd::asio::util::get_callback_adapter(
      [this](int r) { handle_write_object(r); }), nullptr, ...);
}
```

`add_write_ops` chooses the RADOS op (`:664`):

```cpp
void ObjectWriteRequest<I>::add_write_ops(neorados::WriteOp* wr) {
  if (this->m_full_object) {
    wr->write_full(bufferlist{m_write_data});
  } else {
    wr->write(this->m_object_off, bufferlist{m_write_data});
  }
  util::apply_op_flags(m_op_flags, 0U, wr);
}

void ObjectDiscardRequest<I>::add_write_ops(neorados::WriteOp* wr) {
  switch (m_discard_action) {
  case DISCARD_ACTION_REMOVE:            wr->remove(); break;
  case DISCARD_ACTION_REMOVE_TRUNCATE:   wr->create(false); // fall through
  case DISCARD_ACTION_TRUNCATE:          wr->truncate(this->m_object_off); break;
  case DISCARD_ACTION_ZERO:              wr->zero(this->m_object_off, this->m_object_len); break;
  }
}
```

A full-object write uses `write_full` so the OSD can drop the old extents
wholesale. A discard that covers an entire object *removes* the object,
which is how a TRIM in the guest returns space to the cluster; partial
discards become `zero` ops, which BlueStore turns into hole-punches in the
extent map. `rbd_discard_granularity_bytes` (`ImageRequest.cc:590`) rounds
small discards away so they do not fragment objects.

The allocation hint (`ObjectRequest.cc:136`) tells BlueStore the expected
object and write sizes so it can pick blob and checksum sizes
(`_choose_write_options` in the disk-internals doc):

```cpp
  if (image_ctx.enable_alloc_hint) {
    wr->set_alloc_hint(image_ctx.get_object_size(),
                       image_ctx.get_object_size(),
                       alloc_hint_flags);
```

### 5.3 Clones and copy-up

A cloned image starts with *no* data objects. Reads fall through to the
parent snapshot; the first write to any object must first materialize it.
`handle_write_object` (`:522`) turns the `-ENOENT` from `assert_exists` into a
`copyup()`:

```cpp
  r = filter_write_result(r);
  if (r == -ENOENT) {
    if (m_copyup_enabled) {
      copyup();
      return;
    }
  }
```

`CopyupRequest::read_from_parent` (`src/librbd/io/CopyupRequest.cc:164`)
issues an ordinary image read against the parent `ImageCtx` for the whole
object's extent, then `copyup()` (`:400`) sends the parent data plus the
pending write in one RADOS op:

```cpp
  if (m_image_ctx->enable_sparse_copyup) {
    cls_client::sparse_copyup(op, m_copyup_extent_map, m_copyup_data);
  } else {
    ...
    cls_client::copyup(op, thick_bl);
  }
```

The class method is idempotent by construction (`cls/rbd/cls_rbd.cc:2703`):

```cpp
int copyup(cls_method_context_t hctx, bufferlist *in, bufferlist *out)
{
  // check for existence; if child object exists, just return success
  if (cls_cxx_stat(hctx, NULL, NULL) == 0)
    return 0;
  return cls_cxx_write(hctx, 0, in->length(), in);
}
```

so two racing copy-ups of the same object cannot corrupt it. If the child
has snapshots, the copyup is issued as its own op *before* the write so the
snapshot sees parent data rather than the new write (`deep_copyup`,
`CopyupRequest.cc:419`).

### 5.4 Journaling (optional)

With the `journaling` feature (required for RBD mirroring), the
image-dispatch journal layer appends an event describing the write to the
image journal *before* the object write is allowed to proceed, and the
object-dispatch journal layer waits for that append to commit
(`src/librbd/journal/ObjectDispatch.cc:108`):

```cpp
  if (*journal_tid == 0) {
    // non-journaled IO
    return false;
  }
  ...
  *on_finish = new C_CommitIOEvent<I>(m_image_ctx, m_journal, object_no,
                                      object_off, data.length(), *journal_tid,
                                      *object_dispatch_flags, *on_finish);
  ...
  wait_or_flush_event(*journal_tid, *object_dispatch_flags, on_dispatched);
```

The journal is a separate set of RADOS objects (`journal.<id>` header,
`journal_data.<pool>.<id>.<n>` chunks) written with append ops by
`journal::ObjectRecorder` (`src/journal/ObjectRecorder.cc:269`). Every
journaled write therefore costs two RADOS writes. This is a write-ahead log
for *replication*, not for local consistency; a plain RBD image without the
feature is already crash-consistent at object granularity because each
object write is atomic on the OSD.

---

## 6. Read path

`ImageReadRequest::send_request` (`src/librbd/io/ImageRequest.cc:377`) maps
the image extents exactly as writes do and issues one read per object, with
optional readahead when the cache is on:

```cpp
  if (this->m_image_area == ImageArea::DATA &&
      image_ctx.cache && image_ctx.readahead_max_bytes > 0 &&
      !(m_op_flags & LIBRADOS_OP_FLAG_FADVISE_RANDOM)) {
    readahead(get_image_ctx(&image_ctx), image_extents, m_io_context);
  }
  ...
  for (auto &oe : object_extents) {
    auto req_comp = new io::ReadResult::C_ObjectReadRequest(
      aio_comp, {{oe.offset, oe.length, std::move(oe.buffer_extents)}});
    auto req = ObjectDispatchSpec::create_read(
      &image_ctx, OBJECT_DISPATCH_LAYER_NONE, oe.object_no,
      &req_comp->extents, m_io_context, m_op_flags, m_read_flags,
      this->m_trace, nullptr, req_comp);
    req->send();
  }
```

`ObjectReadRequest::read_object` (`src/librbd/io/ObjectRequest.cc:223`):

```cpp
  auto read_snap_id = this->m_io_context->get_read_snap();
  if (read_snap_id == image_ctx->snap_id &&
      image_ctx->object_map != nullptr &&
      !image_ctx->object_map->object_may_exist(this->m_object_no)) {
    image_ctx->asio_engine->post([this]() { read_parent(); });   // skip RADOS entirely
    return;
  }
  ...
  neorados::ReadOp read_op;
  for (auto& extent: *this->m_extents) {
    if (extent.length >= image_ctx->sparse_read_threshold_bytes) {
      read_op.sparse_read(extent.offset, extent.length, &extent.bl,
                          &extent.extent_map);
    } else {
      read_op.read(extent.offset, extent.length, &extent.bl);
    }
  }
  ...
  image_ctx->rados_api.execute(
    {data_object_name(this->m_ictx, this->m_object_no)},
    *this->m_io_context, std::move(read_op), ...);
```

Three consequences:

- **Snapshot reads** are just reads with a different `snap` in the
  `IOContext`; the OSD serves the right clone object.
- **Sparse reads** (`sparse_read`) return an extent map alongside the data
  so librbd can avoid shipping and zero-filling holes in large reads; it is
  what makes `rbd export` and mirroring of thin images efficient.
- **`-ENOENT` is not an error.** `handle_read_object` (`:264`) treats it as
  "read from parent or zero-fill":

```cpp
  if (r == -ENOENT) {
    read_parent();
    return;
  }
```

`read_parent` → `io::util::read_parent` re-issues the read against the
parent image at the parent snapshot. With `rbd_clone_copy_on_read` the data
is then also written into the child via a `CopyupRequest` (`:307`), so the
next read is local.

---

## 7. What the OSD sees

From the OSD's point of view an RBD workload is:

| Guest action              | RADOS op(s) on `rbd_data.<id>.<n>`                        | BlueStore effect (see disk-internals doc)                     |
|---------------------------|------------------------------------------------------------|---------------------------------------------------------------|
| 4 KiB write               | `write(off, 4096)` with snapc, alloc hint                  | small write: deferred on HDD, direct on SSD; onode+extent update |
| 1 MiB sequential write    | `write` per touched object, coalesced by ObjectCacher/scheduler | new blobs allocated, direct aio                              |
| Full 4 MiB object write   | `write_full`                                               | old extents released wholesale                                |
| TRIM of a whole object    | `remove`                                                   | onode deleted, extents freed, discard issued                  |
| TRIM of part of an object | `zero(off, len)` (if ≥ discard granularity)                | hole punched in extent map                                    |
| Read of untouched region  | none (object map) or `read` → `-ENOENT`                    | none                                                          |
| Snapshot create           | header omap update (`snapshot_add`); no data I/O           | subsequent writes trigger OSD-side clone of the object        |
| Clone first write         | `assert_exists`+`write` → ENOENT → `copyup`+`write`        | new object of up to object_size bytes                         |

There is no image-level journal or log unless the journaling or
persistent-write-log features are enabled, so RBD's durability is exactly
RADOS durability: a write is acknowledged when the primary and all replicas
(or all EC shards) have committed the transaction described in the
disk-internals document.

---

## 8. File index

| Concern                                | File                                                      |
|----------------------------------------|-----------------------------------------------------------|
| Object naming constants                | `src/include/rbd_types.h`                                 |
| Image open, layout, format string      | `src/librbd/ImageCtx.cc`                                  |
| Image create/remove/clone              | `src/librbd/image/CreateRequest.cc`, `CloneRequest.cc`, `RemoveRequest.cc` |
| Header object class (OSD side)         | `src/cls/rbd/cls_rbd.cc`, client stubs `cls_rbd_client.cc` |
| Striping                               | `src/osdc/Striper.cc`, `src/librbd/io/Utils.cc`           |
| Dispatch layers                        | `src/librbd/io/ImageDispatcher.cc`, `ObjectDispatcher.cc`, `Types.h` |
| Image-level request splitting          | `src/librbd/io/ImageRequest.cc`                           |
| Per-object RADOS ops                   | `src/librbd/io/ObjectRequest.cc`                          |
| Copy-up for clones                     | `src/librbd/io/CopyupRequest.cc`                          |
| Object map                             | `src/librbd/ObjectMap.cc`, `src/librbd/object_map/`       |
| In-memory cache                        | `src/librbd/cache/ObjectCacherObjectDispatch.cc`, `src/osdc/ObjectCacher.cc` |
| Persistent write-log cache             | `src/librbd/cache/pwl/`                                   |
| Journaling                             | `src/librbd/Journal.cc`, `src/librbd/journal/`, `src/journal/` |
| Exclusive lock                         | `src/librbd/ExclusiveLock.cc`, `src/librbd/managed_lock/` |
| Kernel client helper                   | `src/krbd.cc`                                             |
| NVMe-oF gateway                        | `src/nvmeof/`                                             |
| Tunables                               | `src/common/options/rbd.yaml.in`                          |
