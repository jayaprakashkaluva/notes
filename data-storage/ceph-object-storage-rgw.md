# Ceph Object Storage (RGW): How an S3 Object Becomes RADOS Objects

Companion to `ceph-disk-io-internals.md` (OSD/BlueStore), `ceph-block-storage-rbd.md`
and `ceph-file-storage-cephfs.md`. This document covers the **RADOS Gateway
(RGW)**: how an S3/Swift object is split into a head object and tail
stripes, how the bucket index is maintained transactionally, and how a PUT
and a GET flow through the gateway to `librados`. Source references are to
`C:\opensource\cpeh` (commit `9438aabf3f7`); RGW's RADOS driver lives under
`src/rgw/driver/rados/`.

---

## 1. The mental model

RGW is a stateless HTTP frontend. All state is in RADOS, spread across
several pools per zone:

```
 S3 / Swift client
   │  HTTP PUT/GET/DELETE
   ▼
 radosgw  (src/rgw)
   │  RGWOp (rgw_op.cc)  →  SAL (rgw_sal_rados.cc)  →  RGWRados (driver/rados/rgw_rados.cc)
   │  put: data processor pipeline (rgw_putobj_processor.cc) → RadosWriter → librados aio
   │  get: manifest iteration (iterate_obj) → librados aio reads
   ▼
 RADOS pools (defaults for zone "default"):
   default.rgw.buckets.data     head objects + tail stripes            ("<marker>_<key>", "<marker>__shadow_…")
   default.rgw.buckets.index    bucket index shards, omap              (".dir.<bucket_id>.<shard>")
   default.rgw.buckets.non-ec   multipart upload metadata, appendable tails
   default.rgw.meta             users, bucket instances, bucket entrypoints (namespaced)
   default.rgw.log              garbage collection, data/metadata sync logs, lifecycle
   default.rgw.control          watch/notify objects for cache invalidation
```

The two objects that matter for a single S3 object are:

1. **The head object** in the data pool. It carries the first
   `rgw_max_chunk_size` bytes of data (4 MiB by default) *and* every
   attribute as RADOS xattrs: ETag, content type, ACL, user metadata, and
   crucially `user.rgw.manifest`, which describes where the rest of the
   data is.
2. **The bucket index entry**, an omap key on one of the bucket's index
   shard objects. It is what `ListObjects` reads and what accounts quota.

Tail data is in additional data-pool objects that are never looked up by
name from the outside; only the manifest knows them.

---

## 2. Object layout: head, manifest, tail stripes

### 2.1 The manifest

`RGWObjManifest` (`src/rgw/driver/rados/rgw_obj_manifest.h:199`) is stored
in the head object's `user.rgw.manifest` xattr:

```cpp
  bool explicit_objs{false}; /* really old manifest? */
  ...
  uint64_t obj_size{0};
  rgw_obj obj;               /* head object */
  uint64_t head_size{0};
  uint64_t max_head_size{0};
  std::string prefix;
  rgw_bucket_placement tail_placement; /* might be different than the original bucket, ... */
  std::map<uint64_t, RGWObjManifestRule> rules;
  std::string tail_instance; /* tail object's instance */
```

Modern manifests are *implicit*: instead of listing every tail object they
store a random `prefix` and a rule (`:134`):

```cpp
struct RGWObjManifestRule {
  uint32_t start_part_num;
  uint64_t start_ofs;
  uint64_t part_size; /* each part size, 0 if there's no part size, meaning it's unlimited */
  uint64_t stripe_max_size; /* underlying obj max size */
  std::string override_prefix;
```

and derive tail names arithmetically (`src/rgw/rgw_obj_manifest.cc`,
`get_implicit_location`):

```cpp
  if (!cur_part_id) {
    if (ofs < max_head_size) {
      location->set_placement_rule(head_placement_rule);
      *location = obj;                              // the head object itself
      return;
    } else {
      char buf[16];
      snprintf(buf, sizeof(buf), "%d", (int)cur_stripe);
      oid += buf;                                   // "<prefix><stripe>"
      ns = RGW_OBJ_NS_SHADOW;                       // → "…__shadow_<prefix><stripe>"
    }
  } else {
    char buf[32];
    if (cur_stripe == 0) {
      snprintf(buf, sizeof(buf), ".%d", (int)cur_part_id);
      oid += buf;                                   // "<prefix>.<part>"
      ns= RGW_OBJ_NS_MULTIPART;                     // → "…__multipart_<prefix>.<part>"
    } else {
      snprintf(buf, sizeof(buf), ".%d_%d", (int)cur_part_id, (int)cur_stripe);
      oid += buf;                                   // "<prefix>.<part>_<stripe>"
      ns = RGW_OBJ_NS_SHADOW;
    }
  }
  ...
  // Always overwrite instance with tail_instance
  // to get the right shadow object location
  loc.key.set_instance(tail_instance);
```

The prefix is generated once per upload (`generator::create_begin`,
`rgw_obj_manifest.cc:239`):

```cpp
  if (manifest->get_prefix().empty()) {
    char buf[33];
    gen_rand_alphanumeric(cct, buf, sizeof(buf) - 1);

    string oid_prefix = ".";
    oid_prefix.append(buf);
    oid_prefix.append("_");

    manifest->set_prefix(oid_prefix);
  }
```

A RADOS object name for a tail stripe therefore looks like
`<bucket_marker>__shadow_.<32 random chars>_<n>`; the random prefix is what
makes overwriting an S3 object safe: the new upload writes to a fresh prefix
while readers may still be walking the old manifest, and the old tail is
garbage-collected later (§4.3).

### 2.2 Sizes

| Option                      | Default | Role                                                                 |
|-----------------------------|---------|----------------------------------------------------------------------|
| `rgw_max_chunk_size`        | 4 MiB   | Largest single RADOS write RGW issues; also max head-object data size |
| `rgw_obj_stripe_size`       | 4 MiB   | Size of each tail stripe object                                      |
| `rgw_put_obj_min_window_size` | 16 MiB | Bytes of PUT data allowed in flight to RADOS before back-pressure   |
| `rgw_get_obj_window_size`   | 16 MiB  | Same for GET reads                                                   |
| `rgw_get_obj_max_req_size`  | 4 MiB   | Largest single RADOS read                                            |
| `rgw_multipart_min_part_size` | 5 MiB | S3 minimum part size                                                 |

`get_max_chunk_size` rounds `rgw_max_chunk_size` down to a multiple of the
pool's alignment when the data pool is erasure-coded, so stripes line up
with EC stripe units.

---

## 3. PUT: the data processor pipeline

### 3.1 Reading the request body

`RGWPutObj::execute` (`src/rgw/rgw_op.cc:4417`) builds a chain of
`DataProcessor` filters, then pumps the request body through it in
`rgw_max_chunk_size` pieces (`:4685`):

```cpp
  do {
    bufferlist data;
    if (fst > lst)
      break;
    if (copy_source.empty()) {
      len = get_data(data);
    } else {
      off_t cur_lst = min<off_t>(fst + s->cct->_conf->rgw_max_chunk_size - 1, lst);
      op_ret = get_data(fst, cur_lst, data);
      ...
    }
    ...
    if (need_calc_md5) {
      hash.Update((const unsigned char *)data.c_str(), data.length());
    }
    op_ret = filter->process(std::move(data), ofs);
    ...
    ofs += len;
  } while (len > 0);

  // flush any data in filters
  op_ret = filter->process({}, ofs);
```

The filter stack, outermost first, is: checksum (Lua/cksum) → compression
(`RGWPutObj_Compress`, if `rgw_compression_type` or the placement target
enables it) → encryption (SSE) → the object processor. Compression and
encryption are applied *before* striping, so tail objects hold compressed
or encrypted bytes and the `user.rgw.compression` xattr records block
boundaries for later random reads.

### 3.2 Splitting into head and stripes

The object processor is a small pipeline of its own
(`src/rgw/driver/rados/rgw_putobj_processor.cc` and `src/rgw/rgw_putobj.cc`):

```
 HeadObjectProcessor  →  StripeProcessor  →  ChunkProcessor  →  RadosWriter
   captures first        cuts at             cuts at             one librados aio
   head_chunk_size       stripe_max_size     rgw_max_chunk_size  write per chunk
```

`HeadObjectProcessor::process` (`rgw_putobj_processor.cc:96`) holds back
the first chunk so that, at completion time, it can be written together
with the attributes in one atomic head-object op:

```cpp
int HeadObjectProcessor::process(bufferlist&& data, uint64_t logical_offset)
{
  const bool flush = (data.length() == 0);

  // capture the first chunk for special handling
  if (data_offset < head_chunk_size || data_offset == 0) {
    if (flush) {
      // flush partial chunk
      return process_first_chunk(std::move(head_data), &processor);
    }

    auto remaining = head_chunk_size - data_offset;
    auto count = std::min<uint64_t>(data.length(), remaining);
    data.splice(0, count, &head_data);
    data_offset += count;

    if (data_offset == head_chunk_size) {
      // process the first complete chunk
      int r = process_first_chunk(std::move(head_data), &processor);
      ...
    }
    ...
  }
  // send everything else through the processor
  auto write_offset = data_offset;
  data_offset += data.length();
  return processor->process(std::move(data), write_offset);
}
```

When the stripe processor crosses a stripe boundary it calls back into
`ManifestObjectProcessor::next` (`:261`), which advances the manifest
generator and points the writer at the next tail object:

```cpp
int ManifestObjectProcessor::next(uint64_t offset, uint64_t *pstripe_size)
{
  // advance the manifest
  int r = manifest_gen.create_next(offset);
  ...
  rgw_raw_obj stripe_obj = manifest_gen.get_cur_obj(store);

  uint64_t chunk_size = 0;
  r = store->get_max_chunk_size(stripe_obj.pool, &chunk_size, dpp);
  ...
  r = writer.set_stripe_obj(stripe_obj);
  ...
  chunk = ChunkProcessor(&writer, chunk_size);
  *pstripe_size = manifest_gen.cur_stripe_max_size();
  return 0;
}
```

`ChunkProcessor::process` (`src/rgw/rgw_putobj.cc:20`) accumulates until it
has a full chunk, then emits it; `RadosWriter::process` (`:168`) is the
actual RADOS write:

```cpp
int RadosWriter::process(bufferlist&& bl, uint64_t offset)
{
  bufferlist data = std::move(bl);
  const uint64_t cost = data.length();
  if (cost == 0) { // no empty writes, use aio directly for creates
    return 0;
  }
  librados::ObjectWriteOperation op;
  add_write_hint(op);
  if (offset == 0) {
    op.write_full(data);
  } else {
    op.write(offset, data);
  }
  constexpr uint64_t id = 0; // unused
  auto c = aio->get(stripe_obj.obj, Aio::librados_op(stripe_obj.ioctx,
						     std::move(op), y, &trace),
		    cost, id);
  return process_completed(c, &written);
}
```

`aio` is a throttle (`src/rgw/rgw_aio_throttle.cc`) sized by
`rgw_put_obj_min_window_size`: `get()` blocks when more than the window is
in flight, which is how a slow cluster back-pressures the HTTP body read.
`Aio::librados_op` (`src/rgw/rgw_aio.cc:52`) is a plain `aio_operate`.
`written` records every tail object successfully created so they can be
deleted if the upload fails or is superseded.

The write hint tells BlueStore not to try compressing already-compressed
data (`:146`):

```cpp
void RadosWriter::add_write_hint(librados::ObjectWriteOperation& op) {
  const RGWObjStateManifest *sm = obj_ctx.get_state(head_obj);
  const bool compressed = sm->state.compressed;
  uint32_t alloc_hint_flags = 0;
  if (compressed) {
    alloc_hint_flags |= librados::ALLOC_HINT_FLAG_INCOMPRESSIBLE;
  }
  op.set_alloc_hint2(0, 0, alloc_hint_flags);
}
```

### 3.3 Completion: head write + bucket index, transactionally

`AtomicObjectProcessor::complete` (`rgw_putobj_processor.cc:371`) drains the
tail writes and then hands the first chunk, the manifest and the attrs to
`RGWRados::Object::Write::write_meta`:

```cpp
  int r = writer.drain();
  ...
  const uint64_t actual_size = get_actual_size();
  r = manifest_gen.create_next(actual_size);
  ...
  RGWRados::Object::Write obj_op(&op_target);
  obj_op.meta.data = &first_chunk;
  obj_op.meta.manifest = &manifest;
  obj_op.meta.ptag = &unique_tag; /* use req_id as operation tag */
  obj_op.meta.if_match = if_match;
  obj_op.meta.if_nomatch = if_nomatch;
  ...
  obj_op.meta.flags = PUT_OBJ_CREATE;
  obj_op.meta.modify_tail = true;

  r = obj_op.write_meta(actual_size, accounted_size, attrs, rctx,
                        writer.get_trace(), flags & rgw::sal::FLAG_LOG_OP);
  ...
  if (!obj_op.meta.canceled) {
    // on success, clear the set of objects for deletion
    writer.clear_written();
  }
```

`_do_write_meta` (`src/rgw/driver/rados/rgw_rados.cc:3239`) is the heart of
RGW's consistency model. Abbreviated to its RADOS ops, in order:

```cpp
  ObjectWriteOperation op;
  ...
  r = target->prepare_atomic_modification(rctx.dpp, op, reset_obj, ptag, meta.modify_tail, set_attr_id_tag, rctx.y);
  //   → op.cmpxattr(RGW_ATTR_ID_TAG, LIBRADOS_CMPXATTR_OP_EQ, state->obj_tag)  if object exists
  //   → op.create(true) / op.create(false)
  //   → op.setxattr(RGW_ATTR_ID_TAG, new write_tag); op.setxattr(RGW_ATTR_TAIL_TAG, ...)

  op.mtime2(&mtime_ts);
  if (meta.data) {
    /* if we want to overwrite the data, we also want to overwrite the
       xattrs, so just remove the object */
    op.write_full(*meta.data);                        // head data (≤ 4 MiB)
    if (state->compressed) {
      op.set_alloc_hint2(0, 0, librados::ALLOC_HINT_FLAG_INCOMPRESSIBLE);
    }
  }
  ...
  if (meta.manifest) {
    bufferlist bl;
    encode(*meta.manifest, bl);
    op.setxattr(RGW_ATTR_MANIFEST, bl);              // "user.rgw.manifest"
  }
  for (iter = attrs.begin(); iter != attrs.end(); ++iter) {
    op.setxattr(name.c_str(), bl);                    // user.rgw.etag, .acl, .content_type, x-amz-meta-* …
    ...
  }
  ...
  if (!index_op->is_prepared()) {
    r = index_op->prepare(rctx.dpp, CLS_RGW_OP_ADD, &state->write_tag, rctx.y);   // (1) index: pending
    if (r < 0)
      return r;
  }
  auto& ioctx = ref.ioctx;
  r = rgw_rados_operate(rctx.dpp, ref.ioctx, ref.obj.oid, std::move(op), rctx.y, 0, &trace, &epoch); // (2) head object
  if (r < 0) { /* we can expect to get -ECANCELED if object was replaced under,
                or -ENOENT if was removed, or -EEXIST if it did not exist
                before and now it does */
    ...
    goto done_cancel;
  }
  poolid = ioctx.get_id();
  r = target->complete_atomic_modification(rctx.dpp, meta.keep_tail, rctx.y);   // old tail → GC
  ...
  r = index_op->complete(rctx.dpp, poolid, epoch, size, accounted_size,
                        meta.set_mtime, etag, content_type,
                        storage_class, meta.owner,
			 meta.category, meta.remove_objs, rctx.y,
			 meta.user_data, meta.appendable, log_op);             // (3) index: committed
```

Three RADOS round-trips, each atomic on its own OSD, form a two-phase
commit across two objects:

1. **Prepare** on the bucket index shard: record "operation `tag` is
   pending on key `name`".
2. **Write** the head object. `cmpxattr` on `user.rgw.idtag` makes it fail
   with `-ECANCELED` if another writer replaced the object in between, so
   concurrent PUTs to the same key serialize without a lock. The
   `write_full` plus all `setxattr`s are one `ObjectWriteOperation`, so
   the head is never observed half-updated.
3. **Complete** on the index shard with the PG epoch and version from
   step 2: turn the pending entry into a live entry with size, etag and
   mtime, and update bucket stats.

If RGW dies between 2 and 3, the next `ListObjects` that encounters the
stale pending entry checks the head object (`dir_suggest_changes`) and
repairs the index. If it dies between 1 and 2, the pending entry expires
the same way. Tail objects written before a failed completion are cleaned
up by the writer's destructor (it deletes everything in `written` that was
not cleared).

---

## 4. The bucket index

### 4.1 Shards

The index of bucket instance `<id>` is `.dir.<id>` in the index pool
(`src/rgw/services/svc_bi_rados.cc:26`), sharded into
`.dir.<id>.<shard>` (with a generation number after resharding). New
buckets get `bucket_index_max_shards = 11` shards
(`src/rgw/rgw_zone_types.h:338`) unless `rgw_override_bucket_index_max_shards`
is set, and **dynamic resharding** (`rgw_dynamic_resharding = true`) adds
shards once any shard exceeds `rgw_max_objs_per_shard = 100000` entries.
An object's shard is `hash(name) % num_shards`, so a PUT touches exactly
one shard and listing merges all of them.

### 4.2 Entries are omap; ops run in the OSD

Everything is done by the `rgw` object class (`src/cls/rgw/cls_rgw.cc`),
so the read-modify-write of an entry happens inside the OSD under the PG
lock. Prepare (`rgw_bucket_prepare_op`, `:939`):

```cpp
  rgw_bucket_dir_entry entry;
  rc = read_key_entry(hctx, op.key, &idx, &entry);        // omap get
  ...
  bool noent = (rc == -ENOENT);
  if (noent) { // no entry, initialize fields
    entry.key = op.key;
    entry.ver = rgw_bucket_entry_ver();
    entry.exists = false;
    entry.locator = op.locator;
  }

  // fill in proper state
  rgw_bucket_pending_info info;
  info.timestamp = real_clock::now();
  info.state = CLS_RGW_STATE_PENDING_MODIFY;
  info.op = op.op;
  entry.pending_map.insert(pair<string, rgw_bucket_pending_info>(op.tag, info));

  // write out new key to disk
  rc = write_entry(hctx, entry, idx, header, false);
```

Complete (`rgw_bucket_complete_op`, `:1152`) removes the tag from
`pending_map`, rejects stale epochs, and re-accounts the header stats:

```cpp
  if (op.tag.size()) {
    auto pinter = entry.pending_map.find(op.tag);
    if (pinter == entry.pending_map.end()) {
      return -EINVAL;
    }
    entry.pending_map.erase(pinter);
  }

  if (op.tag.size() && op.op == CLS_RGW_OP_CANCEL) {
  } else if (op.ver.pool == entry.ver.pool &&
             op.ver.epoch && op.ver.epoch <= entry.ver.epoch) {
    op.op = CLS_RGW_OP_CANCEL;          // an older write completing late: ignore
  }
  ...
  entry.ver = op.ver;
  if (op.op == CLS_RGW_OP_CANCEL) {
    ...
  } else if (op.op == CLS_RGW_OP_DEL) {
    // unaccount deleted entry
    unaccount_entry(header, entry);
    ...
  } else if (op.op == CLS_RGW_OP_ADD) {
    // unaccount overwritten entry, account the new one
    ...
    entry.meta = op.meta;
    entry.exists = true;
    ...
  }
  write_entry(hctx, entry, idx, header);    // omap set
  write_bucket_header(hctx, &header);       // omap header: per-category num_entries / total_size
```

`write_entry` (`:772`) is a `cls_cxx_map_set_val`; the header, which holds
the bucket's usage counters (`rgw_bucket_dir_header::stats`), is the omap
header. Both are RocksDB writes on the index OSD (see the disk-internals
doc §4.2), which is why index pools are placed on SSD/NVMe and why a
100,000-entry shard cap exists: listing a shard is a RocksDB range scan.

The client-side call that issues the prepare
(`RGWRados::cls_obj_prepare_op`, `rgw_rados.cc:10488`) also guards against
concurrent resharding:

```cpp
  ObjectWriteOperation o;
  o.assert_exists(); // bucket index shard must exist

  cls_rgw_obj_key key(obj.key.get_index_key_name(), obj.key.instance);
  cls_rgw_guard_bucket_resharding(o, -ERR_BUSY_RESHARDING);
  cls_rgw_bucket_prepare_op(o, op, tag, key, obj.key.get_loc());
  int ret = bs.bucket_obj.operate(dpp, std::move(o), y);
```

Complete is issued asynchronously (`cls_obj_complete_op`, `:10506`,
`aio_operate` with a completion manager), so the HTTP 200 does not wait for
the index write. That is the documented eventual-consistency window of
bucket listings.

### 4.3 Overwrites and garbage collection

When a PUT replaces an existing key, step 2 above changes `idtag`, and
`complete_atomic_modification` pushes the *old* manifest's tail objects
onto the GC queue (objects in `default.rgw.log:gc`). The `rgw_gc_*` worker
deletes them after `rgw_gc_obj_min_wait` (2 h by default) so that a GET
that started before the overwrite can finish reading the old tail. DELETE
follows the same path: index prepare/complete with `CLS_RGW_OP_DEL`, head
object removed, tails to GC.

---

## 5. Multipart uploads

Each part is uploaded as its own object with its own manifest and head
(`MultipartObjectProcessor::prepare_head`, `rgw_putobj_processor.cc:475`):

```cpp
  manifest.set_multipart_part_rule(stripe_size, part_num);

  r = manifest_gen.create_begin(store->ctx(), &manifest,
				bucket_info.placement_rule,
				&tail_placement_rule,
				target_obj.bucket, target_obj);
  ...
  rgw_raw_obj stripe_obj = manifest_gen.get_cur_obj(store);
  RGWSI_Tier_RADOS::raw_obj_to_obj(head_obj.bucket, stripe_obj, &head_obj);
  head_obj.index_hash_source = target_obj.key.name;

  // point part uploads at the part head instead of the final multipart head
  writer.set_head_obj(head_obj);
```

so part 3 of an upload lives at `…__multipart_<prefix>.3` (head) and
`…__multipart_<prefix>.3_<n>` (stripes), in the `non-ec` pool if the data
pool is erasure-coded (EC pools cannot do the partial overwrites the
`.meta` object needs). `CompleteMultipartUpload`
(`RadosMultipartUpload::complete`, `src/rgw/driver/rados/rgw_sal_rados.cc:4315`)
does not copy any data; it concatenates the part manifests into one:

```cpp
        manifest.append(dpp, obj_part.manifest, store->svc()->zone->get_zonegroup(), store->svc()->zone->get_zone_params());
        auto manifest_prefix = part->info.manifest.get_prefix();
        ...
  obj_op.meta.manifest = &manifest;
  obj_op.meta.remove_objs = &remove_objs;   // per-part index entries to drop
  obj_op.meta.ptag = &tag; /* use req_id as operation tag */
```

and writes a head object with zero or few data bytes whose manifest has
one rule per part. Reads of a multipart object therefore hop from the head
straight into part objects; the ETag is the familiar `<md5-of-md5s>-<n>`.

---

## 6. GET: walking the manifest

`RGWRados::Object::Read::iterate` (`rgw_rados.cc:8383`) sets up a read
window and calls `iterate_obj` (`:8409`):

```cpp
  const uint64_t chunk_size = cct->_conf->rgw_get_obj_max_req_size;
  const uint64_t window_size = cct->_conf->rgw_get_obj_window_size;

  auto aio = rgw::make_throttle(window_size, y);
  get_obj_data data(store, cb, &*aio, ofs, y);
  ...
  int r = store->iterate_obj(dpp, source->get_ctx(), source->get_bucket_info(), state.obj,
                             ofs, end, chunk_size, _get_obj_iterate_cb, &data, y);
```

`iterate_obj` first fetches the object state, which is a `stat` of the head
object with all xattrs (and, with prefetch, the head's data bytes), decodes
the manifest, then walks stripes:

```cpp
  if (manifest) {
    /* now get the relevant object stripe */
    RGWObjManifest::obj_iterator iter = manifest->obj_find(dpp, ofs);
    RGWObjManifest::obj_iterator obj_end = manifest->obj_end(dpp);

    for (; iter != obj_end && ofs <= end; ++iter) {
      off_t stripe_ofs = iter.get_stripe_ofs();
      off_t next_stripe_ofs = stripe_ofs + iter.get_stripe_size();

      while (ofs < next_stripe_ofs && ofs <= end) {
        read_obj = iter.get_location().get_raw_obj(this);
        uint64_t read_len = std::min(len, iter.get_stripe_size() - (ofs - stripe_ofs));
        read_ofs = iter.location_ofs() + (ofs - stripe_ofs);

        if (read_len > max_chunk_size) {
          read_len = max_chunk_size;
        }

        reading_from_head = (read_obj == head_obj);
        r = cb(dpp, read_obj, ofs, read_ofs, read_len, reading_from_head, astate, arg);
        ...
        len -= read_len;
        ofs += read_len;
      }
    }
  }
```

Range requests are therefore cheap: `obj_find(ofs)` jumps straight to the
right stripe without touching earlier ones. The per-chunk callback
(`get_obj_iterate_cb`, `:8330`) issues one RADOS read, guarded on the head
by an atomic test so a concurrent overwrite is detected rather than served
as a mix of old and new stripes:

```cpp
  if (is_head_obj) {
    /* only when reading from the head object do we need to do the atomic test */
    int r = append_atomic_test(dpp, astate, op);        // cmpxattr idtag == what we stat'ed
    ...
    if (astate && obj_ofs < astate->data.length()) {
      // head data was prefetched with the stat: serve it without another read
      r = d->client_cb->handle_data(astate->data, obj_ofs, chunk_len);
      ...
    }
  }
  ...
  op.read(read_ofs, len, nullptr, nullptr);

  const uint64_t cost = len;
  const uint64_t id = obj_ofs; // use logical object offset for sorting replies

  auto completed = d->aio->get(obj.obj, rgw::Aio::librados_op(obj.ioctx, std::move(op), d->yield), cost, id);

  return d->flush(std::move(completed));
```

Up to `rgw_get_obj_window_size` bytes of reads are in flight at once;
`flush` hands completed chunks to the HTTP layer strictly in offset order.
Decompression and decryption happen in the callback chain above this,
using the block map stored in `user.rgw.compression`.

---

## 7. What the OSD sees

| S3 operation                 | RADOS ops (in order)                                                                                  | Pool(s)                |
|------------------------------|-------------------------------------------------------------------------------------------------------|------------------------|
| PUT 1 MiB object             | index `prepare` (cls, omap) → head `write_full` + xattrs (one op) → index `complete` (cls, omap, async) | index, data            |
| PUT 100 MiB object           | index `prepare` → 24× tail `write_full` (4 MiB each, up to 16 MiB in flight) → head `write_full`+xattrs → index `complete` | index, data |
| PUT overwriting a key        | as above; head op guarded by `cmpxattr idtag`; old tails queued to GC (`rgw.log`)                    | index, data, log       |
| Multipart part upload        | tail writes + part head in `non-ec` (if EC) or `data`; part index entry                               | non-ec / data, index   |
| CompleteMultipartUpload      | read `.meta`; head write with merged manifest; index `complete` + remove part entries; no data copy   | non-ec, data, index    |
| GET whole object             | head `stat`+getxattrs (+ prefetched first 4 MiB) → one `read` per remaining stripe                    | data                   |
| GET range                    | head `stat` → `read`s only on the stripes covering the range                                          | data                   |
| DELETE                       | index `prepare` → head `remove` → index `complete`; tails to GC                                       | index, data, log       |
| ListObjects                  | `cls_rgw_bucket_list` on every shard (omap range scan), merge                                         | index                  |
| HEAD / GetObjectAcl          | head `stat` + getxattrs only                                                                          | data                   |

On the OSD side, data-pool writes are large aligned `write_full`s, the best
case for BlueStore (whole-object blobs, no read-modify-write, direct aio on
SSD); index-pool traffic is entirely RocksDB omap I/O through the object
class. Both then follow the paths in `ceph-disk-io-internals.md`.

---

## 8. File index

| Concern                                    | File                                                          |
|--------------------------------------------|---------------------------------------------------------------|
| S3/Swift op handlers                       | `src/rgw/rgw_op.cc` (`RGWPutObj`, `RGWGetObj`, `RGWDeleteObj`, `RGWCompleteMultipart`), `rgw_rest_s3.cc` |
| Store abstraction layer                    | `src/rgw/rgw_sal.h`, `src/rgw/driver/rados/rgw_sal_rados.cc`  |
| PUT pipeline                               | `src/rgw/rgw_putobj.cc`, `src/rgw/driver/rados/rgw_putobj_processor.cc` |
| RADOS driver (head write, reads, index)    | `src/rgw/driver/rados/rgw_rados.cc`                           |
| Manifest                                   | `src/rgw/driver/rados/rgw_obj_manifest.h`, `rgw_obj_manifest.cc`, `src/rgw/rgw_obj_manifest.cc` |
| Async I/O and throttling                   | `src/rgw/rgw_aio.cc`, `rgw_aio_throttle.cc`                   |
| Bucket index object class (OSD side)       | `src/cls/rgw/cls_rgw.cc`, client `cls_rgw_client.cc`          |
| Bucket index service / sharding            | `src/rgw/services/svc_bi_rados.cc`, `src/rgw/driver/rados/rgw_reshard.cc` |
| Attribute names                            | `src/rgw/rgw_common.h` (`RGW_ATTR_*`)                         |
| Zone / pool layout                         | `src/rgw/rgw_zone.cc`, `rgw_zone_types.h`                     |
| Compression / encryption filters           | `src/rgw/rgw_compression.cc`, `rgw_crypt.cc`                  |
| Garbage collection                         | `src/rgw/driver/rados/rgw_gc.cc`, `src/cls/rgw_gc/`           |
| Multipart                                  | `src/rgw/driver/rados/rgw_sal_rados.cc` (`RadosMultipartUpload`) |
| Tunables                                   | `src/common/options/rgw.yaml.in`                              |
