# ext4 Filesystem Internals — `fs/ext4/` Deep Dive

> Notes from reading the Linux kernel source at `fs/ext4/` (~70K lines, kernel ~7.3-rc, 2026).
> Target depth: staff/principal engineer. Sources are cited as `file:line` into the tree.

---

## 1. What This Is

`fs/ext4/` is the implementation of **ext4**, the default general-purpose journaling
filesystem for most Linux distributions. It is the fourth generation of the ext lineage
(ext2 → ext3 → ext4) and remains **backward compatible**: the same driver mounts ext2 and
ext3 images (see `EXT2_FEATURE_*_SUPP` in `ext4.h:2334`), with behavior gated entirely by
**feature flags** stored in the superblock rather than by version numbers.

The subsystem splits into roughly these responsibilities:

| Area | Files | Size |
|---|---|---|
| Superblock, mount, error handling | `super.c` | 7.6K lines |
| Block allocation (buddy + preallocation) | `mballoc.c`, `balloc.c` | 8.3K lines |
| Inode lifecycle, buffered I/O, writeback, DIO | `inode.c`, `page-io.c`, `readpage.c`, `file.c` | 9K lines |
| Extent tree (on-disk mapping) | `extents.c`, `ext4_extents.h`, `migrate.c`, `move_extent.c` | 7.7K lines |
| Extent status tree (in-memory cache) | `extents_status.c/.h` | 2.6K lines |
| Directories & htree index | `namei.c`, `dir.c`, `hash.c` | 5.3K lines |
| Journaling glue to jbd2 | `ext4_jbd2.c/.h`, `fsync.c` | ~1K lines |
| Fast commit (fine-grained journal) | `fast_commit.c/.h` | 2.9K lines |
| Inode allocation | `ialloc.c` | 1.6K lines |
| Extended attributes | `xattr*.c` | 3.4K lines |
| Legacy indirect-block mapping (ext2/3 compat) | `indirect.c` | 1.5K lines |
| Inline data (tiny files in the inode) | `inline.c` | 2K lines |
| Online resize, orphan handling, multi-mount protection | `resize.c`, `orphan.c`, `mmp.c` | 3.3K lines |
| Crypto/verity/casefold integration | `crypto.c`, `verity.c` | ~0.6K lines |
| KUnit tests | `*-test.c` | 2.9K lines |

The actual journal engine is **not here** — it lives in `fs/jbd2/`. ext4 is a *client* of
jbd2: it wraps every metadata mutation in jbd2 handles and buys "credits" against a
running transaction (§6).

---

## 2. What Problem It Solves

At the highest level: **persist a POSIX file hierarchy on block storage such that a crash
at any instant never leaves metadata inconsistent, while staying fast on both spinning
disks and flash, from ~MB-scale volumes up to 64-bit block counts.**

The individual mechanisms each answer a specific historical pain point:

1. **Crash consistency without full fsck** — journaling via jbd2. ext2 required an O(disk-size)
   fsck after crash; ext3/4 replay a bounded journal instead.
2. **Scalability of block mapping** — ext2/3's indirect block scheme costs one metadata block
   per 1024 data blocks and describes each block individually. Extents (§4) describe up to
   32768 contiguous blocks in 12 bytes.
3. **Allocation quality / fragmentation** — mballoc (§5) does multi-block allocation with
   buddy bitmaps, preallocation, and locality grouping, instead of ext3's block-at-a-time
   reservation window.
4. **Delayed allocation** — block allocation is deferred from `write()` to writeback, so the
   allocator sees whole dirty ranges at once and can allocate them contiguously.
5. **Large directory lookup** — linear dirent scans became O(n); htree (§7) gives hashed
   B-tree lookup while keeping the on-disk format readable by old kernels.
6. **fsync latency** — full jbd2 commits are block-granular and force unrelated dirty
   metadata out. Fast commit (§8) logs logical deltas (TLVs) instead.
7. **Metadata corruption detection** — `metadata_csum` puts crc32c on every metadata
   structure (superblock, bitmaps, extent blocks, dirents, MMP block — see e.g.
   `ext4_extent_tail`, `ext4.h:2211`).

---

## 3. On-Disk Layout Fundamentals

### Block groups and flex_bg

The volume is divided into **block groups** (`blocks_per_group` = 8 × blocksize, so 32768
blocks / 128 MiB with 4K blocks). Each group classically holds: superblock copy (sparse),
group descriptor table, block bitmap, inode bitmap, inode table, data blocks.

- **`sparse_super`**: superblock backups only in groups 0 and powers of 3, 5, 7.
- **`flex_bg`** (`INCOMPAT`, `ext4.h:2226`): bitmaps and inode tables of `2^k` groups are
  packed together into the first group of a "flex group", turning small metadata I/O into
  large contiguous runs and freeing large contiguous data areas.
- **`meta_bg`**: moves group descriptors into each group's own region, removing the limit
  that the GDT (which must fit its growth into a reserved area) imposes on resize.
- **`64bit`**: 64-bit block numbers via `_hi` fields in group descriptors; group descriptors
  grow from 32 to 64 bytes.
- **`bigalloc`**: allocation unit becomes a *cluster* of `2^n` blocks. Bitmaps track clusters,
  not blocks. This bifurcates the entire allocator accounting into "clusters" (see `pa_len`
  in clusters, `mballoc.c:67`; the `partial_cluster` machinery in `ext4_extents.h:122` exists
  purely to decide whether a cluster shared between extents can be freed during truncate).

### The inode

60 bytes of `i_block[]` in the inode are *polymorphic*, interpreted per-inode-flags:

- `EXT4_EXTENTS_FL` → extent tree root (header + up to 4 extents), see §4.
- Legacy → 12 direct + 1/2/3-level indirect pointers (`indirect.c`).
- `EXT4_INLINE_DATA_FL` → file data itself lives here (+ overflow into the xattr space),
  `inline.c`. Small files and small directories need zero data blocks.
- Fast symlinks store the target string directly (`ext4_inode_is_fast_symlink`, `inode.c:148`).

`extra_isize` extends the 128-byte ext2 inode (typically to 256 bytes) for nanosecond +
crash-safe timestamps (y2038-safe encoding — tested in `inode-test.c`), project quota IDs,
and in-inode xattr space.

### Feature-flag compatibility model (worth internalizing)

Three classes (`ext4.h:2196-2233`), enforced at mount:

- **COMPAT**: kernel may mount even if it doesn't understand the feature (e.g. `dir_index`,
  `fast_commit`, `orphan_file`).
- **RO_COMPAT**: unknown feature ⇒ mount read-only allowed, read-write refused
  (e.g. `metadata_csum`, `bigalloc`, `quota`, `verity`, `project`).
- **INCOMPAT**: unknown feature ⇒ refuse mount entirely (e.g. `extents`, `64bit`, `casefold`,
  `encrypt`, `inline_data`, `largedir`, `mmp`).

This is the *design pattern* that let ext4 evolve for 20 years without breaking old
kernels/tools: any format change must first be classified by what an ignorant
implementation can safely do.

---

## 4. Extent Tree (`extents.c`, `ext4_extents.h`)

### On-disk format

A B+-tree keyed by logical block. All records are 12 bytes:

```c
struct ext4_extent {          // leaf level
    __le32 ee_block;          // first logical block covered
    __le16 ee_len;            // length (with MSB overload, below)
    __le16 ee_start_hi;       // physical block, high 16 bits
    __le32 ee_start_lo;       //                 low 32 bits  → 48-bit physical
};
struct ext4_extent_idx {      // interior levels
    __le32 ei_block; __le32 ei_leaf_lo; __le16 ei_leaf_hi; __u16 ei_unused;
};
```

Every node (including the root inside `i_block`) starts with an `ext4_extent_header`
(magic `0xf30a`, entries, max, depth). Max depth is **5** (`ext4_extents.h:87`). Non-root
nodes end with a crc32c tail; the clever bit (`ext4_extents.h:43`): `blocksize % 12 >= 4`
for all power-of-two block sizes ≥ 512, so a 4-byte checksum always fits after the record
array without rebalancing the layout.

### The `ee_len` MSB trick — unwritten extents

`ext4_extents.h:132-150`: if MSB of `ee_len` is set, the extent is **unwritten**
(allocated but never written — reads as zeros). So an initialized extent maxes at 2^15 =
32768 blocks, an unwritten one at 32767. Unwritten extents are the mechanism behind:

- `fallocate()` — preallocate without exposing stale disk contents (a *security* invariant:
  never map uninitialized disk blocks as readable data).
- Direct I/O and buffered writeback into preallocated space — write first, then *convert*
  unwritten→written in a small metadata transaction after the data I/O completes
  (`EXT4_HT_EXT_CONVERT`, end-of-I/O work in `page-io.c`).

Extent split/merge logic for partial conversions (write lands in the middle of an
unwritten extent → up to 3-way split) is a large fraction of `extents.c`'s complexity and
has its own KUnit suite (`extents-test.c`).

### Path handling

Lookups build an `ext4_ext_path[]` array (one entry per level: bh + positions,
`ext4_extents.h:105`) which insert/split/truncate then mutate. Truncate walks the tree
iteratively with this path rather than recursively.

---

## 5. Block Allocation — mballoc (`mballoc.c`)

The most algorithmically dense file in the directory. Origin: Alex Tomas / ClusterFS
(Lustre), 2003-2006. The 400-line design comment at `mballoc.c:43-407` is the canonical
reference; summary of the architecture:

### 5.1 Buddy cache

Per group, an in-memory **buddy bitmap** is maintained alongside the on-disk bitmap:
free-extent information at every power-of-two order. Both live in the page cache of a
private, never-persisted inode (`sbi->s_buddy_cache`), two blocks per group
(`[bitmap][buddy]`, `mballoc.c:101`), so they are demand-loaded and reclaimable like any
page cache. Loading a group = `ext4_mb_load_buddy()`.

Fundamental identity (`mballoc.c:252`):

```
in-core buddy = on-disk bitmap + preallocation descriptors (PAs)
```

i.e. the buddy marks *both* persistent allocations and non-persistent preallocations as
used. The comment at `mballoc.c:229-332` builds a full concurrency table proving which
operations need which locks — the key relaxations being: a referenced buddy is
initialized; a block used in a referenced buddy can't be reallocated; setting an
already-set bit is idempotent, so on-disk bitmap and PA may transiently both claim a block.

### 5.2 Preallocation (PA)

Two flavors (`mballoc.c:235-249`):

- **Inode PA** — attached to one inode, positioned by *logical* offset so subsequent file
  growth is physically contiguous. Requests are *normalized* (rounded up by file-size
  heuristics in `ext4_mb_normalize_request`) and the surplus is kept as PA.
- **Locality-group PA** — per-CPU shared pools (`s_locality_groups[cpu]`) used for *small*
  files (< `s_mb_stream_request`, default 16 blocks) so that small files land next to each
  other on disk. Consumed front-to-back by any inode.

A per-CPU `discard_pa_seq` sequence counter (`mballoc.c:436`) lets the ENOSPC path detect
"PAs were discarded concurrently, retry" without adding synchronization to the fast path.

### 5.3 Allocation criteria ladder (CR levels)

Group scanning proceeds through escalating criteria:

1. `CR_POWER2_ALIGNED` — power-of-2-aligned request; find a group whose *largest free
   order* ≥ request order. O(1) with optimize_scan.
2. `CR_GOAL_LEN_FAST` — group whose *average fragment size* ≥ goal length. O(1) lookup.
3. `CR_BEST_AVAIL_LEN` — same structures, but proactively trims the goal length so a
   fast lookup can still succeed (trade allocation size for speed).
4. `CR_GOAL_LEN_SLOW` — linear scan, best-effort.
5. `CR_ANY_FREE` — anything free (desperation).

With the `mb_optimize_scan` mount option, two arrays of **xarrays** index all groups
(`mballoc.c:132-162`): by largest-free-order and by ⌊log2(avg fragment size)⌋
(avg = `bb_free / bb_fragments`). Writers take `xa_lock`; readers are RCU. This replaced
older list-based structures and turns "find a suitable group" from O(#groups) into O(1) —
this matters at 100+ TB where there are millions of block groups.

Rotational-device nuance (`mballoc.c:213-221`): out-of-order group selection fills the disk
non-linearly → seeks. `mb_max_linear_groups` forces N linear scans first on HDDs
(default `MB_DEFAULT_LINEAR_LIMIT`, 0 on non-rotational).

Within a group, best-extent search is bounded by `mb_min_to_scan`/`mb_max_to_scan`;
`s_stripe` aligns allocations to RAID stripes.

### 5.4 Freeing and journaling interaction

Freed blocks cannot be reused until the transaction that freed them commits (else a crash
could replay metadata pointing at reused blocks). `ext4_free_data_cachep` objects park
freed extents per-transaction; buddy bits are returned only at commit callback time. TRIM
(`FITRIM` → `ext4_try_to_trim_range`) also runs through the buddy layer.

---

## 6. Journaling Integration (`ext4_jbd2.h/.c`)

ext4 does **physical metadata journaling** through jbd2:

- Every mutation path opens a handle: `ext4_journal_start(inode, EXT4_HT_xxx, credits)`.
  The `EXT4_HT_*` enum (`ext4_jbd2.h:111`) classifies handle types for diagnostics.
- **Credits** = worst-case count of metadata blocks the handle may dirty. The header is a
  catalog of the arithmetic: `EXT4_SINGLEDATA_TRANS_BLOCKS` is 20 with extents (5 tree
  levels each needing bitmap+group-descriptor) vs 8 for indirect (`ext4_jbd2.h:33`);
  add xattr blocks, quota (quota files are themselves journaled files!), directory index
  growth (`EXT4_INDEX_EXTRA_TRANS_BLOCKS`, 12).
- Long operations (truncate of a huge file) can't reserve worst-case up front; they
  **extend or restart** the transaction mid-flight when credits run low
  (`ext4_datasem_ensure_credits` — note it may *drop and reacquire* `i_data_sem`, which is
  why multi-transaction extent updates need the heavier locking protocol in §9).
- Before modifying any journaled buffer: `ext4_journal_get_write_access(bh)`; after:
  `ext4_handle_dirty_metadata(bh)`. jbd2 handles copy-on-write of buffers that are part of
  the committing transaction.

### Data modes

- **`data=ordered`** (default): only metadata is journaled, but data blocks are flushed
  *before* the metadata that references them commits → no stale-data exposure.
- **`data=writeback`**: no ordering; stale data possible after crash; fastest.
- **`data=journal`**: data goes through the journal too; double-write; disables delalloc,
  DIO and dax. Dirtying is special-cased throughout `inode.c` (see the journalled-data
  helper at `inode.c:1142`).

### Orphan handling (`orphan.c`)

Inodes that are unlinked-but-open or mid-truncate at crash time must be cleaned at
recovery. Classic mechanism: an on-disk singly-linked list threaded through
`i_dtime` from the superblock — a global contention point (every list update modifies the
superblock buffer). The **`orphan_file`** feature (`ext4.h:2194`) replaces it with a
preallocated file of orphan slot blocks; CPUs pick blocks via a cheap hash
(`raw_smp_processor_id()*13 % of_blocks`, `orphan.c:28`) and claim slots with atomics —
lock-free in the common path. `RO_COMPAT_ORPHAN_PRESENT` flags "orphan file may be
non-empty" so old kernels can't mount r/w and miss the cleanup.

---

## 7. Directories and htree (`namei.c`, `dir.c`, `hash.c`)

- Base format: variable-length dirents in directory blocks (name, inode, `file_type` with
  the `filetype` feature).
- **htree** (`dir_index` COMPAT feature): a hashed B-tree hidden *inside* a normal-looking
  directory. Block 0 is a `dx_root` (`namei.c:247`) disguised as `.`/`..` plus what old
  kernels see as an empty/garbage block they can fall back to linear-scanning — this is why
  it's COMPAT, not INCOMPAT. Interior nodes map hash→block; leaves are ordinary dirent
  blocks. 2 levels normally; **`largedir`** enables 3 levels and >2GB directories
  (`ext4.h:2230`, depth checks at `namei.c:842`).
- Hash: half-MD4 (default) or TEA over the name, seeded per-filesystem (`s_hash_seed`) to
  resist collision attacks; `hash.c` + KUnit `hash-test.c`. Collisions across leaf splits
  are handled by a "continuation" bit in the hash ordering.
- `readdir` on htree directories iterates in *hash order* (an rbtree is built in memory,
  `ext4_htree_fill_tree`, `namei.c:1151`) so that `telldir`/`seekdir` cookies remain
  stable across leaf splits — a subtle POSIX requirement that dictates the design.
- **casefold** (INCOMPAT): per-directory case-insensitive lookup via Unicode folding;
  interacts with fscrypt and disables inline_data for those dirs. Dirent checksums ride in
  a phantom tail dirent (`dx_tail`/`dirent_tail`).

---

## 8. Fast Commit (`fast_commit.c`)

Design doc at `fast_commit.c:17-183`. The problem: a jbd2 commit is coarse — an fsync of
one small file pays for the full running transaction (all dirty metadata blocks of *all*
files, descriptor blocks, revoke blocks, two flushes). Fast commit adds a **logical
redo log** in a reserved journal area:

- Format: TLV records (`ext4_fc_tl`), three delta classes (`fast_commit.c:27-45`):
  dirent ops (ADD/UNLINK/CREAT), range ops (ADD_RANGE/DEL_RANGE), and inode records.
- Commit protocol (9 steps, `fast_commit.c:52-72`): flush data of tracked inodes → briefly
  lock the journal barrier → *snapshot* inode state under `EXT4_STATE_FC_COMMITTING` →
  unlock (new handles proceed; concurrent updates get *requeued* rather than blocked) →
  write dirent + inode TLVs → write TAIL (crc + tid) for atomicity.
- Multiple fast commits stack after one full commit; replay applies every valid TAIL-ed
  region (`fast_commit.c:101-105`).
- **Idempotence principle** (`fast_commit.c:107-154`, worth quoting): *log outcomes, not
  procedures*. `mv B A` is logged as "link A→ino11; unlink B; inode 11 state", so replay
  can crash and rerun arbitrarily. This is the standard trick for making logical
  replay idempotent (cf. message-delivery dedup, CRDT convergence).
- Anything unsupported (xattr changes, etc.) calls `ext4_fc_mark_ineligible()` → next
  commit silently falls back to a full jbd2 commit. Bounded snapshot cost: > 1024 inodes
  or > 2048 ranges under the journal barrier → fall back (`fast_commit.c:194`).

Key insight: fast commit is an *optimization layered on* jbd2, never a replacement — full
commits remain the periodic checkpoint and the fallback for every hard case, which is what
keeps the replay code small enough to trust.

---

## 9. Extent Status Tree (`extents_status.c`) — the in-memory mapping cache

An **rbtree per inode** (`i_es_lock`) caching logical→physical mapping state with four
statuses: **written / unwritten / delayed / hole**. Design comment at
`extents_status.c:58-174`.

Why it exists (`extents_status.c:65-94`): before it, "is this range delayed-allocated?"
was answered by trawling the page cache (FIEMAP, SEEK_HOLE/DATA, bigalloc quota decisions,
writeback) — slow and buggy. Now `ext4_map_blocks()` consults the ES tree *first* and
treats it as authoritative.

Critical consistency contract (`extents_status.c:125-154`) — this is the part reviewers
get wrong:

1. All mapping creation/query goes through the ES tree (fast-commit replay excepted).
2. Updating the on-disk extent tree requires exclusive `i_data_sem` + atomic ES update.
   If the operation spans multiple transactions (credits may force `i_data_sem` to drop —
   §6), you must instead hold `i_rwsem` **and** `invalidate_lock` exclusively, evict page
   cache in the range, and rebuild/drop the ES tree (punch-hole pattern).
3. Any mapping query must hold at least one of: `i_rwsem`, `invalidate_lock`, or the
   folio lock covering the range.

Memory management: a shrinker reclaims written/unwritten/hole entries under pressure, but
**never delayed entries** — those carry reservation accounting that exists nowhere else
(`extents_status.c:50-55`). "Pending reservation" objects additionally track per-cluster
delalloc state for bigalloc.

---

## 10. Write Paths (`inode.c`, `page-io.c`, `file.c`)

### Buffered writes + delayed allocation

`write()` → `ext4_da_write_begin`: no blocks allocated; a **delayed** ES entry is inserted
and space is *reserved* (`s_dirtyclusters_counter`, quota reservation). At writeback,
`ext4_do_writepages` (`inode.c:2764`) gathers the longest run of dirty+delayed pages
(`mpage_da_data`), allocates it in one `ext4_map_blocks()` call inside one handle
(`mpage_map_and_submit_extent`, `inode.c:2452`), and submits bios via `page-io.c`.
Allocation happens *once per extent*, not once per block — delalloc is what makes mballoc's
normalization effective.

Notable: buffered I/O is still buffer_head-based in this tree; **iomap** is used for
direct I/O (via `iomap_dio_rw`), DAX, `bmap`, fiemap, and atomic-write probing
(`ext4_iomap_ops`, `inode.c:3462+`, including `IOMAP_F_ATOMIC_BIO`). The buffered-iomap
conversion is a long-running upstream effort; check `ext4_map_blocks` callers before
assuming either model.

Writeback of pages that need unwritten conversion defers the conversion to end-of-bio
workqueue context (`page-io.c`) because block state can't be modified in interrupt context
and needs a handle.

### ENOSPC subtleties

Delayed allocation means `write()` succeeds before blocks exist → writeback can hit
ENOSPC. ext4 mitigates with reserved cluster accounting (worst-case metadata reservation
per delayed extent) and the mballoc retry/PA-discard dance (§5.2). Bigalloc makes the
accounting per-cluster with the "pending reservation" tree.

### Locking hierarchy (memorize)

```
i_rwsem  →  invalidate_lock (mapping)  →  transaction start  →  i_data_sem
```

plus `s_writepages_rwsem` taken read around writeback (`ext4_writepages_down_read`,
`inode.c:3031`) and write around operations that change the mapping *mode* of an inode
(e.g. `migrate.c` indirect→extents conversion). `i_data_sem` read = query mapping tree;
write = modify it. Page faults take `invalidate_lock` shared, which is what serializes
punch-hole against faults without taking `i_rwsem` in the fault path.

---

## 11. Supporting Machinery

- **`ialloc.c`** — inode allocation, Orlov allocator: spread top-level directories across
  groups (future growth room), pack files near their parent directory. With `flex_bg`,
  works on flex-group granularity. Handles lazy inode-table zeroing (`inode_readahead`,
  `li_request` list zeroes inode tables in a kernel thread post-mkfs).
- **`resize.c`** — online *grow* only (no shrink), group-at-a-time or meta_bg; must
  carefully journal new group metadata before flipping `s_blocks_count`.
- **`mmp.c`** — multi-mount protection (INCOMPAT_MMP): a heartbeat block rewritten every
  interval by a kthread; a second node refusing to mount if the sequence advances.
  Guards shared-storage (SAN) double-mount corruption. Checksummed like everything else
  (`mmp.c:11`).
- **`block_validity.c`** — rbtree of "system zones" (bitmaps, inode tables, journal);
  every mapped extent is checked against it so a corrupted extent tree can't cause reads
  or writes *over metadata*. Cheap runtime defense against fuzzed/corrupt images.
- **`xattr.c`** — in-inode xattrs (after the fixed inode fields) → single external block
  (shared, refcounted, mbcache-deduplicated) → `ea_inode` feature for large values (each
  value in its own inode, refcounted by hash). Deduplication + journaling interaction makes
  this file trickier than it looks (~3.2K lines).
- **`crypto.c` / `verity.c`** — thin glue to fs/crypto (fscrypt: per-file keys, encrypted
  filenames force ciphertext dirent handling in namei) and fs/verity (Merkle tree appended
  past EOF, hidden from stat/read).
- **`fsmap.c`** — `GETFSMAP` ioctl: physical-space inventory (who owns each block range),
  reverse-mapping-style reporting built by walking bitmaps + known metadata.
- **`sysfs.c`** — `/sys/fs/ext4/<dev>/` tunables (all the mballoc knobs from §5,
  `err_out` behavior, etc.).
- **Error handling policy** (`super.c`): `errors=remount-ro|panic|continue`;
  `ext4_error()` marks the sb with error flags + first/last error location persisted in
  the superblock (invaluable for forensics), optionally aborts the journal. Recent trees
  also count errors into the superblock asynchronously to avoid deadlocking on the sb
  buffer lock from arbitrary contexts.
- **KUnit** (`.kunitconfig`, `*-test.c`) — unit tests run mballoc/extents/inode-time/hash
  logic against a **stubbed block layer** (`kunit/static_stub.h` hooks in `mballoc.c:21`,
  `extents_status.c:19`) — notable as one of the few places the kernel unit-tests fs
  internals without a block device.

---

## 12. Staff-Level Takeaways / Design Patterns Worth Stealing

1. **Compatibility as a type system.** COMPAT/RO_COMPAT/INCOMPAT classifies every format
   change by what an *ignorant* reader can safely do. This single idea bought 20+ years of
   forward evolution. (Compare: protobuf field semantics, API versioning policies.)
2. **Disguise new structures as old ones.** htree roots masquerade as legacy directory
   blocks; the extent checksum hides in the `blocksize % 12` slack. Backward compatibility
   through *structural camouflage* rather than version bumps.
3. **Log outcomes, not procedures** (fast commit) to get idempotent replay for free.
4. **Reserve worst-case, spend actual** — jbd2 credits are a pessimistic admission-control
   system; operations that can't bound worst-case restart transactions at safe points.
   The price is the §9 locking contract — every "we might drop the lock and re-take it"
   creates a protocol others must follow.
5. **Cache derived state in reclaimable memory, but know which entries are load-bearing.**
   The buddy cache lives in page cache (reclaimable); ES-tree written/hole entries are
   shrinkable, but *delayed* entries hold reservation state and must never be reclaimed.
   Distinguishing "cache" from "the only copy of accounting state" in the same structure
   is subtle and is exactly where bugs live.
6. **Escalating-effort search ladders** (CR_POWER2_ALIGNED → … → CR_ANY_FREE) with O(1)
   indexes for the early rungs: get the common case fast and keep a correct slow path,
   rather than one clever middle-ground algorithm.
7. **Percpu sequence counters for optimistic fallback detection** (`discard_pa_seq`):
   fast path samples one CPU's counter; only the failure path pays for the full sum.
8. **The concurrency table as documentation** (`mballoc.c:288`): enumerate every pair of
   concurrent operations and argue each cell. When lock relaxation is justified bit-by-bit
   ("setting an already-set bit is idempotent"), write the proof down in the source.

---

## 13. Where to Start Reading

Suggested order for building a mental model from source:

1. `ext4.h` — skim `struct ext4_super_block`, `struct ext4_inode`, feature flags, inode
   state flags (`EXT4_STATE_*`), `struct ext4_sb_info`, `struct ext4_inode_info`.
2. `ext4_jbd2.h` — the credit macros teach the cost model of every operation.
3. `extents.c:ext4_ext_map_blocks()` — the central mapping function; everything meets here.
4. `inode.c:ext4_map_blocks()` (`inode.c:677`) then `ext4_do_writepages()` — the I/O spine.
5. `mballoc.c:43-407` comment, then `ext4_mb_new_blocks()`.
6. `extents_status.c:58-174` comment — the locking contract.
7. `fast_commit.c:17-183` comment — the replay model.
